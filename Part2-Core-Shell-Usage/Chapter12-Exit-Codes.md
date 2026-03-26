# Chapter 12: Exit Codes and Command Status

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand exit codes and their role in shell scripting
- Use `$?` to check command success or failure
- Write scripts that handle errors based on exit codes
- Use `true`, `false`, and `test` for control flow
- Understand exit codes in pipelines with `PIPESTATUS`

---

## 12.1 What Is an Exit Code?

Every command in Linux returns an **exit code** (also called **exit status** or **return code**) — an integer between 0 and 255.

```
Exit Code 0     = SUCCESS (the command worked)
Exit Code 1-255 = FAILURE (something went wrong)
```

This is a convention, not an enforcement. But virtually every well-written program follows it.

```bash
# Successful command
ls /tmp
echo $?
# 0

# Failed command
ls /nonexistent_directory
echo $?
# 2        (ls returns 2 for "cannot access")

# The special variable $? holds the exit code of the LAST command
```

---

## 12.2 Common Exit Codes

| Code | Meaning | Example |
|------|---------|---------|
| 0 | Success | Command completed normally |
| 1 | General error | Miscellaneous errors |
| 2 | Misuse of shell command | Invalid arguments, missing keywords |
| 126 | Permission denied | Command found but not executable |
| 127 | Command not found | Typo in command name |
| 128 | Invalid exit argument | `exit 3.14` (not an integer) |
| 128+N | Fatal signal N | Killed by signal (e.g., 137 = 128+9 = SIGKILL) |
| 130 | Ctrl+C (SIGINT) | User interrupted (128+2) |
| 137 | SIGKILL | Process was killed with `kill -9` (128+9) |
| 143 | SIGTERM | Process was terminated with `kill` (128+15) |
| 255 | Exit status out of range | `exit -1` wraps to 255 |

```bash
# Command not found
asdfgh
echo $?    # 127

# Permission denied
touch /root/forbidden
echo $?    # 1 (or could be other non-zero)

# Check specific program exit codes
grep "pattern" file.txt    # 0 if found, 1 if not found, 2 if error
diff file1 file2           # 0 if same, 1 if different, 2 if error
ping -c 1 host             # 0 if reachable, 1 or 2 if not
```

---

## 12.3 Using Exit Codes

### With `&&` and `||`

Exit codes power the `&&` and `||` operators from Chapter 8:

```bash
# && runs second command only if first succeeds (exit 0)
mkdir newdir && echo "Created successfully"

# || runs second command only if first fails (exit non-zero)
mkdir existingdir || echo "Failed to create directory"

# Combined pattern
gcc program.c -o program && echo "Build succeeded" || echo "Build failed"
```

### With if Statements

`if` checks the exit code of a command:

```bash
if grep -q "error" logfile.txt; then
    echo "Errors found in log!"
fi

# The -q flag makes grep quiet (no output), it only sets the exit code
```

### Setting Exit Codes in Scripts

```bash
#!/bin/bash
# Your script should exit with meaningful codes

# exit 0 — success
# exit 1 — general error

if [ ! -f "$1" ]; then
    echo "Error: File not found: $1" >&2
    exit 1
fi

echo "Processing $1..."
exit 0
```

---

## 12.4 The true and false Commands

`true` and `false` are commands whose sole purpose is to return exit codes:

```bash
true
echo $?    # 0

false
echo $?    # 1

# Useful in loops and conditions
while true; do           # Infinite loop
    echo "Running..."
    sleep 1
done

if true; then
    echo "This always executes"
fi

# The : (colon) builtin is equivalent to true
:
echo $?    # 0
```

---

## 12.5 Exit Codes in Pipelines

### The Problem

In a pipeline, `$?` only gives the exit code of the **last** command:

```bash
# grep fails (returns 1) but wc succeeds (returns 0)
grep "nonexistent" file.txt | wc -l
echo $?    # 0  ← This is wc's exit code, NOT grep's!
```

