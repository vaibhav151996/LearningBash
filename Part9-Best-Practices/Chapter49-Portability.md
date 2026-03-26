# Chapter 49: Portability and POSIX Compliance

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand the difference between POSIX sh and Bash
- Write portable scripts that work across systems
- Know which features are Bash-specific
- Handle differences between Linux and macOS
- Decide when to use Bash features vs POSIX compliance

---

## 49.1 POSIX sh vs Bash

POSIX `sh` is a standard; Bash is an implementation that extends it.

```
┌──────────────────────────────────────────────────────┐
│                     Bash                              │
│  ┌────────────────────────────────────────────────┐  │
│  │                POSIX sh                         │  │
│  │  - basic variables             - if/else        │  │
│  │  - functions (name() { })      - while/for      │  │
│  │  - pipes, redirects            - case           │  │
│  │  - $(), backticks              - test / [ ]     │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  Bash-only features:                                 │
│  - [[ ]]             - arrays          - $RANDOM     │
│  - (( ))             - ${var//p/r}     - shopt       │
│  - {1..10}           - <<<             - BASH_*      │
│  - <() >()           - declare -A      - select      │
│  - extglob           - local (partial) - coproc      │
└──────────────────────────────────────────────────────┘
```

---

## 49.2 Bash-Specific Features

Features that will NOT work in `/bin/sh`, dash, or other POSIX shells:

```bash
# [[ ]] extended test
[[ "$str" =~ regex ]]        # Bash only
[[ "$str" == pattern* ]]     # Bash only

# (( )) arithmetic
(( x++ ))                    # Bash only

# Arrays
arr=(a b c)                  # Bash only
declare -A map               # Bash only

# Brace expansion
echo {1..10}                 # Bash only
cp file{,.bak}               # Bash only

# Here strings
grep x <<< "$str"            # Bash only

# Process substitution
diff <(cmd1) <(cmd2)         # Bash only

# Advanced parameter expansion
${var^^}                     # Bash 4+ only
${var,,}                     # Bash 4+ only
${var//old/new}              # Bash only (though some shells support it)

# select menu
select opt in a b c; do ...  # Bash only
```

---

## 49.3 POSIX-Compatible Alternatives

```bash
# Instead of [[ ]], use [ ] (test)
# Bash:
[[ -f "$file" && -r "$file" ]]
# POSIX:
[ -f "$file" ] && [ -r "$file" ]

# Instead of (( )), use $(( )) or expr
# Bash:
(( count++ ))
# POSIX:
count=$((count + 1))

# Instead of arrays, use positional parameters or strings
# Bash:
arr=("one" "two" "three")
for item in "${arr[@]}"; do echo "$item"; done
# POSIX:
set -- "one" "two" "three"
for item in "$@"; do echo "$item"; done

# Instead of <<<, use echo |
# Bash:
grep "x" <<< "$string"
# POSIX:
echo "$string" | grep "x"

# Instead of ${var,,}, use tr
# Bash:
lower="${str,,}"
# POSIX:
lower=$(echo "$str" | tr '[:upper:]' '[:lower:]')

# Instead of == in [ ], use =
# Bash:
[ "$a" == "$b" ]
# POSIX:
[ "$a" = "$b" ]
```

---

## 49.4 Linux vs macOS Differences

macOS ships with BSD versions of common tools, which differ from GNU versions:

```bash
# sed -i (in-place editing)
# Linux (GNU sed):
sed -i 's/old/new/' file.txt
# macOS (BSD sed):
sed -i '' 's/old/new/' file.txt

# Portable:
sed -i.bak 's/old/new/' file.txt && rm file.txt.bak

# date command
# Linux:
date -d "2024-01-15" +%s
# macOS:
date -j -f "%Y-%m-%d" "2024-01-15" +%s

# readlink -f (canonical path)
# Linux:
readlink -f "$path"
# macOS (no -f by default):
# Install coreutils: brew install coreutils → greadlink -f
# Or use: cd "$(dirname "$path")" && pwd -P

# grep -P (PCRE)
# Linux: available
grep -P '\d+' file
# macOS: not available, use grep -E instead
grep -E '[0-9]+' file

# stat command (completely different syntax)
# Linux:
stat -c %s file.txt          # File size
# macOS:
stat -f %z file.txt          # File size

# mktemp
# Linux:
mktemp /tmp/myapp.XXXXXX
# macOS: works the same (compatible)
```

---

## 49.5 Writing Portable Scripts

```bash
#!/usr/bin/env bash    # More portable than #!/bin/bash

# Detect OS
case "$(uname -s)" in
    Linux*)  OS="Linux" ;;
    Darwin*) OS="macOS" ;;
    CYGWIN*|MINGW*|MSYS*) OS="Windows" ;;
    *) OS="Unknown" ;;
esac

# Portable sed -i
sed_inplace() {
    if [[ "$OS" == "macOS" ]]; then
        sed -i '' "$@"
    else
        sed -i "$@"
    fi
}

# Portable readlink
realpath_portable() {
    if command -v realpath >/dev/null 2>&1; then
        realpath "$1"
    elif command -v greadlink >/dev/null 2>&1; then
        greadlink -f "$1"
    else
        cd "$(dirname "$1")" && echo "$(pwd -P)/$(basename "$1")"
    fi
}

# Check for GNU vs BSD tools
if date --version >/dev/null 2>&1; then
    DATE_CMD="gnu"
else
    DATE_CMD="bsd"
fi
```

---

## 49.6 When to Use Bash vs POSIX sh

```
Use POSIX sh when:
├── Script must run on minimal systems (containers, embedded)
├── Script is very simple (no arrays, no regex matching needed)
├── Targeting dash / ash / BusyBox environments
└── Maximum compatibility is required

Use Bash when:
├── Associative arrays or indexed arrays are needed
├── Regex matching with [[ =~ ]] is needed
├── Advanced parameter expansion saves complexity
├── Interactive features (select, readline) are needed
├── You control the target environment
└── The script is complex enough to benefit from Bash features
```

> **Practical Advice:** Most scripts can safely target Bash. Use POSIX sh only when you specifically need to run on systems without Bash (Alpine Docker images, embedded systems, etc.).

---

## Exercises

### Exercise 49.1: Portability Audit
Take a Bash script and identify every Bash-specific feature. Write POSIX-compatible alternatives where possible.

### Exercise 49.2: Cross-Platform Tool
Write a script that works on both Linux and macOS to: check disk usage, show top processes, and display system uptime — handling tool differences.

---

## Summary

- POSIX sh is the portable standard; Bash extends it significantly
- `[[ ]]`, arrays, `<<<`, process substitution, and `(( ))` are Bash-only
- macOS uses BSD tools — `sed -i`, `date`, `stat`, `readlink` differ from GNU
- Use `#!/usr/bin/env bash` for better portability
- Detect OS with `uname -s` and adapt accordingly
- Choose POSIX sh for minimal environments; Bash for feature-rich scripting

---

**Next Chapter:** [Chapter 50: Debugging Techniques →](Chapter50-Debugging.md)
