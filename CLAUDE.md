# CLAUDE.md - claude-utils

See README.md for full project documentation, script descriptions, and setup instructions.

## For Claude: working in this repo

### What this repo is

Shell scripts that drive autonomous Claude Code task execution. Two entry points:

- **Brief flow** (recommended for multi-task work): `briefs/` → `claude-brief` → `backlog-for-review/` → (user review) → `inbox/` → `claude-task`/`claude-tasks` → `done/`
- **Standard flow**: `backlog/` → `claude-prep` → `backlog-for-review/` → (user review) → `inbox/` → `claude-task` → `done/`

### Conventions

- All scripts: `#!/bin/bash`, no external dependencies beyond what README lists.
- Scripts must be safe to run with an empty source folder (exit 0, print nothing or a short message).
- `config` is sourced as key=value. Never hardcode user-specific values (email, paths beyond `$HOME`).
- Symlink names in `~/bin/` matter: `init` is exposed as `claude-init` to avoid clashing with system commands.

### What to watch out for

- `claude-task` calls itself via `$HOME/bin/claude-task --run` for the internal tmux execution. If you rename the script, update that self-reference.
- `claude-tasks` also references `$HOME/bin/claude-task` directly. Keep both in sync if the name changes.
- `claude-prep` uses the same self-call pattern (`$HOME/bin/claude-prep --run`). Keep symlink name in sync.
- `claude-brief` uses the same self-call pattern (`$HOME/bin/claude-brief --run`). Keep symlink name in sync.
- `claude-preps` and `claude-briefs` reference their respective single-task scripts via `$HOME/bin/`.
- `mtask` sources `config` at runtime via `realpath "$0"`, so it works correctly through the symlink.
- `claude-prep` can output either a single task file (raw Markdown) or multiple delimiter-separated task files if the task size rule triggers a split. The post-processor detects which case applies automatically.
- `claude-brief` archives the original brief to `~/tasks/briefs/done/` and embeds a `## Brief` path in every generated task. Do not move or rename that archive — tasks reference it at runtime.

### Auth-failure circuit-breaker

An auth/credentials failure (expired token, `API Error: 401`) means no work happened, so it must never be treated as a task-quality failure:

- `claude-task` detects it (`is_auth_failure`) right after the main `claude -p` call. The task is requeued to `inbox/` **unchanged** — no retry burned, never sent to `review/`. A marker file `~/tasks/.auth-cooldown` is dropped.
- `claude-eval` returns a new exit code **3** when the evaluator call itself can't authenticate (verdict is meaningless). `claude-task` treats exit 3 the same way (defer, no retry).
- The launcher pauses the unattended (no-arg / cron) path while `~/tasks/.auth-cooldown` is younger than `auth_cooldown_min` minutes (config, default 30). An explicit task argument bypasses the pause. The marker is cleared on the first successful (non-401) run and ages out on its own.
- To resume immediately after re-authenticating: run a task manually, or `rm ~/tasks/.auth-cooldown`.

### Killed / interrupted run (exit 143)

A non-zero exit from the main `claude -p` call with **no parseable `.result`** means the run was interrupted, not that the task is bad. The classic cause is a headless task parking itself to wait on an event that never fires (a Monitor event, `ScheduleWakeup`, or a task-notification) — there is no event loop in `claude -p`, so it eventually gets `SIGTERM`'d (**exit 143**).

- `claude-task` catches this right after the auth-failure check: the task is requeued to `inbox/` **unchanged** — no retry burned, never sent to the evaluator (a blank report reads as FAIL), never left orphaned in `run/`. If the run exits non-zero but *does* produce a `.result`, it falls through to normal evaluation.
- The autonomous PREAMBLE explicitly forbids waiting on async events: tasks must run tests/commands **synchronously**, block on their output, and record anything unfinished in `## Summary` rather than parking. This removes the root cause.
- Before this fix, exit 143 with an empty report could pass the evaluator and hit the final branch, which logged `FAILED (exit code 143)` and `exit`ed **without archiving the task file** — leaving orphans in `~/tasks/run/`. If you see stale files there, that's the signature.

### Capturing `claude -p --output-format json` (never merge stderr)

Every `claude -p ... --output-format json` capture MUST send stdout and stderr to
**separate** files (`> "$JSON_TMP" 2>"$ERR_TMP"`), never `2>&1`. Claude prints
warnings to stderr (e.g. `Ignoring N permissions.allow entries ... this workspace
has not been trusted`); merged into stdout they prepend non-JSON text, `jq` fails
to parse, and `.result` / `.modelUsage` come back empty. That silently produced
blank task reports AND blank evaluator verdicts — and a blank verdict was being
read as FAIL, sending a task that actually passed through 10 retries into
`review/`. Detection helpers (`is_auth_failure`) scan **both** files. As defence
in depth: if `.result` is empty but the capture is non-empty, `claude-task` falls
back to dumping the raw stdout/stderr into the report (never a silent blank), and
`claude-eval` returns inconclusive **3** (defer, no retry) on any empty verdict,
not FAIL. Trust the workspace once interactively to stop the warning at source.

### Task size rule

Both `claude-prep` and `claude-brief` enforce a size constraint in their preambles: a task touching more than 5 files or spanning more than 3 distinct logical phases must be split into smaller tasks connected with `## Depends-on`. This keeps each task within the model's effective context window (~40%) and prevents silent quality degradation from context compaction.

### Adding a script

1. Write it in this directory with a `#!/bin/bash` shebang.
2. `chmod +x scriptname`
3. Symlink: `ln -s ~/Scripts/claude-utils/scriptname ~/bin/scriptname`
4. Document in README.md (user-facing) and add a note here if there are gotchas.
