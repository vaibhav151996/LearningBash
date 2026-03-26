# Chapter 50: Debugging Techniques

## Learning Objectives

By the end of this chapter, you will be able to:

- Use Bash's built-in debugging tools
- Apply systematic debugging strategies
- Create debug-enabled scripts with toggle support
- Use external debugging tools
- Trace and diagnose common script issues

---

## 50.1 Debug Mode (set -x)

The most powerful debugging tool in Bash:

```bash
#!/bin/bash
set -x    # Enable: prints every command before execution

name="Alice"
greeting="Hello, $name"
echo "$greeting"

# Output:
# + name=Alice
# + greeting='Hello, Alice'
# + echo 'Hello, Alice'
# Hello, Alice
```

### Selective Debugging

```bash
# Debug only a section
set -x
problematic_function
set +x

# Debug a specific function
my_func() {
    set -x
    # ... function body ...
    set +x
}
```

---

## 50.2 Custom Debug Prompt (PS4)

```bash
# Default PS4 is "+ "
# Customize to show file, line, and function:
export PS4='+(${BASH_SOURCE[0]}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'

set -x
my_function
# +(script.sh:42): main(): my_function
# +(script.sh:15): my_function(): local result=data
```

### Other Useful PS4 Formats

```bash
# With timestamp
export PS4='+[$(date +%H:%M:%S)] ${BASH_SOURCE}:${LINENO}: '

# Minimal with line numbers
export PS4='+${LINENO}: '

# Full call stack
export PS4='+${BASH_SOURCE}:${LINENO} ${FUNCNAME[*]}: '
```

---

## 50.3 Togglable Debug Mode

```bash
#!/bin/bash

DEBUG="${DEBUG:-false}"

debug() {
    [[ "$DEBUG" == true ]] && echo "DEBUG: $*" >&2
}

my_function() {
    debug "Entering my_function with args: $*"
    local result
    result=$(process "$1")
    debug "Result: $result"
    echo "$result"
}

# Usage:
# ./script.sh              Normal run
# DEBUG=true ./script.sh    Debug mode
```

### Verbose Levels

```bash
VERBOSITY="${VERBOSITY:-0}"

log()   { echo "$*"; }
debug() { (( VERBOSITY >= 1 )) && echo "DEBUG: $*" >&2 || true; }
trace() { (( VERBOSITY >= 2 )) && echo "TRACE: $*" >&2 || true; }

# Usage:
# VERBOSITY=0 ./script.sh     Normal
# VERBOSITY=1 ./script.sh     Debug messages
# VERBOSITY=2 ./script.sh     Trace + Debug
```

---

## 50.4 The BASH_* Debug Variables

```bash
echo "$BASH_SOURCE"      # Current script file path
echo "$BASH_LINENO"      # Line number of calling function
echo "$FUNCNAME"         # Current function name
echo "${FUNCNAME[@]}"    # Function call stack
echo "$BASH_COMMAND"     # Command about to execute (in traps)
echo "$LINENO"           # Current line number
echo "$BASH_SUBSHELL"    # Subshell nesting level

# Print a stack trace
stacktrace() {
    local i
    echo "Stack trace:" >&2
    for ((i=1; i<${#FUNCNAME[@]}; i++)); do
        echo "  $i: ${BASH_SOURCE[$i]}: line ${BASH_LINENO[$((i-1))]}: ${FUNCNAME[$i]}()" >&2
    done
}

# Auto-stacktrace on error
trap 'echo "Error at line $LINENO: $BASH_COMMAND" >&2; stacktrace' ERR
```

---

## 50.5 ERR Trap for Automatic Error Reporting

