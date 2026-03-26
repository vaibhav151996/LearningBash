# Chapter 29: Traps and Signals

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand Unix signals and how they work
- Use `trap` to catch and handle signals
- Implement cleanup routines that always run
- Build signal-safe scripts
- Handle common signals like INT, TERM, HUP, and EXIT

---

## 29.1 What Are Signals?

Signals are software interrupts sent to a process. They tell a process that something happened:

```
┌──────────────┬────────┬─────────────────────────────────┐
│ Signal       │ Number │ When Sent                       │
├──────────────┼────────┼─────────────────────────────────┤
│ SIGHUP       │ 1      │ Terminal closed / hangup        │
│ SIGINT       │ 2      │ Ctrl+C pressed                  │
│ SIGQUIT      │ 3      │ Ctrl+\ pressed                  │
│ SIGKILL      │ 9      │ kill -9 (cannot be caught!)     │
│ SIGTERM      │ 15     │ kill command (default signal)   │
│ SIGCHLD      │ 17     │ Child process stopped/exited    │
│ SIGCONT      │ 18     │ Resume after stop               │
│ SIGSTOP      │ 19     │ Stop process (cannot be caught!)│
│ SIGTSTP      │ 20     │ Ctrl+Z pressed                  │
│ SIGUSR1      │ 10     │ User-defined signal 1           │
│ SIGUSR2      │ 12     │ User-defined signal 2           │
└──────────────┴────────┴─────────────────────────────────┘
```

```bash
# List all signals
kill -l

# Send signals
kill -TERM $pid      # Send SIGTERM (polite termination)
kill -INT $pid       # Send SIGINT (like Ctrl+C)
kill -9 $pid         # Send SIGKILL (forced — cannot be caught)
kill -HUP $pid       # Send SIGHUP (reload configuration)
```

---

## 29.2 The trap Command

`trap` registers a handler to run when a signal is received:

```bash
trap 'commands' SIGNAL [SIGNAL...]
```

```bash
#!/bin/bash

# Catch Ctrl+C
trap 'echo "Ctrl+C pressed! Exiting..."; exit 1' INT

echo "Running... Press Ctrl+C to stop"
while true; do
    sleep 1
    echo "Working..."
done
```

### Trap Syntax Variations

```bash
# String command
trap 'echo "caught signal"' INT TERM

# Function reference
cleanup() { echo "Cleaning up..."; }
trap cleanup EXIT

# Reset to default behavior
trap - INT              # Remove INT handler

# Ignore a signal
trap '' INT             # INT signal is now ignored (even Ctrl+C)

# Show current traps
trap -p                 # Print all active traps
```

---

## 29.3 The EXIT Pseudo-Signal

`EXIT` is not a real signal — it's a Bash pseudo-signal that fires when the shell exits **for any reason**:

```bash
#!/bin/bash

tmpfile=$(mktemp)
trap 'rm -f "$tmpfile"; echo "Cleaned up"' EXIT

echo "Working with $tmpfile"
echo "data" > "$tmpfile"

# No matter how the script exits (success, error, signal),
# the trap runs and cleans up the temp file
```

```
                     Trigger EXIT trap?
Normal exit ──────────── YES
exit command ─────────── YES  
set -e failure ──────── YES
Ctrl+C (SIGINT) ─────── YES (if INT trap also exits)
SIGTERM ──────────────── YES (if TERM trap also exits)
SIGKILL ──────────────── NO! (cannot be caught)
```

---

## 29.4 Cleanup Pattern (Production Pattern)

```bash
#!/usr/bin/env bash
set -euo pipefail

TMPDIR=""
LOCKFILE=""

cleanup() {
    local exit_code=$?
    
    # Remove temp directory
    if [[ -n "$TMPDIR" && -d "$TMPDIR" ]]; then
        rm -rf "$TMPDIR"
    fi
    
    # Release lock
    if [[ -n "$LOCKFILE" && -d "$LOCKFILE" ]]; then
        rmdir "$LOCKFILE"
    fi
    
    # Restore terminal if needed
    # stty sane 2>/dev/null
    
    exit "$exit_code"    # Preserve original exit code
}

trap cleanup EXIT
trap 'echo "Interrupted" >&2; exit 130' INT
trap 'echo "Terminated" >&2; exit 143' TERM

TMPDIR=$(mktemp -d)
LOCKFILE="/var/lock/myapp.lock"
mkdir "$LOCKFILE" || die "Another instance is running"

# ... main logic ...
```

