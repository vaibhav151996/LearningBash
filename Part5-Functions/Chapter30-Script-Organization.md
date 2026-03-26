# Chapter 30: Script Organization

## Learning Objectives

By the end of this chapter, you will be able to:

- Structure scripts for readability and maintainability
- Implement the `main()` function pattern
- Organize code into logical sections
- Build modular, testable scripts
- Apply professional script templates

---

## 30.1 The main() Pattern

The most important organizational pattern in Bash:

```bash
#!/usr/bin/env bash
set -euo pipefail

# --- Constants ---
readonly VERSION="1.0.0"

# --- Functions ---
usage() {
    cat <<EOF
Usage: $(basename "$0") [options] <target>
Options:
    -h    Show help
    -v    Verbose mode
    -V    Show version
EOF
}

parse_args() {
    while getopts ":hvV" opt; do
        case $opt in
            h) usage; exit 0 ;;
            v) VERBOSE=true ;;
            V) echo "$VERSION"; exit 0 ;;
            *) usage >&2; exit 1 ;;
        esac
    done
    shift $((OPTIND - 1))
    TARGET="${1:?Missing target argument}"
}

# --- Main ---
main() {
    parse_args "$@"
    # ... main logic ...
}

main "$@"
```

### Why main()?

```bash
# Without main() — functions must be defined before use:
helper()   { ...; }
process()  { helper; }    # helper must be above
run()      { process; }   # process must be above
run                       # Must be at the bottom

# With main() — order doesn't matter:
main()     { run; }       # Can call functions defined below
run()      { process; }
process()  { helper; }
helper()   { ...; }
main "$@"                 # Execute last, after all definitions
```

---

## 30.2 Script Template

```bash
#!/usr/bin/env bash
#
# script-name — Brief description of what this script does
#
# Usage:
#   script-name [options] <arguments>
#
# Author: Your Name
# Date:   2024-01-01

set -euo pipefail

# ============================================================
# Constants
# ============================================================
readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
readonly VERSION="1.0.0"

VERBOSE=false
DRY_RUN=false

# ============================================================
# Logging
# ============================================================
log()   { printf '[%s] %s\n' "$(date +%T)" "$*"; }
debug() { [[ "$VERBOSE" == true ]] && log "DEBUG: $*" || true; }
warn()  { log "WARN: $*" >&2; }
error() { log "ERROR: $*" >&2; }
die()   { error "$@"; exit 1; }

# ============================================================
# Cleanup
# ============================================================
cleanup() {
    local exit_code=$?
    # ... cleanup logic ...
    exit "$exit_code"
}
trap cleanup EXIT
trap 'die "Interrupted"' INT TERM

# ============================================================
# Functions
# ============================================================
usage() {
    cat <<EOF
Usage: $SCRIPT_NAME [options] <target>

Description:
    Brief description of what the script does.

Options:
    -h          Show this help message
    -v          Enable verbose output
    -n          Dry run (show what would be done)
    -V          Show version

Examples:
    $SCRIPT_NAME -v /path/to/target
    $SCRIPT_NAME -n myfile.txt
EOF
}

parse_args() {
    while getopts ":hvnV" opt; do
        case $opt in
            h) usage; exit 0 ;;
            v) VERBOSE=true ;;
            n) DRY_RUN=true ;;
            V) echo "$SCRIPT_NAME version $VERSION"; exit 0 ;;
            :) die "Option -$OPTARG requires an argument" ;;
            ?) die "Unknown option: -$OPTARG" ;;
        esac
    done
    shift $((OPTIND - 1))
    
    # Validate required arguments
    (( $# >= 1 )) || { usage >&2; exit 1; }
    readonly TARGET="$1"
}

validate() {
    command -v required_tool >/dev/null 2>&1 \
        || die "required_tool not found"
    [[ -e "$TARGET" ]] || die "Target not found: $TARGET"
}

process() {
    local target="$1"
    debug "Processing: $target"
    
    if [[ "$DRY_RUN" == true ]]; then
        log "[DRY RUN] Would process: $target"
        return 0
    fi
    
    # ... actual processing ...
}

# ============================================================
# Main
# ============================================================
main() {
    parse_args "$@"
    validate
    log "Starting $SCRIPT_NAME v$VERSION"
    process "$TARGET"
    log "Done"
}

main "$@"
```

