# CLAUDE.md - claude-utils

See README.md for full project documentation, script descriptions, and setup instructions.

## For Claude: working in this repo

### What this repo is

Shell scripts that drive autonomous Claude Code task execution. The core loop:
`claude-task` moves a `.md` file from `inbox/` to `run/`, calls `claude --dangerously-skip-permissions -p`, writes output to `done/`, then notifies the user.

### Conventions

- All scripts: `#!/bin/bash`, no external dependencies beyond what README lists.
- Scripts must be safe to run with an empty inbox (exit 0, print nothing or a short message).
- `config` is sourced as key=value. Never hardcode user-specific values (email, paths beyond `$HOME`).
- Symlink names in `~/bin/` matter: `init` is exposed as `claude-init` to avoid clashing with system commands.

### What to watch out for

- `claude-task` calls itself via `$HOME/bin/claude-task --run` for the internal tmux execution. If you rename the script, update that self-reference on line 100.
- `claude-tasks` also references `$HOME/bin/claude-task` directly. Keep both in sync if the name changes.
- `mtask` sources `config` at runtime via `realpath "$0"`, so it works correctly through the symlink.

### Adding a script

1. Write it in this directory with a `#!/bin/bash` shebang.
2. `chmod +x scriptname`
3. Symlink: `ln -s ~/Scripts/claude-utils/scriptname ~/bin/scriptname`
4. Document in README.md (user-facing) and add a note here if there are gotchas.
