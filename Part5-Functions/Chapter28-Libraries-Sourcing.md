# Chapter 28: Libraries and Sourcing

## Learning Objectives

By the end of this chapter, you will be able to:

- Source external files to reuse functions and variables
- Build reusable Bash libraries
- Understand the difference between sourcing and executing
- Organize code across multiple files
- Implement include guards and library patterns

---

## 28.1 Sourcing Files

The `source` command (or `.`) reads and executes a file in the **current** shell:

```bash
# These are equivalent:
source ./lib/utils.sh
. ./lib/utils.sh
```

```
Executing (./script.sh):         Sourcing (source script.sh):
┌──────────────────────┐         ┌──────────────────────┐
│ Parent Shell         │         │ Current Shell         │
│                      │         │                      │
│ ┌──────────────────┐ │         │ Variables from file   │
│ │ Child Process    │ │         │ are set HERE          │
│ │ (separate env)   │ │         │                      │
│ │ vars stay here   │ │         │ Functions defined     │
│ └──────────────────┘ │         │ are available HERE    │
│                      │         │                      │
│ Vars NOT available   │         │ Everything merges     │
│ in parent            │         │ into current shell    │
└──────────────────────┘         └──────────────────────┘
```

```bash
# lib/config.sh
APP_NAME="MyApp"
APP_VERSION="2.0"
log() { echo "[$APP_NAME] $*"; }

# main.sh
#!/bin/bash
source ./lib/config.sh
log "Starting version $APP_VERSION"
# [MyApp] Starting version 2.0
```

---

## 28.2 Building a Library

### Library File: lib/utils.sh

```bash
#!/bin/bash
# lib/utils.sh — Common utility functions

# Prevent double-sourcing
[[ -n "${_UTILS_SH_LOADED:-}" ]] && return 0
readonly _UTILS_SH_LOADED=1

# --- Logging ---
log()   { printf '[%s] %s\n' "$(date +%T)" "$*"; }
warn()  { printf '[%s] WARN: %s\n' "$(date +%T)" "$*" >&2; }
error() { printf '[%s] ERROR: %s\n' "$(date +%T)" "$*" >&2; }
die()   { error "$@"; exit 1; }

# --- String Utilities ---
to_upper() { echo "${1^^}"; }
to_lower() { echo "${1,,}"; }
trim()     { local s="$1"; s="${s#"${s%%[![:space:]]*}"}"; s="${s%"${s##*[![:space:]]}"}"; echo "$s"; }

# --- Validation ---
is_integer()  { [[ "$1" =~ ^-?[0-9]+$ ]]; }
is_empty()    { [[ -z "$1" ]]; }
file_exists() { [[ -f "$1" ]]; }
```

### Using the Library

```bash
#!/bin/bash
# main.sh

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/lib/utils.sh"

log "Script started"

if is_integer "$1"; then
    log "Processing number: $1"
else
    die "Expected integer, got: $1"
fi
```

---

## 28.3 Reliable Source Paths

The biggest challenge with sourcing: finding the file reliably.

```bash
# BAD — depends on working directory
source lib/utils.sh              # Breaks if run from another directory

# GOOD — relative to script location
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/lib/utils.sh"

# GOOD — with error handling
source_lib() {
    local lib="$1"
    if [[ -f "$lib" ]]; then
        source "$lib"
    else
        echo "ERROR: Library not found: $lib" >&2
        exit 1
    fi
}

source_lib "$SCRIPT_DIR/lib/utils.sh"
source_lib "$SCRIPT_DIR/lib/config.sh"
```

---

## 28.4 Include Guards

Prevent a library from being loaded multiple times:

```bash
# lib/database.sh
# Include guard
[[ -n "${_DATABASE_SH:-}" ]] && return 0
readonly _DATABASE_SH=1

# Library code...
db_connect() { ... ; }
db_query() { ... ; }
```

Why? If `main.sh` sources `a.sh` and `b.sh`, and both source `utils.sh`, without guards `utils.sh` runs twice.

---

## 28.5 Project Structure

```
myproject/
├── bin/
│   └── myapp                # Main entry point
├── lib/
│   ├── utils.sh             # Common utilities
│   ├── config.sh            # Configuration handling
│   ├── database.sh          # Database functions
│   └── ui.sh                # User interface functions
├── etc/
│   └── myapp.conf           # Configuration file
├── tests/
│   ├── test_utils.sh        # Tests for utils
│   └── test_database.sh     # Tests for database
└── README.md
```

```bash
#!/bin/bash
# bin/myapp — Main entry point

set -euo pipefail

readonly APP_DIR="$(cd "$(dirname "$0")/.." && pwd)"
readonly LIB_DIR="$APP_DIR/lib"
readonly ETC_DIR="$APP_DIR/etc"

# Load libraries
source "$LIB_DIR/utils.sh"
source "$LIB_DIR/config.sh"
source "$LIB_DIR/database.sh"

main() {
    load_config "$ETC_DIR/myapp.conf"
    db_connect
    # ... application logic ...
}

main "$@"
```

---

## 28.6 Configuration Files

```bash
# etc/myapp.conf
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp
LOG_LEVEL=info
MAX_RETRIES=3
```

```bash
# lib/config.sh
[[ -n "${_CONFIG_SH:-}" ]] && return 0
readonly _CONFIG_SH=1

load_config() {
    local config_file="$1"
    
    [[ -f "$config_file" ]] || die "Config not found: $config_file"
    
    # Validate: only KEY=VALUE lines (security!)
    while IFS= read -r line; do
        # Skip comments and empty lines
        [[ "$line" =~ ^[[:space:]]*# ]] && continue
        [[ -z "$line" ]] && continue
        
        # Validate format
        if [[ "$line" =~ ^[A-Za-z_][A-Za-z0-9_]*= ]]; then
            eval "$line"    # Or use declare: declare -g "$line"
        else
            warn "Invalid config line: $line"
        fi
    done < "$config_file"
}
```

> **Security Note:** Never blindly `source` a config file — it could contain arbitrary commands. Parse and validate line by line.

---

## 28.7 Common Mistakes

### Mistake 1: Sourcing with Relative Path

```bash
source lib/utils.sh    # Only works if you cd to project root first

# Fix:
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/lib/utils.sh"
```

### Mistake 2: Not Checking if Source Succeeded

```bash
source "$LIB/utils.sh"    # Silently fails if file doesn't exist

# Fix:
source "$LIB/utils.sh" || { echo "Failed to load utils" >&2; exit 1; }
```

---

## Exercises

### Exercise 28.1: Build a Library
Create a library `lib/strings.sh` with functions: `str_repeat` (repeat a string N times), `str_pad` (pad string to width), `str_contains`, `str_starts_with`, `str_ends_with`. Write a main script that sources and uses them.

### Exercise 28.2: Multi-File Project
Create a simple log analyzer with separate library files for parsing, filtering, and reporting.

---

## Summary

- `source file.sh` (or `. file.sh`) executes a file in the current shell
- Sourced files share variables and functions with the current shell
- Always use `SCRIPT_DIR`-relative paths for reliable sourcing
- Use include guards to prevent double-loading
- Organize large projects into `bin/`, `lib/`, `etc/`, `tests/`
- Never blindly source config files — validate content for security

---

**Next Chapter:** [Chapter 29: Traps and Signals →](Chapter29-Traps-Signals.md)
