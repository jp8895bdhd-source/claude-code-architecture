# Hook Pattern: Event-Driven Automation

## Overview

Hooks are shell scripts triggered at specific lifecycle events in Claude Code. They enable passive automation — things that happen without explicit user action at key moments.

Hooks are configured in `~/.claude/settings.json` under the "hooks" key and live in `~/.claude/hooks/`.

```json
{
  "hooks": {
    "SessionStart": "~/.claude/hooks/session-start.sh",
    "PreToolUse": "~/.claude/hooks/pre-tool-use.sh",
    "PostToolUse": "~/.claude/hooks/post-tool-use.sh",
    "Stop": "~/.claude/hooks/stop.sh",
    "TaskCompleted": "~/.claude/hooks/task-completed.sh"
  }
}
```

Each hook receives context (tool name, session ID, etc.) via environment variables and can control tool execution via exit codes:
- **Exit 0** — Allow/proceed
- **Exit 1** — Deny/skip  
- **Exit 2** — Prompt user

---

## Example 1: Token Optimization Hook (PreToolUse)

**File:** `~/.claude/hooks/pre-tool-use.sh`

This hook intercepts every Bash tool call and rewrites commands to save tokens. It's the hot path — runs hundreds of times per session.

```bash
#!/bin/bash
# Token optimization hook: rewrite commands for token efficiency
# Triggered: Before every bash tool call
# Exit code: 0 = proceed with potentially rewritten command
#            1 = deny tool use
#            2 = prompt user (usually for send operations)

set -e

# Context passed by Claude Code
TOOL_NAME="$1"        # e.g., "bash"
COMMAND="$2"          # raw command string
SESSION_ID="$3"       # unique session identifier

RTK_VERSION="1.2.5"
RTK_BIN="$HOME/.cargo/bin/rtk"
EXPECTED_VERSION="1.2.5"

# Version guard: ensure rtk is installed and correct version
if [[ ! -f "$RTK_BIN" ]]; then
    echo "[WARN] rtk not found at $RTK_BIN - skipping optimization" >&2
    echo "$COMMAND"  # Return unmodified command
    exit 0
fi

INSTALLED_VERSION=$("$RTK_BIN" --version 2>/dev/null | awk '{print $2}' || echo "unknown")
if [[ "$INSTALLED_VERSION" != "$EXPECTED_VERSION" ]]; then
    echo "[WARN] rtk version mismatch (expected $EXPECTED_VERSION, got $INSTALLED_VERSION)" >&2
    echo "$COMMAND"  # Return unmodified command
    exit 0
fi

# Check if this is a command that benefits from optimization
# rtk handles: git, grep, find, jq, curl, docker, kubernetes, etc.
FIRST_WORD=$(echo "$COMMAND" | awk '{print $1}')

case "$FIRST_WORD" in
    git|grep|find|jq|curl|docker|kubectl|aws|gh)
        # Run through token optimizer
        OPTIMIZED=$("$RTK_BIN" proxy "$COMMAND" 2>/dev/null || echo "$COMMAND")
        echo "$OPTIMIZED"
        exit 0
        ;;
    *)
        # Not a candidate for optimization
        echo "$COMMAND"
        exit 0
        ;;
esac
```

### How It Works

1. **User runs a bash command** (e.g., `git log --all --oneline --graph --decorate`)
2. **PreToolUse hook fires** with the command as input
3. **Hook passes to rtk binary:** `rtk proxy "git log --all --oneline --graph --decorate"`
4. **rtk rewrites it:** `git log --oneline` (removes verbose flags, reduces output)
5. **Hook returns rewritten command** to Claude Code
6. **Bash runs the optimized version**, which returns less output
7. **Claude processes fewer tokens** — 60-90% savings on average

### Token Savings Example

```
Original:     git log --all --graph --decorate --pretty=format:"%h %d %s"
Output:       ~2,000 tokens (full commit graph, decorations, all branches)

Rewritten:    git log --oneline -20
Output:       ~400 tokens (last 20 commits, minimal info)

Savings:      80% reduction
```

### Exit Codes in PreToolUse

