# Chapter 11: Pipes and Redirections

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand file descriptors and standard I/O streams
- Use output redirection (`>`, `>>`) and input redirection (`<`)
- Redirect standard error (`2>`, `2>&1`)
- Use pipes (`|`) to chain commands together
- Combine pipes and redirections fluently
- Understand and use `tee` for splitting output

---

## 11.1 File Descriptors and Standard Streams

Every running process on Linux has three I/O channels open by default:

```
┌────────────────────────────────┐
│         Your Process           │
│                                │
│  ┌──────────────────────────┐  │
│  │ FD 0: stdin  ← keyboard │  │  Standard Input
│  │ FD 1: stdout → terminal │  │  Standard Output
│  │ FD 2: stderr → terminal │  │  Standard Error
│  └──────────────────────────┘  │
│                                │
│  FD 3, 4, 5, ... (open files) │
└────────────────────────────────┘
```

| File Descriptor | Name | Default Destination | Purpose |
|----------------|------|-------------------|---------|
| 0 | `stdin` | Keyboard | Input to the program |
| 1 | `stdout` | Terminal | Normal output |
| 2 | `stderr` | Terminal | Error messages |

**Key insight:** `stdout` and `stderr` both appear on your terminal by default, but they are **separate streams**. This separation is vital — you can redirect errors to a log while keeping normal output visible, or vice versa.

---

## 11.2 Output Redirection

### `>` — Redirect stdout (Overwrite)

The `>` operator redirects standard output to a file, **replacing** its contents:

```bash
# Save command output to a file
echo "Hello, World" > greeting.txt

# Save directory listing to a file
ls -la > filelist.txt

# Save date to a file
date > timestamp.txt

# WARNING: > overwrites the file completely!
echo "First line" > file.txt
echo "Second line" > file.txt
cat file.txt
# Second line    ← First line is GONE
```

### `>>` — Redirect stdout (Append)

```bash
# Append to a file (doesn't overwrite)
echo "Line 1" > log.txt
echo "Line 2" >> log.txt
echo "Line 3" >> log.txt
cat log.txt
# Line 1
# Line 2
# Line 3
```

### Creating and Truncating Files

```bash
# Create an empty file (or truncate an existing one to zero length)
> empty_file.txt

# This is the fastest way to empty a file:
> /var/log/myapp.log    # File now exists but is empty
```

---

## 11.3 Input Redirection

### `<` — Redirect stdin

Instead of typing input or piping from another command, read from a file:

```bash
# Count words in a file (two equivalent approaches)
wc -w < document.txt      # Input redirection
wc -w document.txt         # Passing filename as argument

# Sort a file
sort < unsorted.txt

# Read from a file in a while loop (very common pattern)
while read -r line; do
    echo "Processing: $line"
done < data.txt
```

**Difference between `<` and passing a filename:**
When you use `<`, the command receives input via stdin and doesn't know the filename. When you pass a filename as an argument, the command opens the file itself. The results are often the same, but some commands behave differently:

```bash
wc -w < file.txt        # Output: 42
wc -w file.txt           # Output: 42 file.txt   (includes filename)
```

---

## 11.4 Redirecting Standard Error

### `2>` — Redirect stderr

```bash
# Redirect errors to a file
ls /nonexistent 2> errors.log
cat errors.log
# ls: cannot access '/nonexistent': No such file or directory

# Suppress errors (send to /dev/null)
ls /nonexistent 2>/dev/null
# (no output — error was discarded)

# Common pattern: suppress errors from find
find / -name "*.conf" 2>/dev/null
```

### `2>>` — Append stderr

```bash
# Append errors to a log file
command_that_might_fail 2>> error.log
```

---

## 11.5 Combining stdout and stderr

### Redirect Both to the Same File

```bash
# Method 1: Redirect stdout, then stderr to stdout's destination
command > output.log 2>&1

# Method 2: Bash shorthand (Bash 4+)
command &> output.log

# Method 3: Append both
command >> output.log 2>&1
command &>> output.log
```

### Understanding `2>&1`

`2>&1` means "redirect file descriptor 2 (stderr) to wherever file descriptor 1 (stdout) is currently going."

**Order matters!**

```bash
# CORRECT: stdout goes to file, stderr follows stdout to the file
command > output.log 2>&1

# WRONG: stderr goes to terminal (stdout's CURRENT destination),
# THEN stdout goes to file. stderr still goes to terminal!
command 2>&1 > output.log
```

```
CORRECT order:  command > file 2>&1
Step 1: stdout → file
Step 2: stderr → (wherever stdout is) → file
Result: Both go to file ✓

WRONG order:    command 2>&1 > file
Step 1: stderr → (wherever stdout is) → terminal
Step 2: stdout → file
Result: stdout in file, stderr on terminal ✗
```

### Redirect stderr and stdout to Different Files

```bash
# stdout to one file, stderr to another
command > output.log 2> error.log

# Process output and errors separately
./deploy.sh > deploy_results.txt 2> deploy_errors.txt
```

### Suppress All Output

