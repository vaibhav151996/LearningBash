# Chapter 46: Subshells and Command Grouping

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand when subshells are created
- Use explicit subshells for isolation
- Group commands with `{ }` and `( )`
- Avoid common subshell pitfalls
- Choose between subshells and command groups

---

## 46.1 What Is a Subshell?

A subshell is a child copy of the current shell. It inherits variables and settings but **changes in the subshell don't affect the parent**.

```
┌─────────────────────────┐
│ Parent Shell             │
│   var="hello"            │
│                          │
│   ┌───────────────────┐  │
│   │ Subshell (copy)   │  │
│   │   var="hello"     │  │ ← inherited
│   │   var="changed"   │  │ ← modified locally
│   └───────────────────┘  │
│                          │
│   echo $var → "hello"    │ ← parent is unaffected
└─────────────────────────┘
```

---

## 46.2 When Subshells Are Created

```bash
# 1. Explicit subshell with ( )
(cd /tmp; echo "$PWD")    # /tmp
echo "$PWD"               # original directory (unchanged)

# 2. Pipes (each segment runs in a subshell)
echo "hello" | read -r word
echo "$word"              # empty! (read was in subshell)

# 3. Command substitution
result=$(echo "hello")    # Runs in a subshell

# 4. Background processes
command &                 # Runs in a subshell

# 5. Process substitution
while read -r line; do ... done < <(command)    # command in subshell
```

---

## 46.3 Explicit Subshells: ( )

```bash
# Isolate side effects
(
    cd /some/directory
    export SPECIAL_VAR="value"
    source some_config.sh
    run_command
)
# Parent's pwd, environment, and variables are untouched

# Temporary environment changes
(
    export PATH="/custom/bin:$PATH"
    export LC_ALL=C
    process_data
)

# Parallel execution
(task1) &
(task2) &
(task3) &
wait    # Wait for all
```

---

## 46.4 Command Groups: { }

Braces group commands in the **current** shell — no subshell:

```bash
# Group commands (runs in CURRENT shell)
{
    echo "Line 1"
    echo "Line 2"
    echo "Line 3"
} > output.txt          # All three lines redirect to one file

# Compare:
( commands )    # Subshell: changes are isolated
{ commands; }   # Current shell: changes affect parent

# Important syntax: braces need spaces and semicolons
{ echo "hello"; echo "world"; }    # Correct
{echo "hello"}                      # WRONG — no space after {
```

### Practical Difference

```bash
x=1

# Subshell — x stays 1
(x=99)
echo $x    # 1

# Command group — x changes
{ x=99; }
echo $x    # 99
```

---

## 46.5 Grouped Redirection

```bash
# Redirect a block of commands
{
    echo "=== Report ==="
    date
    echo "Users logged in:"
    who
    echo "Disk usage:"
    df -h
} > report.txt       # Entire block to one file

# Redirect errors from a block
{
    command1
    command2
    command3
} 2> errors.log

# Pipe a block
{
    echo "header"
    cat data.txt
    echo "footer"
} | process_input
```

---

## 46.6 The Pipe Subshell Problem

The biggest subshell gotcha:

```bash
# Problem: variables set in a pipe are lost
total=0
cat numbers.txt | while read -r num; do
    total=$((total + num))
done
echo "Total: $total"    # 0 — total was incremented in a subshell!

# Solution 1: Process substitution
total=0
while read -r num; do
    total=$((total + num))
done < <(cat numbers.txt)
echo "Total: $total"    # Correct!

# Solution 2: Redirect from file directly
total=0
while read -r num; do
    total=$((total + num))
done < numbers.txt
echo "Total: $total"    # Correct!

# Solution 3: lastpipe (Bash 4.2+)
shopt -s lastpipe
total=0
cat numbers.txt | while read -r num; do
    total=$((total + num))
done
echo "Total: $total"    # Correct with lastpipe!
```

---

## 46.7 Subshell Detection

```bash
# Check if you're in a subshell
if [[ $$ -ne $BASHPID ]]; then
    echo "Running in a subshell"
else
    echo "Running in the main shell"
fi

# $$ = PID of the main shell (same in subshells)
# $BASHPID = PID of the current process (different in subshells)

echo "Main: $$"
(echo "Subshell: $$ vs $BASHPID")    # $$ same, BASHPID different
```

---

## 46.8 Performance Considerations

```bash
# Subshells have overhead (fork)
# Avoid in tight loops:

# BAD — forks a subshell for each iteration
for i in $(seq 1 1000); do
    result=$(echo "$i * 2" | bc)    # Two forks per iteration!
done

# GOOD — use arithmetic expansion (no fork)
for i in $(seq 1 1000); do
    result=$((i * 2))               # Pure bash, no fork
done
```

---

## Exercises

### Exercise 46.1: Isolation Testing
Write a script that demonstrates the difference between `( )` and `{ }` by modifying variables, changing directories, and modifying environment — showing what persists and what doesn't.

### Exercise 46.2: Parallel Processing
Write a script that processes 4 files in parallel using subshells, waits for all to complete, and collects results.

---

## Summary

- `( commands )` runs in a **subshell** — changes are isolated from parent
- `{ commands; }` runs in the **current shell** — changes persist
- Pipes create subshells — variables set inside pipes are lost
- Fix pipe variable loss with process substitution `< <(cmd)` or `lastpipe`
- `$BASHPID` differs from `$$` in subshells
- Use subshells for isolation; use groups for combined redirection
- Avoid unnecessary subshells in loops for performance

---

**Next Chapter:** [Chapter 47: Bash Options and Shopt →](Chapter47-Bash-Options.md)