---

## 29.5 Signal Handling Patterns

### Graceful Shutdown

```bash
#!/bin/bash

RUNNING=true

shutdown() {
    echo "Shutting down gracefully..."
    RUNNING=false
}

trap shutdown INT TERM

while $RUNNING; do
    echo "Processing... (PID $$)"
    sleep 2
done

echo "Cleanup complete. Exiting."
```

### Config Reload on SIGHUP

```bash
#!/bin/bash

load_config() {
    echo "Loading configuration..."
    source /etc/myapp/config.sh
}

trap load_config HUP

load_config    # Initial load

while true; do
    # Main processing loop
    process_work
    sleep 5
done

# Reload config: kill -HUP $pid
```

### Progress Save on SIGUSR1

```bash
#!/bin/bash

PROGRESS=0

save_progress() {
    echo "$PROGRESS" > /tmp/progress.txt
    echo "Progress saved: $PROGRESS%"
}

trap save_progress USR1

for (( PROGRESS=0; PROGRESS<=100; PROGRESS++ )); do
    sleep 1
    # ... do work ...
done

# Check progress: kill -USR1 $pid
```

---

## 29.6 Trap in Functions

```bash
# Traps are global — setting a trap in a function replaces any previous one

setup_trap() {
    trap 'echo "Handler A"' EXIT
}

main() {
    trap 'echo "Handler B"' EXIT     # This REPLACES Handler A
}

setup_trap
main
# Only "Handler B" runs on exit

# To append, save and restore:
append_trap() {
    local existing
    existing=$(trap -p EXIT | sed "s/trap -- '//;s/' EXIT//")
    trap "${existing:+$existing; }$1" EXIT
}
```

---

## 29.7 Common Mistakes

### Mistake 1: Using Single Quotes When You Need Variables Expanded NOW

```bash
file="/tmp/myfile.txt"

# Variables expanded at TRAP TIME (when signal fires):
trap 'rm -f "$file"' EXIT    # If $file changes later, new value used

# Variables expanded at DEFINITION TIME:
trap "rm -f $file" EXIT       # Captures current value of $file
```

### Mistake 2: Forgetting That SIGKILL Can't Be Caught

```bash
trap 'cleanup' KILL    # This does NOTHING — SIGKILL cannot be trapped
```

### Mistake 3: Not Exiting After Signal Handler

```bash
# BAD — script continues running after Ctrl+C
trap 'echo "Caught!"' INT

# GOOD — clean up and exit
trap 'echo "Caught!"; exit 130' INT
```

---

## 29.8 Exit Codes for Signals

Convention: exit code = 128 + signal number:

```
Signal    Number    Exit Code
SIGHUP    1         129
SIGINT    2         130
SIGQUIT   3         131
SIGTERM   15        143
```

```bash
trap 'exit 130' INT      # Correct exit code for Ctrl+C
trap 'exit 143' TERM     # Correct exit code for kill
```

---

## Exercises

### Exercise 29.1: Robust Cleanup
Write a script that creates 3 temp files-and a temp directory, starts a background process, and ensures all are cleaned up on any exit (normal, error, Ctrl+C).

### Exercise 29.2: Graceful Server
Write a simulated server that counts requests in a loop, saves the count to a file on SIGUSR1, reloads config on SIGHUP, and shuts down cleanly on SIGTERM.

---

## Summary

- Signals are notifications sent to processes (SIGINT, SIGTERM, SIGHUP, etc.)
- `trap 'commands' SIGNAL` registers a signal handler
- `EXIT` pseudo-signal fires on any script exit — use it for cleanup
- `SIGKILL` (9) and `SIGSTOP` (19) cannot be caught or ignored
- Always `exit` from signal handlers to prevent scripts from continuing unexpectedly
- Convention: exit code = 128 + signal number
- Use traps for: temp file cleanup, lock release, graceful shutdown, config reload

---

**Next Chapter:** [Chapter 30: Script Organization →](Chapter30-Script-Organization.md)
