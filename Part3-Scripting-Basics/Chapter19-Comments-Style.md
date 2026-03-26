# Chapter 19: Comments, Style, and Readability

## Learning Objectives

By the end of this chapter, you will be able to:

- Write clear, useful comments
- Follow established Bash style conventions
- Structure scripts for maximum readability
- Name variables and functions meaningfully
- Apply the Google Shell Style Guide principles

---

## 19.1 Comments

```bash
# Single-line comment
echo "Hello"  # Inline comment

# Multi-line comments (each line needs #)
# This script performs a backup of the home directory.
# It creates a compressed archive and stores it in /backups.
# Run daily via cron.

# Block comment trick using : and here-doc (not recommended for production)
: << 'COMMENT'
This is technically a multi-line comment.
But it's actually a here-doc passed to the null command.
Prefer individual # lines instead.
COMMENT
```

### What to Comment

```bash
# YES: Explain WHY, not WHAT
# Retry 3 times because the API has intermittent timeouts
for i in 1 2 3; do
    curl -s "$url" && break
    sleep 5
done

# NO: Don't explain obvious code
# Increment counter
((counter++))           # This comment adds nothing

# YES: Document non-obvious behavior
# Parameter expansion: remove longest match from beginning
filepath="/var/log/nginx/access.log"
filename="${filepath##*/}"   # Result: access.log

# YES: Document workarounds
# Redirect stderr to avoid "broken pipe" errors from head
sort huge_file.txt 2>/dev/null | head -10
```

---

## 19.2 Script Header

Every script should start with a header:

```bash
#!/usr/bin/env bash
#
# backup_database.sh — Create a compressed backup of the PostgreSQL database
#
# Usage:
#   ./backup_database.sh [-d database] [-o output_dir] [-r retention_days]
#
# Options:
#   -d    Database name (default: myapp)
#   -o    Output directory (default: /backups)
#   -r    Days to retain old backups (default: 30)
#
# Examples:
#   ./backup_database.sh
#   ./backup_database.sh -d production -o /mnt/backup -r 90
#
# Dependencies: pg_dump, gzip
# Author: John Doe <john@example.com>
# Created: 2026-03-26
```

---

## 19.3 Variable Naming

```bash
# Constants: UPPER_SNAKE_CASE
readonly MAX_RETRIES=3
readonly DEFAULT_PORT=8080
readonly CONFIG_DIR="/etc/myapp"

# Regular variables: lower_snake_case
current_user="$(whoami)"
file_count=0
output_file="/tmp/results.txt"

# Loop variables: short, descriptive
for file in *.txt; do ...
for user in "${users[@]}"; do ...
for i in {1..10}; do ...

# Avoid: single letters (except loop counters), vague names
# BAD:
x=5
tmp="something"
data="important"

# GOOD:
retry_count=5
temp_filename="staging_config.tmp"
user_email="admin@example.com"
```

---

## 19.4 Code Structure

### Logical Sections

```bash
#!/usr/bin/env bash

# ============================================================
# Configuration
# ============================================================
readonly SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
readonly LOG_FILE="/var/log/myapp.log"

# ============================================================
# Functions
# ============================================================
log() {
    printf '[%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*"
}

die() {
    printf 'ERROR: %s\n' "$*" >&2
    exit 1
}

usage() {
    cat << EOF
Usage: $(basename "$0") [options] <argument>

Options:
    -h, --help      Show this help message
    -v, --verbose   Enable verbose output
EOF
}

# ============================================================
# Validation
# ============================================================
[[ $# -eq 0 ]] && { usage; exit 1; }

# ============================================================
# Main
# ============================================================
main() {
    log "Starting..."
    # ... main logic ...
    log "Complete."
}

main "$@"
```

### Indentation

Use **4 spaces** or **2 spaces** consistently. Never mix tabs and spaces.

```bash
# 4-space indent (common)
if [[ -f "$file" ]]; then
    while IFS= read -r line; do
        if [[ -n "$line" ]]; then
            process_line "$line"
        fi
    done < "$file"
fi
```

### Line Length

Keep lines under **80 characters**. Use backslash continuation for long lines:

```bash
# Long command — break with backslash
curl -X POST \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer ${TOKEN}" \
    -d "{\"name\": \"${name}\"}" \
    "https://api.example.com/users"

# Long pipeline — break after |
cat /var/log/syslog \
    | grep "error" \
    | sort \
    | uniq -c \
    | sort -rn \
    | head -20
```

---

## 19.5 Function Style

```bash
# Use the function_name() syntax (POSIX compatible)
process_file() {
    local filename="$1"
    local output_dir="${2:-/tmp}"
    
    # Validate
    [[ -f "$filename" ]] || { echo "Not found: $filename" >&2; return 1; }
    
    # Process
    local base
    base="$(basename "$filename")"
    cp "$filename" "${output_dir}/${base}.bak"
    
    return 0
}

# Avoid the 'function' keyword (not POSIX)
# function process_file { ... }   # Not portable
```

---

## 19.6 Best Practices Checklist

1. **Always quote variables**: `"$var"`, not `$var`
2. **Use `[[ ]]` over `[ ]`** in Bash scripts (safer, more features)
3. **Use `$(command)` over backticks** `` `command` `` (nestable, clearer)
4. **Prefer `printf` over `echo`** for portable, predictable output
5. **Use `local` in functions** to avoid polluting global scope
6. **Check return values** or use `set -e`
7. **Send errors to stderr**: `echo "Error" >&2`
8. **Use `readonly` for constants**
9. **Use `main()` function pattern** for complex scripts
10. **Run ShellCheck** on your scripts

---

## 19.7 Using printf Instead of echo

`printf` is more portable and predictable:

```bash
# echo varies between systems
echo -e "Hello\tWorld"    # Works on some systems, literal on others

# printf is consistent everywhere
printf "Hello\tWorld\n"
printf "Name: %s, Age: %d\n" "$name" "$age"
printf "%05d\n" 42          # 00042  (zero-padded)
printf "%.2f\n" 3.14159     # 3.14   (limited precision)
```

---

## Exercises

### Exercise 19.1: Style Review
Take a script you've written in a previous exercise and rewrite it following the conventions in this chapter.

### Exercise 19.2: Template Script
Create a template script file that you can copy for new projects, including header, configuration, functions, validation, and main sections.

---

## Summary

- Comment **why**, not **what** — explain non-obvious decisions
- Use a **header block** with description, usage, options, and examples
- Name variables: `UPPER_CASE` for constants, `lower_case` for variables
- Structure scripts: header → config → functions → validation → main
- Indent consistently; keep lines under 80 characters
- **Always quote**, prefer `[[ ]]`, use `$(...)`, use `local`, send errors to stderr
- `printf` is more portable than `echo`

---

**Next Chapter:** [Part 4, Chapter 20: Conditional Statements →](../Part4-Control-Flow/Chapter20-Conditionals.md)
