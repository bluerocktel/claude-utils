# Scheduled jobs (`~/tasks/scheduled/`)

Job definitions read by `claude-scheduled`. Drop one `.md` file per job into
`~/tasks/scheduled/`. Cron fires `claude-scheduled` often (e.g. every 30 min);
the script alone decides which jobs are *due* and dispatches them.

The file's basename (without `.md`) is the job **slug** — it names the per-job
state stamp (`~/tasks/scheduled/.state/<slug>.run`) and prefixes every rendered
task file (`<slug>-YYYY-MM-DD_HH-MM.md`).

## Job file format

```markdown
## Kind: template
## Schedule: weekly-mon
## Project: claude-utils       (optional)
## Model: opus                 (optional; ignored by the template kind)

# Body heading

The Markdown body below the headers becomes the rendered task body.
```

Headers use the inline `## Field: value` form.

### `## Kind`

| Kind       | Behaviour                                                              |
|------------|-----------------------------------------------------------------------|
| `template` | Deterministic recurring task; the body is copied into the pipeline.   |
| `monitor`  | Agentic watcher — runs one `claude -p` that performs the watch, diffs against a saved snapshot, and on a real signal writes a dated report to `~/Notes/` **and** a suggestion stub to `~/tasks/suggestions/`. **Silent** (snapshot only, no report/stub) when nothing changed. See `market-monitor.md`. |
| `planner`  | Agentic — reads recent done/, project roadmaps, the queues and review/ failures, and proposes a ranked set of next-task stubs into `~/tasks/suggestions/` (capped at `plan_ahead_max`). See `weekly-planner.md`. |

### `## Schedule` grammar

Due-ness is **interval-since-last-run**, not a cron expression.

| Schedule       | Due when …                                                        |
|----------------|-------------------------------------------------------------------|
| `daily`        | ~24h since the last run                                           |
| `weekly`       | ~7d since the last run                                            |
| `weekly-<dow>` | today is `<dow>` (mon/tue/wed/thu/fri/sat/sun) and not run today  |
| `monthly`      | the calendar month changed, or ~28d elapsed                       |
| `every:<N>h`   | at least `N` hours (positive integer) since the last run         |

A job with no/invalid `## Schedule` is skipped with a warning.

### `## Project` (optional)

If set, the rendered task carries a `## Project` line (so downstream per-project
locking and the follow-up suggester work). It also controls autonomy routing:

- **Default** — the rendered task lands in `~/tasks/backlog-for-review/` (you
  review it before it runs).
- **Pure automation** — if `~/tasks/projects/<project>.md` contains a line
  `## Automation: auto`, the task is rendered straight into `~/tasks/inbox/`.

## Suggested cron line

```
*/30 * * * * $HOME/bin/claude-scheduled >> $HOME/tasks/cron.log 2>&1
```
