# claude-utils

Shell utilities for running and managing [Claude Code](https://claude.ai/code) tasks autonomously.

Write a task as a Markdown file, drop it in `~/tasks/inbox/`, and let Claude handle it, either on demand or automatically via cron.

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
for script in claude-task claude-tasks claude-plan claude-plans ltask vtask mtask; do
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

Creates `~/tasks/{plan,planned,inbox,run,done}/` and drops a `sample-task.md` in inbox. Run once per machine.

```
claude-init
```

### `claude-plan`

Generates an implementation plan for a task in `~/tasks/plan/`. Runs Claude in planning mode (no code written, no files edited) and writes a structured, ready-to-execute Markdown file to `~/tasks/planned/`. Review and edit the plan there, then move it to `~/tasks/inbox/` to execute with `claude-task`.

```
claude-plan                  # picks oldest task from ~/tasks/plan/
claude-plan path/to/task.md  # plan a specific file
```

### `claude-plans`

Launches planning for all `.md` files in `~/tasks/plan/` in parallel, each in its own tmux session.

```
claude-plans
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

### `ltask`

Lists tasks in `~/tasks/plan/`, `~/tasks/planned/`, `~/tasks/inbox/`, and `~/tasks/run/`.

```
ltask
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

## Task folder layout

```
~/tasks/
  projects/ # project context files (one per project, see "Project context" section)
  plan/     # rough task descriptions waiting for a planning pass
  planned/  # Claude-generated plans awaiting review (move to inbox/ to execute)
  inbox/    # .md files ready to run
  run/      # tasks currently executing
  done/     # completed result files (timestamped)
  DONE.md   # running log of all completed/failed tasks
  cron.log  # cron output (if using crontab automation)
```

### Planning workflow

For complex tasks, use the two-phase flow:

1. Write a rough description and drop it in `~/tasks/plan/`
2. Run `claude-plan` — Claude analyzes and outputs a structured plan to `~/tasks/planned/`
3. Review and optionally edit the plan file in `planned/`
4. Move it to `~/tasks/inbox/` and run `claude-task` as normal

For simple tasks, skip planning and drop directly into `~/tasks/inbox/`.

## Crontab automation

To pick up and run inbox tasks automatically, add `claude-task` to your crontab:

```
crontab -e
```

Recommended entry (every 15 minutes):

```
*/15 * * * * $HOME/bin/claude-task >> $HOME/tasks/cron.log 2>&1
```

`claude-task` exits silently with no error when the inbox is empty, so frequent polling is safe.

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
  projects/   # project context files (one per project)
  plan/       # rough task descriptions waiting for a planning pass
  planned/    # Claude-generated plans awaiting review
  inbox/      # .md files ready to run
  run/        # tasks currently executing
  done/       # completed result files (timestamped)
  DONE.md     # running log of all completed/failed tasks
  cron.log    # cron output (if using crontab automation)
```

## Writing a task

Tasks are plain Markdown files. See `~/tasks/inbox/sample-task.md` for a template. A good task includes:

- **Project**: alias referencing `~/tasks/projects/{name}.md` (replaces manual context)
- **Instructions**: numbered steps with clear expected actions
- **Expected outcome**: what success looks like

Claude runs fully autonomously: no questions, no prompts. Be explicit.

## Task output format

Each result file in `~/tasks/done/` ends with two standard sections appended by Claude:

- **`## Summary`**: what was done, key decisions made, and any follow-up actions to be aware of.
- **`## Suggested Commit Message`**: a ready-to-use git commit message (subject line + optional body) describing all changes made. Claude does not run `git commit` itself.

## License

MIT
