<h1 align="center">autopilot</h1>

<p align="center">
  <strong>Autonomous work mode for Claude Code — works through plans without stopping, with push notifications</strong>
</p>

<p align="center">
  <a href="https://github.com/danielcregg/autopilot/blob/main/LICENSE"><img src="https://img.shields.io/badge/License-MIT-blue.svg" alt="License: MIT"></a>
  <img src="https://img.shields.io/badge/Version-3.0.0-green.svg" alt="Version 3.0.0">
  <img src="https://img.shields.io/badge/Claude%20Code-blueviolet.svg" alt="Claude Code">
  <img src="https://img.shields.io/badge/Claude%20Desktop-blueviolet.svg" alt="Claude Desktop">
</p>

<p align="center">
  Set it and forget it. Claude works through complex plans autonomously — no stopping for questions, no waiting idle, real-time push notifications to your phone. The only native Claude Code skill for unattended autonomous work.
</p>

---

## The Problem

Claude Code stops frequently to ask questions, waits for your approval on every step, and sits idle while background tasks run. If you step away for an hour, you come back to find it waiting for you instead of working.

## The Solution

```
/autopilot Build a REST API with auth, database, tests, and deploy to staging
```

Claude works through the entire plan autonomously:
- Breaks it into tasks and executes them in order
- Parallelizes independent work using background tasks
- Uses Codex/Gemini for code reviews instead of asking you
- Sends push notifications to your phone so you can monitor from anywhere
- Includes circuit breakers — doesn't loop forever on failures
- Tracks progress via the task system so you always know where things stand

---

## Quick Start

```bash
# Install
mkdir -p ~/.claude/skills
git clone https://github.com/danielcregg/autopilot.git ~/.claude/skills/autopilot
```

For **Claude Desktop**: Settings > Skills > Upload skill > drag and drop `SKILL.md`.

---

## Three Modes

| Mode | Invoke With | What It Does |
|------|------------|--------------|
| **Execute** | `/autopilot <plan>` | Work through a plan from start to finish |
| **Continue** | `/autopilot continue` | Resume from where you left off |
| **Monitor** | `/autopilot monitor` | Watch running services, alert on problems |

Modes combine — execute a plan while monitoring services in parallel.

---

## How It Looks in Practice

### You start the work:
```
/autopilot Build a Python Flask API with JWT auth, SQLite database,
CRUD endpoints for users and posts, unit tests, and deploy to Railway.
```

### Claude works autonomously:

```
[claude] Starting task 1/8: Project scaffolding
  📱 → "Task 1/8: Setting up Flask project structure"

[claude] Task 1/8 complete. Starting task 2/8: Database models
  📱 → "Task 1/8 complete. Moving to database models."

[claude] Task 2/8 complete. Tasks 3 and 4 are independent — parallelizing.
  Starting task 3/8: Auth middleware (foreground)
  Starting task 4/8: CRUD endpoints (background)
  📱 → "Tasks 3 & 4 running in parallel"

[claude] Task 3 done. Checking on task 4... still running.
  Starting task 5/8: Unit tests (while waiting)
  📱 → "Auth complete. Tests started. CRUD still running."

[claude] Task 4/8 background complete. Task 5 tests failing...
  Attempt 1: Fix import error → still failing
  Attempt 2: Fix fixture → still failing
  Attempt 3: Asking Codex for review...
  Codex found: missing database teardown in conftest.py
  Applied fix → tests passing ✓
  📱 → "Tests fixed with Codex help. 6/8 tasks done."

[claude] All tasks complete. Running final verification...
  ✓ All tests pass (23/23)
  ✓ Lint clean
  ✓ Deploy successful
  📱 → "🎉 ALL DONE: API deployed to Railway. 8/8 tasks complete."
```

### You check your phone whenever you want:

```
📱 14:01  Task 1/8: Setting up Flask project structure
📱 14:03  Task 1/8 complete. Moving to database models.
📱 14:08  Tasks 3 & 4 running in parallel
📱 14:15  Auth complete. Tests started. CRUD still running.
📱 14:22  Tests fixed with Codex help. 6/8 tasks done.
📱 14:31  🎉 ALL DONE: API deployed. 8/8 tasks complete.
```

---

## Push Notifications (Optional)

