# Chapter 25: Defensive Scripting Techniques

## Learning Objectives

By the end of this chapter, you will be able to:

- Validate all inputs before using them
- Handle edge cases gracefully
- Write scripts that fail safely
- Protect against common security pitfalls
- Use defensive patterns for file and directory operations

---

## 25.1 Input Validation

Never trust user input:

```bash
validate_integer() {
    local value="$1" name="$2"
    if [[ ! "$value" =~ ^-?[0-9]+$ ]]; then
        die "$name must be an integer, got: '$value'"
    fi
}

validate_file_exists() {
    local file="$1"
    [[ -f "$file" ]] || die "File not found: $file"
    [[ -r "$file" ]] || die "File not readable: $file"
}

validate_dir_writable() {
    local dir="$1"
    [[ -d "$dir" ]] || die "Directory not found: $dir"
    [[ -w "$dir" ]] || die "Directory not writable: $dir"
}

validate_not_empty() {
    local value="$1" name="$2"
    [[ -n "$value" ]] || die "$name must not be empty"
}

# Usage
validate_not_empty "$1" "Filename argument"
validate_file_exists "$1"
validate_integer "$2" "Count argument"
```

---

## 25.2 Default Values

```bash
# Parameter expansion for defaults
name="${1:-anonymous}"          # Default if unset or empty
config="${CONFIG_FILE:-/etc/myapp/config}"
log_level="${LOG_LEVEL:-info}"

# Assign default if unset
: "${TIMEOUT:=30}"             # Sets TIMEOUT to 30 if not already set
: "${VERBOSE:=false}"
```

---

## 25.3 Safe File Operations

```bash
# Always quote paths (spaces, glob chars!)
rm "$file"           # Safe
rm $file             # DANGEROUS — word splitting

# Use -- to prevent option injection
rm -- "$file"        # Safe even if $file starts with -
grep -- "$pattern" "$file"

# Safe temp files
tmpfile=$(mktemp) || die "Cannot create temp file"
tmpdir=$(mktemp -d) || die "Cannot create temp dir"

# Avoid TOCTOU (Time-of-check, time-of-use) races
# BAD: check then act
if [[ ! -f "$file" ]]; then
    touch "$file"    # Another process might create it between check and touch
fi

# BETTER: just do it and handle errors
touch "$file" 2>/dev/null || die "Cannot create $file"
```

---

## 25.4 Preventing Dangerous rm

```bash
# NEVER do this:
rm -rf "$DIR/"       # If DIR is empty, this becomes rm -rf /

# Defensive pattern:
[[ -n "$DIR" ]] || die "DIR is empty!"
[[ "$DIR" != "/" ]] || die "Refusing to delete /"
rm -rf "${DIR:?ERROR: DIR is unset or empty}"

# The :? expansion causes an error if the variable is unset or empty
${var:?message}     # If var is unset/empty, print message and exit
```

---

## 25.5 Checking Dependencies

```bash
require_command() {
    command -v "$1" >/dev/null 2>&1 || die "Required command not found: $1"
}

require_root() {
    (( EUID == 0 )) || die "This script must be run as root"
}

require_bash_version() {
    local minimum="$1"
    if [[ "${BASH_VERSINFO[0]}" -lt "$minimum" ]]; then
        die "Bash $minimum+ required (current: $BASH_VERSION)"
    fi
}

require_command curl
require_command jq
require_bash_version 4
```

---

## 25.6 Readonly Variables for Safety

```bash
# Prevent accidental modification
readonly DATABASE_URL="postgres://prod-server/mydb"
readonly MAX_RETRIES=5

# If anything tries to change them:
DATABASE_URL="something"    # ERROR: DATABASE_URL: readonly variable

# Use for critical constants
readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
readonly LOG_FILE="/var/log/${SCRIPT_NAME%.sh}.log"
```

---

## 25.7 Atomic Operations

```bash
# Problem: partial writes if interrupted
echo "$data" > "$config_file"    # File is empty briefly during write

# Solution: write to temp, then atomically rename
tmpfile=$(mktemp "${config_file}.XXXXXX")
echo "$data" > "$tmpfile"
mv "$tmpfile" "$config_file"     # mv is atomic on same filesystem

# For critical files, sync first
echo "$data" > "$tmpfile"
sync "$tmpfile"
mv "$tmpfile" "$config_file"
```

---

## 25.8 Lock Files

Prevent multiple instances from running simultaneously:

```bash
LOCKFILE="/var/lock/${SCRIPT_NAME}.lock"

acquire_lock() {
    if ! mkdir "$LOCKFILE" 2>/dev/null; then
        die "Another instance is running (lockfile: $LOCKFILE)"
    fi
    trap 'rm -rf "$LOCKFILE"' EXIT
}

acquire_lock
# ... do work ...
# Lock is released automatically on exit via trap
```

> **Why `mkdir`?** `mkdir` is atomic — it either succeeds or fails. Unlike file-based locking, there's no race condition.

---

## 25.9 Security Considerations

```bash
# Set secure PATH
export PATH="/usr/local/bin:/usr/bin:/bin"

# Set secure umask (owner-only by default)
umask 077

# Don't expose secrets in command line (visible via ps)
# BAD:
curl -u "user:$PASSWORD" https://api.example.com

# BETTER:
curl -u "user:$(cat /run/secrets/password)" https://api.example.com
# or use .netrc or config file

# Don't store passwords in variables if avoidable
# Variables can leak via /proc, core dumps, error messages
```

---

## 25.10 Defensive Scripting Checklist

```
┌─────────────────────────────────────────────┐
│     DEFENSIVE SCRIPTING CHECKLIST           │
├─────────────────────────────────────────────┤
│ □ Shebang: #!/usr/bin/env bash              │
│ □ Strict mode: set -euo pipefail            │
│ □ All variables quoted: "$var"              │
│ □ Inputs validated before use               │
│ □ Dependencies checked at startup           │
│ □ Temp files cleaned up (trap EXIT)         │
│ □ Error messages to stderr (>&2)            │
│ □ Meaningful exit codes                     │
│ □ No hardcoded passwords/secrets            │
│ □ Safe rm (never rm -rf $UNQUOTED)          │
│ □ Constants marked readonly                 │
│ □ Lock file if single-instance required     │
│ □ Atomic writes for critical files          │
│ □ PATH set explicitly                       │
└─────────────────────────────────────────────┘
```

---

## Exercises

### Exercise 25.1: Safe Deployment Script
Write a script that copies files to a target directory with:
- Input validation
- Lock file to prevent concurrent runs
- Atomic operations
- Full cleanup on failure

### Exercise 25.2: Validate Configuration
Write a function that reads a config file and validates that all required keys are present with non-empty values.

---

## Summary

- **Validate everything**: inputs, files, directories, dependencies
- **Use defaults**: `${var:-default}` and `${var:?error message}`
- **Quote everything**: `"$var"`, `"$@"`, always
- **Use `readonly`** for constants to prevent accidental changes
- **Atomic writes**: write to temp, then `mv` into place
- **Lock files**: use `mkdir` for atomic locking
- **Cleanup always**: `trap cleanup EXIT`
- **Security**: restrict PATH, set umask, never expose secrets

---

**Next Chapter:** [Chapter 26: Functions →](../Part5-Functions/Chapter26-Functions.md)
