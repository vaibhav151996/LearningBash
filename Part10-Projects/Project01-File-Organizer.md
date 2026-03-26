# Project 1: File Organizer

## Project Overview

Build a script that automatically organizes files in a directory by sorting them into subdirectories based on file type, date, or custom rules.

---

## Requirements

- Organize files by extension (Documents, Images, Videos, Archives, etc.)
- Support a dry-run mode to preview changes
- Handle filename conflicts (append number)
- Support undo (generate a reverse script)
- Log all operations
- Accept custom rules via config file

---

## Complete Implementation

```bash
#!/usr/bin/env bash
#
# organize.sh — Organize files into categorized directories
#
# Usage: organize.sh [-n] [-u] [-c config] [-l logfile] <directory>

set -euo pipefail

readonly SCRIPT_NAME="$(basename "$0")"
readonly VERSION="1.0.0"

# Defaults
DRY_RUN=false
UNDO_FILE=""
LOG_FILE=""
CONFIG_FILE=""

# --- Logging ---
log() { printf '[%s] %s\n' "$(date '+%H:%M:%S')" "$*"; }
info() { log "INFO: $*"; [[ -n "$LOG_FILE" ]] && log "INFO: $*" >> "$LOG_FILE"; }
warn() { log "WARN: $*" >&2; }

# --- Category Mapping ---
declare -A CATEGORIES=(
    # Documents
    [pdf]=Documents [doc]=Documents [docx]=Documents [txt]=Documents
    [odt]=Documents [rtf]=Documents [xlsx]=Documents [csv]=Documents
    [pptx]=Documents [md]=Documents
    
    # Images
    [jpg]=Images [jpeg]=Images [png]=Images [gif]=Images
    [bmp]=Images [svg]=Images [webp]=Images [ico]=Images [tiff]=Images
    
    # Videos
    [mp4]=Videos [mkv]=Videos [avi]=Videos [mov]=Videos
    [wmv]=Videos [flv]=Videos [webm]=Videos
    
    # Audio
    [mp3]=Audio [wav]=Audio [flac]=Audio [aac]=Audio
    [ogg]=Audio [wma]=Audio [m4a]=Audio
    
    # Archives
    [zip]=Archives [tar]=Archives [gz]=Archives [bz2]=Archives
    [xz]=Archives [7z]=Archives [rar]=Archives
    
    # Code
    [sh]=Code [py]=Code [js]=Code [ts]=Code [html]=Code
    [css]=Code [java]=Code [c]=Code [cpp]=Code [go]=Code
    [rs]=Code [rb]=Code [php]=Code
    
    # Data
    [json]=Data [xml]=Data [yaml]=Data [yml]=Data
    [sql]=Data [db]=Data [sqlite]=Data
)

# --- Usage ---
usage() {
    cat <<EOF
Usage: $SCRIPT_NAME [OPTIONS] <directory>

Organize files into categorized subdirectories.

Options:
    -n          Dry run (preview without moving)
    -u FILE     Generate undo script
    -c FILE     Load custom category config
    -l FILE     Log operations to file
    -h          Show this help
    -V          Show version

Categories: Documents, Images, Videos, Audio, Archives, Code, Data
Uncategorized files go to "Other/".

Examples:
    $SCRIPT_NAME ~/Downloads
    $SCRIPT_NAME -n ~/Downloads          # Preview only
    $SCRIPT_NAME -u undo.sh ~/Downloads  # With undo script
EOF
}

# --- Load Custom Config ---
load_config() {
    local config="$1"
    [[ -f "$config" ]] || { warn "Config not found: $config"; return 1; }
    
    while IFS='=' read -r ext category; do
        [[ "$ext" =~ ^[[:space:]]*# ]] && continue
        [[ -z "$ext" ]] && continue
        ext=$(echo "$ext" | xargs)
        category=$(echo "$category" | xargs)
        CATEGORIES[$ext]="$category"
    done < "$config"
    info "Loaded config: $config"
}

# --- Get Unique Filename ---
get_unique_path() {
    local dest="$1"
    if [[ ! -e "$dest" ]]; then
        echo "$dest"
        return
    fi
    
    local dir base ext counter
    dir="$(dirname "$dest")"
    base="$(basename "$dest")"
    ext="${base##*.}"
    base="${base%.*}"
    
    counter=1
    while [[ -e "$dir/${base}_${counter}.${ext}" ]]; do
        ((counter++))
    done
    echo "$dir/${base}_${counter}.${ext}"
}

# --- Move File ---
move_file() {
    local src="$1" dest_dir="$2"
    local filename dest
    
    filename="$(basename "$src")"
    dest=$(get_unique_path "$dest_dir/$filename")
    
    if [[ "$DRY_RUN" == true ]]; then
        log "[DRY RUN] $src -> $dest"
        return
    fi
    
    mkdir -p "$dest_dir"
    mv "$src" "$dest"
    info "Moved: $src -> $dest"
    
    # Write undo command
    if [[ -n "$UNDO_FILE" ]]; then
        echo "mv $(printf '%q' "$dest") $(printf '%q' "$src")" >> "$UNDO_FILE"
    fi
}

# --- Organize ---
organize() {
    local target_dir="$1"
    local moved=0 skipped=0
    
    [[ -d "$target_dir" ]] || { warn "Not a directory: $target_dir"; return 1; }
    
    info "Organizing: $target_dir"
    [[ "$DRY_RUN" == true ]] && info "DRY RUN — no files will be moved"
    
    # Setup undo file
    if [[ -n "$UNDO_FILE" && "$DRY_RUN" == false ]]; then
        echo "#!/bin/bash" > "$UNDO_FILE"
        echo "# Undo script generated $(date)" >> "$UNDO_FILE"
        echo "set -euo pipefail" >> "$UNDO_FILE"
        chmod +x "$UNDO_FILE"
    fi
    
    shopt -s nullglob
    for file in "$target_dir"/*; do
        # Skip directories
        [[ -f "$file" ]] || continue
        
        # Get extension (lowercase)
        local ext="${file##*.}"
        ext="${ext,,}"
        
        # Skip files without extensions
        if [[ "$ext" == "$(basename "$file")" ]]; then
            ext=""
        fi
        
        # Determine category
        local category="${CATEGORIES[$ext]:-Other}"
        
        move_file "$file" "$target_dir/$category"
        ((moved++))
    done
    shopt -u nullglob
    
    info "Done: $moved files organized"
}

# --- Parse Arguments ---
parse_args() {
    while getopts ":hVnu:c:l:" opt; do
        case $opt in
            h) usage; exit 0 ;;
            V) echo "$VERSION"; exit 0 ;;
            n) DRY_RUN=true ;;
            u) UNDO_FILE="$OPTARG" ;;
            c) CONFIG_FILE="$OPTARG" ;;
            l) LOG_FILE="$OPTARG" ;;
            :) warn "Option -$OPTARG requires argument"; usage; exit 1 ;;
            ?) warn "Unknown option: -$OPTARG"; usage; exit 1 ;;
        esac
    done
    shift $((OPTIND - 1))
    
    (( $# >= 1 )) || { warn "Missing directory argument"; usage; exit 1; }
    echo "$1"
}

# --- Main ---
main() {
    local target_dir
    target_dir=$(parse_args "$@")
    
    [[ -n "$CONFIG_FILE" ]] && load_config "$CONFIG_FILE"
    
    organize "$target_dir"
}

main "$@"
```

---

## Testing

```bash
# Create test files
mkdir -p /tmp/test_organize
cd /tmp/test_organize
touch photo.jpg document.pdf script.sh data.json video.mp4 song.mp3 archive.zip unknown.xyz

# Dry run
./organize.sh -n /tmp/test_organize

# Real run with undo
./organize.sh -u undo.sh /tmp/test_organize

# Undo
./undo.sh
```

---

## Extensions

1. Add recursive mode to organize subdirectories
2. Organize by date (year/month directories)
3. Add file size categories (small, medium, large)
4. Support regex-based rules in the config file
5. Add interactive mode asking for confirmation per file

---

**Next Project:** [Project 2: System Monitor Dashboard →](Project02-System-Monitor.md)