Uses [ntfy.sh](https://ntfy.sh) — a free, open-source push notification service. No account needed.

1. Claude generates a unique topic when the skill starts
2. You subscribe on your phone: download the ntfy app → subscribe to the topic
3. You get real-time updates as Claude works

**No ntfy? No problem.** The skill works without notifications — it just tracks progress via the task system instead.

---

## Key Features

### Unified Naming (tmux + Claude + ntfy)

One name for everything. When the skill starts, it generates a session name like `myproject-437` and uses it for:

```
tmux session:    tmux attach -t myproject-437
Claude session:  claude --resume myproject-437
ntfy topic:      https://ntfy.sh/myproject-437
```

- The 3-digit random suffix makes the ntfy topic hard to guess
- tmux sessions survive SSH disconnects — reconnect and see exactly where things stand
- Claude sessions can be resumed by name if context is cleared

### Circuit Breaker

Three consecutive failures on the same task? Claude stops retrying, notifies you, and moves to the next task. No infinite loops.

```
Attempt 1: Try the straightforward approach
Attempt 2: Debug, try a different approach
Attempt 3: Ask Codex/Gemini for help
→ All failed? Notify user, move on.
```

### Parallel Execution

Independent tasks run simultaneously. Claude doesn't wait for one to finish before starting another.

### External AI Reviews

Instead of asking you for code reviews, Claude asks Codex or Gemini:
- Before deploying or running expensive operations
- When debugging persistent failures
- For architecture decisions

### Zero Idle Time

```
NEVER: Stop to ask a question
NEVER: Wait idle for a background task
NEVER: Give up without trying 3 approaches

ALWAYS: Send a notification
ALWAYS: Track progress
ALWAYS: Move to the next task
```

---

## Monitor Mode

Watch running services and get alerted on problems:

```
/autopilot monitor api.example.com staging-db.internal
```

| Check Type | Default Interval | Examples |
|---|---|---|
| Critical services | 15 min | API endpoints, databases |
| Infrastructure | 30 min | SSH hosts, containers, disk |
| Batch jobs | 1 hr | CI pipelines, long builds |
| Full status report | 8 hr | Aggregate everything |

Only notifies on **changes or problems** — no spam.

---

## Compared to Alternatives

| Feature | autopilot | Ralph Loop (bash) | pickle-rick | wiggum |
|---------|:------------:|:-----------------:|:-----------:|:------:|
| Native Claude Code skill | Yes | No | No | No |
| Unified naming (tmux + Claude + ntfy) | Yes | No | No | No |
| tmux session management | Yes | No | Partial | No |
| Push notifications | Yes (ntfy) | No | No | No |
| Service monitoring (via /loop) | Yes | No | No | No |
| External AI reviews | Yes (Codex + Gemini) | No | No | No |
| Circuit breakers | Yes | No | Yes | Yes |
| Parallel execution | Yes | No | Partial | No |
| Progress tracking | Yes (tasks) | Via git | Via git | Via git |
| Works inside Claude | Yes | External wrapper | External wrapper | External wrapper |
| Rate limit recovery | Yes | No | Yes | No |
| Uses Claude Code built-ins (/loop, /rename) | Yes | N/A | N/A | N/A |
| No external setup | Yes | Needs bash script | Needs bash script | Needs brew |

---

## When Does Claude Contact You?

Only for:
1. **Destructive actions** — delete data, force push, deploy to production
2. **Genuinely stuck** — 3 attempts + Codex review all failed
3. **Plan change needed** — fundamental assumption was wrong
4. **All done** — final summary report
5. **Service down** — monitored service is unresponsive

Everything else is handled autonomously with push notifications.

---

## Installation

### Claude Code (CLI)

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/danielcregg/autopilot.git ~/.claude/skills/autopilot
```

### Claude Desktop

1. Download `SKILL.md` from this repo
2. Open Claude Desktop > **Settings** > **Skills**
3. Click **Upload skill**
4. Drag and drop the `SKILL.md` file

---

## Usage Examples

### Build a full project
```
/autopilot Build a Next.js dashboard with Supabase auth, Prisma ORM,
CRUD for projects and tasks, Tailwind styling, and deploy to Vercel.
```

### Resume previous work
```
/autopilot continue
```

### Monitor production services
```
/autopilot monitor api.myapp.com db.myapp.com
```

### Refactor a codebase
```
/autopilot Refactor the auth module: extract middleware, add refresh
tokens, write integration tests, update API docs.
```

### Execute from a plan file
```
/autopilot Execute the plan in PLAN.md
```

---

## Compatibility

| Platform | Supported |
|----------|:---------:|
| Claude Code (CLI) | Yes |
| Claude Desktop | Yes |
| macOS / Linux / Windows | Yes |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| **3.0.0** | 2026-03-25 | tmux session management with unified naming. Leverages built-in `/loop` for monitoring instead of custom loops. Uses `/rename` for Claude session naming. |
| 2.0.0 | 2026-03-25 | Generalized for any project. Circuit breakers, rate limit recovery, Gemini integration. Published to GitHub. |
| 1.0 | 2026-02 | Initial skill for HPC autonomous work. |

---

## Contributing

Contributions welcome. Ideas for improvement:
- Additional monitoring check patterns
- Integration with other notification services (Slack, Discord, email)
- More external AI review integrations
- Cost tracking and token budget awareness

1. Fork this repository
2. Create a feature branch
3. Open a Pull Request

---

## License

[MIT](LICENSE)
