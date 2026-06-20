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

### Task size rule

Both `claude-prep` and `claude-brief` enforce a size constraint in their preambles: a task touching more than 5 files or spanning more than 3 distinct logical phases must be split into smaller tasks connected with `## Depends-on`. This keeps each task within the model's effective context window (~40%) and prevents silent quality degradation from context compaction.

### Adding a script

1. Write it in this directory with a `#!/bin/bash` shebang.
2. `chmod +x scriptname`
3. Symlink: `ln -s ~/Scripts/claude-utils/scriptname ~/bin/scriptname`
4. Document in README.md (user-facing) and add a note here if there are gotchas.
