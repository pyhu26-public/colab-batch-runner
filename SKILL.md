---
name: colab-batch-runner
description: Use when the user wants to run a long-running job (GPU inference batch, model download+warm-up, data processing loop) on a Google Colab session via the `colab` CLI, especially when it must run unattended for many minutes or hours. Also use when a `colab exec` session has died mid-job ("Connection was lost", session shows as "not found" or "[?]" in `colab sessions`) and the user wants to recover/resume it. Trigger phrases: "colab에서 배치 돌려줘", "run this on a colab GPU session", "colab 세션이 자꾸 끊겨", "resume the colab batch".
---

# colab-batch-runner

Drives long unattended jobs on a `colab` CLI session (session create → package
install → upload → execute) in a way that survives the tool's real-world
instability, instead of losing hours of GPU compute to a dropped connection.

## The core problem (observed empirically, not documented anywhere)

`colab exec` talks to the remote kernel over a websocket that is **not
reliable for long or gappy sessions**:

- A multi-minute **idle gap between two separate `colab exec` calls** (e.g.
  you ran setup, then paused to write a summary or wait for user confirmation
  before running the next command) reliably kills the session. The next
  `colab exec`/`colab status` call fails with `RuntimeError: Connection was
  lost`, and afterwards the session is **unrecoverable** — `status`, `exec`,
  `stop`, and `url` all report "not found" for that session name, even though
  `colab sessions` may still list the underlying VM (often with a `[?]`
  prefix, meaning the CLI can no longer drive it at all).
- Even a **single continuous `colab exec` call** can drop mid-run after
  running successfully for a long time (observed: ~478 successful iterations
  over several hours before disconnecting). Treat this as a normal failure
  mode to design around, not a bug to keep retrying past.
- `colab update --install` (upgrading the CLI) is worth trying once if
  reconnection fails outright, but does not fix the long-running-job dropout
  above.

**Conclusion: assume any `colab exec` call can die at any time, with no way
to reconnect afterward.** The job must be designed so that dying costs at
most "redo the current chunk", never "lose everything and start over".

## Required shape of the work itself

Before touching the `colab` CLI, make sure the actual job function is:

1. **Checkpointed in external state**, not in the kernel's memory or local
   session filesystem. A database row, a completion-marker file in cloud
   storage, an appended log the next run can diff against — anything that
   survives the session dying. Do not rely on `/content/...` files on the VM
   surviving to a later session.
2. **Idempotent / naturally resumable** — the job's own "what's left to do"
   query should just skip already-completed items when re-run, so restarting
   it after a crash requires no manual bookkeeping (no offsets to track by
   hand).
3. **Committing progress incrementally**, not buffering all results to write
   at the end. If item N's work is fully persisted before starting item N+1,
   a mid-batch disconnect only ever loses the one in-flight item.

If the job doesn't already look like this, restructure it first — the
`colab` orchestration below assumes it.

## Orchestration pattern

```bash
SESSION=my-batch-job-1     # bump the suffix each time you must recreate the session
GPU=L4                     # T4 | L4 | G4 | H100 | A100 (omit --gpu for CPU-only)

# 1. Create the session
colab --auth adc new --session "$SESSION" --gpu "$GPU"
colab --auth adc status -s "$SESSION"     # confirm hardware/IDLE

# 2. Install packages (fast — check what's already preinstalled first so you
#    only install what's actually missing; Colab GPU runtimes usually already
#    have torch/transformers/pandas/psycopg2 etc.)
colab --auth adc install -s "$SESSION" <missing-pkg-1> <missing-pkg-2>

# 3. Upload everything the job needs — secrets/env file, setup script, work
#    script — back-to-back, no thinking/reporting pause between these calls.
colab --auth adc upload ./local/secrets.env /content/secrets.env -s "$SESSION"
colab --auth adc upload ./local/setup.py    /content/setup.py    -s "$SESSION"
colab --auth adc upload ./local/work.py     /content/work.py     -s "$SESSION"

# 4. THE CRITICAL STEP: run setup + the actual work in ONE continuous
#    `colab exec` call, with a --timeout comfortably longer than the whole
#    job could ever take. Do not split "run setup" and "then run the work"
#    into two separate exec invocations — the gap between them (even just the
#    time it takes you to read the setup output and decide to continue) is
#    exactly the kind of idle gap that kills the session.
colab --auth adc exec -s "$SESSION" --timeout 50000 -f /dev/stdin <<'PYEOF'
import os
os.chdir('/content')
# ... load secrets.env into os.environ ...
exec(compile(open('/content/setup.py').read(), 'setup.py', 'exec'), globals())
print('=== SETUP DONE, STARTING WORK ===')
exec(compile(open('/content/work.py').read(), 'work.py', 'exec'), globals())
print('=== WORK DONE ===')
PYEOF

# 5. When this exits 0, the job is actually finished. Always stop the
#    session explicitly — Colab compute keeps billing until you do.
colab --auth adc stop -s "$SESSION"
```

`colab exec -f <path>` reads a **local** file path and ships its contents to
run remotely — it is not a reference to something already uploaded via
`colab upload`. Piping a heredoc through `/dev/stdin` (as above) is the
simplest way to inline a driver script without creating a throwaway local
file.

## Recovering after a disconnect

When step 4 above fails with `RuntimeError: Connection was lost` (or exit
code 1 with a similar traceback):

1. **Check the job's external checkpoint state first** (the DB / marker
   files from "Required shape of the work itself" above) to see how far it
   actually got — the successful part is not lost.
2. `colab --auth adc sessions` — if the dead session still appears, try
   `colab --auth adc stop -s "$SESSION"` once to attempt cleanup, but expect
   it may already say "not found"; don't loop on this.
3. If you cannot confirm the old session is gone and stopped, tell the user
   explicitly and ask them to check colab.research.google.com (logged into
   whichever account owns the session) for an orphaned runtime that might
   still be billing — the CLI has no way to reach a session once it's in
   this state.
4. Pick a **new session name** (bump a suffix, e.g. `my-batch-job-2`) and
   repeat steps 1–4 of the orchestration pattern. Because the work is
   idempotent, the same `work.py` naturally picks up only what's left.
5. Repeat until a run completes with exit code 0, then stop the session.

## Gotchas

- **Auth**: `colab` may default to a personal Google account instead of an
  organization one. If the account matters (billing, data access), always
  pass `--auth adc` explicitly rather than relying on the default. If the
  wrong account is being used, that's a `colab` CLI configuration issue to
  resolve with the user, not something this pattern can fix.
- **`colab sessions` showing `[?]`**: this means the CLI still knows the raw
  VM exists but has lost the local connection metadata needed to drive it
  (exec/stop/status/url all fail "not found"). Treat it as dead; don't try
  increasingly creative ways to reattach.
- **`colab log -s <session>`** (optionally `-o file.md` to export full
  content instead of the truncated preview) recovers the complete
  code+output history of everything run through this CLI in a named session
  — useful forensic tool if a previously-working script seems to have been
  lost, but it captures **only** what ran through `colab exec`/the CLI, not
  anything typed directly into a Colab browser notebook.
- Before installing packages, check what's already importable — Colab GPU
  runtimes ship with a lot preinstalled (torch, transformers, pandas,
  psycopg2, jsonschema, PIL...); installing only the real gaps keeps step 2
  fast.
