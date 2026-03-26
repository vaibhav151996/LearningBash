# Chapter 24: Error Handling Patterns

## Learning Objectives

By the end of this chapter, you will be able to:

- Implement robust error handling in scripts
- Use `set -e` for automatic error detection
- Create error-handling functions
- Implement retry logic for unreliable operations
- Build cleanup handlers with `trap`

---

## 24.1 The Die Pattern

The simplest and most common error-handling pattern:

```bash
#!/bin/bash

die() {
    echo "ERROR: $*" >&2
    exit 1
}

# Usage
[[ -f "$config" ]] || die "Config file not found: $config"
cd "$workdir" || die "Cannot change to directory: $workdir"
mkdir -p "$output" || die "Cannot create output directory: $output"
```

---

## 24.2 set -e — Exit on Error

```bash
#!/bin/bash
set -e    # Exit immediately if any command fails

mkdir /tmp/workspace
cd /tmp/workspace
cp important_file.txt .      # If this fails, script stops HERE
process_file important_file.txt
```

### Caveats of set -e

```bash
set -e

# These do NOT cause exit (by design):
false || echo "This is fine"                       # Part of || chain
false && echo "Not reached"                         # Part of && chain
if false; then echo "no"; fi                        # Part of if condition
while false; do echo "no"; done                     # Part of loop condition

# This DOES cause exit:
false                    # Standalone failing command — script exits

# Command in a subshell: depends
(false)                  # Exits the subshell, but may or may not exit parent
```

---

## 24.3 The Strict Mode

Combine multiple safety options:

```bash
#!/bin/bash
set -euo pipefail

# -e: Exit on error
# -u: Error on undefined variables
# -o pipefail: Pipeline fails if any command fails
```

```bash
set -u    # Treat unset variables as errors
echo $undefined_var    # ERROR: undefined_var: unbound variable

set -o pipefail
false | true
echo $?    # 1 (without pipefail, this would be 0)
```

---

## 24.4 Retry Logic

```bash
retry() {
    local max_attempts="$1"
    local delay="$2"
    shift 2
    local cmd=("$@")
    
    local attempt=1
    while (( attempt <= max_attempts )); do
        echo "Attempt $attempt of $max_attempts: ${cmd[*]}"
        if "${cmd[@]}"; then
            return 0
        fi
        echo "Failed. Retrying in ${delay}s..."
        sleep "$delay"
        ((attempt++))
    done
    
    echo "All $max_attempts attempts failed" >&2
    return 1
}

# Usage
retry 3 5 curl -sf https://api.example.com/health
retry 5 2 pg_isready -h localhost
```

---

## 24.5 Cleanup with trap

`trap` registers commands to run when signals or events occur:

```bash
#!/bin/bash
set -euo pipefail

TMPDIR=""

cleanup() {
    echo "Cleaning up..."
    [[ -n "$TMPDIR" && -d "$TMPDIR" ]] && rm -rf "$TMPDIR"
}

# Run cleanup on EXIT (regardless of success or failure)
trap cleanup EXIT

TMPDIR=$(mktemp -d)
echo "Working in $TMPDIR"

# ... do work in $TMPDIR ...
# If script fails anywhere, cleanup still runs!
```

### Common trap Patterns

```bash
# Cleanup on exit
trap 'rm -f "$tmpfile"' EXIT

# Cleanup on error signals
trap 'echo "Interrupted!" >&2; exit 130' INT TERM

# Log file locking
trap 'rm -f /var/lock/myapp.lock' EXIT

# Combined
trap cleanup EXIT
trap 'echo "Signal received"; exit 1' INT TERM HUP
```

---

## 24.6 Complete Error-Handling Template

```bash
#!/usr/bin/env bash
set -euo pipefail

readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

# Logging
log()   { printf '[%s] %s\n' "$(date '+%H:%M:%S')" "$*"; }
warn()  { printf '[%s] WARNING: %s\n' "$(date '+%H:%M:%S')" "$*" >&2; }
error() { printf '[%s] ERROR: %s\n' "$(date '+%H:%M:%S')" "$*" >&2; }
die()   { error "$@"; exit 1; }

# Cleanup
cleanup() {
    local exit_code=$?
    # ... cleanup resources ...
    exit "$exit_code"
}
trap cleanup EXIT
trap 'die "Interrupted"' INT TERM

# Validation
check_dependencies() {
    local deps=("$@")
    for dep in "${deps[@]}"; do
        command -v "$dep" > /dev/null 2>&1 \
            || die "Required command not found: $dep"
    done
}

check_dependencies curl jq gzip

# Main
main() {
    log "Starting $SCRIPT_NAME"
    # ... main logic ...
    log "Complete"
}

main "$@"
```

---

## Exercises

### Exercise 24.1: Robust Script
Write a script that creates a temp directory, does some work, and cleans up even on failure.

### Exercise 24.2: Retry Wrapper
Write a function that retries a command with exponential backoff (1s, 2s, 4s, 8s...).

---

## Summary

- Use `die()` for fatal errors with messages and non-zero exits
- `set -euo pipefail` is the "strict mode" — catches most errors
- `trap cleanup EXIT` ensures cleanup runs regardless of how the script ends
- Retry logic handles transient failures in unreliable operations
- Always send error messages to **stderr** (`>&2`)
- Combine logging, validation, and cleanup for production-grade scripts

---

**Next Chapter:** [Chapter 25: Defensive Scripting Techniques →](Chapter25-Defensive-Scripting.md)