### The Solution: PIPESTATUS

Bash provides the `PIPESTATUS` array, which holds the exit code of every command in the pipeline:

```bash
grep "nonexistent" file.txt | sort | wc -l
echo "${PIPESTATUS[0]}"    # grep's exit code (1)
echo "${PIPESTATUS[1]}"    # sort's exit code (0)
echo "${PIPESTATUS[2]}"    # wc's exit code (0)
echo "${PIPESTATUS[@]}"    # All: 1 0 0
```

### set -o pipefail

With `pipefail`, the pipeline's exit code is the exit code of the **last command that failed** (rightmost non-zero):

```bash
set -o pipefail

grep "nonexistent" file.txt | sort | wc -l
echo $?    # 1  ← Now reflects grep's failure!

# This is essential for robust scripts
set -euo pipefail    # The "strict mode" (see Chapter 47)
```

---

## 12.6 The test Command and `[`

The `test` command evaluates conditional expressions and returns exit code 0 (true) or 1 (false):

```bash
test -f /etc/passwd
echo $?    # 0 (true — the file exists)

test -f /nonexistent
echo $?    # 1 (false — the file doesn't exist)

# [ is a synonym for test (it's actually a command!)
[ -f /etc/passwd ]
echo $?    # 0

# Note: [ requires a closing ]
# The spaces around [ and ] are REQUIRED
```

Common test expressions (preview — detailed in Chapter 20):

```bash
# File tests
[ -f file ]     # True if file exists and is a regular file
[ -d dir ]      # True if directory exists
[ -r file ]     # True if file is readable
[ -w file ]     # True if file is writable
[ -x file ]     # True if file is executable
[ -s file ]     # True if file exists and is not empty

# String tests
[ -z "$var" ]   # True if string is empty
[ -n "$var" ]   # True if string is not empty
[ "$a" = "$b" ] # True if strings are equal

# Numeric tests
[ "$a" -eq "$b" ]  # Equal
[ "$a" -ne "$b" ]  # Not equal
[ "$a" -lt "$b" ]  # Less than
[ "$a" -gt "$b" ]  # Greater than
```

---

## Common Mistakes

1. **Not checking exit codes** — Ignoring `$?` means errors pass silently. Always check or use `set -e`.

2. **Assuming `$?` persists** — `$?` is updated after EVERY command, including `echo`. Check it immediately.

3. **Forgetting pipeline exit codes** — `$?` shows only the last command's status. Use `PIPESTATUS` or `set -o pipefail`.

4. **Using exit codes above 125** — Codes 126-255 have special meanings. Stick to 0-125 for your scripts.

5. **Spaces in `[` commands** — `[$var = "test"]` fails. It must be `[ $var = "test" ]` with spaces.

---

## Exercises

### Exercise 12.1: Exit Code Exploration
Run each command and check `$?`:
1. `ls /`
2. `ls /nonexistent`
3. `grep root /etc/passwd`
4. `grep zzzzz /etc/passwd`
5. `true`
6. `false`

### Exercise 12.2: Pipeline Status
Run this command and examine `PIPESTATUS`:
```bash
cat /etc/passwd | grep "root" | wc -l
echo "${PIPESTATUS[@]}"
```

### Exercise 12.3: Error-Aware Script
Write a script that takes a filename as an argument and exits with code 1 if the file doesn't exist, code 2 if it's not readable, and code 0 if everything is fine.

---

## Summary

- Every command returns an **exit code**: 0 = success, 1-255 = failure
- `$?` holds the exit code of the most recently executed command
- Exit codes drive `&&`, `||`, and `if` statements
- `PIPESTATUS` array holds exit codes for every pipeline stage
- `set -o pipefail` makes a pipeline fail if any stage fails
- `test` (or `[`) evaluates conditions and returns exit codes
- **Always check or handle exit codes** in scripts

---

**Next Chapter:** [Chapter 13: Job Control →](Chapter13-Job-Control.md)
