# Chapter 26: Functions

## Learning Objectives

By the end of this chapter, you will be able to:

- Define and call functions in Bash
- Pass arguments to functions and return values
- Understand function naming conventions
- Use functions to organize and reuse code
- Apply best practices for function design

---

## 26.1 What Is a Function?

A function is a reusable block of code that you can call by name. Functions eliminate code duplication and make scripts easier to read and maintain.

```
Without functions:              With functions:
┌────────────────────┐         ┌────────────────────┐
│ validate input     │         │ validate_input()    │←── defined once
│ process data       │         │ process_data()      │
│ ...                │         │ ...                 │
│ validate input     │  ←dup   │                     │
│ process data       │  ←dup   │ validate_input      │←── called again
│ ...                │         │ process_data        │
│ validate input     │  ←dup   │ validate_input      │←── called again
│ process data       │  ←dup   │ process_data        │
└────────────────────┘         └────────────────────┘
```

---

## 26.2 Defining Functions

Two equivalent syntaxes:

```bash
# Style 1: Modern (preferred)
greet() {
    echo "Hello, $1!"
}

# Style 2: With 'function' keyword
function greet {
    echo "Hello, $1!"
}

# Style 3: Both (redundant but valid)
function greet() {
    echo "Hello, $1!"
}
```

> **Best Practice:** Use `name() { ... }` (Style 1). It's POSIX-compatible, shorter, and the most common convention.

---

## 26.3 Calling Functions

```bash
greet() {
    echo "Hello, $1!"
}

# Call the function (no parentheses, no $ sign)
greet "World"        # Hello, World!
greet "Alice"        # Hello, Alice!

# Functions must be defined BEFORE they are called
# This order matters:
say_hi              # ERROR: command not found
say_hi() { echo "Hi!"; }
```

---

## 26.4 Function Arguments

Functions receive arguments just like scripts — via `$1`, `$2`, `$@`, etc.:

```bash
create_user() {
    local username="$1"
    local email="$2"
    local role="${3:-user}"    # Default to "user"
    
    echo "Creating user: $username"
    echo "  Email: $email"
    echo "  Role:  $role"
}

create_user "alice" "alice@example.com" "admin"
create_user "bob" "bob@example.com"              # role defaults to "user"
```

### Argument Reference Inside Functions

| Variable | Meaning |
|----------|---------|
| `$1`, `$2`... | Positional arguments to the **function** |
| `$@` | All arguments to the function |
| `$#` | Number of arguments passed to the function |
| `$0` | Still the script name (NOT the function name) |
| `$FUNCNAME` | The current function name |

```bash
show_info() {
    echo "Function: $FUNCNAME"
    echo "Arguments: $#"
    echo "All args: $@"
    echo "Script: $0"
}
show_info a b c
# Function: show_info
# Arguments: 3
# All args: a b c
# Script: ./myscript.sh
```

---

## 26.5 Return Values

### Return Codes (Exit Status)

`return` sets the function's exit status (0-255):

```bash
is_even() {
    (( $1 % 2 == 0 ))    # Returns 0 (true) or 1 (false)
}

file_exists() {
    [[ -f "$1" ]]
    return $?             # Explicit (but redundant — last command's status returned anyway)
}

# Using return values
if is_even 42; then
    echo "Even!"
fi

is_even 7 && echo "Even" || echo "Odd"
```

### Returning Data

Since `return` only handles 0-255, use other methods to return data:

```bash
# Method 1: echo (most common)
get_username() {
    echo "$(whoami)"
}
user=$(get_username)

# Method 2: printf (avoids trailing newline issues)
get_date() {
    printf '%s' "$(date +%Y-%m-%d)"
}
today=$(get_date)

# Method 3: Set a global variable (use with caution)
get_result() {
    RESULT="computed value"
}
get_result
echo "$RESULT"

# Method 4: Nameref (Bash 4.3+)
get_value() {
    local -n ref="$1"
    ref="computed value"
}
get_value myvar
echo "$myvar"    # computed value
```

> **Warning:** When using `echo` to return data, the function must NOT echo anything else (debug messages, prompts, etc.), or it will contaminate the return value.

```bash
# BAD — debug output captured in result
get_count() {
    echo "Counting..."          # This gets captured!
    echo "$(wc -l < "$1")"
}
result=$(get_count file.txt)    # result = "Counting...\n42"

# GOOD — debug to stderr, data to stdout
get_count() {
    echo "Counting..." >&2     # Goes to stderr, not captured
    echo "$(wc -l < "$1")"    # Only this is captured
}
result=$(get_count file.txt)   # result = "42"
```

---

## 26.6 One-Line Functions

```bash
# Simple wrappers
die()  { echo "ERROR: $*" >&2; exit 1; }
log()  { printf '[%s] %s\n' "$(date +%T)" "$*"; }
warn() { echo "WARNING: $*" >&2; }
usage() { echo "Usage: $0 [-v] [-f file] target" >&2; exit 1; }
```

---

## 26.7 Common Mistakes

### Mistake 1: Calling Functions with Parentheses

```bash
greet() { echo "Hello, $1!"; }

greet("Alice")    # SYNTAX ERROR!
greet "Alice"     # Correct
```

### Mistake 2: Using echo Return in Conditions

```bash
# This does NOT work as expected
check_file() {
    if [[ -f "$1" ]]; then
        echo "true"
    else
        echo "false"
    fi
}

# BAD — comparing strings
if [[ $(check_file "test.txt") == "true" ]]; then ...

# GOOD — use return codes
check_file() {
    [[ -f "$1" ]]
}
if check_file "test.txt"; then ...
```

### Mistake 3: Forgetting That Functions Share the Script's Variables

```bash
count=0
increment() {
    count=$((count + 1))    # Modifies the GLOBAL count!
}
increment
echo $count    # 1
```

---

## Exercises

### Exercise 26.1: Utility Functions
Write a set of utility functions: `to_upper`, `to_lower`, `trim` (remove leading/trailing whitespace), and `contains` (check if string contains substring).

### Exercise 26.2: Calculator
Write functions `add`, `subtract`, `multiply`, `divide` that take two numbers and print the result. Handle division by zero.

---

## Summary

- Functions are defined with `name() { commands; }` syntax
- Call functions by name: `funcname arg1 arg2` (no parentheses)
- Functions receive arguments via `$1`, `$2`, `$@`, `$#`
- `return N` sets exit status (0-255); use `echo`/`printf` for data
- Functions must be defined before they are called
- Send debug/log output to stderr (`>&2`) to keep stdout clean for return values

---

**Next Chapter:** [Chapter 27: Variable Scope →](Chapter27-Variable-Scope.md)
