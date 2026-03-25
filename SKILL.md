---
name: autopilot
version: 3.0.0
description: |
  Autonomous work mode — Claude works through a plan non-stop without asking
  questions. Sets up a named tmux session with the same name used for the Claude
  session and ntfy push notifications. Uses background tasks, parallel execution,
  Codex/Gemini for code reviews, and circuit breakers. Leverages Claude Code's
  built-in /loop for monitoring and auto mode for permissions. The only native
  skill for autonomous, unattended work with real-time phone notifications.
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

# Autopilot

You are now in **autopilot mode**. You work through tasks autonomously without stopping to ask the user questions. You use background tasks, parallel execution, and push notifications to keep the user informed while you keep working.

## Modes

| Argument | Behaviour |
|---|---|
| `<plan description>` | Execute the described plan autonomously |
| `continue` | Resume from the task list where you left off |
| `monitor <services...>` | Use built-in `/loop` to monitor services periodically |

---

## Step 0: Session Setup (Unified Naming)

Create a unified session name used everywhere — tmux, Claude session, and ntfy topic.

### Generate the Session Name

```bash
PROJECT_NAME=$(basename "$(git rev-parse --show-toplevel 2>/dev/null || basename "$PWD")")
SESSION_SUFFIX=$(shuf -i 100-999 -n 1 2>/dev/null || echo $((RANDOM % 900 + 100)))
SESSION_NAME="${PROJECT_NAME}-${SESSION_SUFFIX}"
echo "Session name: $SESSION_NAME"
```

### Set Up tmux Session

If tmux is available, create a named session so all work happens in a persistent, recoverable terminal:

```bash
# Check if we're already in a tmux session
if [ -z "$TMUX" ]; then
  # Create a new named tmux session (detached)
  tmux new-session -d -s "$SESSION_NAME" 2>/dev/null || true
  echo "tmux session created: $SESSION_NAME"
  echo "Attach with: tmux attach -t $SESSION_NAME"
fi
```

If tmux is not available, skip — the skill works without it.

### Name the Claude Session

Use the built-in `/rename` command to match the tmux session:

```
/rename <SESSION_NAME>
```

This way the user can resume with `claude --resume <SESSION_NAME>`.

### Set Up ntfy Notifications

The ntfy topic uses the same session name (with the random suffix already built in for privacy):

```bash
NTFY_TOPIC="$SESSION_NAME"
echo "ntfy topic: $NTFY_TOPIC"
echo "Subscribe: https://ntfy.sh/$NTFY_TOPIC"
```

**Tell the user** the session name and ntfy topic so they can:
- Attach to tmux: `tmux attach -t <SESSION_NAME>`
- Resume Claude: `claude --resume <SESSION_NAME>`
- Subscribe to notifications: `https://ntfy.sh/<SESSION_NAME>`

If the user provides their own ntfy topic or says they don't want notifications, respect that.

### Summary of Unified Naming

```
SESSION_NAME = project-name-123

tmux session:    tmux attach -t project-name-123
Claude session:  claude --resume project-name-123
ntfy topic:      https://ntfy.sh/project-name-123
```

All three use the same name. One name to remember, one name to share.

---

## Core Rules

1. **Never stop to ask the user a question.** Make a reasonable decision, document it, and move on. If you need a code review or second opinion, use Codex or Gemini.
2. **Never wait idly.** While a background task runs, pick up the next independent task.
3. **Use background tasks for anything that takes more than 30 seconds.** Kick it off, work on something else, check back later.
4. **Track progress visibly.** Update tasks after each step.
5. **Send push notifications** for status updates (if ntfy is configured).
6. **Circuit breaker: 3 strikes.** Three consecutive failures on the same task = stop, notify, move on.

---

## Plan Execution Mode

When given a plan or `continue`:

### 1. Load the Plan
- If given a description: break it into discrete tasks
- If `continue`: check `TaskList` for pending work
- If a plan file exists (e.g., `PLAN.md`): read it

### 2. Create Tasks
Use `TaskCreate` for each step. Set dependencies with `addBlockedBy` where needed.

### 3. Execute Tasks

For each task:

```
1. Mark task as in_progress
2. Notify: "Starting task N/M: <description>"
3. Do the work
4. Run tests / validate
5. Mark task as completed
6. Notify: "Task N/M complete. Moving to N+1."
7. Immediately start the next unblocked task
```

### 4. Parallelize Where Possible

If two tasks are independent:
- Start one as a background task (`run_in_background: true`)
- Work on the other in the foreground
- Check back on the background task when done

### 5. Handle Failures (Circuit Breaker)

```
Attempt 1: Try the straightforward approach
Attempt 2: Debug the error, try a different approach
Attempt 3: Ask Codex/Gemini for help, apply their suggestion

If all 3 attempts fail:
  → Send urgent notification
  → Log the failure with full context
  → Mark task as blocked
  → Move to the next unblocked task
  → DO NOT keep retrying the same thing
```

### 6. Final Report

When all tasks complete (or all remaining are blocked):
- Run final verification (tests, lint, build)
- Send urgent ntfy with full summary
- Output summary to conversation

---

## Monitor Mode

For monitoring, use Claude Code's **built-in `/loop`** command rather than custom background loops. This is more reliable and survives context compaction.

```
/loop 15m Check if https://api.example.com/health returns 200. If not, send ntfy alert.
/loop 30m SSH to staging and check disk usage. Alert if over 85%.
/loop 1h Check if Slurm job $JOB_ID is still running. Notify when complete.
/loop 8h Compile a full status report of all services and send via ntfy.
```

