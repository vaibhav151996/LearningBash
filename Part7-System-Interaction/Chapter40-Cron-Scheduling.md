# Chapter 40: Cron and Scheduling

## Learning Objectives

By the end of this chapter, you will be able to:

- Schedule tasks with cron and crontab
- Write reliable cron job entries
- Use systemd timers as a modern alternative
- Handle logging and error notification for scheduled tasks
- Avoid common cron pitfalls

---

## 40.1 What Is Cron?

Cron is the traditional Unix task scheduler. It runs commands at specified times automatically.

```
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-7, 0 and 7 = Sunday)
│ │ │ │ │
* * * * *  command to execute
```

---

## 40.2 Crontab Basics

```bash
# Edit your crontab
crontab -e

# List your crontab
crontab -l

# Remove your crontab
crontab -r

# Edit another user's crontab (root only)
sudo crontab -u username -e
```

### Common Schedule Examples

```bash
# Every minute
* * * * *  /path/to/script.sh

# Every 5 minutes
*/5 * * * *  /path/to/script.sh

# Every hour at minute 0
0 * * * *  /path/to/script.sh

# Every day at 2:30 AM
30 2 * * *  /path/to/script.sh

# Every Monday at 9 AM
0 9 * * 1  /path/to/script.sh

# First day of every month at midnight
0 0 1 * *  /path/to/script.sh

# Every weekday at 6 PM
0 18 * * 1-5  /path/to/script.sh

# Every 15 minutes during business hours
*/15 9-17 * * 1-5  /path/to/script.sh

# Twice daily at 8 AM and 8 PM
0 8,20 * * *  /path/to/script.sh
```

### Special Strings

```bash
@reboot    /path/to/script.sh     # Once after reboot
@hourly    /path/to/script.sh     # 0 * * * *
@daily     /path/to/script.sh     # 0 0 * * *
@weekly    /path/to/script.sh     # 0 0 * * 0
@monthly   /path/to/script.sh     # 0 0 1 * *
@yearly    /path/to/script.sh     # 0 0 1 1 *
```

---

## 40.3 Writing Reliable Cron Jobs

### Rule 1: Use Absolute Paths

```bash
# BAD — cron doesn't have your PATH
* * * * *  backup.sh

# GOOD — absolute paths for everything
* * * * *  /home/user/scripts/backup.sh
```

### Rule 2: Set PATH and Environment

```bash
# At the top of your crontab:
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
MAILTO=admin@example.com

0 2 * * *  /home/user/scripts/backup.sh
```

### Rule 3: Redirect Output

```bash
# Log stdout and stderr
0 2 * * *  /path/to/script.sh >> /var/log/backup.log 2>&1

# Discard output (only get email on error... if MAILTO is set)
0 2 * * *  /path/to/script.sh > /dev/null 2>&1

# Log with timestamp
0 2 * * *  /path/to/script.sh 2>&1 | logger -t backup
```

### Rule 4: Use Lock Files

```bash
# Prevent overlapping runs
0 * * * *  flock -n /tmp/myjob.lock /path/to/script.sh

# Or inside the script:
LOCKFILE="/tmp/${0##*/}.lock"
exec 200>"$LOCKFILE"
flock -n 200 || { echo "Already running"; exit 1; }
```

---

## 40.4 Cron Job Template

```bash
#!/usr/bin/env bash
# /home/user/scripts/daily-backup.sh
# Cron entry: 0 2 * * * /home/user/scripts/daily-backup.sh

set -euo pipefail

# Cron-safe environment
export PATH="/usr/local/bin:/usr/bin:/bin"
export HOME="/home/user"

readonly SCRIPT_NAME="$(basename "$0")"
readonly LOG_FILE="/var/log/${SCRIPT_NAME%.sh}.log"
readonly LOCK_FILE="/tmp/${SCRIPT_NAME}.lock"

# Logging
log() { printf '[%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*" >> "$LOG_FILE"; }

# Locking
exec 200>"$LOCK_FILE"
flock -n 200 || { log "Already running, exiting"; exit 0; }

# Cleanup
cleanup() {
    local exit_code=$?
    if (( exit_code != 0 )); then
        log "FAILED with exit code $exit_code"
    fi
}
trap cleanup EXIT

# Main
log "Starting backup"
# ... backup commands ...
log "Backup complete"
```

---

## 40.5 systemd Timers (Modern Alternative)

systemd timers offer advantages over cron: dependencies, logging with journald, randomized delays, and better error handling.

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Daily Backup

[Service]
Type=oneshot
ExecStart=/home/user/scripts/backup.sh
User=user

# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup daily

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

```bash
# Enable and start the timer
sudo systemctl enable backup.timer
sudo systemctl start backup.timer

# Check timer status
systemctl list-timers
systemctl status backup.timer

# View logs
journalctl -u backup.service

# Run manually
sudo systemctl start backup.service
```

---

## 40.6 Other Scheduling Tools

```bash
# at — Run once at a specific time
echo "/path/to/script.sh" | at 2:00 AM
echo "/path/to/script.sh" | at now + 1 hour
atq                          # List queued jobs
atrm 5                       # Remove job 5

# batch — Run when system load is low
echo "/path/to/heavy-task.sh" | batch

# anacron — For systems that aren't always on
# Runs missed jobs (good for laptops)
# Config: /etc/anacrontab
```

---

## 40.7 Common Cron Pitfalls

| Pitfall | Solution |
|---------|----------|
| Script works manually, fails in cron | Set PATH explicitly in crontab or script |
| No output/errors visible | Redirect to log file: `>> /var/log/job.log 2>&1` |
| Job runs multiple times simultaneously | Use `flock` for locking |
| Environment variables missing | Set them in script, not in `.bashrc` |
| Cron uses `/bin/sh` not bash | Set `SHELL=/bin/bash` or use `#!/bin/bash` shebang |
| `%` in commands treated as newlines | Escape: `\%` or put commands in a script |

---

## Exercises

### Exercise 40.1: Crontab Design
Write crontab entries for: (a) rotate logs every Sunday at 3 AM, (b) check disk space every 30 minutes, (c) send a weekly report every Friday at 5 PM.

### Exercise 40.2: Reliable Cron Job
Write a cron-safe script that: pulls data from an API, processes it, saves to a database, sends email on failure. Include locking, logging, and error handling.

---

## Summary

- Cron schedules tasks with `* * * * *` (minute hour day month weekday)
- Always use absolute paths, set PATH, redirect output, and use lock files
- `flock` prevents overlapping runs
- systemd timers are the modern alternative with better logging and features
- `at` and `batch` for one-time scheduling
- Always test cron scripts manually first with the same environment

---

**Next Chapter:** [Chapter 41: Systemd and Services →](Chapter41-Systemd-Services.md)
