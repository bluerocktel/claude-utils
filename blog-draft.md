# I stopped running Claude. Now it runs itself.

Most people use Claude Code the same way they use a senior developer: they ask, it answers, they review, they move on. That's already useful. But there's a different mode that most people haven't tried, and once you switch to it, going back feels like driving with the handbrake on.

Here's what my day looks like now. I write a rough description of what I need done, drop it in a folder, and go do something else. By the time I'm back, it's done.

Last week: I needed bilingual compliance documents (GDPR statement + security posture, EN + FR) formatted with our company letterhead, attached to an email, and sent. I wrote a two-paragraph task description. Seventy-seven minutes later, both PDFs were in my inbox, ready to send to a prospect. The pipeline had extracted the exact letterhead specs from a reference PDF, converted fonts to fix a ReportLab incompatibility, generated 8 and 9-page documents, and sent the email via the Gmail API. I didn't touch a keyboard after dropping the task.

That's not magic. It's a pipeline.

---

## The pipeline

The core idea is simple: treat Claude Code as a worker that picks tasks from a queue, not as a chatbot you interrogate.

```
backlog/ → claude-prep → backlog-for-review/ → (you review) → inbox/ → claude-task → done/
```

Every step is automated. The only thing I do manually is review prepared tasks and decide whether to promote them to the execution queue.

**Step 1: draft.** I write rough ideas, sometimes a single sentence, and drop them in `~/tasks/backlog/`. When I have a lot on my mind, I dump everything into one file and let `claude-draft` split it into individual task files.

**Step 2: prep.** `claude-prep` picks the oldest task, adds structure, acceptance criteria, and expected outcomes, and writes it to `~/tasks/backlog-for-review/`. I read the prepared task, adjust if needed, and promote it.

**Step 3: execute.** `claude-task` picks the next task from `~/tasks/inbox/`, runs Claude Code headlessly in a detached tmux session with full tool access, writes the result to `~/tasks/done/`, and sends a desktop notification when it's done.

**Step 4: evaluate.** After execution, `claude-eval` grades the result against the acceptance criteria. If anything fails, the task goes back to the queue with the evaluator's feedback appended. From the second failure onward, a strategist pass adds a `## Retry Strategy` section suggesting a concrete alternative approach. After three failed attempts, the task lands in `~/tasks/review/` for manual inspection.

**Step 5: cron.** All of steps 1-3 run on a 15-minute cron. I drop something in backlog before lunch, and it's executed before I'm back.

---

## What this unlocks

The obvious win is time. Tasks that would take me 30-90 minutes of focused work now happen while I'm in a meeting or asleep.

The less obvious win is parallelism. `claude-tasks` runs every task in `inbox/` simultaneously, each in its own tmux session. Last month I needed a full REST API built from scratch for a new product: authentication, resource endpoints, OpenAPI documentation, quiz delivery, content publication. Five tasks, five tmux sessions, running in parallel. Two hours later I had a working API with documentation.

The even less obvious win is quality enforcement. Because the eval loop grades every task against explicit criteria and feeds failure analysis back into retries, the output is more consistent than what you'd get from an ad-hoc session. The pipeline doesn't get tired. It doesn't skip the tests because it's 11pm.

---

## Project context files

One thing that makes this work at scale: I never repeat project-specific context in task files.

Each project has a single context file at `~/tasks/projects/{name}.md` that defines the working directory, local URL, tech stack notes, and anything else Claude needs to orient itself. Tasks just reference the project alias:

```markdown
## Project
bluerocklms

## Instructions
Add rate limiting to the public API endpoints...
```

Claude reads the project file automatically before starting. The task itself stays focused on the what, not the where-and-how.

---

## The repo

Everything described here is open source: [github.com/bluerocktel/claude-utils](https://github.com/bluerocktel/claude-utils).

It's a set of bash scripts. No framework, no dependencies beyond `claude`, `tmux`, and `s-nail`. Drop it on any Linux machine, set up a cron, and your backlog starts moving on its own.

---

*We build this kind of AI-assisted workflow automation at BlueRockTEL. If you're trying to figure out how to integrate Claude into your team's operations and want a conversation, reach out.*
