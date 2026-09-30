# colab-batch-runner

A [Claude Code](https://claude.com/claude-code) skill/plugin that teaches Claude
how to run long, unattended jobs (GPU inference batches, model downloads,
data-processing loops — anything that takes minutes to hours) on a
[Google Colab](https://colab.research.google.com) session via the
[`colab` CLI](https://pypi.org/project/google-colab-cli/), **without losing
hours of work to a dropped connection.**

## The problem this solves

`colab exec` talks to the remote kernel over a websocket that turns out to be
fragile for anything long-running:

- A multi-minute idle gap between two separate `colab exec` calls reliably
  kills the session — and once that happens, the session is **gone for
  good**. `status`, `exec`, `stop`, `url` all start saying "not found",
  even though `colab sessions` may still list the underlying VM (often
  prefixed `[?]`, meaning the CLI can no longer control it — and you can no
  longer tell from the CLI whether it's still running and billing).
- Even a single continuous `colab exec` call can drop mid-run after hours of
  successful work.

None of this is documented by the `colab` CLI itself — it was found the hard
way, twice, while running a real batch job. This skill packages the pattern
that survives it.

## What you get

Once installed, Claude will automatically apply this pattern whenever you ask
it to run something long on a Colab session — you don't need to explain the
gotchas every time. It knows to:

- Make sure the job checkpoints its own progress externally (e.g. a
  database row per item) so a crash never loses more than the one item that
  was in flight.
- Run **setup + the actual work in one continuous `colab exec` call**,
  instead of pausing between "run setup" and "now run the batch" — that
  pause is exactly what kills sessions.
- Recognize a dead session for what it is instead of trying to reconnect
  forever, spin up a fresh one, and resume — because the job is idempotent,
  resuming needs no manual bookkeeping.
- Always stop the session when done, so Colab compute doesn't keep billing.

## Install

Requires [Claude Code](https://claude.com/claude-code) and the
[`colab` CLI](https://pypi.org/project/google-colab-cli/) already set up
and authenticated.

```bash
git clone https://github.com/pyhu26-public/colab-batch-runner.git \
  ~/.claude/skills/colab-batch-runner
```

Restart Claude Code (or run `/reload-plugins` in an existing session). Check
it loaded:

```bash
claude plugin list
# Skills-directory plugins (.claude/skills/*):
#   colab-batch-runner@skills-dir   Status: ✔ loaded
```

## Use

Just ask Claude Code normally — no special syntax needed:

> "Run this OCR pipeline over 5,000 images on a Colab L4 GPU session, it'll
> take hours"

> "My colab session died with 'Connection was lost', can you recover it and
> keep going?"

Claude will pull in `SKILL.md` for the actual orchestration steps and
`colab` command sequence. Read [`SKILL.md`](./SKILL.md) directly if you want
to see (or copy) the pattern yourself.

## Uninstall

```bash
claude plugin disable colab-batch-runner@skills-dir   # keep the files, just stop loading it
# or
rm -rf ~/.claude/skills/colab-batch-runner            # remove entirely
```

## License

MIT — see [LICENSE](./LICENSE).
