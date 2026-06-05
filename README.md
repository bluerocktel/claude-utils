# claude-utils

Shell utilities for running and managing [Claude Code](https://claude.ai/code) tasks autonomously.

Write a task as a Markdown file, drop it in `~/tasks/backlog/`, let Claude prepare it with acceptance criteria, review it, then execute it autonomously. Or write a high-level brief in `~/tasks/briefs/` and let Claude decompose it into multiple ready-to-run task files in one shot.

> **Note:** `claude-task` uses `--dangerously-skip-permissions`, which allows Claude to run shell commands, edit files, and use all tools without confirmation prompts. Only use this on machines and tasks you trust.

## Requirements

- [Claude Code](https://claude.ai/code) (`claude` CLI on `$PATH`)
- `tmux`
- `s-nail` (for `mtask` email sending)
- `~/bin/` on your `$PATH`

## Install

```bash
git clone https://github.com/bluerocktel/claude-utils ~/Scripts/claude-utils
cd ~/Scripts/claude-utils
cp config.example config
# Edit config with your own values
chmod +x ~/Scripts/claude-utils/*
for script in claude-brief claude-briefs claude-draft claude-prep claude-preps claude-task claude-tasks claude-plan claude-plans claude-eval claude-review-alert ltask vtask mtask dtask; do
  ln -s ~/Scripts/claude-utils/$script ~/bin/$script
done
ln -s ~/Scripts/claude-utils/init ~/bin/claude-init
claude-init
```

## Config

`config` is a key=value file (not committed). Copy from `config.example` and edit:

| Key     | Description                       |
|---------|-----------------------------------|
| `email` | Recipient address used by `mtask` |

## Scripts

### `claude-init`

Creates `~/tasks/{drafts,backlog,backlog-for-review,plan,plan-for-review,inbox,run,done,review}/` and drops a `sample-task.md` in backlog. Run once per machine.

```
claude-init
```

### `claude-brief`

Decomposes a high-level brief into multiple well-scoped task files ready for review. Write a brief describing a feature, fix, or batch of related changes in `~/tasks/briefs/`, then run `claude-brief`. Claude reads the project file, explores the codebase to find exact file paths and line numbers, decides how many tasks are needed, and writes each one to `~/tasks/backlog-for-review/` with full structure and acceptance criteria. Use this when you know what you want to build but don't want to write each task individually.

Brief format (minimal):
```markdown
## Project
myproject

Describe the feature or changes you want. Write as loosely as you like.
Claude will explore the codebase and create as many tasks as needed.
```

```
claude-brief                   # picks oldest brief from ~/tasks/briefs/
claude-brief path/to/brief.md  # process a specific file
```

### `claude-briefs`

Launches decomposition for all `.md` files in `~/tasks/briefs/` in parallel, each in its own tmux session.

```
claude-briefs
```

### `claude-draft`

Splits a brain dump into individual backlog task files. Write all your ideas and cross-project notes into a single `.md` file in `~/tasks/drafts/`, then run `claude-draft` to have Claude parse it and write one focused file per task to `~/tasks/backlog/`. From there, proceed with `claude-prep` as usual.

```
claude-draft                   # picks oldest draft from ~/tasks/drafts/
claude-draft path/to/draft.md  # process a specific file
```

### `claude-prep`

Prepares a rough backlog task by adding structure and acceptance criteria. Picks the oldest `.md` from `~/tasks/backlog/` if no argument is given. Outputs a ready-to-review task file to `~/tasks/backlog-for-review/`. From there, move it to `~/tasks/inbox/` (simple tasks) or `~/tasks/plan/` (complex tasks needing a full implementation plan).

```
claude-prep                  # picks oldest task from ~/tasks/backlog/
claude-prep path/to/task.md  # prepare a specific file
```

### `claude-preps`

Launches preparation for all `.md` files in `~/tasks/backlog/` in parallel, each in its own tmux session.

```
claude-preps
```

### `claude-plan`

Generates an implementation plan for a task in `~/tasks/plan/`. Runs Claude in planning mode (no code written, no files edited) and writes a structured, ready-to-execute Markdown file to `~/tasks/plan-for-review/`. Review and edit the plan there, then move it to `~/tasks/inbox/` to execute with `claude-task`.

```
claude-plan                  # picks oldest task from ~/tasks/plan/
claude-plan path/to/task.md  # plan a specific file
```

### `claude-plans`

Launches planning for all `.md` files in `~/tasks/plan/` in parallel, each in its own tmux session.

```
claude-plans
```

### `claude-eval`

Evaluates a completed task result against its `## Acceptance Criteria`. Called automatically by `claude-task` after each execution — you rarely need to run it directly. If all criteria pass, the task proceeds to `done/`. If any fail, the task is re-queued to `inbox/` with the evaluator's feedback appended. On the second failed attempt and beyond, a strategist pass runs before re-queuing: it reads the full failure history and appends a `## Retry Strategy` section suggesting a concrete alternative approach for the next attempt. After `max_retries` failed attempts (default: 3), the task moves to `review/` for manual inspection.

Tasks without an `## Acceptance Criteria` section skip evaluation entirely.

```
claude-eval path/to/task.md path/to/result.md   # manual evaluation
```

### `claude-task`

Runs a single task file autonomously in a detached tmux session. Picks the oldest `.md` from `~/tasks/inbox/` if no argument is given. Results are written to `~/tasks/done/` (timestamped) and logged to `~/tasks/DONE.md`. Sends a desktop notification on completion or failure.

```
claude-task                  # picks oldest task from inbox
claude-task path/to/task.md  # run a specific file
```

### `claude-tasks`

Launches all `.md` files in `~/tasks/inbox/` in parallel, each in its own tmux session.

```
claude-tasks
```

### `dtask`

Dashboard view of the full task pipeline. Shows items needing your attention, pipeline stage counts, running tasks with elapsed time, today's stats, a per-project breakdown, and recent activity.

```
dtask                # one-shot view
while true; do clear; dtask; sleep 30; done    # live refresh every 30 seconds
```

### `ltask`

Lists tasks across all stages: backlog, backlog-for-review, plan, plan-for-review, inbox, run, and review.

```
ltask
```

### `ltask-cost`

Reads all completed task files in `~/tasks/done/`, aggregates token spend by project and model, and writes a Markdown report to `~/Notes/claude-task-spend-YYYY-WXX.md` for the current ISO week. Re-running overwrites the same file. Tasks with no `## Token Usage` section are skipped silently.

```
ltask-cost
```

### `vtask`

Opens a done task in `$EDITOR`. Defaults to the most recent. Pass `N` to open the Nth-to-last.

```
vtask      # most recent
vtask 2    # second-to-last
```

### `mtask`

Emails a done task to the address set in `config`, using `~/.mailrc` for SMTP settings. Subject is the filename without `.md`.

```
mtask      # most recent
mtask 2    # second-to-last
```

### `claude-review-alert`

Sends a critical desktop notification listing any tasks stuck in `~/tasks/review/` after exhausting all retries. Designed to run from cron every 30 minutes. Exits silently when `review/` is empty.

```
claude-review-alert
```

### `morning-briefing`

Generates a daily Markdown report of the last 24 hours of task activity and writes it to `~/Notes/briefing-YYYY-MM-DD.md`. The report contains four sections: tasks completed, inbox queue, backlog snapshot (with age warnings for items older than 14 days), and a spend table summarising API cost by project. A cron entry runs it automatically at 07:00 each day.

```
morning-briefing           # write report to ~/Notes/briefing-YYYY-MM-DD.md
morning-briefing --print   # write report and also print to stdout
```

**Cron entry (added automatically during install):**
```
0 7 * * * $HOME/bin/morning-briefing >> $HOME/tasks/cron.log 2>&1
```

## Task folder layout

```
~/tasks/
  projects/           # project context files (one per project, see "Project context" section)
  briefs/             # high-level feature descriptions waiting to be decomposed by claude-brief
  briefs/done/        # archived briefs after decomposition (referenced by ## Brief in tasks)
  drafts/             # brain dumps for claude-draft to split into backlog tasks
  backlog/            # rough task ideas, brief descriptions
  backlog-for-review/ # Claude-prepared tasks with acceptance criteria (review before promoting)
  plan/               # tasks needing a full implementation plan
  plan-for-review/    # Claude-generated plans awaiting review (move to inbox/ to execute)
  inbox/              # .md files ready to run
  run/                # tasks currently executing
  done/               # completed result files (timestamped)
  review/             # tasks that failed evaluation after max retries (need manual intervention)
  DONE.md             # running log of all completed/failed/requeued tasks
  cron.log            # cron output (if using crontab automation)
```

### Task workflow

**Fast path: start from a brief (recommended for multi-task work)**

0. Write a high-level description of a feature or set of changes in `~/tasks/briefs/`, then run `claude-brief`: Claude explores the codebase and writes N structured tasks directly to `~/tasks/backlog-for-review/`

**Optional: start from a brain dump**

0. Write all your ideas in one file and drop it in `~/tasks/drafts/`, then run `claude-draft`: Claude splits it into individual files in `~/tasks/backlog/`

**Standard flow**

1. Write a brief idea and drop it in `~/tasks/backlog/`
2. Run `claude-prep`: Claude adds structure, acceptance criteria, and outputs to `~/tasks/backlog-for-review/`
3. Review the prepared task, then:
   - **Simple task:** move to `~/tasks/inbox/` and run `claude-task`
   - **Complex task:** move to `~/tasks/plan/` for a full implementation plan

### Planning workflow (for complex tasks)

1. Move a reviewed task from `backlog-for-review/` to `~/tasks/plan/`
2. Run `claude-plan`: Claude generates a structured plan to `~/tasks/plan-for-review/`
3. Review and optionally edit the plan
4. Move it to `~/tasks/inbox/` and run `claude-task`

## Crontab automation

The full pipeline can run automatically. The only step that requires your attention is reviewing prepared tasks in `backlog-for-review/` and promoting them to `inbox/`.

Add these entries to your crontab (`crontab -e`):

```
*/15 * * * * $HOME/bin/claude-brief        >> $HOME/tasks/cron.log 2>&1
*/15 * * * * $HOME/bin/claude-draft        >> $HOME/tasks/cron.log 2>&1
*/15 * * * * $HOME/bin/claude-prep         >> $HOME/tasks/cron.log 2>&1
*/15 * * * * $HOME/bin/claude-task         >> $HOME/tasks/cron.log 2>&1
*/30 * * * * $HOME/bin/claude-review-alert >> $HOME/tasks/cron.log 2>&1
```

All scripts exit silently when there is nothing to process, so frequent polling is safe. With these entries the flow becomes:

1. Drop a brief in `~/tasks/briefs/` (or a brain dump in `~/tasks/drafts/`, or a rough task in `~/tasks/backlog/`) — cron does the rest.
2. Review prepared tasks in `~/tasks/backlog-for-review/` and move approved ones to `~/tasks/inbox/`.
3. Receive a desktop alert if any task exhausts its retries and lands in `~/tasks/review/`.

## Project context

Instead of repeating working directory, URLs, and stack details in every task, define them once in a project file under `~/tasks/projects/`.

### Project file format

```markdown
## Type
code  # or: writing

## Aliases
- `myproject`

## Working Directory
`~/Projects/myproject/`

## CLAUDE.md
`~/Projects/myproject/CLAUDE.md`

## Local URL
`https://myproject.test`
```

The `## Type` field controls which conventions Claude applies:
- `code`: coding rules apply (implementation plan, careful file editing, etc.)
- `writing`: text rules apply (prose quality, no code conventions)

### Linking a task to a project

Add a `## Project` section at the top of your task file with the project alias:

```markdown
## Project
myproject

## Instructions

Your task here.
```

Claude will read `~/tasks/projects/myproject.md` automatically before starting work.

### Task folder layout (full)

```
~/tasks/
  projects/           # project context files (one per project)
  briefs/             # high-level feature descriptions waiting to be decomposed
  briefs/done/        # archived briefs after decomposition (referenced by ## Brief in tasks)
  drafts/             # brain dumps for claude-draft to split into backlog tasks
  backlog/            # rough task ideas, brief descriptions
  backlog-for-review/ # Claude-prepared tasks with acceptance criteria
  plan/               # tasks needing a full implementation plan
  plan-for-review/    # Claude-generated plans awaiting review
  inbox/              # .md files ready to run
  run/                # tasks currently executing
  done/               # completed result files (timestamped)
  review/             # tasks that failed evaluation after max retries
  DONE.md             # running log of all completed/failed/requeued tasks
  cron.log            # cron output (if using crontab automation)
```

## Task size rule

Both `claude-prep` and `claude-brief` enforce a size constraint: a task that touches more than 5 files or spans more than 3 distinct logical phases is split automatically into multiple smaller tasks connected with `## Depends-on`. This keeps each task within the model's effective context window and avoids silent quality degradation from mid-session context compaction.

If `claude-prep` splits a task, it outputs multiple files to `backlog-for-review/` instead of one, and sends a "Task Split" notification.

## Writing a task

Tasks are plain Markdown files. See `~/tasks/backlog/sample-task.md` for a template. A backlog task can be brief: just describe what you want done. `claude-prep` will add structure and acceptance criteria. A fully prepared task includes:

- **Project**: alias referencing `~/tasks/projects/{name}.md` (replaces manual context)
- **Instructions**: numbered steps with clear expected actions
- **Expected outcome**: what success looks like
- **Acceptance Criteria**: concrete pass/fail items for the evaluator (optional, but recommended for complex tasks)

Claude runs fully autonomously: no questions, no prompts. Be explicit.

### Acceptance Criteria and evaluation

If your task file contains an `## Acceptance Criteria` section, `claude-eval` runs automatically after execution and grades the result against each criterion. Failed tasks are re-queued with the evaluator's feedback appended. From the second failure onward, a strategist pass also runs: it reviews the full failure history and appends a `## Retry Strategy` section with a concrete alternative approach, so repeated attempts don't keep hitting the same wall.

```markdown
## Acceptance Criteria
- All existing tests pass
- No new lint warnings introduced
- Migration runs without errors on empty database
```

The retry limit defaults to 3 and can be changed by setting `max_retries=N` in your `config` file. Tasks that exhaust all retries are moved to `~/tasks/review/` and appear in `ltask` output for manual inspection.

### Task dependencies

Add a `## Depends-on` section to declare that a task must not run until another task has completed successfully. When `claude-task` picks the next task from `inbox/`, it skips any task whose dependency is not yet in `done/` and picks the next unblocked one instead. This keeps tasks within the same project sequential without preventing tasks from other projects from running in parallel.

```markdown
## Depends-on
002-seed-database
```

The value is a task filename without the timestamp prefix or `.md` extension (match the base name you gave the task when you created it). If all tasks in `inbox/` are blocked, `claude-task` exits with a clear message rather than silently doing nothing.

### Test Command

Add a `## Test Command` section to run real shell commands before the LLM evaluator. Each non-blank line is executed in sequence. The first command that exits non-zero immediately fails the task (VERDICT: FAIL) without calling the LLM, saving API cost and giving a precise failure signal.

```markdown
## Test Command
cd ~/Projects/myproject && php artisan test --filter=MyFeatureTest
cd ~/Projects/myproject && npm run lint
```

If the task has a `## Test Command` but no `## Acceptance Criteria`, a passing test is enough to mark the task PASS. Commands have access to the full shell environment.

## Task output format

Each result file in `~/tasks/done/` ends with two standard sections appended by Claude:

- **`## Summary`**: what was done, key decisions made, and any follow-up actions to be aware of.
- **`## Suggested Commit Message`**: a ready-to-use git commit message (subject line + optional body) describing all changes made. Claude does not run `git commit` itself.

## License

MIT
