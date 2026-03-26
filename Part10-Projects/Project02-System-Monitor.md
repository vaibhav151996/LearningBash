# Project 2: System Monitor Dashboard

## Project Overview

Build a terminal-based system monitoring dashboard that displays CPU, memory, disk, network, and process information with real-time updates.

---

## Requirements

- Display CPU usage, memory usage, disk usage, and load averages
- Show top processes by CPU and memory
- Auto-refresh at configurable intervals
- Color-coded alerts for thresholds
- Export report to file

---

## Complete Implementation

```bash
#!/usr/bin/env bash
#
# sysmon.sh — Terminal System Monitor Dashboard
#
# Usage: sysmon.sh [-i interval] [-r report_file] [-1]

set -euo pipefail

readonly VERSION="1.0.0"
INTERVAL=3
REPORT_FILE=""
ONE_SHOT=false

# --- Colors ---
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BLUE='\033[0;34m'
BOLD='\033[1m'
NC='\033[0m'    # No Color

# --- Helpers ---
color_pct() {
    local pct="$1"
    if (( pct >= 90 )); then echo -e "${RED}${pct}%${NC}"
    elif (( pct >= 70 )); then echo -e "${YELLOW}${pct}%${NC}"
    else echo -e "${GREEN}${pct}%${NC}"
    fi
}

bar() {
    local pct="$1" width="${2:-30}" filled empty
    filled=$(( pct * width / 100 ))
    empty=$(( width - filled ))
    
    printf '['
    (( filled > 0 )) && printf '%0.s█' $(seq 1 "$filled")
    (( empty > 0 ))  && printf '%0.s░' $(seq 1 "$empty")
    printf '] '
    color_pct "$pct"
}

header() {
    echo -e "${BOLD}${BLUE}$1${NC}"
    printf '%*s\n' "${#1}" '' | tr ' ' '─'
}

# --- Data Collection ---
get_cpu_usage() {
    # Read two snapshots of /proc/stat
    local cpu1 cpu2
    cpu1=($(head -1 /proc/stat))
    sleep 0.5
    cpu2=($(head -1 /proc/stat))
    
    local idle1=${cpu1[4]} idle2=${cpu2[4]}
    local total1=0 total2=0
    for ((i=1; i<${#cpu1[@]}; i++)); do
        total1=$((total1 + cpu1[i]))
        total2=$((total2 + cpu2[i]))
    done
    
    local total_diff=$((total2 - total1))
    local idle_diff=$((idle2 - idle1))
    
    if (( total_diff > 0 )); then
        echo $(( (total_diff - idle_diff) * 100 / total_diff ))
    else
        echo 0
    fi
}

get_memory_info() {
    local total used free pct
    read -r total used free <<< "$(free -m | awk 'NR==2 {print $2, $3, $4}')"
    pct=$(( used * 100 / total ))
    echo "$total $used $free $pct"
}

get_swap_info() {
    local total used pct
    read -r total used <<< "$(free -m | awk 'NR==3 {print $2, $3}')"
    if (( total > 0 )); then
        pct=$(( used * 100 / total ))
    else
        pct=0
    fi
    echo "$total $used $pct"
}

get_disk_info() {
    df -h --output=source,size,used,avail,pcent,target 2>/dev/null | \
        grep -E '^/dev/' | head -5
}

# --- Display ---
display_dashboard() {
    clear
    
    echo -e "${BOLD}╔══════════════════════════════════════════╗${NC}"
    echo -e "${BOLD}║     System Monitor Dashboard v$VERSION      ║${NC}"
    echo -e "${BOLD}║     $(date '+%Y-%m-%d %H:%M:%S')                    ║${NC}"
    echo -e "${BOLD}╚══════════════════════════════════════════╝${NC}"
    echo
    
    # System Info
    header "SYSTEM"
    printf "  Hostname:  %s\n" "$(hostname)"
    printf "  Kernel:    %s\n" "$(uname -r)"
    printf "  Uptime:    %s\n" "$(uptime -p 2>/dev/null || uptime)"
    printf "  Load Avg:  %s\n" "$(cat /proc/loadavg | cut -d' ' -f1-3)"
    echo
    
    # CPU
    header "CPU"
    local cpu_pct
    cpu_pct=$(get_cpu_usage)
    printf "  Usage:  "
    bar "$cpu_pct" 30
    echo
    printf "  Cores:  %s\n" "$(nproc)"
    echo
    
    # Memory
    header "MEMORY"
    local mem_total mem_used mem_free mem_pct
    read -r mem_total mem_used mem_free mem_pct <<< "$(get_memory_info)"
    printf "  RAM:    "
    bar "$mem_pct" 30
    printf "          %s MB used / %s MB total (%s MB free)\n" "$mem_used" "$mem_total" "$mem_free"
    
    local swap_total swap_used swap_pct
    read -r swap_total swap_used swap_pct <<< "$(get_swap_info)"
    if (( swap_total > 0 )); then
        printf "  Swap:   "
        bar "$swap_pct" 30
        printf "          %s MB used / %s MB total\n" "$swap_used" "$swap_total"
    fi
    echo
    
    # Disk
    header "DISK USAGE"
    printf "  %-20s %6s %6s %6s %5s  %s\n" "DEVICE" "SIZE" "USED" "AVAIL" "USE%" "MOUNT"
    while read -r dev size used avail pct mount; do
        local pct_num="${pct%\%}"
        printf "  %-20s %6s %6s %6s " "$dev" "$size" "$used" "$avail"
        color_pct "$pct_num"
        printf "  %s\n" "$mount"
    done <<< "$(get_disk_info)"
    echo
    
    # Top Processes
    header "TOP 5 CPU PROCESSES"
    printf "  ${BOLD}%-8s %5s %5s  %s${NC}\n" "USER" "%CPU" "%MEM" "COMMAND"
    ps -eo user,%cpu,%mem,comm --sort=-%cpu --no-headers | head -5 | \
        while read -r user cpu mem comm; do
            printf "  %-8s %5s %5s  %s\n" "$user" "$cpu" "$mem" "$comm"
        done
    echo
    
    header "TOP 5 MEMORY PROCESSES"
    printf "  ${BOLD}%-8s %5s %5s  %s${NC}\n" "USER" "%MEM" "%CPU" "COMMAND"
    ps -eo user,%mem,%cpu,comm --sort=-%mem --no-headers | head -5 | \
        while read -r user mem cpu comm; do
            printf "  %-8s %5s %5s  %s\n" "$user" "$mem" "$cpu" "$comm"
        done
    echo
    
    # Footer
    echo -e "${BLUE}Refreshing every ${INTERVAL}s. Press Ctrl+C to exit.${NC}"
}

# --- Report ---
generate_report() {
    local file="$1"
    {
        echo "System Monitor Report — $(date)"
        echo "=================================="
        echo "Hostname: $(hostname)"
        echo "Kernel: $(uname -r)"
        echo "Uptime: $(uptime -p 2>/dev/null || uptime)"
        echo ""
        echo "CPU Usage: $(get_cpu_usage)%"
        echo ""
        echo "Memory:"
        free -h
        echo ""
        echo "Disk Usage:"
        df -h
        echo ""
        echo "Top Processes (CPU):"
        ps -eo pid,user,%cpu,%mem,comm --sort=-%cpu | head -11
        echo ""
        echo "Top Processes (Memory):"
        ps -eo pid,user,%mem,%cpu,comm --sort=-%mem | head -11
    } > "$file"
    echo "Report saved to: $file"
}

# --- Main ---
usage() {
    cat <<EOF
Usage: $0 [OPTIONS]
  -i SEC    Refresh interval (default: 3)
  -r FILE   Generate report and exit
  -1        One-shot (display once, no refresh)
  -h        Show help
EOF
}

while getopts ":i:r:1h" opt; do
    case $opt in
        i) INTERVAL="$OPTARG" ;;
        r) REPORT_FILE="$OPTARG" ;;
        1) ONE_SHOT=true ;;
        h) usage; exit 0 ;;
        *) usage; exit 1 ;;
    esac
done

if [[ -n "$REPORT_FILE" ]]; then
    generate_report "$REPORT_FILE"
    exit 0
fi

trap 'echo -e "\n${GREEN}Goodbye!${NC}"; exit 0' INT

if [[ "$ONE_SHOT" == true ]]; then
    display_dashboard
else
    while true; do
        display_dashboard
        sleep "$INTERVAL"
    done
fi
```

---

## Testing

```bash
chmod +x sysmon.sh

# Live dashboard
./sysmon.sh

# Custom interval
./sysmon.sh -i 5

# One-shot
./sysmon.sh -1

# Generate report
./sysmon.sh -r report.txt
```

---

## Extensions

1. Add network bandwidth monitoring
2. Add disk I/O stats
3. Add per-CPU core usage display
4. Send alerts via email when thresholds are exceeded
5. Store historical data for trend analysis

---

**Next Project:** [Project 3: Log Analyzer →](Project03-Log-Analyzer.md)
