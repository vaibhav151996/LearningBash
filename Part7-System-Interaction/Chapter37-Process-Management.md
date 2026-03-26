# Chapter 37: Process Management

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand how Linux processes work
- View and manage running processes
- Use ps, top, htop, and /proc for process inspection
- Send signals to control processes
- Manage process priorities

---

## 37.1 What Is a Process?

Every running program is a **process** — an instance of a program in memory with its own PID (Process ID).

```
┌─────────────────────────────────────────────┐
│ Process 1 (init/systemd) — PID 1            │
│   ├── Process 200 (sshd)                    │
│   │   └── Process 5001 (bash) ← your shell  │
│   │       ├── Process 5050 (vim)             │
│   │       └── Process 5051 (grep)            │
│   ├── Process 300 (cron)                     │
│   └── Process 400 (nginx)                    │
│       ├── Process 401 (nginx worker)         │
│       └── Process 402 (nginx worker)         │
└─────────────────────────────────────────────┘
```

Every process has:
- **PID**: Process ID (unique number)
- **PPID**: Parent Process ID
- **UID**: User who owns it
- **State**: Running, sleeping, stopped, zombie
- **File descriptors**: Open files, pipes, sockets

---

## 37.2 Viewing Processes

### ps — Process Snapshot

```bash
ps                          # Your processes in current terminal
ps aux                      # All processes, detailed (BSD style)
ps -ef                      # All processes, detailed (System V style)

# Useful columns in 'ps aux':
# USER  PID  %CPU  %MEM  VSZ  RSS  TTY  STAT  START  TIME  COMMAND

# Find specific processes
ps aux | grep nginx
ps -C nginx                 # By exact command name

# Process tree
ps axjf                     # Tree format
pstree                      # Dedicated tree view
pstree -p                   # With PIDs

# Custom output
ps -eo pid,ppid,user,%cpu,%mem,comm --sort=-%cpu | head -20
```

### top / htop — Live Process Monitor

```bash
top                         # Live view, updates every 3 seconds
# Key commands inside top:
#   q     Quit
#   P     Sort by CPU
#   M     Sort by memory
#   k     Kill a process
#   1     Show individual CPUs

htop                        # Enhanced top (more user-friendly)
# Key commands in htop:
#   F5    Tree view
#   F9    Kill process
#   F6    Sort by column
#   /     Search
```

### /proc — Process Filesystem

```bash
# Every process has a directory in /proc
ls /proc/$$                      # Your shell's proc directory

cat /proc/$$/status              # Detailed process status
cat /proc/$$/cmdline | tr '\0' ' '   # Command line
ls -l /proc/$$/fd                # Open file descriptors
cat /proc/$$/environ | tr '\0' '\n'  # Environment variables
cat /proc/$$/maps                # Memory maps

# System-wide info
cat /proc/cpuinfo | grep "model name" | head -1
cat /proc/meminfo | head -5
cat /proc/uptime
cat /proc/loadavg
```

---

## 37.3 Process States

```
┌────────┬───────────────────────────────────────┐
│ State  │ Meaning                               │
├────────┼───────────────────────────────────────┤
│ R      │ Running or runnable                   │
│ S      │ Sleeping (waiting for event)          │
│ D      │ Uninterruptible sleep (usually I/O)   │
│ T      │ Stopped (by signal or debugger)       │
│ Z      │ Zombie (terminated, not reaped)       │
│ I      │ Idle kernel thread                    │
└────────┴───────────────────────────────────────┘
```

---

## 37.4 Killing Processes

```bash
# By PID
kill $pid                  # Send SIGTERM (15) — polite termination
kill -9 $pid               # Send SIGKILL (9) — forced termination
kill -HUP $pid             # Send SIGHUP (1) — reload config

# By name
killall nginx              # Kill all processes named "nginx"
pkill -f "python server"   # Kill processes matching pattern

# Interactive
kill -0 $pid && echo "Process exists" || echo "Process gone"
```

### Proper Shutdown Sequence

```bash
# Step 1: Ask nicely (SIGTERM)
kill $pid

# Step 2: Wait a bit
sleep 5

# Step 3: Force if still running
kill -0 $pid 2>/dev/null && kill -9 $pid
```

---

## 37.5 Process Priority (nice)

```bash
# Run with lower priority (higher nice = lower priority)
nice -n 10 ./heavy_computation.sh      # Nice value 10 (lower priority)
nice -n 19 ./background_task.sh        # Lowest priority

# Change priority of running process
renice 10 -p $pid                      # Set nice to 10
sudo renice -5 -p $pid                # Higher priority (needs root)

# Nice values: -20 (highest priority) to 19 (lowest priority)
# Default: 0
```

---

## 37.6 Process Substitution in Scripts

```bash
# Get PID of current script
echo "My PID: $$"

# Get PID of last background process
./server.sh &
server_pid=$!
echo "Server PID: $server_pid"

# Wait for specific process
wait $server_pid
echo "Server exited with code: $?"

# Check if process is running
is_running() {
    kill -0 "$1" 2>/dev/null
}

if is_running $server_pid; then
    echo "Server is running"
fi
```

---

## 37.7 Practical Script: Process Manager

```bash
#!/bin/bash
# Simple process management script

start_service() {
    local name="$1" cmd="$2" pidfile="/var/run/${name}.pid"
    
    if [[ -f "$pidfile" ]] && kill -0 "$(cat "$pidfile")" 2>/dev/null; then
        echo "$name is already running (PID $(cat "$pidfile"))"
        return 1
    fi
    
    $cmd &
    echo $! > "$pidfile"
    echo "$name started (PID $!)"
}

stop_service() {
    local name="$1" pidfile="/var/run/${name}.pid"
    
    if [[ ! -f "$pidfile" ]]; then
        echo "$name is not running"
        return 1
    fi
    
    local pid=$(cat "$pidfile")
    kill "$pid" 2>/dev/null
    
    # Wait up to 10 seconds for graceful shutdown
    local i=0
    while kill -0 "$pid" 2>/dev/null && (( i < 10 )); do
        sleep 1
        ((i++))
    done
    
    # Force kill if still running
    kill -0 "$pid" 2>/dev/null && kill -9 "$pid"
    
    rm -f "$pidfile"
    echo "$name stopped"
}
```

---

## Exercises

### Exercise 37.1: System Monitor
Write a script that shows: top 5 CPU-consuming processes, top 5 memory-consuming processes, total number of processes, and number of zombie processes.

### Exercise 37.2: Process Watchdog
Write a script that monitors a given process by name. If it dies, restart it and log the event.

---

## Summary

- Every program runs as a process with a unique PID
- `ps aux` shows all processes; `top`/`htop` for live monitoring
- `/proc/$PID/` contains detailed process information
- `kill` sends signals: TERM (15) for graceful, KILL (9) for forced
- `$$` = current PID, `$!` = last background PID
- `nice`/`renice` control process scheduling priority
- Always try SIGTERM before SIGKILL

---

**Next Chapter:** [Chapter 38: Working with Signals →](Chapter38-Signals.md)
