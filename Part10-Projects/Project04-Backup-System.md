# Project 4: Backup System

## Project Overview

Build a comprehensive backup system with full and incremental backups, retention policies, remote sync, and restore capabilities.

---

## Requirements

- Full and incremental backup support
- Compression with configurable algorithms
- Retention policy (keep N daily, N weekly, N monthly)
- Remote backup via rsync/SSH
- Restore from any backup point
- Email notifications on failure
- Lock file to prevent concurrent runs

---

## Complete Implementation

```bash
#!/usr/bin/env bash
#
# backup.sh — Comprehensive Backup System
#
# Usage: backup.sh [-c config] [-t full|incremental] [-r backup_id] [-l]

set -euo pipefail

readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
readonly TIMESTAMP="$(date +%Y%m%d_%H%M%S)"

# --- Defaults ---
BACKUP_DIR="/backup"
SOURCE_DIRS=()
EXCLUDE_PATTERNS=()
COMPRESS="gzip"       # gzip, bzip2, xz, none
RETENTION_DAILY=7
RETENTION_WEEKLY=4
RETENTION_MONTHLY=6
REMOTE_HOST=""
REMOTE_PATH=""
LOCK_FILE="/tmp/backup.lock"
LOG_FILE="/var/log/backup.log"
NOTIFY_EMAIL=""
BACKUP_TYPE="full"

# --- Logging ---
log() { printf '[%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*" | tee -a "$LOG_FILE"; }
error() { log "ERROR: $*" >&2; }
die() { error "$@"; notify_failure "$*"; exit 1; }

# --- Locking ---
acquire_lock() {
    if ! mkdir "$LOCK_FILE" 2>/dev/null; then
        die "Another backup is running (lock: $LOCK_FILE)"
    fi
    trap 'rm -rf "$LOCK_FILE"' EXIT
}

# --- Notification ---
notify_failure() {
    local msg="$1"
    if [[ -n "$NOTIFY_EMAIL" ]] && command -v mail >/dev/null 2>&1; then
        echo "Backup failed on $(hostname) at $(date): $msg" | \
            mail -s "BACKUP FAILURE on $(hostname)" "$NOTIFY_EMAIL"
    fi
}

# --- Config ---
load_config() {
    local config="$1"
    [[ -f "$config" ]] || die "Config not found: $config"
    # shellcheck source=/dev/null
    source "$config"
    log "Loaded config: $config"
}

# --- Compression ---
get_compress_ext() {
    case "$COMPRESS" in
        gzip)  echo ".tar.gz" ;;
        bzip2) echo ".tar.bz2" ;;
        xz)    echo ".tar.xz" ;;
        none)  echo ".tar" ;;
        *)     die "Unknown compression: $COMPRESS" ;;
    esac
}

get_compress_flag() {
    case "$COMPRESS" in
        gzip)  echo "z" ;;
        bzip2) echo "j" ;;
        xz)    echo "J" ;;
        none)  echo "" ;;
    esac
}

# --- Backup ---
do_full_backup() {
    local backup_name="full_${TIMESTAMP}"
    local ext
    ext=$(get_compress_ext)
    local backup_file="${BACKUP_DIR}/${backup_name}${ext}"
    local flag
    flag=$(get_compress_flag)
    
    log "Starting FULL backup: $backup_name"
    mkdir -p "$BACKUP_DIR"
    
    # Build exclude args
    local exclude_args=()
    for pattern in "${EXCLUDE_PATTERNS[@]}"; do
        exclude_args+=("--exclude=$pattern")
    done
    
    # Create archive
    tar -c${flag}f "$backup_file" \
        "${exclude_args[@]}" \
        "${SOURCE_DIRS[@]}" 2>/dev/null || die "tar failed"
    
    # Record snapshot for incremental reference
    echo "$TIMESTAMP" > "${BACKUP_DIR}/last_full_timestamp"
    local size
    size=$(du -sh "$backup_file" | cut -f1)
    
    log "Full backup complete: $backup_file ($size)"
    
    # Sync to remote
    [[ -n "$REMOTE_HOST" ]] && sync_remote "$backup_file"
}

do_incremental_backup() {
    local snapshot_file="${BACKUP_DIR}/last_full_timestamp"
    
    if [[ ! -f "$snapshot_file" ]]; then
        log "No full backup found. Running full backup instead."
        do_full_backup
        return
    fi
    
    local last_full
    last_full=$(cat "$snapshot_file")
    local backup_name="incr_${TIMESTAMP}"
    local ext
    ext=$(get_compress_ext)
    local backup_file="${BACKUP_DIR}/${backup_name}${ext}"
    local flag
    flag=$(get_compress_flag)
    
    log "Starting INCREMENTAL backup since: $last_full"
    mkdir -p "$BACKUP_DIR"
    
    # Find files newer than the snapshot marker
    local newer_file="${BACKUP_DIR}/.snapshot_marker"
    touch -t "$(echo "$last_full" | sed 's/_//;s/\(..\)$/.\1/')" "$newer_file"
    
    local exclude_args=()
    for pattern in "${EXCLUDE_PATTERNS[@]}"; do
        exclude_args+=("--exclude=$pattern")
    done
    
    # Create incremental archive using --newer
    tar -c${flag}f "$backup_file" \
        --newer-mtime="@$(stat -c %Y "$newer_file" 2>/dev/null || stat -f %m "$newer_file")" \
        "${exclude_args[@]}" \
        "${SOURCE_DIRS[@]}" 2>/dev/null || true
    
    local size
    size=$(du -sh "$backup_file" | cut -f1)
    log "Incremental backup complete: $backup_file ($size)"
    
    rm -f "$newer_file"
    [[ -n "$REMOTE_HOST" ]] && sync_remote "$backup_file"
}

# --- Remote Sync ---
sync_remote() {
    local file="$1"
    [[ -n "$REMOTE_HOST" && -n "$REMOTE_PATH" ]] || return 0
    
    log "Syncing to remote: ${REMOTE_HOST}:${REMOTE_PATH}"
    rsync -avz "$file" "${REMOTE_HOST}:${REMOTE_PATH}/" || \
        error "Remote sync failed (continuing)"
    log "Remote sync complete"
}

# --- Restore ---
do_restore() {
    local backup_id="$1"
    local restore_dir="${2:-/tmp/restore_${TIMESTAMP}}"
    
    # Find backup file
    local backup_file
    backup_file=$(find "$BACKUP_DIR" -name "${backup_id}*" -type f | head -1)
    [[ -n "$backup_file" ]] || die "Backup not found: $backup_id"
    
    log "Restoring from: $backup_file"
    log "Restore target: $restore_dir"
    mkdir -p "$restore_dir"
    
    local flag
    flag=$(get_compress_flag)
    tar -x${flag}f "$backup_file" -C "$restore_dir"
    
    log "Restore complete: $restore_dir"
}

# --- Retention ---
apply_retention() {
    log "Applying retention policy..."
    local now
    now=$(date +%s)
    
    # Remove daily backups older than retention period
    find "$BACKUP_DIR" -name "*.tar*" -type f -mtime +"$RETENTION_DAILY" | \
    while read -r file; do
        local basename
        basename=$(basename "$file")
        
        # Keep weekly (every Monday's backup)
        local file_date
        file_date=$(stat -c %Y "$file" 2>/dev/null || stat -f %m "$file")
        local day_of_week
        day_of_week=$(date -d "@$file_date" +%u 2>/dev/null || date -r "$file_date" +%u)
        
        local file_age=$(( (now - file_date) / 86400 ))
        
        # Keep if it's a Monday backup within weekly retention
        if [[ "$day_of_week" == "1" ]] && (( file_age <= RETENTION_WEEKLY * 7 )); then
            continue
        fi
        
        # Keep monthly (1st of month) within monthly retention
        local day_of_month
        day_of_month=$(date -d "@$file_date" +%d 2>/dev/null || date -r "$file_date" +%d)
        if [[ "$day_of_month" == "01" ]] && (( file_age <= RETENTION_MONTHLY * 30 )); then
            continue
        fi
        
        log "Removing old backup: $basename"
        rm -f "$file"
    done
    
    log "Retention policy applied"
}

# --- List Backups ---
list_backups() {
    echo "=== Available Backups ==="
    printf "  %-30s %8s  %s\n" "BACKUP" "SIZE" "DATE"
    printf "  %-30s %8s  %s\n" "------------------------------" "--------" "-------------------"
    
    find "$BACKUP_DIR" -name "*.tar*" -type f | sort | \
    while read -r file; do
        local name size date
        name=$(basename "$file")
        size=$(du -sh "$file" | cut -f1)
        date=$(stat -c '%y' "$file" 2>/dev/null | cut -d'.' -f1 || \
               stat -f '%Sm' "$file" 2>/dev/null)
        printf "  %-30s %8s  %s\n" "$name" "$size" "$date"
    done
}

# --- Main ---
usage() {
    cat <<EOF
Usage: $SCRIPT_NAME [OPTIONS]

Options:
    -c FILE          Load configuration file
    -t TYPE          Backup type: full, incremental (default: full)
    -r BACKUP_ID     Restore from backup
    -d DIR           Restore destination directory
    -l               List available backups
    -p               Apply retention policy
    -h               Show help

Config file example:
    SOURCE_DIRS=(/home /etc /var/www)
    EXCLUDE_PATTERNS=("*.tmp" "*.cache" "node_modules")
    BACKUP_DIR=/backup
    COMPRESS=gzip
    RETENTION_DAILY=7
EOF
}

RESTORE_ID=""
RESTORE_DIR=""
ACTION="backup"

while getopts ":c:t:r:d:lph" opt; do
    case $opt in
        c) load_config "$OPTARG" ;;
        t) BACKUP_TYPE="$OPTARG" ;;
        r) RESTORE_ID="$OPTARG"; ACTION="restore" ;;
        d) RESTORE_DIR="$OPTARG" ;;
        l) ACTION="list" ;;
        p) ACTION="retention" ;;
        h) usage; exit 0 ;;
        *) usage; exit 1 ;;
    esac
done

case "$ACTION" in
    backup)
        acquire_lock
        log "=== Backup started ==="
        case "$BACKUP_TYPE" in
            full)        do_full_backup ;;
            incremental) do_incremental_backup ;;
            *) die "Unknown backup type: $BACKUP_TYPE" ;;
        esac
        apply_retention
        log "=== Backup finished ==="
        ;;
    restore)
        do_restore "$RESTORE_ID" "$RESTORE_DIR"
        ;;
    list)
        list_backups
        ;;
    retention)
        apply_retention
        ;;
esac
```

---

## Sample Configuration

```bash
# /etc/backup.conf
SOURCE_DIRS=(/home /etc /var/www /opt/apps)
EXCLUDE_PATTERNS=("*.tmp" "*.cache" "*.log" "node_modules" ".git")
BACKUP_DIR="/mnt/backup/server1"
COMPRESS="gzip"
RETENTION_DAILY=7
RETENTION_WEEKLY=4
RETENTION_MONTHLY=12
REMOTE_HOST="backup-server"
REMOTE_PATH="/backup/server1"
NOTIFY_EMAIL="admin@example.com"
LOG_FILE="/var/log/backup.log"
```

## Cron Setup

```bash
# Full backup every Sunday at 2 AM
0 2 * * 0  /opt/scripts/backup.sh -c /etc/backup.conf -t full

# Incremental backup Mon-Sat at 2 AM
0 2 * * 1-6  /opt/scripts/backup.sh -c /etc/backup.conf -t incremental
```

---

**Next Project:** [Project 5: Deployment Script →](Project05-Deployment.md)
