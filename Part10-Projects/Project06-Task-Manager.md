# Project 6: CLI Task Manager

## Project Overview

Build a full-featured command-line task manager with add, list, complete, delete, priority, and search capabilities, backed by a simple file-based storage.

---

## Requirements

- Add, list, complete, delete, and edit tasks
- Priority levels (high, medium, low)
- Due dates
- Tags/categories
- Search and filter
- Import/export (CSV, JSON)
- Color-coded output

---

## Complete Implementation

```bash
#!/usr/bin/env bash
#
# task.sh — CLI Task Manager
#
# Usage: task <command> [options]

set -euo pipefail

readonly SCRIPT_NAME="$(basename "$0")"
readonly DATA_DIR="${TASK_DIR:-$HOME/.tasks}"
readonly TASK_FILE="${DATA_DIR}/tasks.tsv"
readonly ARCHIVE_FILE="${DATA_DIR}/archive.tsv"
readonly ID_FILE="${DATA_DIR}/.next_id"

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BLUE='\033[0;34m'
MAGENTA='\033[0;35m'
CYAN='\033[0;36m'
DIM='\033[2m'
BOLD='\033[1m'
NC='\033[0m'

# --- Init ---
init_storage() {
    mkdir -p "$DATA_DIR"
    [[ -f "$TASK_FILE" ]] || echo -e "ID\tSTATUS\tPRIORITY\tDUE\tTAGS\tCREATED\tDESCRIPTION" > "$TASK_FILE"
    [[ -f "$ARCHIVE_FILE" ]] || echo -e "ID\tSTATUS\tPRIORITY\tDUE\tTAGS\tCREATED\tCOMPLETED\tDESCRIPTION" > "$ARCHIVE_FILE"
    [[ -f "$ID_FILE" ]] || echo "1" > "$ID_FILE"
}

next_id() {
    local id
    id=$(cat "$ID_FILE")
    echo $((id + 1)) > "$ID_FILE"
    echo "$id"
}

# --- Priority Colors ---
priority_color() {
    case "$1" in
        high)   echo -e "${RED}$1${NC}" ;;
        medium) echo -e "${YELLOW}$1${NC}" ;;
        low)    echo -e "${GREEN}$1${NC}" ;;
        *)      echo "$1" ;;
    esac
}

status_icon() {
    case "$1" in
        todo)      echo "○" ;;
        doing)     echo "◐" ;;
        done)      echo "●" ;;
        *)         echo "?" ;;
    esac
}

# --- Commands ---

cmd_add() {
    local desc="" priority="medium" due="" tags=""
    
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -p|--priority) priority="$2"; shift 2 ;;
            -d|--due)      due="$2"; shift 2 ;;
            -t|--tag)      tags="${tags:+$tags,}$2"; shift 2 ;;
            *)             desc="${desc:+$desc }$1"; shift ;;
        esac
    done
    
    [[ -n "$desc" ]] || { echo "Usage: $SCRIPT_NAME add [-p high|medium|low] [-d date] [-t tag] description" >&2; return 1; }
    
    local id
    id=$(next_id)
    local created
    created=$(date +%Y-%m-%d)
    
    echo -e "${id}\ttodo\t${priority}\t${due:--}\t${tags:--}\t${created}\t${desc}" >> "$TASK_FILE"
    
    echo -e "${GREEN}Added task #${id}:${NC} $desc"
}

cmd_list() {
    local filter_status="" filter_priority="" filter_tag=""
    
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -s|--status)   filter_status="$2"; shift 2 ;;
            -p|--priority) filter_priority="$2"; shift 2 ;;
            -t|--tag)      filter_tag="$2"; shift 2 ;;
            --all)         filter_status="all"; shift ;;
            *)             shift ;;
        esac
    done
    
    echo ""
    echo -e "${BOLD}  Tasks${NC}"
    echo "  ─────────────────────────────────────────────────────────"
    
    local count=0
    
    tail -n +2 "$TASK_FILE" | sort -t$'\t' -k3,3 | while IFS=$'\t' read -r id status priority due tags created desc; do
        # Apply filters
        [[ -n "$filter_status" && "$filter_status" != "all" && "$status" != "$filter_status" ]] && continue
        [[ -n "$filter_priority" && "$priority" != "$filter_priority" ]] && continue
        [[ -n "$filter_tag" && "$tags" != *"$filter_tag"* ]] && continue
        
        local icon
        icon=$(status_icon "$status")
        local pri_display
        pri_display=$(priority_color "$priority")
        
        # Due date coloring
        local due_display="$due"
        if [[ "$due" != "-" ]]; then
            local due_epoch today_epoch
            today_epoch=$(date +%s)
            due_epoch=$(date -d "$due" +%s 2>/dev/null || echo "0")
            if (( due_epoch > 0 && due_epoch < today_epoch )); then
                due_display="${RED}${due} OVERDUE${NC}"
            fi
        fi
        
        # Status-based formatting
        if [[ "$status" == "done" ]]; then
            printf "  ${DIM}%s #%-4s [%s] %s${NC}\n" "$icon" "$id" "$priority" "$desc"
        else
            printf "  %s #%-4s " "$icon" "$id"
            printf "[%b] " "$pri_display"
            printf "%s" "$desc"
            [[ "$due" != "-" ]] && printf "  %b" "$(echo -e " 📅 $due_display")"
            [[ "$tags" != "-" ]] && printf "  ${CYAN}[%s]${NC}" "$tags"
            printf "\n"
        fi
        ((count++)) || true
    done
    
    echo ""
    echo -e "  ${DIM}$(tail -n +2 "$TASK_FILE" | wc -l) tasks${NC}"
    echo ""
}

cmd_done() {
    local id="$1"
    [[ -n "$id" ]] || { echo "Usage: $SCRIPT_NAME done <id>" >&2; return 1; }
    
    # Update status to done
    if grep -q "^${id}\t" "$TASK_FILE"; then
        local tmpfile
        tmpfile=$(mktemp)
        awk -v id="$id" 'BEGIN{FS=OFS="\t"} NR==1{print;next} $1==id{$2="done"}{print}' "$TASK_FILE" > "$tmpfile"
        mv "$tmpfile" "$TASK_FILE"
        echo -e "${GREEN}✓ Task #${id} completed!${NC}"
    else
        echo "Task #$id not found" >&2
        return 1
    fi
}

cmd_doing() {
    local id="$1"
    [[ -n "$id" ]] || { echo "Usage: $SCRIPT_NAME doing <id>" >&2; return 1; }
    
    if grep -q "^${id}\t" "$TASK_FILE"; then
        local tmpfile
        tmpfile=$(mktemp)
        awk -v id="$id" 'BEGIN{FS=OFS="\t"} NR==1{print;next} $1==id{$2="doing"}{print}' "$TASK_FILE" > "$tmpfile"
        mv "$tmpfile" "$TASK_FILE"
        echo -e "${YELLOW}◐ Task #${id} in progress${NC}"
    else
        echo "Task #$id not found" >&2
    fi
}

cmd_delete() {
    local id="$1"
    [[ -n "$id" ]] || { echo "Usage: $SCRIPT_NAME delete <id>" >&2; return 1; }
    
    if grep -q "^${id}\t" "$TASK_FILE"; then
        local tmpfile
        tmpfile=$(mktemp)
        awk -v id="$id" 'BEGIN{FS=OFS="\t"} $1!=id{print}' "$TASK_FILE" > "$tmpfile"
        mv "$tmpfile" "$TASK_FILE"
        echo -e "${RED}✗ Task #${id} deleted${NC}"
    else
        echo "Task #$id not found" >&2
    fi
}

cmd_search() {
    local query="$1"
    [[ -n "$query" ]] || { echo "Usage: $SCRIPT_NAME search <query>" >&2; return 1; }
    
    echo -e "\n${BOLD}  Search: $query${NC}\n"
    
    grep -i "$query" "$TASK_FILE" | while IFS=$'\t' read -r id status priority due tags created desc; do
        [[ "$id" == "ID" ]] && continue
        local icon
        icon=$(status_icon "$status")
        printf "  %s #%-4s [%s] %s\n" "$icon" "$id" "$priority" "$desc"
    done
    echo ""
}

cmd_export() {
    local format="${1:-csv}"
    
    case "$format" in
        csv)
            echo "ID,Status,Priority,Due,Tags,Created,Description"
            tail -n +2 "$TASK_FILE" | tr '\t' ','
            ;;
        json)
            echo "["
            local first=true
            tail -n +2 "$TASK_FILE" | while IFS=$'\t' read -r id status priority due tags created desc; do
                $first || echo ","
                first=false
                printf '  {"id":%s,"status":"%s","priority":"%s","due":"%s","tags":"%s","created":"%s","description":"%s"}' \
                    "$id" "$status" "$priority" "$due" "$tags" "$created" "$desc"
            done
            echo ""
            echo "]"
            ;;
        *) echo "Supported formats: csv, json" >&2; return 1 ;;
    esac
}

cmd_stats() {
    echo -e "\n${BOLD}  Task Statistics${NC}\n"
    
    local total todo doing done_count
    total=$(( $(tail -n +2 "$TASK_FILE" | wc -l) ))
    todo=$(grep -c $'\ttodo\t' "$TASK_FILE" || true)
    doing=$(grep -c $'\tdoing\t' "$TASK_FILE" || true)
    done_count=$(grep -c $'\tdone\t' "$TASK_FILE" || true)
    
    printf "  Total:       %d\n" "$total"
    printf "  ○ Todo:      %d\n" "$todo"
    printf "  ◐ In Progress: %d\n" "$doing"
    printf "  ● Done:      %d\n" "$done_count"
    
    if (( total > 0 )); then
        local pct=$(( done_count * 100 / total ))
        printf "  Completion:  %d%%\n" "$pct"
    fi
    
    echo ""
    echo "  By Priority:"
    for p in high medium low; do
        local c
        c=$(grep -c $"\t${p}\t" "$TASK_FILE" || true)
        printf "    %-8s %d\n" "$p:" "$c"
    done
    echo ""
}

# --- Usage ---
usage() {
    cat <<EOF
${BOLD}task${NC} — CLI Task Manager

${BOLD}USAGE:${NC}
    $SCRIPT_NAME <command> [options]

${BOLD}COMMANDS:${NC}
    add <desc> [-p priority] [-d due_date] [-t tag]
                            Add a new task
    list [-s status] [-p priority] [-t tag] [--all]
                            List tasks
    done <id>               Mark task as complete
    doing <id>              Mark task as in-progress
    delete <id>             Delete a task
    search <query>          Search tasks
    export [csv|json]       Export tasks
    stats                   Show statistics

${BOLD}EXAMPLES:${NC}
    $SCRIPT_NAME add "Write documentation" -p high -d 2024-02-01 -t work
    $SCRIPT_NAME list -p high
    $SCRIPT_NAME done 5
    $SCRIPT_NAME search "documentation"
    $SCRIPT_NAME export json > tasks.json
EOF
}

# --- Main ---
init_storage

case "${1:-}" in
    add)     shift; cmd_add "$@" ;;
    list|ls) shift; cmd_list "$@" ;;
    done)    shift; cmd_done "${1:-}" ;;
    doing)   shift; cmd_doing "${1:-}" ;;
    delete|rm) shift; cmd_delete "${1:-}" ;;
    search)  shift; cmd_search "${1:-}" ;;
    export)  shift; cmd_export "${1:-csv}" ;;
    stats)   cmd_stats ;;
    help|-h|--help) usage ;;
    "")      cmd_list ;;
    *)       echo "Unknown command: $1" >&2; usage; exit 1 ;;
esac
```

---

## Usage Examples

```bash
# Add tasks
./task.sh add "Write project proposal" -p high -d 2024-03-01 -t work
./task.sh add "Buy groceries" -p low -t personal
./task.sh add "Review pull request #42" -p medium -t work -d 2024-02-15

# List tasks
./task.sh list
./task.sh list -p high
./task.sh list -t work

# Update status
./task.sh doing 1
./task.sh done 2

# Search
./task.sh search "proposal"

# Statistics
./task.sh stats

# Export
./task.sh export json > tasks.json
./task.sh export csv > tasks.csv
```

---

## Extensions

1. Add subtasks support
2. Add recurring tasks
3. Add time tracking (start/stop)
4. Sync with cloud storage (Dropbox, Google Drive)
5. Interactive TUI mode using dialog/whiptail
6. Integration with git hooks (auto-close tasks on commit)

---

**Next:** [Appendix A: Bash Quick Reference →](../Appendices/AppendixA-Quick-Reference.md)