- **0** — Use the (potentially rewritten) command
- **1** — Deny the tool (user doesn't need it)
- **2** — Prompt user ("This will send an email, ok?")

Example: Deny if running as a test:

```bash
if [[ "$SESSION_ID" == *"test"* ]]; then
    exit 1  # Deny bash in test sessions
fi
```

---

## Example 2: Session Lifecycle Hooks

**File:** `~/.claude/hooks/session-start.sh`

Runs at the beginning of every Claude Code session. Initialize state, check prerequisites, configure the environment.

```bash
#!/bin/bash
# Initialize session: set up browser mode, check dependencies, etc.

set -e

SESSION_ID="$1"
HEADLESS_BROWSER="${HEADLESS_BROWSER:-false}"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

# Create session directory for temporary files
SESSION_DIR="/tmp/claude-session-$SESSION_ID"
mkdir -p "$SESSION_DIR"
export SESSION_DIR

# Log session start
echo "[$TIMESTAMP] Session started: $SESSION_ID" >> ~/.claude/session.log

# Check for required tools
for cmd in python3 jq sqlite3; do
    if ! command -v "$cmd" &> /dev/null; then
        echo "[WARN] $cmd not found" >&2
    fi
done

# If user prefers headless browser for this session, configure it
if [[ "$HEADLESS_BROWSER" == "true" ]]; then
    export PLAYWRIGHT_HEADLESS=true
    echo "[INFO] Browser automation in headless mode"
fi

# Load user environment (git config, API keys in credential manager, etc.)
if [[ -f ~/.claude/session-env.sh ]]; then
    source ~/.claude/session-env.sh
fi

exit 0
```

---

## Example 3: PostToolUse Hook

**File:** `~/.claude/hooks/post-tool-use.sh`

Runs after a tool completes. Log the tool call, track execution time, update state.

```bash
#!/bin/bash
# Log tool execution for auditing and optimization

set -e

TOOL_NAME="$1"
COMMAND="$2"
EXIT_CODE="$3"
DURATION_MS="$4"

TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
LOG_FILE="$HOME/.claude/tools.log"

# Append execution record
echo "[$TIMESTAMP] $TOOL_NAME | exit:$EXIT_CODE | ${DURATION_MS}ms | $COMMAND" >> "$LOG_FILE"

# Rotation: keep last 10,000 entries
LINES=$(wc -l < "$LOG_FILE")
if [[ $LINES -gt 10000 ]]; then
    tail -10000 "$LOG_FILE" > "$LOG_FILE.tmp"
    mv "$LOG_FILE.tmp" "$LOG_FILE"
fi

exit 0
```

---

## Example 4: Stop Hook (Async Backup + Notifications)

**File:** `~/.claude/hooks/stop.sh`

Runs when the session ends. Async operations don't block the user from exiting.

```bash
#!/bin/bash
# Run async tasks when session ends: backup, summary email, notifications

set -e

SESSION_ID="$1"
SESSION_DURATION_MINS="$2"

TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

# Function to run in background (doesn't block session end)
run_async() {
    (
        # Backup Claude Code to GitHub
        cd ~/.claude
        git add -A
        git commit -m "Auto-backup: $TIMESTAMP" 2>/dev/null || true
        git push origin main 2>/dev/null || true
    ) &
    
    # Async task tracking
    echo "[$TIMESTAMP] Backup started (async, PID: $!)" >> ~/.claude/session.log
}

# Only run async if session was non-trivial
if [[ $SESSION_DURATION_MINS -gt 2 ]]; then
    run_async
fi

exit 0
```

The ampersand at the end (`&`) makes the operation background. The script exits immediately; the backup continues.

---

## Example 5: TaskCompleted Hook

**File:** `~/.claude/hooks/task-completed.sh`

Runs when a complex task finishes. Send native macOS notifications, post summaries, etc.

```bash
#!/bin/bash
# Notify user when a task completes (non-blocking)

set -e

TASK_NAME="$1"
TASK_DURATION_MINS="$2"
STATUS="$3"  # success or failed

# Only notify for long-running tasks
if [[ $TASK_DURATION_MINS -lt 5 ]]; then
    exit 0
fi

# Send native macOS notification (doesn't block)
osascript -e "
    display notification \"$TASK_NAME completed ($STATUS)\" \
        with title \"Claude Code\" \
        subtitle \"Duration: ${TASK_DURATION_MINS}m\"
" &

exit 0
```

This shows a Notification Center popup on macOS without blocking the session.

---

## Example 6: Custom Permission Hook

**File:** `~/.claude/hooks/pre-tool-use-permissions.sh`

Some commands need explicit user approval. Use exit code 2 to prompt.

```bash
#!/bin/bash
# Permission gates: require confirmation for sensitive operations

TOOL_NAME="$1"
COMMAND="$2"

# Deny git push by default (require explicit mention)
if [[ "$TOOL_NAME" == "bash" && "$COMMAND" == *"git push"* ]]; then
    exit 2  # Prompt user: "This will push to remote. Continue?"
fi

# Deny financial transactions in batch operations
if [[ "$COMMAND" == *"transfer"* || "$COMMAND" == *"payment"* ]]; then
    exit 2  # Require confirmation
fi

# Allow everything else
exit 0
```

When hook returns 2, Claude Code shows a confirmation dialog before proceeding.

---

## Full Hook Chain Example

Here's what happens during a typical session:

```
1. SESSION START (SessionStart hook)
   ✓ Create temp directory
   ✓ Check tool availability
   ✓ Load environment

2. USER TYPES: git status
   
3. PRE-TOOL-USE (PreToolUse hook)
   • Command: "git status"
   • rtk passes through (no optimization)
   • Hook returns: "git status" (unchanged)
   
4. BASH EXECUTES: git status
   
5. POST-TOOL-USE (PostToolUse hook)
   • Log: "2026-04-08 14:22:15 | bash | exit:0 | 142ms | git status"
   
6. [User types 5 more commands...]

7. USER TYPES: git push origin main
   
8. PRE-TOOL-USE (PreToolUse hook)
   • Command: "git push origin main"
   • Hook detects git push
   • Returns exit code 2 (prompt user)
   
9. CLAUDE CODE SHOWS: "This will push to remote. Continue?"
   
10. USER CONFIRMS: "Yes"
    
11. BASH EXECUTES: git push origin main
    
12. SESSION ENDS
    
13. STOP HOOK
    • Start async backup to GitHub (doesn't block)
    • Writes: "2026-04-08 14:23:50 | Session ended | Duration: 8m | Backup started (async)"

14. SESSION EXITS immediately (backup continues in background)
```

---

## Key Design Decisions

### PreToolUse is Synchronous

The token optimization hook runs on every command and must be instant (<10ms). It's written in Bash and calls a compiled Rust binary (rtk) for speed.

### Stop/Backup is Async

No user wants to wait for GitHub backup when exiting a session. The Stop hook spawns background processes and returns immediately.

### Exit Codes Control Flow

Instead of returning JSON or complex data, hooks use Unix exit codes (0 = allow, 1 = deny, 2 = prompt). This is:
- Fast to parse
- Impossible to get wrong
- Follows POSIX conventions

### Session Logging Enables Auditing

PostToolUse logs every tool call with timestamp, duration, exit code. This enables:
- Finding which tools took longest (optimization targets)
- Debugging failures (what command failed and why?)
- Billing/cost tracking (how much did this session cost?)

### Hooks Are Optional

You don't need hooks for Claude Code to work. But they enable advanced automation without modifying core behavior.

---

## Common Hook Patterns

### Rate Limiting
```bash
# Deny web requests if more than 10 in the last minute
RECENT=$(grep "$(date '+%Y-%m-%d %H:%M')" ~/.claude/tools.log | grep curl | wc -l)
if [[ $RECENT -gt 10 ]]; then
    exit 1  # Deny
fi
```

### Cost Tracking
```bash
# PostToolUse: track tokens per tool
TOOL_NAME="$1"
TOKENS_USED="$4"
echo "$TOOL_NAME:$TOKENS_USED" >> ~/.claude/cost.log
```

### Dry-Run Injection
```bash
# PreToolUse: if session is tagged "dry-run", inject --dry-run flags
if [[ "$SESSION_ID" == *"dry-run"* ]]; then
    COMMAND="${COMMAND/docker run/docker run --dry-run}"
fi
echo "$COMMAND"
```
