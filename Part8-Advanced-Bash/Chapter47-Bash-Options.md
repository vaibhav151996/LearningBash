# Chapter 47: Bash Options and Shopt

## Learning Objectives

By the end of this chapter, you will be able to:

- Configure Bash behavior with `set` options
- Use `shopt` for extended Bash features
- Apply the right options for scripting vs interactive use
- Build safer scripts with option combinations
- Debug scripts using Bash options

---

## 47.1 set Options

`set` changes shell behavior. Use `-` to enable, `+` to disable:

```bash
set -e        # Enable: exit on error
set +e        # Disable: don't exit on error
```

### Essential set Options

```bash
set -e          # Exit immediately if a command fails
set -u          # Treat unset variables as errors
set -o pipefail # Pipeline fails if any command fails
set -x          # Print each command before executing (debug)
set -n          # Read commands but don't execute (syntax check)
set -v          # Print each line as read (verbose)
set -f          # Disable filename globbing
set -C          # Don't overwrite files with > (use >| to force)
```

### The Strict Mode Combination

```bash
set -euo pipefail

# -e: Any command failure exits the script
# -u: Using undefined variables is an error
# -o pipefail: Pipeline returns the exit code of the first failure
```

### How -e Works in Detail

```bash
set -e

# These WILL cause exit:
false                               # Direct failure
nonexistent_command                  # Command not found

# These will NOT cause exit (by design):
false || true                       # Part of ||
false && true                       # Part of &&
if false; then :; fi               # In an if condition
while false; do :; done            # In a while condition
false; true                         # Left side of ;  (only last matters? No, -e checks each)

# Gotcha: command substitution
set -e
x=$(false)                         # This DOES exit with -e
local x=$(false)                   # This does NOT exit (local succeeds)
```

---

## 47.2 shopt Options

`shopt` is Bash-specific (not POSIX). It controls extended features:

```bash
shopt -s option_name    # Enable (set)
shopt -u option_name    # Disable (unset)
shopt option_name       # Query status
shopt                   # List all options
```

### Most Useful shopt Options

```bash
# Globbing enhancements
shopt -s nullglob       # Globs with no matches expand to nothing (not literal)
shopt -s globstar       # ** matches directories recursively
shopt -s nocaseglob     # Case-insensitive globbing
shopt -s extglob        # Extended globs: ?(pat), *(pat), +(pat), @(pat), !(pat)
shopt -s dotglob        # Include hidden files (.*) in glob matches

# Directory navigation
shopt -s autocd         # Type directory name to cd into it
shopt -s cdspell        # Auto-correct minor cd typos
shopt -s direxpand      # Expand directory names on tab completion

# History
shopt -s histappend     # Append to history file, don't overwrite
shopt -s cmdhist        # Save multi-line commands as single entry

# Other
shopt -s lastpipe       # Last cmd in pipe runs in current shell
shopt -s checkjobs      # Warn about running jobs before exit
shopt -s failglob       # Glob failures become errors
```

---

## 47.3 Key Options Explained

### nullglob

```bash
# Without nullglob:
for f in *.xyz; do
    echo "$f"        # Prints literal "*.xyz" if no matches
done

# With nullglob:
shopt -s nullglob
for f in *.xyz; do
    echo "$f"        # Loop body never executes if no matches
done
```

### globstar

```bash
shopt -s globstar
# ** matches zero or more directories recursively
for f in **/*.py; do
    echo "$f"        # Finds all .py files in all subdirectories
done
```

### extglob

```bash
shopt -s extglob

# Extended glob patterns:
ls !(*.log)           # All files EXCEPT .log files
ls +(*.jpg|*.png)     # One or more jpg/png files
rm ?(temp)*.txt       # Files optionally starting with "temp"

# ?(pattern)   Zero or one match
# *(pattern)   Zero or more matches
# +(pattern)   One or more matches
# @(pattern)   Exactly one match
# !(pattern)   Anything NOT matching
```

### lastpipe

```bash
shopt -s lastpipe

# Without lastpipe: 'read' is in a subshell, variable is lost
echo "hello" | read -r word
echo "$word"    # empty

# With lastpipe: last command in pipe runs in current shell
shopt -s lastpipe
echo "hello" | read -r word
echo "$word"    # hello
```

---

## 47.4 Debug Mode

```bash
# Trace execution
set -x
cp file1.txt file2.txt
# + cp file1.txt file2.txt    ← printed before executing

# Custom trace prefix
export PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
set -x
# +(script.sh:15): main(): cp file1.txt file2.txt

# Enable debug for specific section only
set -x
problematic_function
set +x

# Syntax check (don't execute)
bash -n script.sh    # Just checks syntax

# Verbose (print lines as read)
set -v
```

---

## 47.5 Practical Option Recipes

### Production Script Header

```bash
#!/usr/bin/env bash
set -euo pipefail
shopt -s nullglob
```

### Safe File Processing

```bash
shopt -s nullglob globstar
for f in **/*.log; do
    process "$f"
done
# Safe: no error if no matches, finds files recursively
```

### Temporary Option Changes

```bash
# Save and restore options
old_opts=$(set +o)       # Save all set options
set -e
risky_operation
eval "$old_opts"         # Restore original options

# Or for shopt:
old_nullglob=$(shopt -p nullglob)
shopt -s nullglob
# ... do work ...
eval "$old_nullglob"    # Restore
```

---

## 47.6 Option Quick Reference

```
┌─────────────────┬──────────────────────────────────────┐
│ Option          │ Effect                               │
├─────────────────┼──────────────────────────────────────┤
│ set -e          │ Exit on error                        │
│ set -u          │ Error on undefined variables         │
│ set -o pipefail │ Pipeline fails on first error        │
│ set -x          │ Print commands before execution      │
│ set -n          │ Syntax check only (no execution)     │
│ set -f          │ Disable globbing                     │
│ set -C          │ No clobber (prevent > overwrite)     │
│ shopt nullglob  │ No-match globs expand to nothing     │
│ shopt globstar  │ ** matches recursively               │
│ shopt extglob   │ Extended glob patterns               │
│ shopt lastpipe  │ Last pipe cmd in current shell       │
│ shopt dotglob   │ Globs match hidden files             │
│ shopt failglob  │ No-match globs cause errors          │
└─────────────────┴──────────────────────────────────────┘
```

---

## Exercises

### Exercise 47.1: Option Exploration
Write a script that demonstrates each major `set` and `shopt` option with before/after examples showing the behavior change.

### Exercise 47.2: Debug Wrapper
Write a function `debug_run` that enables `set -x` with a custom PS4, runs a given command, then restores the original settings.

---

## Summary

- `set -euo pipefail` is "strict mode" — essential for reliable scripts
- `set -x` enables debugging trace; customize with `PS4`
- `shopt -s nullglob` prevents glob patterns from being treated as literal strings
- `shopt -s globstar` enables recursive `**` matching
- `shopt -s extglob` adds powerful pattern operators like `!(pattern)`
- Save and restore options with `set +o` and `shopt -p`
- Use `-` to enable options, `+` to disable them

---

**Next Chapter:** [Chapter 48: Writing Safe Shell Scripts →](../Part9-Best-Practices/Chapter48-Safe-Scripts.md)
