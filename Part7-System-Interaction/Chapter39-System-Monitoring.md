# Chapter 39: System Monitoring

## Learning Objectives

By the end of this chapter, you will be able to:

- Monitor CPU, memory, disk, and network usage
- Build system health-check scripts
- Set up alerts based on resource thresholds
- Parse system information from /proc and standard tools
- Create monitoring dashboards in the terminal

---

## 39.1 CPU Monitoring

```bash
# Load averages (1, 5, 15 minute)
uptime
cat /proc/loadavg

# CPU usage per core
mpstat -P ALL 1 5     # All CPUs, every 1 second, 5 times

# Top CPU consumers
ps -eo pid,ppid,user,%cpu,%mem,comm --sort=-%cpu | head -10

# CPU info
nproc                        # Number of CPUs
cat /proc/cpuinfo | grep "model name" | head -1
lscpu                       # Detailed CPU architecture

# Script: get CPU usage percentage
get_cpu_usage() {
    local idle
    idle=$(top -bn1 | grep "Cpu(s)" | awk '{print $8}')
    echo "scale=1; 100 - $idle" | bc
}
```

---

## 39.2 Memory Monitoring

```bash
# Memory overview
free -h                     # Human-readable
free -m                     # In megabytes

# Detailed memory info
cat /proc/meminfo

# Memory usage percentage
get_memory_usage() {
    free | awk 'NR==2 { printf "%.1f%%\n", ($3/$2)*100 }'
}

# Top memory consumers
ps -eo pid,user,%mem,rss,comm --sort=-%mem | head -10

# Swap usage
swapon --show
```

---

## 39.3 Disk Monitoring

```bash
# Disk space
df -h                       # All filesystems
df -h /                     # Specific mount point
df -i                       # Inode usage

# Directory sizes
du -sh /var/log/*           # Size of each item in /var/log
du -sh /home/* | sort -rh   # Users by disk usage

# Find large files
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null | sort -k5 -rh

# Disk I/O
iostat -x 1 5              # Extended stats, every 1s, 5 times
iotop                      # Like top but for I/O

# Disk usage alert
check_disk() {
    df -h | awk 'NR>1 && $5+0 > 80 { 
        printf "WARNING: %s is %s full (%s used of %s)\n", $6, $5, $3, $2 
    }'
}
```

---

## 39.4 Network Monitoring

```bash
# Network interfaces and addresses
ip addr show
ip -s link show             # With statistics

# Active connections
ss -tunap                   # TCP/UDP, numeric, all, processes
ss -s                       # Connection summary

# Network traffic per interface
sar -n DEV 1 5             # Network stats per second

# Bandwidth monitoring
iftop                       # Live bandwidth by connection
nethogs                     # Bandwidth by process

# Port listening
ss -tlnp                   # Which processes are listening
```

---

## 39.5 Complete Health Check Script

```bash
#!/usr/bin/env bash
set -euo pipefail

# Simple system health check
health_check() {
    echo "=========================================="
    echo "  System Health Check — $(date)"
    echo "=========================================="
    echo
    
    # Hostname & uptime
    echo "--- System ---"
    echo "Hostname: $(hostname)"
    echo "Uptime:   $(uptime -p 2>/dev/null || uptime)"
    echo "Kernel:   $(uname -r)"
    echo
    
    # CPU
    echo "--- CPU ---"
    echo "Cores: $(nproc)"
    echo "Load:  $(cat /proc/loadavg | cut -d' ' -f1-3)"
    echo
    
    # Memory
    echo "--- Memory ---"
    free -h | awk '
        NR==1 { printf "%-10s %10s %10s %10s\n", "", $1, $2, $3 }
        NR==2 { printf "%-10s %10s %10s %10s\n", "RAM:", $2, $3, $4 }
        NR==3 { printf "%-10s %10s %10s %10s\n", "Swap:", $2, $3, $4 }
    '
    echo
    
    # Disk
    echo "--- Disk Usage ---"
    df -h | awk 'NR==1 || $5+0 > 0' | column -t
    echo
    
    # Top processes
    echo "--- Top 5 CPU Processes ---"
    ps -eo user,%cpu,%mem,pid,comm --sort=-%cpu | head -6
    echo
    
    echo "--- Top 5 Memory Processes ---"
    ps -eo user,%mem,%cpu,pid,comm --sort=-%mem | head -6
    echo
    
    # Warnings
    echo "--- Warnings ---"
    local warnings=0
    
    # High disk usage
    while read -r line; do
        echo "DISK: $line"
        ((warnings++))
    done < <(df -h | awk 'NR>1 && $5+0 > 80 { print $6, "is", $5, "full" }')
    
    # High memory usage
    local mem_pct
    mem_pct=$(free | awk 'NR==2 { printf "%d", ($3/$2)*100 }')
    if (( mem_pct > 85 )); then
        echo "MEMORY: ${mem_pct}% used"
        ((warnings++))
    fi
    
    # High load
    local cores load
    cores=$(nproc)
    load=$(cat /proc/loadavg | cut -d'.' -f1)
    if (( load > cores * 2 )); then
        echo "LOAD: Load average is critically high"
        ((warnings++))
    fi
    
    if (( warnings == 0 )); then
        echo "No warnings — system is healthy"
    fi
    
    echo
    echo "=========================================="
}

health_check
```

---

## 39.6 Scheduled Monitoring

```bash
# Add to crontab for regular monitoring
# crontab -e
# */5 * * * * /path/to/health_check.sh >> /var/log/health.log 2>&1

# Alert on high resource usage
alert_if_needed() {
    local metric="$1" value="$2" threshold="$3"
    if (( value > threshold )); then
        echo "ALERT: $metric at ${value}% (threshold: ${threshold}%)" |
            mail -s "Server Alert: $metric" admin@example.com
    fi
}
```

---

## Exercises

### Exercise 39.1: Dashboard
Build a terminal dashboard that refreshes every 5 seconds showing CPU, memory, disk, and top processes (use `watch` or a loop with `clear`).

### Exercise 39.2: Capacity Planning
Write a script that logs CPU, memory, and disk usage to a CSV file every minute. After a day, generate a report with min/max/average for each metric.

---

## Summary

- CPU: `uptime`, `mpstat`, `/proc/loadavg`, `nproc`
- Memory: `free -h`, `/proc/meminfo`, `ps --sort=-%mem`
- Disk: `df -h`, `du -sh`, `find -size`, `iostat`
- Network: `ss`, `ip addr`, `iftop`, `nethogs`
- Combine these into health-check scripts with threshold alerts
- Schedule monitoring with cron for continuous oversight

---

**Next Chapter:** [Chapter 40: Cron and Scheduling →](Chapter40-Cron-Scheduling.md)