```bash
# Discard everything — stdout AND stderr
command > /dev/null 2>&1
command &> /dev/null           # Bash shorthand
```

---

## 11.6 Pipes

A **pipe** (`|`) connects the stdout of one command to the stdin of the next command. This is one of the most powerful features of Unix/Linux.

```
┌──────────┐  stdout   ┌──────────┐  stdout   ┌──────────┐
│ Command1 │──────────>│ Command2 │──────────>│ Command3 │
└──────────┘    stdin→  └──────────┘    stdin→  └──────────┘
                  │                       │
                pipe                    pipe
```

### Basic Pipe Examples

```bash
# Count files in a directory
ls | wc -l

# Find a specific process
ps aux | grep nginx

# Sort and remove duplicates
cat data.txt | sort | uniq

# Find the 10 largest files
du -sh * | sort -rh | head -10

# Search command history
history | grep "ssh"

# Count unique IP addresses in a log
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -20
```

### How Pipes Work Internally

When Bash encounters a pipe:

1. Creates a **pipe** — a small buffer in kernel memory
2. Forks two child processes (one for each side)
3. Connects the left command's stdout to the pipe's input
4. Connects the right command's stdin to the pipe's output
5. Both commands run **simultaneously** (in parallel!)

```
┌─ Process 1 ──────┐     ┌──── Kernel Pipe ────┐     ┌─ Process 2 ──────┐
│                   │     │                      │     │                   │
│   ls -la          │     │  ┌────────────────┐  │     │   grep ".txt"    │
│                   │     │  │ Buffer (64KB)  │  │     │                   │
│  stdout (fd 1) ───┼────>│  │                │  │────>│── stdin (fd 0)   │
│                   │     │  └────────────────┘  │     │                   │
└───────────────────┘     └──────────────────────┘     └───────────────────┘
```

The commands in a pipeline run **concurrently**, not sequentially. The data flows through as it's produced. This means:

```bash
# This doesn't wait for sort to finish before head starts.
# Data flows through in real-time.
sort huge_file.txt | head -5
# head reads 5 lines and exits
# sort receives SIGPIPE and stops — it doesn't need to sort the entire file!
```

### Common Pipe Patterns

```bash
# Filter: select specific lines
cat /etc/passwd | grep "/bin/bash"

# Transform: modify output
echo "HELLO WORLD" | tr 'A-Z' 'a-z'
# hello world

# Count
ls -la | wc -l

# Extract columns
# (Note: ps is just an example — we'll learn awk in Ch. 34)
ps aux | awk '{print $1, $11}' | head -20

# Paginate long output
ls -la /etc | less

# Chain multiple transformations
cat access.log | grep "ERROR" | cut -d' ' -f1,4 | sort | uniq -c | sort -rn
```

---

## 11.7 Pipes and stderr

By default, **pipes only connect stdout**. stderr goes directly to the terminal.

```bash
# stderr is NOT piped — it appears on terminal
ls /nonexistent | grep "error"
# ls: cannot access '/nonexistent': No such file or directory
# (grep receives nothing on stdin — the error went to the terminal)
```

### Pipe stderr

```bash
# Method 1: Redirect stderr to stdout first, then pipe
ls /nonexistent 2>&1 | grep "cannot"
# ls: cannot access '/nonexistent': No such file or directory

# Method 2: Bash 4+ shorthand — pipe both stdout and stderr
ls /nonexistent |& grep "cannot"
```

---

## 11.8 The tee Command

`tee` reads from stdin, writes to stdout AND to a file simultaneously. It's like a T-junction in plumbing.

```
                    ┌────────────────┐
stdin ──────────────│      tee       │──────────── stdout
                    │                │
                    └───────┬────────┘
                            │
                            ▼
                         file.txt
```

```bash
# Save output AND display it
ls -la | tee filelist.txt

# Append instead of overwrite
ls -la | tee -a filelist.txt

# Write to multiple files
ls -la | tee file1.txt file2.txt file3.txt

# Pipe AND save intermediate results
cat data.txt | sort | tee sorted.txt | uniq | tee unique.txt | wc -l

# Use with sudo (common pattern for writing to protected files)
echo "new line" | sudo tee -a /etc/hosts
# (echo runs as you, tee runs as root to write the file)
```

---

## 11.9 Here Documents and Here Strings

### Here Documents (`<<`)

A **here document** feeds a block of text as stdin to a command:

```bash
cat << EOF
This is line 1
This is line 2
Variable expansion works: $USER
This is the last line
EOF
```

Output:
```
This is line 1
This is line 2
Variable expansion works: john
This is the last line
```

The delimiter (commonly `EOF` but can be any word) marks the start and end.

```bash
# Suppress variable expansion by quoting the delimiter
cat << 'EOF'
This is literal: $USER
No expansion: $(date)
EOF
# This is literal: $USER
# No expansion: $(date)

# Suppress leading tabs (<<- instead of <<)
	cat <<- EOF
		Indented content
		Still indented
	EOF
# Indented content
# Still indented
```

