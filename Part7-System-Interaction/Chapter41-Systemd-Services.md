# Chapter 41: Systemd and Services

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand systemd and its role in modern Linux
- Manage services with systemctl
- Create custom service unit files
- View logs with journalctl
- Understand targets, dependencies, and boot process

---

## 41.1 What Is systemd?

systemd is the init system and service manager for most modern Linux distributions. PID 1.

```
┌─────────────────────────────────────────────┐
│ Kernel boots → starts systemd (PID 1)       │
│   ├── Mounts filesystems                     │
│   ├── Sets up networking                     │
│   ├── Starts services (sshd, nginx, etc.)    │
│   └── Reaches "multi-user.target" or         │
│       "graphical.target"                     │
└─────────────────────────────────────────────┘
```

Units managed by systemd:
- **service** — daemons and processes
- **timer** — scheduled tasks (like cron)
- **mount** — filesystem mount points
- **target** — groups of units (like runlevels)
- **socket** — network sockets

---

## 41.2 Managing Services

```bash
# Status
sudo systemctl status nginx           # Detailed status
systemctl is-active nginx              # "active" or "inactive"
systemctl is-enabled nginx             # "enabled" or "disabled"

# Start / Stop / Restart
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx           # Stop + start
sudo systemctl reload nginx            # Reload config without restart

# Enable / Disable (boot persistence)
sudo systemctl enable nginx            # Start on boot
sudo systemctl disable nginx           # Don't start on boot
sudo systemctl enable --now nginx      # Enable AND start immediately

# List services
systemctl list-units --type=service              # Running services
systemctl list-units --type=service --state=failed  # Failed services
systemctl list-unit-files --type=service         # All service files
```

---

## 41.3 Creating a Custom Service

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application Server
Documentation=https://myapp.example.com/docs
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.yml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal
Environment=NODE_ENV=production
EnvironmentFile=/etc/myapp/env

# Security hardening
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/lib/myapp /var/log/myapp

[Install]
WantedBy=multi-user.target
```

```bash
# After creating the unit file:
sudo systemctl daemon-reload           # Reload unit files
sudo systemctl enable --now myapp      # Enable and start
sudo systemctl status myapp            # Check status
```

### Service Types

```
Type=simple      Default. Process started by ExecStart is the main process
Type=forking     Process forks into background (traditional daemons)
Type=oneshot     For scripts that run and exit (like batch jobs)
Type=notify      Service notifies systemd when it's ready
Type=idle        Like simple, but waits until other jobs finish
```

---

## 41.4 journalctl — Viewing Logs

```bash
# All logs
journalctl

# Logs for a specific service
journalctl -u nginx
journalctl -u nginx --since today
journalctl -u nginx --since "2024-01-15 10:00" --until "2024-01-15 12:00"

# Follow logs in real-time
journalctl -u nginx -f

# Recent logs
journalctl -u nginx -n 50             # Last 50 lines
journalctl -u nginx --since "5 min ago"

# By priority
journalctl -p err                     # Errors and above
journalctl -p warning -u nginx        # Warnings from nginx

# Boot logs
journalctl -b                         # Current boot
journalctl -b -1                      # Previous boot
journalctl --list-boots               # List all boots

# Kernel messages
journalctl -k

# Output formats
journalctl -u nginx -o json-pretty    # JSON format
journalctl -u nginx -o short-precise  # Precise timestamps

# Disk usage
journalctl --disk-usage
sudo journalctl --vacuum-size=100M    # Clean up to 100MB
```

---

## 41.5 Practical: Deploy a Bash Service

```bash
#!/bin/bash
# /opt/myworker/worker.sh — A background worker script

set -euo pipefail

readonly QUEUE_DIR="/var/spool/myworker"
readonly LOG_TAG="myworker"

log() { logger -t "$LOG_TAG" "$*"; }

process_job() {
    local job_file="$1"
    log "Processing: $job_file"
    # ... process the job ...
    mv "$job_file" "${job_file}.done"
    log "Completed: $job_file"
}

main() {
    log "Worker started (PID $$)"
    mkdir -p "$QUEUE_DIR"
    
    while true; do
        for job in "$QUEUE_DIR"/*.job; do
            [[ -f "$job" ]] || continue
            process_job "$job"
        done
        sleep 5
    done
}

main
```

```ini
# /etc/systemd/system/myworker.service
[Unit]
Description=My Worker Service
After=network.target

[Service]
Type=simple
ExecStart=/opt/myworker/worker.sh
Restart=always
RestartSec=10
User=worker

[Install]
WantedBy=multi-user.target
```

---

## 41.6 Targets (Runlevels)

```bash
# View current target
systemctl get-default

# Set default target
sudo systemctl set-default multi-user.target     # Console only
sudo systemctl set-default graphical.target       # GUI

# Switch targets
sudo systemctl isolate multi-user.target

# Common targets:
# poweroff.target     — Shut down
# rescue.target       — Single-user / rescue mode
# multi-user.target   — Multi-user, no GUI (like runlevel 3)
# graphical.target    — Multi-user with GUI (like runlevel 5)
```

---

## 41.7 Common Mistakes

| Mistake | Solution |
|---------|----------|
| Forgot `daemon-reload` | Always run after editing unit files |
| Service crashes, won't restart | Set `Restart=on-failure` and `RestartSec=5` |
| Environment not set | Use `Environment=` or `EnvironmentFile=` |
| Log output missing | Set `StandardOutput=journal` |
| Wrong permissions | Set correct `User=` and file permissions |

---

## Exercises

### Exercise 41.1: Custom Service
Create a systemd service for a Bash script that monitors a directory for new files and processes them. Include proper restart, logging, and security settings.

### Exercise 41.2: Service Management Script
Write a Bash script that takes a service name and actions (status, start, stop, logs) as arguments and provides a user-friendly interface to systemctl and journalctl.

---

## Summary

- systemd manages services, timers, mounts, and the boot process
- `systemctl` controls services: start, stop, enable, disable, status
- Unit files in `/etc/systemd/system/` define custom services
- `journalctl` provides powerful log viewing and filtering
- Always `daemon-reload` after editing unit files
- Use security directives (`ProtectSystem`, `NoNewPrivileges`) for hardening
- Targets replace traditional runlevels

---

**Next Chapter:** [Chapter 42: Arrays →](../Part8-Advanced-Bash/Chapter42-Arrays.md)
