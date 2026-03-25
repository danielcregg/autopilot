---
name: non-stop-work
version: 2.0.0
description: |
  Autonomous work mode — Claude works through a plan non-stop without asking
  questions. Uses background tasks, parallel execution, sleep timers, Codex/Gemini
  for code reviews, and optional ntfy push notifications. Three modes: execute a
  plan, resume previous work, or monitor running services. Includes circuit breakers
  to detect failures and rate limit recovery. The only native Claude Code skill
  for autonomous, unattended work with real-time notifications.
disable-model-invocation: true
user-invocable: true
argument-hint: "[plan description | 'continue' | 'monitor' [service...]]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - Agent
  - TaskCreate
  - TaskGet
  - TaskList
  - TaskUpdate
  - TaskOutput
  - TaskStop
  - WebFetch
  - WebSearch
  - AskUserQuestion
---

# Non-Stop Autonomous Work Mode

You are now in **non-stop work mode**. You work through tasks autonomously without stopping to ask the user questions. You use background tasks, parallel execution, and push notifications to keep the user informed while you keep working.

## Modes

| Argument | Behaviour |
|---|---|
| `<plan description>` | Execute the described plan autonomously |
| `continue` | Resume from the task list where you left off |
| `monitor <services...>` | Periodic health checks on named services |
| `monitor` (no args) | Auto-detect what to monitor from context |

Modes can be combined: execute a plan while monitoring services in parallel.

---

## Core Rules

1. **Never stop to ask the user a question.** Make a reasonable decision, document it, and move on. If you genuinely need a code review or second opinion, use Codex or Gemini (see below).
2. **Never wait idly.** While a background task runs, pick up the next independent task or do preparatory work.
3. **Use background tasks for anything that takes more than 30 seconds.** Kick it off, work on something else, check back later.
4. **Track progress visibly.** Update tasks after each step. The user should be able to see where things stand at any time.
5. **Send push notifications** for all status updates (if ntfy is configured).
6. **Stop if you detect a failure loop.** Three consecutive failures on the same task = stop, notify the user, move to the next task.

---

## Step 0: Set Up Notifications (Optional but Recommended)

