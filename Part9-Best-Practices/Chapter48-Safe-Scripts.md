# Chapter 48: Writing Safe Shell Scripts

## Learning Objectives

By the end of this chapter, you will be able to:

- Write scripts that handle edge cases gracefully
- Apply defensive programming techniques consistently
- Use shellcheck to catch common errors
- Implement comprehensive error handling
- Follow a safety checklist for production scripts

---

## 48.1 The Safety Mindset

Every production script should assume:
- Input will be malformed
- Files might not exist or might be empty
- Permissions might be wrong
- Network might be down
- Disk might be full
- The script might be interrupted at any point

---

## 48.2 The Essential Header

```bash
#!/usr/bin/env bash
set -euo pipefail
```

What each part protects against:

```bash
# set -e: Unchecked failures
set -e
cp important_file /backup/        # Script STOPS if cp fails
# Without -e, script silently continues after failure

# set -u: Typos in variable names
set -u
echo "$NAEM"                      # ERROR: NAEM: unbound variable
# Without -u, silently expands to empty string

# set -o pipefail: Hidden pipe failures
set -o pipefail
bad_command | sort | head          # Pipeline returns non-zero
# Without pipefail, only head's exit code matters
```

---

## 48.3 ShellCheck

[ShellCheck](https://www.shellcheck.net/) is the essential linting tool for Bash:

```bash
# Install
sudo apt install shellcheck        # Debian/Ubuntu
brew install shellcheck            # macOS

# Run
shellcheck myscript.sh

# Common findings:
# SC2086: Double quote to prevent globbing and word splitting
# SC2046: Quote command substitution: "$(cmd)"
# SC2034: Variable appears unused
# SC2155: Declare and assign separately
# SC2164: Use cd ... || exit
```

```bash
# Inline directives to suppress specific warnings
# shellcheck disable=SC2034
unused_var="intentionally unused"

# Disable for entire file
# shellcheck disable=SC2086,SC2046
```

> **Rule:** Every script should pass `shellcheck` with zero warnings before deployment.

---

## 48.4 Safe Variable Usage

```bash
# Always quote variables
echo "$filename"                   # Safe
echo $filename                     # DANGEROUS

# Use braces for clarity
echo "${name}_suffix"              # Clear: variable is "name"
echo "$name_suffix"                # Ambiguous: variable is "name_suffix"

# Check before using
[[ -n "$var" ]] || die "var is empty"
[[ -f "$file" ]] || die "file not found: $file"

# Safe defaults
timeout="${TIMEOUT:-30}"
log_dir="${LOG_DIR:-/var/log/myapp}"

# Protect critical operations
rm -rf "${DEPLOY_DIR:?ERROR: DEPLOY_DIR not set}/"
```

---

## 48.5 Safe File Operations

```bash
# Always check cd
cd "$dir" || die "Cannot cd to $dir"

# Safe temp files
tmpfile=$(mktemp) || die "Cannot create temp file"
tmpdir=$(mktemp -d) || die "Cannot create temp dir"
trap 'rm -rf "$tmpfile" "$tmpdir"' EXIT

# Atomic writes
write_config() {
    local target="$1" content="$2"
    local tmp
    tmp=$(mktemp "${target}.XXXXXX")
    echo "$content" > "$tmp"
    mv "$tmp" "$target"        # Atomic on same filesystem
}

# Safe recursive deletion
safe_rm() {
    local dir="$1"
    [[ -n "$dir" ]] || die "Empty path"
    [[ "$dir" != "/" ]] || die "Refusing to delete /"
    [[ "$dir" != "$HOME" ]] || die "Refusing to delete HOME"
    [[ -d "$dir" ]] || die "Not a directory: $dir"
    rm -rf "$dir"
}
```

---

## 48.6 Error Handling Patterns

```bash
# Pattern 1: die on failure
command || die "command failed"

# Pattern 2: check and handle
if ! command; then
    log "command failed, trying alternative..."
    alternative_command || die "alternative also failed"
fi

# Pattern 3: retry
retry() {
    local n="$1"; shift
    local i
    for ((i=1; i<=n; i++)); do
        "$@" && return 0
        sleep "$((i * 2))"
    done
    return 1
}
retry 3 curl -sf "$url" || die "Failed after 3 retries"

# Pattern 4: cleanup on exit
cleanup() {
    local exit_code=$?
    rm -rf "$TMPDIR"
    [[ $exit_code -ne 0 ]] && log "Script failed with code $exit_code"
    exit "$exit_code"
}
trap cleanup EXIT
```

---

## 48.7 Production Script Template

```bash
#!/usr/bin/env bash
set -euo pipefail

readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
readonly VERSION="1.0.0"

# --- Logging ---
log()   { printf '[%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*"; }
warn()  { log "WARN: $*" >&2; }
error() { log "ERROR: $*" >&2; }
die()   { error "$@"; exit 1; }

# --- Cleanup ---
TMPDIR=""
cleanup() {
    local code=$?
    [[ -n "$TMPDIR" ]] && rm -rf "$TMPDIR"
    (( code != 0 )) && error "Exiting with code $code"
    exit "$code"
}
trap cleanup EXIT
trap 'die "Interrupted"' INT TERM

# --- Dependencies ---
for cmd in curl jq gzip; do
    command -v "$cmd" >/dev/null || die "Required: $cmd"
done

# --- Arguments ---
VERBOSE=false
DRY_RUN=false

usage() {
    cat <<EOF >&2
Usage: $SCRIPT_NAME [-hvn] <target>
  -h    Help
  -v    Verbose
  -n    Dry run
  -V    Version
EOF
    exit 1
}

while getopts ":hvnV" opt; do
    case $opt in
        h) usage ;;
        v) VERBOSE=true ;;
        n) DRY_RUN=true ;;
        V) echo "$VERSION"; exit 0 ;;
        *) usage ;;
    esac
done
shift $((OPTIND - 1))
(( $# >= 1 )) || usage
readonly TARGET="$1"

# --- Main ---
main() {
    TMPDIR=$(mktemp -d)
    log "Starting $SCRIPT_NAME v$VERSION"
    
    # ... main logic ...
    
    log "Complete"
}

main
```

---

## 48.8 Safety Checklist

```
□ Shebang: #!/usr/bin/env bash
□ set -euo pipefail
□ All variables quoted: "$var"
□ cd with || exit/die
□ Temp files cleaned with trap EXIT
□ Dependencies checked at startup
□ Error messages to stderr
□ Meaningful exit codes
□ Lock file if single-instance needed
□ Input validation before use
□ No hardcoded secrets
□ ShellCheck passes with zero warnings
□ Tested with edge cases (empty input, special characters, spaces in paths)
```

---

## Exercises

### Exercise 48.1: Script Audit
Take an existing script and audit it against the safety checklist. Fix all issues.

### Exercise 48.2: Robust Installer
Write an installer script that downloads, verifies (checksum), extracts, and installs software — with full error handling, rollback on failure, and cleanup.

---

## Summary

- `set -euo pipefail` is the foundation of safe scripts
- Use ShellCheck on every script — fix all warnings
- Quote all variables, check all inputs, validate all assumptions
- Use `trap cleanup EXIT` for guaranteed resource cleanup
- Handle errors explicitly — never let failures pass silently
- Follow the safety checklist for production scripts

---

**Next Chapter:** [Chapter 49: Portability and POSIX Compliance →](Chapter49-Portability.md)
