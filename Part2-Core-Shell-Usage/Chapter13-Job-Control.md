# Chapter 13: Job Control

## Learning Objectives

By the end of this chapter, you will be able to:

- Run commands in the foreground and background
- Suspend and resume processes
- Manage multiple jobs in a single terminal
- Use `&`, `fg`, `bg`, `jobs`, `wait`, `nohup`, and `disown`
- Understand when to use terminal multiplexers

---

## 13.1 Foreground vs. Background Processes

By default, commands run in the **foreground** — the shell waits for the command to finish before giving you back the prompt.

```bash
# Foreground: shell waits for sleep to finish (10 seconds)
sleep 10
# (you wait... no prompt for 10 seconds)
```

You can run a command in the **background** by appending `&`:

```bash
# Background: shell immediately returns the prompt
sleep 10 &
# [1] 12345           ← Job number [1], Process ID 12345
# $ █                 ← Prompt is available immediately
```

---

## 13.2 The `&` Operator

```bash
# Run a long process in the background
./long_running_script.sh &

# Run multiple background processes
./process1.sh &
./process2.sh &
./process3.sh &

# The output from background processes still goes to the terminal
# (unless redirected)
find / -name "*.log" &
# Output appears mixed with your typing — messy!

# Better: redirect output
find / -name "*.log" > results.txt 2>/dev/null &
```

---

## 13.3 Suspending and Resuming

### Ctrl+Z — Suspend a Foreground Process

```bash
# Start a long-running command
ping google.com
# ... ping is running ...
# Press Ctrl+Z
# [1]+  Stopped                 ping google.com
```

The process is **stopped** (paused), not killed. It still exists in memory.

### fg — Resume in Foreground

```bash
# Resume the most recently suspended job in the foreground
fg

# Resume a specific job (by job number)
fg %1
fg %2
```

### bg — Resume in Background

```bash
# Resume the stopped job in the background
bg

# Resume a specific job in the background
bg %1
```

### Common Workflow

```bash
# 1. Start a command
vim bigfile.txt

# 2. Need to run another command — suspend vim
# Press Ctrl+Z
# [1]+  Stopped                 vim bigfile.txt

# 3. Run your command
grep "error" /var/log/syslog

# 4. Go back to vim
fg
```

---

## 13.4 The jobs Command

`jobs` lists all background and stopped jobs:

```bash
sleep 100 &
sleep 200 &
ping localhost > /dev/null &

jobs
# [1]   Running                 sleep 100 &
# [2]-  Running                 sleep 200 &
# [3]+  Running                 ping localhost > /dev/null &
```

| Symbol | Meaning |
|--------|---------|
| `+` | Current job (default for `fg` and `bg`) |
| `-` | Previous job |
| No symbol | Other jobs |

### Job Specifiers

```bash
fg %1          # Job number 1
fg %+          # Current job (same as fg)
fg %-          # Previous job
fg %string     # Job whose command starts with "string"
fg %?string    # Job whose command contains "string"
```

---

## 13.5 Waiting for Background Jobs

### wait — Wait for Background Processes

```bash
# Wait for all background jobs to finish
./job1.sh &
./job2.sh &
./job3.sh &
wait
echo "All jobs completed"

# Wait for a specific job
./job1.sh &
pid1=$!         # $! holds the PID of the last background process

./job2.sh &
pid2=$!

wait $pid1
echo "Job 1 finished with status $?"

wait $pid2
echo "Job 2 finished with status $?"
```

### Parallel Execution Pattern

```bash
#!/bin/bash
# Process files in parallel, then wait for all to complete

for file in data/*.csv; do
    process_file "$file" &
done

wait    # Wait for ALL background jobs
echo "All files processed"
```

---

## 13.6 nohup — Survive Logout

When you close a terminal or log out, the shell sends the **SIGHUP** (hangup) signal to all its jobs, which typically kills them.

`nohup` makes a command immune to SIGHUP:

```bash
# Run a command that survives logout
nohup ./long_running_task.sh &

# Output goes to nohup.out by default
# You can specify a different output file:
nohup ./long_running_task.sh > output.log 2>&1 &

# Now you can safely close the terminal or log out
```

### disown — Remove a Job from Shell's Job Table

`disown` removes a running job from the shell's job list, preventing SIGHUP:

```bash
# Start a background process
./server.sh &

# Remove it from the job table
disown

# Or disown a specific job
./server.sh &
disown %1

# disown -h: don't remove from table, just don't send SIGHUP
./server.sh &
disown -h %1
```

**Difference:**
- `nohup` — Use BEFORE starting the command (prevents SIGHUP from the start)
- `disown` — Use AFTER starting the command (retroactively removes SIGHUP handling)

---

## 13.7 Terminal Multiplexers: tmux and screen

For serious terminal work, use a **terminal multiplexer**. These create persistent terminal sessions that survive disconnection.

### tmux (Recommended)

```bash
# Start a new tmux session
tmux

# Start a named session
tmux new -s mysession

# Detach from session (leave it running)
# Press Ctrl+B, then d

# List sessions
tmux ls

# Reattach to a session
tmux attach -t mysession

# Kill a session
tmux kill-session -t mysession
```

Key tmux commands (press `Ctrl+B` first):

| Key | Action |
|-----|--------|
| `d` | Detach from session |
| `c` | Create a new window |
| `n` | Next window |
| `p` | Previous window |
| `%` | Split pane vertically |
| `"` | Split pane horizontally |
| Arrow keys | Switch between panes |

### Why Use a Multiplexer?

1. **Persistence** — Sessions survive network disconnection (essential for SSH)
2. **Multiple windows** — Multiple terminals in one connection
3. **Split panes** — See multiple outputs simultaneously
4. **Shared sessions** — Multiple users can view/control the same session

---

## 13.8 Practical Scenario

```bash
# Scenario: Start a web server, monitor logs, and keep working

# Terminal multiplexer approach (recommended):
tmux new -s webdev

# Pane 1: Start the server
python3 -m http.server 8080

# Ctrl+B "  (split horizontally)
# Pane 2: Monitor access log
tail -f /var/log/nginx/access.log

# Ctrl+B %  (split vertically)
# Pane 3: Your working shell
vim app.py

# Ctrl+B d  (detach — everything keeps running)
# Close the terminal — everything survives!
# Later: tmux attach -s webdev
```

---

## Common Mistakes

1. **Forgetting about background job output** — Background processes still write to the terminal. Always redirect their output.

2. **Background jobs dying on logout** — Use `nohup` or `disown` (or better, `tmux`/`screen`) for long-running tasks.

3. **Not waiting for background jobs** — In scripts, forgetting `wait` means you don't know when (or if) background jobs finished.

4. **Using `&` without checking status** — Background errors are silent by default. Check `wait`'s exit status.

---

## Exercises

### Exercise 13.1: Background Basics
1. Run `sleep 30` in the background
2. Check the job list with `jobs`
3. Bring it to the foreground with `fg`
4. Suspend it with `Ctrl+Z`
5. Resume it in the background with `bg`
6. Wait for it to finish with `wait`

### Exercise 13.2: Parallel Processing
Write a script that runs three `sleep` commands (5, 10, 15 seconds) in parallel, waits for all of them, then prints "Done."

### Exercise 13.3: Surviving Logout
Use `nohup` to run a script that writes the date to a file every second for 60 seconds. Close the terminal, reopen it, and verify the output file is still being updated.

---

## Summary

- `&` runs commands in the **background**; the shell doesn't wait
- `Ctrl+Z` **suspends** a foreground process; `fg` resumes it; `bg` sends it to background
- `jobs` lists all shell jobs; use `%N` to reference specific jobs
- `wait` blocks until background processes finish (essential in scripts)
- `$!` holds the PID of the most recent background process
- `nohup` prevents SIGHUP on logout; `disown` removes from the job table
- **Terminal multiplexers** (tmux, screen) provide persistent, multi-window terminal sessions

---

## Part 2 Summary: Core Shell Usage Complete

You now understand:

1. **Command syntax** — How Bash tokenizes, expands, and executes commands
2. **Built-ins vs. externals** — The lookup order and performance implications
3. **Globbing** — Filename pattern matching with `*`, `?`, `[...]`
4. **Pipes and redirections** — Connecting commands and controlling I/O streams
5. **Exit codes** — How commands report success and failure
6. **Job control** — Running, suspending, and managing background processes

In Part 3, we'll begin writing **Bash scripts** — putting all these concepts into reusable, automated programs.

---

**Next Chapter:** [Part 3, Chapter 14: Your First Bash Script →](../Part3-Scripting-Basics/Chapter14-First-Script.md)
