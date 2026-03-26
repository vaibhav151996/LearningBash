# Chapter 23: Loop Control — break, continue, exit

## Learning Objectives

By the end of this chapter, you will be able to:

- Use `break` to exit loops early
- Use `continue` to skip iterations
- Use `exit` to terminate scripts with status codes
- Use `return` to exit functions
- Control nested loops with numbered `break` and `continue`

---

## 23.1 break — Exit a Loop

```bash
# Stop when we find what we're looking for
for file in /var/log/*; do
    if grep -q "CRITICAL" "$file" 2>/dev/null; then
        echo "Found critical error in: $file"
        break
    fi
done

# Exit an infinite loop
while true; do
    read -rp "Enter command (quit to exit): " cmd
    [[ "$cmd" == "quit" ]] && break
    echo "Running: $cmd"
done
```

### Breaking Nested Loops

`break N` breaks out of N levels of loops:

```bash
# break 2 exits both loops
for i in {1..5}; do
    for j in {1..5}; do
        if (( i * j > 12 )); then
            echo "Breaking at i=$i, j=$j"
            break 2    # Exit BOTH loops
        fi
        printf "%d×%d=%d  " "$i" "$j" "$((i * j))"
    done
done
echo "Done"
```

---

## 23.2 continue — Skip an Iteration

```bash
# Skip non-.txt files
for file in *; do
    [[ "$file" != *.txt ]] && continue
    echo "Processing: $file"
done

# Skip blank lines
while IFS= read -r line; do
    [[ -z "$line" ]] && continue      # Skip empty lines
    [[ "$line" == \#* ]] && continue  # Skip comments
    echo "Config: $line"
done < config.txt
```

### Continuing in Nested Loops

```bash
# continue 2 skips to the next iteration of the outer loop
for dir in /var/*/; do
    for file in "$dir"*.log; do
        [[ ! -f "$file" ]] && continue 2    # Skip to next dir
        process "$file"
    done
done
```

---

## 23.3 exit — Terminate the Script

```bash
#!/bin/bash
# exit stops the entire script

if [[ ! -f "$1" ]]; then
    echo "Error: File not found: $1" >&2
    exit 1    # Non-zero = failure
fi

echo "Processing..."
# ... work ...
exit 0        # Explicit success
```

### exit vs. return

| Command | Scope | Use Case |
|---------|-------|----------|
| `exit N` | Terminates the entire script/shell | End a script with status |
| `return N` | Exits the current function only | End a function with status |
| `break` | Exits the current loop | Leave a loop early |
| `continue` | Skips to the next iteration | Skip one loop iteration |

```bash
my_function() {
    [[ -z "$1" ]] && return 1    # Return from function, not exit script
    echo "Processing: $1"
    return 0
}

my_function ""
echo "Script continues"    # This still runs (return, not exit)
```

---

## Exercises

### Exercise 23.1: Search and Break
Write a script that searches for a file by name in `/etc` and breaks when found.

### Exercise 23.2: Skip and Continue
Write a loop over numbers 1-20 that skips multiples of 3.

---

## Summary

- `break` exits the current loop; `break N` exits N nested levels
- `continue` skips to the next iteration; `continue N` for nested loops
- `exit N` terminates the entire script with status N
- `return N` exits from a function with status N
- Use `exit 0` for success, `exit 1` for errors

---

**Next Chapter:** [Chapter 24: Error Handling Patterns →](Chapter24-Error-Handling.md)