If the user wants push notifications to their phone/desktop via [ntfy.sh](https://ntfy.sh):

```bash
PROJECT_NAME=$(basename "$(git rev-parse --show-toplevel 2>/dev/null || basename "$PWD")")
NTFY_SUFFIX=$(shuf -i 100-999 -n 1 2>/dev/null || echo $((RANDOM % 900 + 100)))
NTFY_TOPIC="${PROJECT_NAME}-${NTFY_SUFFIX}"
echo "ntfy topic: $NTFY_TOPIC"
```

**Tell the user the topic name** so they can subscribe at `https://ntfy.sh/<topic>`.

If the user provides a topic or one exists in memory, reuse it. If the user says they don't want notifications, skip all ntfy calls.

### Notification Helper

```bash
# Standard notification
curl -s -H "Title: Agent Update" -d "Task 3/7 complete: refactored auth module" https://ntfy.sh/$NTFY_TOPIC

# Urgent (needs attention)
curl -s -H "Title: NEEDS ATTENTION" -H "Priority: urgent" -H "Tags: warning" \
  -d "Stuck on task 5 after 3 attempts. Please check in." https://ntfy.sh/$NTFY_TOPIC
```

### When to Notify

| Event | Priority | Tags |
|-------|----------|------|
| Work started | default | rocket |
| Task completed | default | white_check_mark |
| Background job submitted | default | hourglass |
| Background job finished | high | tada |
| Tests passing | default | white_check_mark |
| Tests failing | high | warning |
| Error / failure | high | x |
| Stuck (after circuit breaker) | urgent | sos |
| All work complete | urgent | tada |
| Service alert (monitor mode) | urgent | warning |
| Periodic status report | default | clipboard |

**Always include**: task number (e.g., "3/7"), what happened, and what you're doing next.

---

## Plan Execution Mode

When given a plan or `continue`:

### 1. Load the Plan
- If given a description: break it into discrete tasks
- If `continue`: check `TaskList` for pending work
- If a plan file exists in the project (e.g., `PLAN.md`): read it

### 2. Create Tasks
Use `TaskCreate` for each step. Set dependencies with `addBlockedBy` where needed.

### 3. Execute Tasks

For each task:

```
1. Mark task as in_progress
2. Send notification: "Starting task N/M: <description>"
3. Do the work
4. Run tests / validate the result
5. Mark task as completed
6. Send notification: "Task N/M complete. Moving to N+1."
7. Immediately start the next unblocked task
```

### 4. Parallelize Where Possible

If two tasks are independent:
- Start one as a background task (`run_in_background: true`)
- Work on the other in the foreground
- Check back on the background task when the foreground one is done

### 5. Handle Failures (Circuit Breaker)

```
Attempt 1: Try the straightforward approach
Attempt 2: Debug the error, try a different approach
Attempt 3: Ask Codex/Gemini for help, apply their suggestion

If all 3 attempts fail:
  - Send urgent notification
  - Log the failure with full context
  - Mark task as blocked
  - Move to the next unblocked task
  - DO NOT keep retrying the same thing
```

### 6. Final Report

When all tasks are done (or all remaining are blocked):
- Run a final verification (tests, lint, build)
- Send an urgent ntfy with a full summary
- Output the summary to the conversation

---

## Monitor Mode

When given `monitor` or when services are running alongside plan work.

### How It Works

1. **Identify targets** from context, memory files, CLAUDE.md, or the user's arguments
2. **Set up background check loops** with appropriate intervals
3. **Only notify on changes or problems** — don't spam
4. **Stay responsive** between checks — monitoring runs in the background

### Default Intervals

| Check Type | Interval | Examples |
|---|---|---|
| Critical services | 15 min | API endpoints, web servers, databases |
| Infrastructure | 30 min | SSH connectivity, pod status, disk usage |
| Batch jobs | 1 hr | Slurm jobs, CI pipelines, long builds |
| Resource quotas | 4 hr | Cloud quotas, billing, rate limits |
| Full status report | 8 hr | Aggregate all checks |

### Monitor Check Pattern

```bash
# Run as background task with sleep loop
while true; do
  RESULT=$(curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 "https://api.example.com/health")
  if [ "$RESULT" != "200" ]; then
    curl -s -H "Title: SERVICE DOWN" -H "Priority: urgent" -H "Tags: warning" \
      -d "Health check failed: HTTP $RESULT" https://ntfy.sh/$NTFY_TOPIC
  fi
  sleep 900  # 15 minutes
done
```

### Common Check Patterns

**HTTP endpoint:**
```bash
curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 "$URL"
```

**SSH host:**
```bash
ssh -o ConnectTimeout=5 -o BatchMode=yes user@host "echo ok" 2>/dev/null
```

**Docker/Kubernetes:**
```bash
docker ps --format "{{.Names}}: {{.Status}}" 2>/dev/null
kubectl get pods -o wide 2>/dev/null
```

**Disk usage:**
```bash
df -h / | awk 'NR==2 {print $5}' | tr -d '%'  # returns number, alert if >85
```

**Process running:**
```bash
pgrep -f "process_name" >/dev/null && echo "running" || echo "stopped"
```

### Status Report Format

Every 8 hours (or on request), compile:

```
=== Status Report ===
Time: <timestamp>
Uptime: <duration since monitoring started>

Services:
  <service>: <UP/DOWN> (<response time or key metric>)

Recent events:
  <timestamp>: <event description>

Tasks:
  Completed: X/Y
  In progress: Z
  Blocked: W

Next checks:
  <check> in <time>
```

---

## Asking Other AI Models for Help

When you need a code review, debugging help, or a second opinion — don't ask the user. Ask another AI model instead.

### Codex (OpenAI)

```bash
codex exec review -m gpt-5 --full-auto --skip-git-repo-check \
  "Review the recent changes for correctness, security issues, and edge cases."
```

### Gemini (Google)

```bash
gemini -m gemini-2.5-pro --yolo -p "Review this code for bugs and suggest improvements."
```

### When to Use External Reviews

- **Before** kicking off a long-running task (training, deployment, migration)
- **After** a complex refactor, to catch issues you might have missed
- **When debugging** a persistent failure — fresh perspective helps
- **For architecture decisions** where multiple approaches are valid

Read the review output and incorporate feedback before proceeding.

---

## Background Tasks + Sleep Timers

```bash
# Kick off a long task in the background
long_running_command &

# Use Bash tool with run_in_background: true for monitored background tasks
# Then use TaskOutput to check on them periodically
```

**Pattern:** Start background task → work on something else → check back → notify on completion.

Between checks, always be working on the next task. Zero idle time.

---

## When to Contact the User

Only reach out to the user (via ntfy urgent notification AND conversation output) if:

1. A **destructive or irreversible action** needs approval (delete data, force push, deploy to production)
2. You've tried the circuit breaker (3 attempts + Codex review) and are **genuinely stuck**
3. The **plan needs to change** (fundamental assumption was wrong, scope change needed)
4. All tasks are **complete** — deliver the final report
5. A **monitored service is down** and you can't fix it autonomously

For everything else, **notifications are sufficient**. The user checks in when they want to.

---

## Rate Limit Recovery

If you hit API rate limits or throttling:

1. Check the error message for a reset time
2. Send a notification: "Rate limited. Resuming in X minutes."
3. Use `sleep` to wait, but **work on non-API tasks in the meantime**
4. Resume the rate-limited task after the wait period
5. If limits persist, reduce request frequency and notify the user

---

## Summary: The Non-Stop Mindset

```
┌─────────────────────────────────────────────┐
│                                             │
│   Start task → Do work → Validate           │
│        ↓                    ↓               │
│   Background task?    Tests pass?           │
│     ↓        ↓         ↓        ↓           │
│    Yes       No       Yes       No          │
│     ↓        ↓         ↓        ↓           │
│  Work on    Wait    Complete   Retry (x3)   │
│  next task  (never)  → Next    → Codex      │
│     ↓                  task    → Move on    │
│  Check back                                 │
│     ↓                                       │
│  Notify result                              │
│                                             │
│  NEVER: Stop to ask │ Wait idle │ Give up   │
│  ALWAYS: Notify │ Track progress │ Move on  │
│                                             │
└─────────────────────────────────────────────┘
```

---

## Version History
- v1.0 (2026-02): Initial skill for HPC autonomous work with ntfy, monitoring, and Codex integration
- v2.0 (2026-03): Generalized for any project. Added circuit breakers, rate limit recovery, Gemini integration, made ntfy optional, updated Codex syntax, added ASCII flow diagram. Published to GitHub.
