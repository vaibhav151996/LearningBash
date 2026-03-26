# Chapter 21: Case Statements and Pattern Matching

## Learning Objectives

By the end of this chapter, you will be able to:

- Write `case` statements for multi-way branching
- Use glob patterns in case branches
- Replace complex if/elif chains with clean case statements
- Parse command-line options and user input with case

---

## 21.1 The case Statement

`case` matches a value against multiple patterns:

```bash
case "$variable" in
    pattern1)
        commands
        ;;
    pattern2)
        commands
        ;;
    pattern3|pattern4)     # Multiple patterns with |
        commands
        ;;
    *)                     # Default (catch-all)
        commands
        ;;
esac
```

### Basic Example

```bash
#!/bin/bash
read -rp "Enter a fruit: " fruit

case "$fruit" in
    apple)
        echo "Apples are red or green"
        ;;
    banana)
        echo "Bananas are yellow"
        ;;
    orange|tangerine)
        echo "Citrus fruit!"
        ;;
    *)
        echo "Unknown fruit: $fruit"
        ;;
esac
```

---

## 21.2 Pattern Matching in case

Case patterns use **glob syntax** (not regex):

```bash
case "$input" in
    [Yy]|[Yy]es)            # Y, y, Yes, yes
        echo "Confirmed"
        ;;
    [Nn]|[Nn]o)              # N, n, No, no
        echo "Denied"
        ;;
    *.txt)                    # Ends with .txt
        echo "Text file"
        ;;
    [0-9]*)                   # Starts with a digit
        echo "Starts with a number"
        ;;
    ???)                      # Exactly 3 characters
        echo "Three-character string"
        ;;
    "")                       # Empty string
        echo "Empty input"
        ;;
    *)
        echo "No match"
        ;;
esac
```

---

## 21.3 Practical Uses

### Parsing Command-Line Options

```bash
#!/bin/bash
while [[ $# -gt 0 ]]; do
    case "$1" in
        -v|--verbose)
            VERBOSE=true
            shift
            ;;
        -o|--output)
            OUTPUT="$2"
            shift 2
            ;;
        -h|--help)
            echo "Usage: $0 [-v] [-o file] [args...]"
            exit 0
            ;;
        --)
            shift
            break
            ;;
        -*)
            echo "Unknown option: $1" >&2
            exit 1
            ;;
        *)
            break
            ;;
    esac
done
```

### Menu System

```bash
#!/bin/bash
while true; do
    echo ""
    echo "=== System Management ==="
    echo "1) Show disk usage"
    echo "2) Show memory usage"
    echo "3) Show logged-in users"
    echo "4) Show system uptime"
    echo "q) Quit"
    echo ""
    read -rp "Choose an option: " choice

    case "$choice" in
        1) df -h ;;
        2) free -h ;;
        3) who ;;
        4) uptime ;;
        q|Q) echo "Goodbye!"; exit 0 ;;
        *) echo "Invalid option: $choice" ;;
    esac
done
```

### Service Control Script

```bash
#!/bin/bash
case "${1,,}" in    # ${1,,} converts to lowercase
    start)
        echo "Starting service..."
        ;;
    stop)
        echo "Stopping service..."
        ;;
    restart)
        echo "Restarting service..."
        ;;
    status)
        echo "Checking status..."
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status}" >&2
        exit 1
        ;;
esac
```

---

## 21.4 Fall-Through with ;;&  and ;&

Bash 4+ supports fall-through behavior:

```bash
# ;; — Normal: stop after first match (default)
# ;&  — Fall through: execute the next clause unconditionally
# ;;& — Continue: test the next pattern too

# Example with ;;&
case "$num" in
    *0) echo "Divisible by 10" ;;&    # Continue testing
    *[05]) echo "Divisible by 5" ;;&  # Continue testing
    *[02468]) echo "Even" ;;          # Stop
    *) echo "Odd" ;;
esac

# Input: 20
# Output:
# Divisible by 10
# Divisible by 5
# Even
```

---

## Common Mistakes

1. **Forgetting `;;`** — Each clause must end with `;;`, `;&`, or `;;&`.
2. **Forgetting `esac`** — `case` must be closed with `esac` (case spelled backward).
3. **Not quoting the variable** — `case $var in` risks word splitting. Use `case "$var" in`.
4. **Forgetting the `*` default** — Always handle unexpected input.

---

## Exercises

### Exercise 21.1: File Type Classifier
Write a script that takes a filename and uses `case` on its extension to classify it (`.txt` → text, `.sh` → script, `.jpg`/`.png` → image, etc.).

### Exercise 21.2: Interactive Calculator
Write a menu-driven calculator using `case`.

---

## Summary

- `case` matches a value against **glob patterns** — cleaner than long if/elif chains
- Use `|` for multiple patterns: `yes|Yes|YES)`
- Always include a `*)` default case
- Fall-through: `;&` (unconditional) and `;;&` (continue testing) — Bash 4+
- `case` is ideal for option parsing, menus, and dispatch logic

---

**Next Chapter:** [Chapter 22: Loops — for, while, until, select →](Chapter22-Loops.md)