---

## 30.3 Section Organization

Order sections logically:

```
┌─────────────────────────────────────────┐
│ 1. Shebang & Header Comment            │
│ 2. set options (set -euo pipefail)      │
│ 3. Constants (readonly)                 │
│ 4. Global variables                     │
│ 5. Logging functions                    │
│ 6. Cleanup / trap setup                 │
│ 7. Helper functions                     │
│ 8. Core logic functions                 │
│ 9. Argument parsing                     │
│ 10. main() function                     │
│ 11. main "$@" invocation                │
└─────────────────────────────────────────┘
```

---

## 30.4 Modular Design

Break large scripts into sourced modules:

```bash
# bin/deploy
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/../lib/logging.sh"
source "$SCRIPT_DIR/../lib/validation.sh"
source "$SCRIPT_DIR/../lib/deploy_helpers.sh"

main() {
    validate_environment
    log "Deploying to $TARGET_ENV"
    build_artifacts
    run_tests
    deploy_to_server
    verify_deployment
    log "Deployment complete"
}

main "$@"
```

Each library focuses on one concern:

```bash
# lib/logging.sh — Only logging
# lib/validation.sh — Only input validation
# lib/deploy_helpers.sh — Only deployment operations
```

---

## 30.5 Making Scripts Testable

```bash
# mylib.sh — Testable functions
add() { echo $(( $1 + $2 )); }
is_valid_email() { [[ "$1" =~ ^[^@]+@[^@]+\.[^@]+$ ]]; }

# Only run main if script is executed directly (not sourced)
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
```

```bash
# test_mylib.sh
#!/bin/bash
source ./mylib.sh

assert_equal() {
    local expected="$1" actual="$2" label="$3"
    if [[ "$expected" == "$actual" ]]; then
        echo "PASS: $label"
    else
        echo "FAIL: $label (expected '$expected', got '$actual')"
    fi
}

assert_equal "5" "$(add 2 3)" "add 2+3"
assert_equal "0" "$(add 0 0)" "add 0+0"

is_valid_email "user@example.com" && echo "PASS: valid email" || echo "FAIL: valid email"
is_valid_email "invalid" && echo "FAIL: invalid email" || echo "PASS: invalid email"
```

---

## 30.6 Common Mistakes

### Mistake 1: No main() Function

```bash
# BAD — code runs as it's parsed
parse_args "$@"
validate
process    # If parse_args is defined below, this fails

# GOOD — everything in main(), called at the end
main() { parse_args "$@"; validate; process; }
main "$@"
```

### Mistake 2: Mixing Concerns

```bash
# BAD — one function does everything
do_everything() {
    parse_config
    connect_to_db
    validate_input
    transform_data
    generate_report
    send_email
}

# GOOD — separate functions, called from main
main() {
    local config data report
    config=$(parse_config "$CONFIG_FILE")
    connect_to_db "$config"
    validate_input "$@"
    data=$(transform_data "$input")
    report=$(generate_report "$data")
    send_email "$report"
}
```

---

## Exercises

### Exercise 30.1: Refactor a Script
Take a monolithic script (50+ lines of sequential commands) and reorganize it using the main() pattern, proper sections, argument parsing, and error handling.

### Exercise 30.2: Multi-File Project
Build a backup utility with: `bin/backup` (main script), `lib/archive.sh` (compression functions), `lib/remote.sh` (upload functions), `lib/notify.sh` (notification functions).

---

## Summary

- Use the `main()` pattern — define functions in any order, call `main "$@"` at the end
- Follow a consistent section order: constants → logging → cleanup → functions → main
- Keep functions focused on a single responsibility
- Use `source` to split large scripts into libraries
- Make scripts testable: guard main execution with `[[ "${BASH_SOURCE[0]}" == "${0}" ]]`
- Include usage help, version, verbose mode, and dry-run as standard features

---

**Next Chapter:** [Chapter 31: The Unix Text Philosophy →](../Part6-Working-With-Data/Chapter31-Text-Philosophy.md)
