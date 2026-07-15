# I stopped running Claude. Now it runs itself.

Most people use Claude Code the same way they use a senior developer: they ask, it answers, they review, they move on. That's already useful. But there's a different mode that most people haven't tried, and once you switch to it, going back feels like driving with the handbrake on.

Here's what my day looks like now. I write a rough description of what I need done, drop it in a folder, and go do something else. By the time I'm back, it's done.

Last week: I needed bilingual compliance documents (GDPR statement + security posture, EN + FR) formatted with our company letterhead, attached to an email, and sent. I wrote a two-paragraph task description. Seventy-seven minutes later, both PDFs were in my inbox, ready to send to a prospect. The pipeline had extracted the exact letterhead specs from a reference PDF, converted fonts to fix a ReportLab incompatibility, generated 8 and 9-page documents, and sent the email via the Gmail API. I didn't touch a keyboard after dropping the task.

That's not magic. It's a pipeline.

---

## The pipeline

The core idea is simple: treat Claude Code as a worker that picks tasks from a queue, not as a chatbot you interrogate.

There are two ways to feed it work.

**The brief path** (for anything non-trivial):

```
briefs/ → claude-brief → backlog-for-review/ → (you review) → inbox/ → claude-task → done/
```

**The standard path** (for single, well-understood tasks):

```
backlog/ → claude-prep → backlog-for-review/ → (you review) → inbox/ → claude-task → done/
```

In both cases, the only thing I do manually is review prepared tasks and decide whether to promote them to the execution queue.

**Step 1: describe.** I write a high-level description of a feature or investigation — loosely, in plain language — and drop it in `~/tasks/briefs/`. A single brief can describe work that spans multiple files, components, or even multiple execution steps. I don't think about structure at this point. That's the machine's job.

**Step 2: decompose.** `claude-brief` picks up the file, reads the project context, explores the actual codebase with grep and glob to find exact file paths and line numbers, and decomposes the brief into N focused task files. Each task is scoped to fit within a single focused session. Tasks that depend on a prior task's output are automatically chained with a `## Depends-on` field. Every task keeps a `## Brief` pointer back to the original description so the executing agent always has the full intent available.

**Step 3: prep** (standard path). For simpler, already-scoped work I still drop rough ideas into `~/tasks/backlog/` and let `claude-prep` add structure, acceptance criteria, and expected outcomes. If a task turns out to be too large, prep splits it automatically.

**Step 4: execute.** `claude-task` picks the next unblocked task from `~/tasks/inbox/`, runs Claude Code headlessly in a detached tmux session with full tool access, writes the result to `~/tasks/done/`, and sends a desktop notification when it's done. Blocked tasks — those whose `## Depends-on` dependency hasn't landed in `done/` yet — are skipped and picked up on the next pass.

**Step 5: evaluate.** After execution, `claude-eval` grades the result against the acceptance criteria. If anything fails, the task goes back to the queue with the evaluator's feedback appended. From the second failure onward, a strategist pass adds a `## Retry Strategy` section suggesting a concrete alternative approach. After three failed attempts, the task lands in `~/tasks/review/` for manual inspection.

**Step 6: cron.** All of steps 2-4 run on a 15-minute cron. I drop a brief before lunch, and a set of sequenced tasks are executing before I'm back.

---

## What this unlocks

The obvious win is time. Tasks that would take me 30-90 minutes of focused work now happen while I'm in a meeting or asleep.

The less obvious win is parallelism with automatic sequencing. `claude-tasks` runs every unblocked task in `inbox/` simultaneously, each in its own tmux session. Tasks that produce output for the next step wait their turn automatically — no manual orchestration. Last month I needed a full REST API built from scratch for a new product: authentication, resource endpoints, OpenAPI documentation, quiz delivery, content publication. Five tasks, five tmux sessions, running in parallel. Two hours later I had a working API with documentation.

The even less obvious win is decomposition quality. When I write a brief, Claude explores the actual codebase before producing a single task file. The result is tasks that reference real line numbers, real class names, real file paths — not guesses. A task that says "update `Telecom0516Service.php` line 65" is not the same as one that says "find and update the billing service." The former executes reliably. The latter is a lottery.

The quality enforcement compounds through the eval loop: every task is graded against explicit criteria, and failure analysis feeds back into retries. The pipeline doesn't get tired. It doesn't skip the tests because it's 11pm.

---

## Project context files

One thing that makes this work at scale: I never repeat project-specific context in task files.

Each project has a single context file at `~/tasks/projects/{name}.md` that defines the working directory, local URL, tech stack notes, and anything Claude needs to orient itself — including execution conventions like which Docker container to use for running commands. Tasks just reference the project alias:

```markdown
## Project
bluerocklms

## Instructions
Add rate limiting to the public API endpoints...
```

Claude reads the project file automatically before starting. The task itself stays focused on the what, not the where-and-how. When a convention changes — a new container, a different test runner, a renamed command — I update it in one place and every future task inherits it.

---

## The repo

Everything described here is open source: [github.com/bluerocktel/claude-utils](https://github.com/bluerocktel/claude-utils).

It's a set of bash scripts. No framework, no dependencies beyond `claude`, `tmux`, and `s-nail`. Drop it on any Linux machine, set up a cron, and your backlog starts moving on its own.

---

*We build this kind of AI-assisted workflow automation at BlueRockTEL. If you're trying to figure out how to integrate Claude into your team's operations and want a conversation, reach out.*
