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
- The main call is now wrapped in `timeout $((task_timeout_min*60))` (config `task_timeout_min`, default 45), so an overrun fails deterministically as **exit 124** and flows through the same deferral path — no more relying on an arbitrary external killer.

### Per-project serialization (the `.locks/` lock)

`claude-tasks` launches every inbox task in parallel, and tasks that share a `## Project` mutate the **same git working tree and DB**. Running them concurrently races — one task's half-written file breaks another's test run (this was the cause of a test task retrying 4× in one batch).

- In `--run` mode `claude-task` extracts `## Project` and holds an exclusive `flock` on `~/tasks/.locks/<project>.lock` for the whole run (including evaluation, since the eval's `## Test Command` hits the same tree). Same-project tasks serialize; different projects still run in parallel.
- If the lock isn't acquired within `project_lock_wait_min` minutes (config, default 60), the task is requeued **unchanged** (no retry burned) for a later pass. Tasks with no `## Project` are never locked.

### Verification & evaluator timeouts

- `claude-eval` wraps each `## Test Command` in `timeout $((test_timeout_min*60))` (config, default 20). A timeout returns **exit 4** = inconclusive (slow/hung suite, often concurrency-induced — not a quality signal and not auth).
- `claude-task` treats eval **exit 4** like a deferral: requeue unchanged, no retry burned, and — unlike auth **exit 3** — it does **not** drop `.auth-cooldown` or pause the queue. Keep 3 (auth) and 4 (test timeout) distinct: 3 pauses the queue, 4 does not.
- **`## Test Command` extraction handles fenced, continued blocks.** The awk grabs every line between `## Test Command` and the next `## ` heading — including the ` ``` ` fence delimiters of a fenced block. Two guards in the run loop make that safe: fence-delimiter lines (`` ``` `` / `~~~`) are skipped (a bare fence line fed to `bash -c` aborts with "unexpected EOF looking for matching" and used to masquerade as a FAIL), and backslash line-continuations (`cmd && \`) are accumulated into one logical command before executing (so a multi-line Test Command runs as a unit instead of each physical line's dangling `\` erroring). Single-line, unfenced commands are unaffected.

### Dead dependency chains

A task blocked by a `## Depends-on` whose dependency landed in `review/` (i.e. failed) can never unblock on its own. When the no-arg path finds every inbox task blocked, `claude-task` scans for deps sitting in `review/` and fires a critical `notify-send` so they don't pile up silently.

### Model aliases

`## Model: opus|sonnet|haiku|fable` map to `claude-opus-4-8`, `claude-sonnet-5`, `claude-haiku-4-5-20251001`, `claude-fable-5` in both `claude-task` and `claude-eval`. **Keep the two files in sync** when models change — a task's run and its evaluation should use consistent tiers.

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

### Follow-up suggester

After a task completes **and passes evaluation**, `claude-task` calls `generate_suggestions` to propose follow-up task stubs into `~/tasks/suggestions/`. **This reflex is now SAME-PROJECT-ONLY** — it proposes natural continuations in the same project. Cross-project propagation (`## Relationships` `mirror`/`content`) and strategic multi-task synthesis were removed from here and are now owned by the `planner` kind in `claude-scheduled` (see the Scheduled-job engine subsection), which can read across pipeline state the per-task reflex structurally cannot see. Watch out for:

- **Fires from exactly one place: the verified-success branch.** Every earlier exit (auth fail, interrupt/timeout, eval defer 3/4, retry, review) returns before it. Do not add a second call site — reaching that branch is the *only* proof the run succeeded and passed eval.
- **Must run BEFORE the `done/` archive `mv`.** The suggester reads `$TASK_FILE`; moving it first would break the read. The call is guarded `|| true` so a suggester hiccup never blocks archival.
- **Reads the result report, never a `git diff`.** Sibling tasks under `claude-tasks` mutate the same tree concurrently; the report (`## Summary` etc.) is scoped to *this* task, a diff is not.
- **No cross-project targets here anymore.** The reflex emits only `kind: same-project` stubs, all in `$PROJECT`, and its prompt explicitly forbids mirror/content propagation. `## Relationships`-driven cross-project work now lives in the `planner` kind. `$PROJECT` is already captured at the top of `--run`; no project → no suggestions.
- **Model-alias sync:** the suggester reuses `$MODEL_FLAG`, so it tracks the task's `## Model` automatically — nothing extra to keep in sync with `claude-eval`.
- **Noise control:** capped at `suggest_max` (config, default 3), deduped via `slug_exists` against `inbox/backlog/backlog-for-review/suggestions/done`, and the prompt makes silence the default. Disable with `suggest_enabled=0`.
- **Not yet built (deliberate):** no TTL sweep of stale stubs, and dedup is slug-based only (near-duplicates are caught at the user's review step).

### Scheduled-job engine (`claude-scheduled`)

The pipeline is **pull-based**: nothing runs until a file lands in `~/tasks/inbox/`. `claude-scheduled` is the push side — a single cron-driven driver that generates work on a schedule from job definitions in `~/tasks/scheduled/*.md`. Cron fires it often (suggested `*/30 * * * *`); the script alone decides what runs.

- **Scheduling model — interval-since-last-run, NOT cron expressions.** Per-job state lives in `~/tasks/scheduled/.state/<slug>.run` (contents = last-run epoch; `<slug>` is the job filename without `.md`). `is_due` reads that epoch and compares elapsed time against the `## Schedule` grammar: `daily` (~24h), `weekly` (~7d), `weekly-<dow>` (today matches dow AND not already run today), `monthly` (month changed or ~28d), `every:<N>h` (≥ N hours). The schedule token is validated **before** the never-run shortcut, so an invalid/missing schedule is skipped with a warning even on a job that has never run. State is stamped (`date +%s`) **only on a successful dispatch**.
- **Double-gated autonomy (template kind).** Default target is `~/tasks/backlog-for-review/` (the review gate). A job renders straight into `~/tasks/inbox/` **only** when it has a `## Project` AND `~/tasks/projects/<project>.md` contains a line `## Automation: auto`. Both conditions required; the chosen route is logged. The rendered file is `<slug>-YYYY-MM-DD_HH-MM.md` (minute granularity — a same-minute re-dispatch overwrites rather than duplicates) and carries a next-line `## Project` header when the job declared one, so downstream `.locks/` serialization and the suggester keep working.
- **Auth-cooldown behaviour.** `template` jobs make no `claude` call, so they run even while `~/tasks/.auth-cooldown` is active. Agentic kinds (`planner`, and `monitor` once added) skip while the marker is younger than `auth_cooldown_min` and — importantly — do **not** stamp state, so they retry after the cooldown clears. The `AUTH_COOLDOWN_ACTIVE` boolean is computed up front with the same `find "$COOLDOWN" -mmin -"$AUTH_COOLDOWN_MIN"` test as `claude-task`.
- **`planner` kind (implemented).** An agentic cross-project planner. On a due, non-cooled-down run it gathers read-only, bounded context — the last ~15 `~/tasks/done/` files by mtime (their `## Summary` sections), every `~/tasks/projects/*.md` (goals/roadmap + `## Relationships`; the job's `## Project` is prioritised when set), the current queue filenames in `inbox/`/`backlog/`/`backlog-for-review/`/`suggestions/`, and the failures in `review/` — then runs one `claude -p` (wrapped in `timeout $((scheduled_timeout_min*60))`, separate stdout/stderr, `jq -r '.result // ""'`) asking for a **ranked** stub set capped at `plan_ahead_max` (config, default 5). Outcomes: `is_auth_failure` → touch `~/tasks/.auth-cooldown` + defer, **no** state stamp; `timeout` (rc 124) / empty `.result` → defer, **no** state stamp; `NONE` or a completed run → stamp state. Stubs use the SAME `===SUGGESTION===` fenced format + awk splitter + `slug_exists` dedup as the per-task suggester, so they flow through the pipeline unchanged; `notify-send` fires only if any stub was written. `is_auth_failure`/`slug_exists` are copied from `claude-task` (kept in sync) and defined **guarded** (`declare -F …`) so the future `monitor` task can add them too without a double definition.
- **`monitor` kind (implemented).** An agentic watcher built on the same skeleton as `planner`. On a due, non-cooled-down run it reads the job's saved snapshot (`$STATE_DIR/<slug>.snapshot.md`, treated as empty on the first run), then runs **one** `claude -p` (wrapped in `timeout $((scheduled_timeout_min*60))`, separate stdout/stderr, `jq -r '.result // ""'`) that performs the watch described in the job body and diffs its findings against that snapshot. The prompt forbids parking on async events (same headless rule as the autonomous PREAMBLE) and demands a machine-parseable reply: an always-present `===SNAPSHOT===` block, a `===SIGNAL===` yes/no block, and on `yes` a `===REPORT===` block plus (only when the job declares a `## Project`) one `===SUGGESTION===` stub. Blocks are sliced with the `extract_block` awk helper (prints between a start marker and the first following `===END===`). Outcomes mirror `planner`: `is_auth_failure` (checked on BOTH the stdout and stderr capture files) → touch `~/tasks/.auth-cooldown` + defer, **no** state stamp; `timeout` (rc 124) / empty `.result` → defer, **no** state stamp. Otherwise the snapshot block is written to the snapshot path (every successful run); on signal `yes` the report goes to `~/Notes/monitor-<slug>-YYYY-MM-DD.md` and, if the stub slug is new per `slug_exists`, a stub to `~/tasks/suggestions/<slug>.md` (a `notify-send` fires with the report path); on signal `no` nothing but the snapshot is written. **State is stamped only on a run that reached the snapshot write (signal yes OR no)** — never on an auth/timeout/empty deferral. The silence-on-no-signal is the whole point: no "nothing changed" noise. `is_auth_failure`/`slug_exists` are the copies from `claude-task` (kept in sync, guarded `declare -F`); the stub reuses the per-task suggester's `===SUGGESTION===` fenced format + `slug_exists` dedup + "body is everything after the first `---`" split.
- **Shared config for agentic kinds.** The `## Model` → `MODEL_FLAG` parser and `scheduled_timeout_min` config are used by both `planner` and `monitor` even though `template` ignores both.
- Config: `scheduled_enabled` (default 1; `0` exits immediately), `scheduled_timeout_min` (default 30), and `plan_ahead_max` (default 5, planner stub cap). Do **not** install the cron line — document it in README and leave it to the user.

### Adding a script

1. Write it in this directory with a `#!/bin/bash` shebang.
2. `chmod +x scriptname`
3. Symlink: `ln -s ~/Scripts/claude-utils/scriptname ~/bin/scriptname`
4. Document in README.md (user-facing) and add a note here if there are gotchas.