The `/loop` command handles the scheduling. Your job is just to define what to check and what to do with the result.

### Quick Health Check Patterns

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
df -h / | awk 'NR==2 {print $5}' | tr -d '%'
```

**Process check:**
```bash
pgrep -f "process_name" >/dev/null && echo "running" || echo "stopped"
```

---

## Push Notifications via ntfy

Uses [ntfy.sh](https://ntfy.sh) — free, open-source, no account needed.

### Notification Helper

```bash
# Standard update
curl -s -H "Title: $SESSION_NAME" -d "Task 3/7 complete: auth module done" https://ntfy.sh/$NTFY_TOPIC

# Urgent (needs attention)
curl -s -H "Title: $SESSION_NAME - ATTENTION" -H "Priority: urgent" -H "Tags: warning" \
  -d "Stuck on task 5 after 3 attempts." https://ntfy.sh/$NTFY_TOPIC

# All done
curl -s -H "Title: $SESSION_NAME - COMPLETE" -H "Priority: urgent" -H "Tags: tada" \
  -d "All 8 tasks complete. Tests passing. Ready for review." https://ntfy.sh/$NTFY_TOPIC
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
| Stuck (circuit breaker triggered) | urgent | sos |
| All work complete | urgent | tada |
| Service alert (monitor mode) | urgent | warning |

**Always include**: task number (e.g., "3/7"), what happened, what's next.

---

## Asking Other AI Models for Help

Don't ask the user. Ask another AI model instead.

### Codex (OpenAI)

```bash
codex exec review -m gpt-5 --full-auto --skip-git-repo-check \
  "Review the recent changes for correctness, security, and edge cases."
```

### Gemini (Google)

```bash
gemini -m gemini-2.5-pro --yolo -p "Review this code for bugs and improvements."
```

### When to Use External Reviews

- **Before** deploying or running expensive operations
- **After** a complex refactor
- **When debugging** a persistent failure (fresh perspective)
- **For architecture decisions** with multiple valid approaches

---

## tmux Session Management

If tmux is available, use it for persistent terminal sessions that survive SSH disconnects.

### Creating Additional Panes

For parallel visual work, split the tmux session:

```bash
# Split horizontally — run tests in bottom pane
tmux split-window -v -t "$SESSION_NAME" "npm test --watch"

# Split vertically — run dev server in right pane
tmux split-window -h -t "$SESSION_NAME" "npm run dev"
```

### Running Long Tasks in tmux

For tasks that need their own terminal (servers, watchers, build processes):

```bash
# Create a named window for the dev server
tmux new-window -t "$SESSION_NAME" -n "server" "npm run dev"

# Create a named window for tests
tmux new-window -t "$SESSION_NAME" -n "tests" "npm test --watch"
```

### Checking tmux Status

```bash
# List all windows in the session
tmux list-windows -t "$SESSION_NAME"

# Capture output from a specific pane
tmux capture-pane -t "$SESSION_NAME" -p | tail -20
```

### Cleanup on Completion

When all work is done:

```bash
# Kill the tmux session (optional — user may want to keep it)
# tmux kill-session -t "$SESSION_NAME"
echo "Work complete. tmux session '$SESSION_NAME' is still running."
echo "Attach with: tmux attach -t $SESSION_NAME"
```

---

## Rate Limit Recovery

If you hit API rate limits:

1. Check the error for a reset time
2. Notify: "Rate limited. Resuming in X minutes."
3. **Work on non-API tasks in the meantime**
4. Resume after the wait period
5. If limits persist, reduce frequency and notify

---

## When to Contact the User

Only for:
1. **Destructive actions** — delete data, force push, deploy to production
2. **Genuinely stuck** — 3 attempts + Codex review all failed
3. **Plan change needed** — fundamental assumption was wrong
4. **All done** — final summary report
5. **Service down** — can't fix autonomously

Everything else: handle it, notify via ntfy, keep working.

---

## The Non-Stop Mindset

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  Start task → Do work → Validate                │
│       ↓                    ↓                    │
│  Background?          Tests pass?               │
│    ↓       ↓           ↓        ↓               │
│   Yes      No         Yes       No              │
│    ↓       ↓           ↓        ↓               │
│  Work on   Continue   Complete  Retry (x3)      │
│  next task            → Next    → Ask Codex     │
│    ↓                   task     → Move on       │
│  Check back                                     │
│    ↓                                            │
│  Notify result                                  │
│                                                 │
│  NEVER: Stop to ask │ Wait idle │ Loop forever  │
│  ALWAYS: Notify │ Track progress │ Move on      │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## Leveraging Built-in Claude Code Features

This skill works best with these Claude Code features:

| Feature | How to Use |
|---------|-----------|
| **Auto mode** | Enable in settings for zero permission prompts |
| **`/loop`** | Use for monitoring instead of custom sleep loops |
| **`/rename`** | Align Claude session name with tmux + ntfy |
| **`claude --resume`** | Resume a named session after disconnection |
| **Background subagents** | Delegate independent research tasks |
| **`-p` headless mode** | Chain multiple Claude invocations in scripts |

---

## Version History
- v1.0 (2026-02): Initial skill for HPC autonomous work with ntfy, monitoring, and Codex integration
- v2.0 (2026-03): Generalized for any project. Added circuit breakers, rate limit recovery, Gemini integration.
- v3.0 (2026-03): Added tmux session management with unified naming (tmux + Claude session + ntfy topic all share the same name). Replaced custom monitoring loops with Claude Code's built-in `/loop`. Added section on leveraging built-in features. Published to GitHub.
