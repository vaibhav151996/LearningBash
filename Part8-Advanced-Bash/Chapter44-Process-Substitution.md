# Chapter 44: Process Substitution and Coprocesses

## Learning Objectives

By the end of this chapter, you will be able to:

- Use process substitution to treat command output as files
- Compare command outputs without temporary files
- Use coprocesses for two-way communication
- Understand named pipes (FIFOs)
- Apply these techniques to solve real problems

---

## 44.1 Process Substitution

Process substitution creates a temporary file descriptor from a command's output:

```bash
# <(command) — command output as a readable "file"
# >(command) — writable "file" that feeds into a command

diff <(ls dir1/) <(ls dir2/)           # Compare two directory listings
diff <(sort file1.txt) <(sort file2.txt)   # Compare sorted versions

# What's happening under the hood:
echo <(echo hello)    # /dev/fd/63  (a file descriptor path)
```

```
Regular pipe:              Process substitution:
cmd1 | cmd2                cmd2 <(cmd1)

cmd1 stdout ──────▶ cmd2 stdin    cmd1 stdout ──▶ /dev/fd/N ──▶ cmd2 argument
                                  (appears as filename)
```

---

## 44.2 Practical Uses

### Compare Without Temp Files

```bash
# Compare remote and local file lists
diff <(ssh remote "ls /opt/app/") <(ls /opt/app/)

# Compare sorted versions of files
diff <(sort -u file_a.txt) <(sort -u file_b.txt)

# Compare configs from different servers
diff <(ssh server1 cat /etc/nginx/nginx.conf) \
     <(ssh server2 cat /etc/nginx/nginx.conf)
```

### Feed Multiple Inputs

```bash
# paste two command outputs side by side
paste <(cut -d: -f1 /etc/passwd) <(cut -d: -f3 /etc/passwd)

# join sorted outputs
join <(sort file1.txt) <(sort file2.txt)
```

### Avoid Subshell Variable Loss

```bash
# Problem: pipe creates a subshell, variables are lost
count=0
cat file.txt | while read -r line; do
    ((count++))
done
echo "$count"    # 0 — subshell's count is gone!

# Solution: process substitution keeps read in current shell
count=0
while read -r line; do
    ((count++))
done < <(cat file.txt)
echo "$count"    # Correct count!
```

### Output Process Substitution

```bash
# Write to multiple destinations simultaneously
echo "log entry" | tee >(gzip > log.gz) >(wc -l > count.txt) > log.txt

# Process and log simultaneously
command 2> >(logger -t myapp) | process_output
```

---

## 44.3 Named Pipes (FIFOs)

A named pipe is a file in the filesystem that acts as a pipe:

```bash
# Create a named pipe
mkfifo /tmp/mypipe

# Terminal 1: Write to pipe (blocks until reader connects)
echo "Hello from writer" > /tmp/mypipe

# Terminal 2: Read from pipe
cat /tmp/mypipe    # "Hello from writer"

# Clean up
rm /tmp/mypipe
```

### Use Case: Producer-Consumer

```bash
#!/bin/bash
pipe="/tmp/work_pipe"
mkfifo "$pipe"
trap 'rm -f "$pipe"' EXIT

# Producer (background)
produce() {
    for i in {1..10}; do
        echo "task_$i" > "$pipe"
        sleep 1
    done
}

# Consumer
consume() {
    while read -r task < "$pipe"; do
        echo "Processing: $task"
    done
}

produce &
consume
```

---

## 44.4 Coprocesses (Bash 4+)

A coprocess runs a command in the background with two-way pipes:

```bash
# Start a coprocess
coproc BC { bc -l; }

# Send input
echo "scale=4; 22/7" >&"${BC[1]}"

# Read output
read -r result <&"${BC[0]}"
echo "$result"    # 3.1428

# Send more
echo "sqrt(2)" >&"${BC[1]}"
read -r result <&"${BC[0]}"
echo "$result"    # 1.4142

# Close input (signal EOF)
exec {BC[1]}>&-

# Wait for coprocess to finish
wait "$BC_PID"
```

### Coprocess File Descriptors

```
┌──────────────┐     ${COPROC[1]}     ┌──────────────┐
│ Your script  │ ───── stdin ────────▶ │  Coprocess   │
│              │                       │  (e.g., bc)  │
│              │ ◀──── stdout ──────── │              │
└──────────────┘     ${COPROC[0]}     └──────────────┘
```

---

## 44.5 Here Strings and Here Documents (Recap for Context)

```bash
# Here string — single line of input
grep "pattern" <<< "$variable"
bc <<< "2 + 2"

# Here document — multi-line input
cat <<EOF
Hello, $name!
Today is $(date).
EOF

# Here document with no expansion
cat <<'EOF'
Variables like $name are literal here.
No $(command) expansion either.
EOF
```

---

## 44.6 Common Mistakes

### Mistake 1: Process Substitution Is Not Available in sh

```bash
#!/bin/sh
diff <(ls dir1) <(ls dir2)    # ERROR in /bin/sh!

#!/bin/bash
diff <(ls dir1) <(ls dir2)    # Works in bash
```

### Mistake 2: Forgetting Process Substitution Creates Subshells

```bash
# Variables set inside <() are lost to the parent
while read -r line; do
    total=$((total + 1))
done < <(command)          # total IS preserved (read is in current shell)

echo "data" > >(while read -r x; do var=$x; done)    # var is NOT preserved
```

---

## Exercises

### Exercise 44.1: Config Diff
Write a script that compares the effective (non-comment, non-blank) lines of two config files using process substitution.

### Exercise 44.2: Interactive Calculator
Use a coprocess with `bc` to create an interactive calculator that maintains state between calculations.

---

## Summary

- `<(command)` creates a file descriptor from command output — use as a filename argument
- `>(command)` creates a writable file descriptor that feeds into a command
- Process substitution avoids temporary files and subshell variable loss
- Named pipes (`mkfifo`) enable inter-process communication via the filesystem
- Coprocesses provide two-way communication with a background process
- These features are Bash-specific — not available in POSIX `sh`

---

**Next Chapter:** [Chapter 45: Advanced Here Documents →](Chapter45-Here-Documents.md)
