# Chapter 22: Loops — for, while, until, select

## Learning Objectives

By the end of this chapter, you will be able to:

- Use `for` loops (list, C-style, and range-based)
- Use `while` and `until` loops
- Build interactive menus with `select`
- Read files and command output in loops
- Choose the right loop for each situation

---

## 22.1 The for Loop

### List Form

```bash
# Iterate over a list of values
for fruit in apple banana cherry; do
    echo "I like $fruit"
done

# Iterate over files
for file in *.txt; do
    echo "Processing: $file"
    wc -l "$file"
done

# Iterate over command output
for user in $(cut -d: -f1 /etc/passwd); do
    echo "User: $user"
done

# Iterate over a range (brace expansion)
for i in {1..10}; do
    echo "Number: $i"
done

# Range with step
for i in {0..100..5}; do
    echo "$i"    # 0, 5, 10, 15, ..., 100
done

# Iterate over arguments
for arg in "$@"; do
    echo "Argument: $arg"
done
```

### C-Style for Loop

```bash
# Familiar to C/Java/JavaScript programmers
for ((i = 0; i < 10; i++)); do
    echo "Index: $i"
done

# Multiple variables
for ((i = 0, j = 10; i < j; i++, j--)); do
    echo "i=$i, j=$j"
done

# Count down
for ((i = 10; i >= 0; i--)); do
    echo "$i..."
done
echo "Launch!"
```

### Iterating Over Arrays

```bash
# Array iteration (covered in detail in Chapter 42)
files=("report.txt" "data.csv" "notes.md")

for file in "${files[@]}"; do
    echo "File: $file"
done
```

---

## 22.2 The while Loop

`while` runs as long as the condition command succeeds (exit 0):

```bash
# Basic counter
count=1
while [ $count -le 5 ]; do
    echo "Count: $count"
    ((count++))
done

# Infinite loop (with break condition)
while true; do
    read -rp "Enter 'quit' to exit: " input
    [[ "$input" == "quit" ]] && break
    echo "You entered: $input"
done

# Read lines from a file (THE standard pattern)
while IFS= read -r line; do
    echo "Line: $line"
done < data.txt

# Read from command output
while IFS= read -r line; do
    echo "Process: $line"
done < <(ps aux)
```

### Reading Lines — The Correct Way

```bash
# CORRECT: preserves whitespace, handles backslashes
while IFS= read -r line; do
    process "$line"
done < "$file"

# WRONG: pipe creates subshell — variables lost after loop
cat "$file" | while read line; do
    count=$((count + 1))    # This count is lost!
done
echo "$count"  # Empty or wrong

# CORRECT: redirect, not pipe
count=0
while IFS= read -r line; do
    ((count++))
done < "$file"
echo "$count"  # Correct!
```

---

## 22.3 The until Loop

`until` is the opposite of `while` — it runs until the condition succeeds:

```bash
# Wait for a file to appear
until [ -f /tmp/ready.flag ]; do
    echo "Waiting for ready signal..."
    sleep 1
done
echo "Ready!"

# Wait for a service to come up
until curl -s http://localhost:8080/health > /dev/null 2>&1; do
    echo "Waiting for server..."
    sleep 2
done
echo "Server is up!"

# Retry with limit
attempts=0
max_attempts=5
until [ $attempts -ge $max_attempts ]; do
    if ./deploy.sh; then
        echo "Deploy succeeded"
        break
    fi
    ((attempts++))
    echo "Attempt $attempts failed, retrying..."
    sleep 5
done

if [ $attempts -ge $max_attempts ]; then
    echo "Deploy failed after $max_attempts attempts" >&2
    exit 1
fi
```

---

## 22.4 The select Loop

`select` creates a numbered menu automatically:

```bash
#!/bin/bash
echo "Select your editor:"
select editor in vim nano emacs "Visual Studio Code" quit; do
    case "$editor" in
        vim|nano|emacs)
            echo "Setting EDITOR=$editor"
            export EDITOR="$editor"
            break
            ;;
        "Visual Studio Code")
            echo "Setting EDITOR=code"
            export EDITOR="code"
            break
            ;;
        quit)
            echo "Bye!"
            exit 0
            ;;
        *)
            echo "Invalid choice. Try again."
            ;;
    esac
done
```

Output:
```
Select your editor:
1) vim
2) nano
3) emacs
4) Visual Studio Code
5) quit
#? 2
Setting EDITOR=nano
```

The `PS3` variable controls the select prompt:

```bash
PS3="Enter your choice: "    # Instead of default "#? "
select opt in ...; do
    ...
done
```

---

## 22.5 Loop Patterns

### Processing Command Output Safely

```bash
# CORRECT: use process substitution for command output
while IFS= read -r line; do
    echo "$line"
done < <(find . -name "*.sh" -type f)

# CORRECT: use a for loop with null-delimited results (handles filenames with newlines)
while IFS= read -r -d '' file; do
    echo "File: $file"
done < <(find . -name "*.sh" -print0)
```

### Parallel Processing in Loops

```bash
# Process files in parallel (basic)
for file in *.csv; do
    process_file "$file" &
done
wait    # Wait for all background jobs
echo "All done"

# Limit parallelism
max_jobs=4
for file in *.csv; do
    # Wait if too many background jobs
    while (( $(jobs -r | wc -l) >= max_jobs )); do
        sleep 0.1
    done
    process_file "$file" &
done
wait
```

### Countdown Timer

```bash
for ((i = 10; i > 0; i--)); do
    printf "\rCountdown: %2d" "$i"
    sleep 1
done
echo -e "\rCountdown: DONE!"
```

---

## Common Mistakes

1. **Looping over `ls` output** — `for f in $(ls)` breaks on filenames with spaces. Use `for f in *` instead.

2. **Pipe-while variable loss** — `cmd | while read; do ...; done` — variables inside are lost. Use `while ... done < <(cmd)`.

3. **Not using nullglob** — `for f in *.csv; do` processes the literal `*.csv` if no files match. Use `shopt -s nullglob`.

4. **Infinite loops without break** — Always ensure there's an exit condition.

5. **Forgetting `done`** — Every `for`, `while`, `until`, and `select` needs `done`.

---

## Exercises

### Exercise 22.1: Multiplication Table
Write a script that prints the multiplication table for a given number (1-12).

### Exercise 22.2: File Renamer
Write a script that renames all `.jpeg` files to `.jpg` in the current directory.

### Exercise 22.3: Monitoring Loop
Write a loop that checks if a URL responds every 5 seconds and prints the HTTP status code.

### Exercise 22.4: Menu System
Use `select` to build a menu that lets users choose to view date, uptime, disk usage, or quit.

---

## Summary

- **`for`** — iterate over lists, ranges, files, or use C-style syntax
- **`while`** — loop while condition is true (exit 0)
- **`until`** — loop until condition is true (opposite of while)
- **`select`** — auto-generate numbered menus
- Read files: `while IFS= read -r line; do ... done < file`
- Read commands: `while ... done < <(command)` — NOT `command | while`
- Use `shopt -s nullglob` in scripts with glob-based loops

---

**Next Chapter:** [Chapter 23: Loop Control — break, continue, exit →](Chapter23-Loop-Control.md)
