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
for script in claude-task claude-tasks ltask vtask mtask; do
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

Creates `~/tasks/{inbox,run,done}/` and drops a `sample-task.md` in inbox. Run once per machine.

```
claude-init
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

Lists tasks currently in `~/tasks/inbox/` and `~/tasks/run/`.

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
  inbox/    # .md files waiting to be picked up
  run/      # tasks currently executing
  done/     # completed result files (timestamped)
  DONE.md   # running log of all completed/failed tasks
  cron.log  # cron output (if using crontab automation)
```

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

## Writing a task

Tasks are plain Markdown files. See `~/tasks/inbox/sample-task.md` for a template. A good task includes:

- **Context**: working directory, relevant files, background
- **Tasks**: numbered steps with clear expected actions
- **Expected outcome**: what success looks like

Claude runs fully autonomously: no questions, no prompts. Be explicit.

## License

MIT