```bash
#!/bin/bash
set -euo pipefail

on_error() {
    local exit_code=$?
    local line_no=$1
    local command="$BASH_COMMAND"
    
    echo "═══════════════════════════════════" >&2
    echo "ERROR in ${BASH_SOURCE[1]}:$line_no" >&2
    echo "Command: $command" >&2
    echo "Exit code: $exit_code" >&2
    echo "═══════════════════════════════════" >&2
    
    # Print context (surrounding lines)
    if [[ -f "${BASH_SOURCE[1]}" ]]; then
        echo "Context:" >&2
        awk -v line="$line_no" 'NR>=line-2 && NR<=line+2 {
            printf "%s %4d: %s\n", (NR==line ? ">>>" : "   "), NR, $0
        }' "${BASH_SOURCE[1]}" >&2
    fi
}

trap 'on_error $LINENO' ERR
```

---

## 50.6 Common Debugging Strategies

### Strategy 1: Binary Search

```bash
# Narrow down the problem by commenting out halves:
main() {
    step_1     # Works?
    step_2     # Works?
    step_3     # Problem here?
    step_4
    step_5
}
# Comment out step_3-5, test. Then narrow further.
```

### Strategy 2: Echo Checkpoints

```bash
echo "CHECKPOINT 1: before processing" >&2
process_data
echo "CHECKPOINT 2: after processing, status=$?" >&2
validate_results
echo "CHECKPOINT 3: after validation" >&2
```

### Strategy 3: Variable Inspection

```bash
# Dump variable state
dump_vars() {
    echo "=== Variable Dump ===" >&2
    echo "  PWD=$PWD" >&2
    echo "  file=$file" >&2
    echo "  count=$count" >&2
    echo "  args=($*)" >&2
    echo "===================" >&2
}
```

### Strategy 4: Trap DEBUG

```bash
# Run a command before every line
trap 'echo "[$LINENO] $BASH_COMMAND"' DEBUG

# Or conditional:
trap '[[ $LINENO -ge 20 && $LINENO -le 30 ]] && echo "L$LINENO: $BASH_COMMAND"' DEBUG
```

---

## 50.7 External Debugging Tools

### bashdb (Bash Debugger)

```bash
# Install
sudo apt install bashdb

# Run
bashdb script.sh

# Commands (gdb-like):
# n        Next line
# s        Step into function
# c        Continue
# b 20     Breakpoint at line 20
# p $var   Print variable
# bt       Backtrace
# q        Quit
```

### Syntax Checking

```bash
# Bash syntax check
bash -n script.sh

# ShellCheck (comprehensive lint)
shellcheck script.sh

# Check specific issues
shellcheck -e SC2086 script.sh   # Exclude specific warning
```

### strace — System Call Tracing

```bash
# See what system calls a script makes
strace -f bash script.sh         # Follow forks
strace -e open bash script.sh   # Only file opens
strace -e network bash script.sh # Only network calls
```

---

## 50.8 Debugging Checklist

```
When a script fails:
1. □ Read the error message carefully
2. □ Check which line failed (add set -x or echo)
3. □ Verify variable values at the point of failure
4. □ Check file existence and permissions
5. □ Test the failing command manually
6. □ Verify quoting (run with set -x to see expansion)
7. □ Check for whitespace issues (spaces in filenames, CRLF)
8. □ Run shellcheck for static analysis
9. □ Test in isolation (extract and test the failing section)
10. □ Check environment differences (PATH, shell version)
```

---

## Exercises

### Exercise 50.1: Debug a Broken Script
Given an intentionally buggy script, use debugging techniques to find and fix all 5 bugs.

### Exercise 50.2: Debug Framework
Build a reusable debug library that provides: togglable debug messages, stack traces on error, variable inspection, and automatic error context display.

---

## Summary

- `set -x` is the primary debugging tool — prints commands before execution
- Customize `PS4` for richer debug output (file, line, function)
- Use `trap 'handler' ERR` for automatic error reporting with context
- `BASH_SOURCE`, `LINENO`, `FUNCNAME` provide runtime introspection
- Toggle debugging with `DEBUG=true` environment variable
- Use `shellcheck` for static analysis — catch bugs before runtime
- When stuck: binary search, echo checkpoints, variable dumps

---

**Next Chapter:** [Chapter 51: Performance and Optimization →](Chapter51-Performance.md)
