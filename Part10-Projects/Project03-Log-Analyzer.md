# Project 3: Log Analyzer

## Project Overview

Build a comprehensive log analysis tool that parses web server access logs, generates statistics, detects anomalies, and produces formatted reports.

---

## Requirements

- Parse Apache/Nginx combined log format
- Generate traffic statistics (requests, bytes, status codes)
- Identify top pages, IPs, and user agents
- Detect suspicious activity (high error rates, brute force attempts)
- Output reports in text and CSV format

---

## Complete Implementation

```bash
#!/usr/bin/env bash
#
# logalyzer.sh — Web Server Log Analyzer
#
# Usage: logalyzer.sh [-f format] [-t top_n] [-o output] <logfile>

set -euo pipefail

readonly SCRIPT_NAME="$(basename "$0")"
TOP_N=10
OUTPUT_FORMAT="text"
OUTPUT_FILE=""

# --- Usage ---
usage() {
    cat <<EOF
Usage: $SCRIPT_NAME [OPTIONS] <logfile>

Analyze web server access logs.

Options:
    -t N          Show top N results (default: 10)
    -o FILE       Write report to file
    -c            CSV output format
    -s SECTION    Show specific section:
                    summary, status, pages, ips, hours, agents, errors
    -h            Show help

Examples:
    $SCRIPT_NAME access.log
    $SCRIPT_NAME -t 20 -o report.txt access.log
    $SCRIPT_NAME -s errors access.log
EOF
}

# --- Analysis Functions ---

analyze_summary() {
    local logfile="$1"
    local total_requests total_bytes first_entry last_entry unique_ips
    
    total_requests=$(wc -l < "$logfile")
    total_bytes=$(awk '{sum += $10} END {print sum+0}' "$logfile")
    first_entry=$(head -1 "$logfile" | grep -oP '\[\K[^\]]+' | head -1)
    last_entry=$(tail -1 "$logfile" | grep -oP '\[\K[^\]]+' | head -1)
    unique_ips=$(awk '{print $1}' "$logfile" | sort -u | wc -l)
    
    echo "=== TRAFFIC SUMMARY ==="
    printf "  Total Requests:   %'d\n" "$total_requests"
    printf "  Total Data:       %s\n" "$(numfmt --to=iec-i "$total_bytes" 2>/dev/null || echo "${total_bytes} bytes")"
    printf "  Unique IPs:       %'d\n" "$unique_ips"
    printf "  Period:           %s\n" "${first_entry:-unknown}"
    printf "  To:               %s\n" "${last_entry:-unknown}"
    echo
}

analyze_status_codes() {
    local logfile="$1"
    echo "=== HTTP STATUS CODES ==="
    printf "  %-6s %-8s %s\n" "CODE" "COUNT" "DESCRIPTION"
    printf "  %-6s %-8s %s\n" "------" "--------" "-----------"
    
    awk '{print $9}' "$logfile" | sort | uniq -c | sort -rn | \
    while read -r count code; do
        local desc=""
        case "$code" in
            200) desc="OK" ;;
            301) desc="Moved Permanently" ;;
            302) desc="Found (Redirect)" ;;
            304) desc="Not Modified" ;;
            400) desc="Bad Request" ;;
            401) desc="Unauthorized" ;;
            403) desc="Forbidden" ;;
            404) desc="Not Found" ;;
            500) desc="Internal Server Error" ;;
            502) desc="Bad Gateway" ;;
            503) desc="Service Unavailable" ;;
            *) desc="" ;;
        esac
        printf "  %-6s %-8s %s\n" "$code" "$count" "$desc"
    done
    echo
}

analyze_top_pages() {
    local logfile="$1" n="$2"
    echo "=== TOP $n REQUESTED PAGES ==="
    printf "  %-8s %s\n" "HITS" "URL"
    printf "  %-8s %s\n" "--------" "---"
    
    awk '{print $7}' "$logfile" | sort | uniq -c | sort -rn | head -"$n" | \
    while read -r count url; do
        printf "  %-8s %s\n" "$count" "$url"
    done
    echo
}

analyze_top_ips() {
    local logfile="$1" n="$2"
    echo "=== TOP $n IP ADDRESSES ==="
    printf "  %-8s %-18s %s\n" "HITS" "IP ADDRESS" "REQUESTS/MIN"
    printf "  %-8s %-18s %s\n" "--------" "------------------" "------------"
    
    awk '{print $1}' "$logfile" | sort | uniq -c | sort -rn | head -"$n" | \
    while read -r count ip; do
        printf "  %-8s %-18s\n" "$count" "$ip"
    done
    echo
}

analyze_hourly() {
    local logfile="$1"
    echo "=== REQUESTS BY HOUR ==="
    
    awk '{
        match($4, /:[0-9]{2}:/, arr)
        hour = substr($4, index($4, ":")+1, 2)
        hours[hour]++
    }
    END {
        for (h=0; h<24; h++) {
            hh = sprintf("%02d", h)
            count = (hh in hours) ? hours[hh] : 0
            bar = ""
            blocks = int(count / 10)
            if (blocks > 50) blocks = 50
            for (i=0; i<blocks; i++) bar = bar "█"
            printf "  %s:00  %6d  %s\n", hh, count, bar
        }
    }' "$logfile"
    echo
}

analyze_errors() {
    local logfile="$1" n="$2"
    echo "=== TOP $n 404 ERRORS ==="
    printf "  %-8s %s\n" "HITS" "URL"
    printf "  %-8s %s\n" "--------" "---"
    
    awk '$9 == 404 {print $7}' "$logfile" | sort | uniq -c | sort -rn | head -"$n" | \
    while read -r count url; do
        printf "  %-8s %s\n" "$count" "$url"
    done
    echo
    
    echo "=== TOP $n 5XX ERRORS ==="
    printf "  %-8s %-6s %s\n" "HITS" "CODE" "URL"
    
    awk '$9 >= 500 && $9 < 600 {print $9, $7}' "$logfile" | sort | uniq -c | sort -rn | head -"$n" | \
    while read -r count code url; do
        printf "  %-8s %-6s %s\n" "$count" "$code" "$url"
    done
    echo
}

analyze_suspicious() {
    local logfile="$1"
    echo "=== SUSPICIOUS ACTIVITY ==="
    
    # IPs with high error rates
    echo "  IPs with >50% error rate (min 10 requests):"
    awk '{
        ip=$1; code=$9
        total[ip]++
        if (code >= 400) errors[ip]++
    } END {
        for (ip in total) {
            if (total[ip] >= 10) {
                rate = (errors[ip]+0) * 100 / total[ip]
                if (rate > 50)
                    printf "    %-18s %d/%d requests failed (%.0f%%)\n", ip, errors[ip]+0, total[ip], rate
            }
        }
    }' "$logfile" | sort -t'(' -k2 -rn | head -10
    
    echo
    echo "  IPs with >100 requests to login/admin:"
    awk '$7 ~ /(login|admin|wp-login|phpmyadmin)/i {ips[$1]++}
        END {for (ip in ips) if (ips[ip]>100) printf "    %-18s %d attempts\n", ip, ips[ip]}
    ' "$logfile" | sort -t' ' -k2 -rn
    echo
}

# --- Generate Full Report ---
full_report() {
    local logfile="$1"
    
    echo "╔══════════════════════════════════════════╗"
    echo "║        Log Analysis Report               ║"
    echo "║        $(date '+%Y-%m-%d %H:%M:%S')                ║"
    echo "║        File: $(basename "$logfile")       ║"
    echo "╚══════════════════════════════════════════╝"
    echo
    
    analyze_summary "$logfile"
    analyze_status_codes "$logfile"
    analyze_top_pages "$logfile" "$TOP_N"
    analyze_top_ips "$logfile" "$TOP_N"
    analyze_hourly "$logfile"
    analyze_errors "$logfile" "$TOP_N"
    analyze_suspicious "$logfile"
}

# --- Main ---
SECTION=""

while getopts ":t:o:s:ch" opt; do
    case $opt in
        t) TOP_N="$OPTARG" ;;
        o) OUTPUT_FILE="$OPTARG" ;;
        s) SECTION="$OPTARG" ;;
        c) OUTPUT_FORMAT="csv" ;;
        h) usage; exit 0 ;;
        *) usage; exit 1 ;;
    esac
done
shift $((OPTIND - 1))

(( $# >= 1 )) || { usage; exit 1; }
LOGFILE="$1"
[[ -f "$LOGFILE" ]] || { echo "File not found: $LOGFILE" >&2; exit 1; }

# Run analysis
run_analysis() {
    if [[ -n "$SECTION" ]]; then
        case "$SECTION" in
            summary) analyze_summary "$LOGFILE" ;;
            status)  analyze_status_codes "$LOGFILE" ;;
            pages)   analyze_top_pages "$LOGFILE" "$TOP_N" ;;
            ips)     analyze_top_ips "$LOGFILE" "$TOP_N" ;;
            hours)   analyze_hourly "$LOGFILE" ;;
            errors)  analyze_errors "$LOGFILE" "$TOP_N" ;;
            suspicious) analyze_suspicious "$LOGFILE" ;;
            *) echo "Unknown section: $SECTION" >&2; exit 1 ;;
        esac
    else
        full_report "$LOGFILE"
    fi
}

if [[ -n "$OUTPUT_FILE" ]]; then
    run_analysis > "$OUTPUT_FILE"
    echo "Report saved to: $OUTPUT_FILE"
else
    run_analysis
fi
```

---

## Testing

Generate a sample log file and run the analyzer:

```bash
# Generate sample access log entries
for i in {1..100}; do
    echo "192.168.1.$((RANDOM % 10 + 1)) - - [20/Jan/2024:$((RANDOM % 24)):$((RANDOM % 60)):$((RANDOM % 60)) +0000] \"GET /page$((RANDOM % 20)) HTTP/1.1\" $((RANDOM % 2 == 0 ? 200 : 404)) $((RANDOM * 10))"
done > /tmp/sample_access.log

./logalyzer.sh /tmp/sample_access.log
./logalyzer.sh -s hours /tmp/sample_access.log
./logalyzer.sh -t 5 -o /tmp/report.txt /tmp/sample_access.log
```

---

## Extensions

1. Support multiple log formats (JSON logs, syslog)
2. Add geolocation lookup for IP addresses
3. Generate HTML reports with charts
4. Real-time mode that tails the log
5. Time-window analysis (specific date ranges)

---

**Next Project:** [Project 4: Backup System →](Project04-Backup-System.md)