### Here Strings (`<<<`)

A **here string** feeds a single string as stdin:

```bash
# Feed a string to a command
grep "hello" <<< "hello world"
# hello world

# Variable as input
name="John Doe"
read first last <<< "$name"
echo "First: $first, Last: $last"
# First: John, Last: Doe

# Equivalent to echo + pipe, but without forking an extra process
wc -w <<< "one two three"
# 3
```

---

## 11.10 Advanced Redirection

### Opening Additional File Descriptors

You can open custom file descriptors beyond 0, 1, and 2:

```bash
# Open fd 3 for writing to a file
exec 3> output.txt

# Write to fd 3
echo "Line 1" >&3
echo "Line 2" >&3

# Close fd 3
exec 3>&-

# Open fd 4 for reading from a file
exec 4< input.txt
read -r line <&4
echo "$line"
exec 4<&-    # Close fd 4
```

### Read and Write from the Same File Descriptor

```bash
# Open fd 3 for reading AND writing
exec 3<> file.txt
read -r line <&3
echo "New data" >&3
exec 3>&-
```

### Redirecting in the Middle of a Script

```bash
# Save original stdout, redirect to file
exec 3>&1          # Save stdout in fd 3
exec > logfile.txt # Redirect stdout to logfile

echo "This goes to logfile"
echo "This too"

exec 1>&3          # Restore stdout from fd 3
exec 3>&-          # Close fd 3

echo "This goes to terminal again"
```

---

## 11.11 Practical Examples

### Example 1: Process a Log File Pipeline

```bash
# Find the top 10 IP addresses hitting your web server
cat /var/log/nginx/access.log \
    | awk '{print $1}' \
    | sort \
    | uniq -c \
    | sort -rn \
    | head -10
```

### Example 2: Separate Output and Errors

```bash
# Run a command, save output and errors separately
./build.sh > build_output.log 2> build_errors.log

# Check if there were errors
if [ -s build_errors.log ]; then
    echo "Build had errors:"
    cat build_errors.log
fi
```

### Example 3: Interactive File Creation

```bash
# Create a configuration file with a here document
cat > /tmp/config.ini << 'EOF'
[database]
host = localhost
port = 5432
name = myapp

[server]
port = 8080
debug = false
EOF
```

### Example 4: Pipeline with tee for Debugging

```bash
# Debug a pipeline by saving intermediate results
cat data.csv \
    | tee /tmp/step1-raw.txt \
    | grep -v "^#" \
    | tee /tmp/step2-no-comments.txt \
    | sort -t',' -k2 \
    | tee /tmp/step3-sorted.txt \
    | head -20
```

---

## Common Mistakes

1. **Using `>` when you meant `>>`** — `>` overwrites; `>>` appends. Accidentally truncating a log file is painful.

2. **Wrong order for `2>&1`** — `cmd > file 2>&1` is correct. `cmd 2>&1 > file` sends errors to the terminal.

3. **Forgetting that pipes only carry stdout** — Use `2>&1 |` or `|&` to include stderr.

4. **Assuming pipe commands are sequential** — They're concurrent. The right side starts before the left side finishes.

5. **Not using `tee` with `sudo`** — `sudo echo "text" > /etc/file` fails because the redirect is done by your shell (not root). Use `echo "text" | sudo tee /etc/file`.

6. **Useless use of cat** — `cat file | grep pattern` can be simplified to `grep pattern file`. However, `cat` at the start of a pipeline can improve readability.

---

## Exercises

### Exercise 11.1: Basic Redirection
1. Save the output of `date` to a file called `today.txt`
2. Append the output of `whoami` to the same file
3. Display the file contents

### Exercise 11.2: Error Handling
1. Run `ls /nonexistent /tmp` and redirect stdout to `output.txt` and stderr to `errors.txt`
2. Verify both files contain expected content

### Exercise 11.3: Pipe Chain
Write a single pipeline that:
1. Lists all files in `/etc`
2. Shows only `.conf` files
3. Counts how many there are

### Exercise 11.4: tee
Write a command that shows you the output of `ps aux | grep bash` while also saving it to `bash_processes.txt`.

### Exercise 11.5: Here Document
Use a here document to create a file called `greeting.sh` that contains a valid Bash script printing "Hello, World!".

---

## Summary

- Every process has three standard streams: **stdin (0)**, **stdout (1)**, **stderr (2)**
- `>` redirects stdout (overwrite); `>>` appends; `<` redirects stdin
- `2>` redirects stderr; `2>&1` merges stderr into stdout
- `&>` redirects both stdout and stderr (Bash shorthand)
- **Pipes** (`|`) connect stdout of one command to stdin of the next
- Pipes run commands **concurrently**, not sequentially
- `tee` splits output to both stdout and a file
- **Here documents** (`<<`) and **here strings** (`<<<`) provide inline input
- Order matters: `cmd > file 2>&1` ≠ `cmd 2>&1 > file`

---

**Next Chapter:** [Chapter 12: Exit Codes and Command Status →](Chapter12-Exit-Codes.md)
