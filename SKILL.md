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

Create a unified session name used everywhere — tmux, Claude session, and ntfy topic. Run all setup in a **single bash block** so variables persist:

```bash
# Generate session name (6-digit suffix for ntfy privacy)
PROJECT_NAME=$(basename "$(git rev-parse --show-toplevel 2>/dev/null || basename "$PWD")")
SESSION_SUFFIX=$(command -v shuf &>/dev/null && shuf -i 100000-999999 -n 1 || \
  command -v gshuf &>/dev/null && gshuf -i 100000-999999 -n 1 || \
  echo $((RANDOM * RANDOM % 900000 + 100000)))
SESSION_NAME="${PROJECT_NAME}-${SESSION_SUFFIX}"

# Save session name to file (persists across bash tool calls)
echo "$SESSION_NAME" > /tmp/autopilot-session-name

# Set up tmux if available (optional — skip if not installed)
if command -v tmux &>/dev/null && [ -z "$TMUX" ]; then
  # Handle collision: regenerate if session exists
  while tmux has-session -t "$SESSION_NAME" 2>/dev/null; do
    SESSION_SUFFIX=$((RANDOM * RANDOM % 900000 + 100000))
    SESSION_NAME="${PROJECT_NAME}-${SESSION_SUFFIX}"
    echo "$SESSION_NAME" > /tmp/autopilot-session-name
  done
  tmux new-session -d -s "$SESSION_NAME"
  echo "tmux session: $SESSION_NAME"
  echo "Attach with: tmux attach -t $SESSION_NAME"
else
  echo "tmux not available — skipping (skill works without it)"
fi

# Check ntfy.sh reachability (optional — skip if offline)
if curl -s -o /dev/null -w "%{http_code}" --connect-timeout 3 https://ntfy.sh 2>/dev/null | grep -q "200"; then
  echo "ntfy topic: $SESSION_NAME"
  echo "Subscribe: https://ntfy.sh/$SESSION_NAME"
  echo "true" > /tmp/autopilot-ntfy-enabled
else
  echo "ntfy.sh not reachable — notifications disabled"
  echo "false" > /tmp/autopilot-ntfy-enabled
fi

echo "Session name: $SESSION_NAME"
```

After running the setup, **rename the Claude session** to match. This is a Claude Code meta-command (not bash) — type it directly:

```
/rename <SESSION_NAME>
```

### Reading the Session Name in Later Bash Calls

Since bash variables don't persist between tool calls, read from the file:

```bash
SESSION_NAME=$(cat /tmp/autopilot-session-name)
NTFY_ENABLED=$(cat /tmp/autopilot-ntfy-enabled 2>/dev/null || echo "false")
```

### Sending Notifications

Only send if ntfy is enabled:

```bash
SESSION_NAME=$(cat /tmp/autopilot-session-name)
NTFY_ENABLED=$(cat /tmp/autopilot-ntfy-enabled 2>/dev/null || echo "false")
if [ "$NTFY_ENABLED" = "true" ]; then
  curl -s -H "Title: $SESSION_NAME" -d "Your message here" https://ntfy.sh/$SESSION_NAME
fi
```

### Summary of Unified Naming

```
SESSION_NAME = myproject-847291

tmux session:    tmux attach -t myproject-847291
Claude session:  claude --resume myproject-847291
ntfy topic:      https://ntfy.sh/myproject-847291
State file:      /tmp/autopilot-session-name
```

All three use the same name. One name to remember, one name to share.
The 6-digit suffix (900,000 possibilities) makes the ntfy topic hard to guess.

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

Always read session name from file (variables don't persist between bash calls):

```bash
SESSION_NAME=$(cat /tmp/autopilot-session-name 2>/dev/null || echo "autopilot")
NTFY_ENABLED=$(cat /tmp/autopilot-ntfy-enabled 2>/dev/null || echo "false")

# Only send if ntfy is enabled
if [ "$NTFY_ENABLED" = "true" ]; then
  # Standard update
  curl -s -H "Title: $SESSION_NAME" -d "Task 3/7 complete: auth module done" https://ntfy.sh/$SESSION_NAME

  # Urgent (needs attention)
  curl -s -H "Title: $SESSION_NAME - ATTENTION" -H "Priority: urgent" -H "Tags: warning" \
    -d "Stuck on task 5 after 3 attempts." https://ntfy.sh/$SESSION_NAME

  # All done
  curl -s -H "Title: $SESSION_NAME - COMPLETE" -H "Priority: urgent" -H "Tags: tada" \
    -d "All 8 tasks complete. Tests passing. Ready for review." https://ntfy.sh/$SESSION_NAME
fi
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

Don't ask the user. Ask another AI model instead. Always check availability first.

### Codex (OpenAI) — if installed

```bash
if command -v codex &>/dev/null; then
  codex exec review -m gpt-5 --full-auto --skip-git-repo-check \
    "Review the recent changes for correctness, security, and edge cases."
else
  echo "Codex not available — skipping external review"
fi
```

### Gemini (Google) — if installed

```bash
if command -v gemini &>/dev/null; then
  gemini -m gemini-2.5-pro --yolo -p "Review this code for bugs and improvements."
else
  echo "Gemini not available — skipping external review"
fi
```

### When External AI Isn't Available

If neither `codex` nor `gemini` is installed:
1. Use a Claude Code `Explore` subagent for a fresh-perspective review
2. Document the issue with full context
3. Mark task as needing review in the final summary
4. Continue with the next task

### When to Use External Reviews

- **Before** deploying or running expensive operations
- **After** a complex refactor
- **When debugging** a persistent failure (attempt 3 in circuit breaker)
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
1. **Destructive actions** — delete data, force push, deploy to **production** (staging/dev deploys are OK autonomously)
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
