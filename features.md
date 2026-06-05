# Claude Code features to explore

Derived from the "Claude Code Full Course (4 hours)" transcript, filtered against the current headless workflow (`claude-task` picks tasks from `~/tasks/inbox/`, runs in detached tmux, no interactive session).

---

## Worth implementing

### Git worktrees for same-repo parallelism

`claude-tasks` parallelizes across task files, but two tasks targeting the same project share one working directory. Git worktrees give each `claude-task` instance an isolated copy of the repo on its own branch, allowing e.g. a migration task and a UI task on the same project to run concurrently without file conflicts. Requires adding a worktree convention to the relevant project's CLAUDE.md.

### Webhook trigger for the pipeline

The course shows exposing a lightweight HTTPS endpoint (Modal, Flask) that drops a task file directly into `~/tasks/inbox/`, bypassing the 15-minute cron cycle. Useful for event-driven triggers: GitHub PR merged, form submitted, external service callback. The cron approach requires up to 15 minutes of latency; a webhook fires immediately.

### Thinking budget control via CLI flag

`--thinking-budget` can be passed to the `claude` CLI invocation (rather than set interactively). Setting a low budget for cheap passes (`claude-prep`, `claude-eval`) and a high budget only for `claude-plan` would reduce cost without degrading output quality where it matters. Worth checking whether the current `claude` CLI version supports this flag.

### Skill `scripts/` subfolder

Skills can include a `scripts/` subfolder with shell or Python scripts that the skill orchestrates, rather than relying purely on LLM instructions. For multi-step data pipelines (API calls, scraping, sheet writes), moving the deterministic steps into scripts makes the skill cheaper, faster, and more reproducible.

---

## Niche / heavier

### `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

Setting this env var in `settings.json` unlocks an experimental mode where a team-lead Claude instance spawns fully independent peer instances that maintain their own context and communicate with each other. Heavier than the current subagent-via-Task pattern. Potentially useful for adversarial plan review (one agent writes the plan, another critiques it).

### Chrome DevTools / Playwright MCP for headless browser

The course demonstrates automating web UIs that lack APIs (purchasing flows, form submission, ClickUp tasks). The Playwright MCP is already available in the current environment (`mcp__playwright__*`), so no setup needed. This is a technique gap, not a tooling gap.
