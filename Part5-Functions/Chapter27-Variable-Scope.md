# Chapter 27: Variable Scope

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand global vs local variable scope in Bash
- Use `local` to create function-scoped variables
- Avoid accidental side effects from global variables
- Use `declare` and namerefs for advanced scoping
- Apply best practices for variable management

---

## 27.1 Global Scope (The Default)

By default, **all variables in Bash are global**:

```bash
#!/bin/bash

name="Global"

change_name() {
    name="Changed Inside Function"    # Modifies the GLOBAL variable!
}

echo "$name"       # Global
change_name
echo "$name"       # Changed Inside Function  ← Side effect!
```

This is one of Bash's biggest pitfalls. Any variable you set inside a function is visible everywhere:

```bash
process() {
    i=0              # Accidentally creates/overwrites global 'i'
    result="done"    # Overwrites any existing global 'result'
}
```

---

## 27.2 Local Variables

Use `local` to restrict a variable to the function:

```bash
#!/bin/bash

name="Global"

change_name() {
    local name="Local"    # Creates a separate, function-scoped variable
    echo "Inside: $name"  # Local
}

echo "Before: $name"      # Global
change_name                # Inside: Local
echo "After: $name"       # Global  ← unchanged!
```

```
┌──────────────────────────────────────┐
│ Global Scope                         │
│   name="Global"                      │
│                                      │
│   ┌──────────────────────────────┐   │
│   │ change_name() scope          │   │
│   │   local name="Local"         │   │
│   │   (shadows global 'name')    │   │
│   └──────────────────────────────┘   │
│                                      │
│   name is still "Global" out here    │
└──────────────────────────────────────┘
```

---

## 27.3 Local Scope Rules

### Rule 1: Local Variables Are Visible to Called Functions

```bash
outer() {
    local secret="hidden"
    inner
}

inner() {
    echo "$secret"    # "hidden" — can see outer's local!
}

outer        # hidden
echo "$secret"    # (empty) — not visible at global scope
```

This is called **dynamic scoping** — Bash looks up the call stack for variables.

### Rule 2: Local Declarations Happen at Runtime

```bash
my_func() {
    echo "Before local: x=$x"    # Sees global x
    local x="local_value"
    echo "After local: x=$x"     # Sees local x
}

x="global_value"
my_func
# Before local: x=global_value
# After local: x=local_value
echo "Global: x=$x"    # global_value
```

### Rule 3: local Only Works Inside Functions

```bash
local var="test"    # ERROR: local: can only be used in a function
```

---

## 27.4 Common Patterns

### Pattern 1: Always Declare Local Variables

```bash
process_file() {
    local filename="$1"
    local line_count
    local status
    
    line_count=$(wc -l < "$filename")
    status=$?
    
    echo "$filename: $line_count lines"
    return $status
}
```

### Pattern 2: Local and Assignment Separately for Error Checking

```bash
# BAD — local masks the exit code of the command
my_func() {
    local result=$(failing_command)    # $? is always 0 (from local)
    echo "status: $?"                 # Always 0!
}

# GOOD — separate declaration and assignment
my_func() {
    local result
    result=$(failing_command)
    echo "status: $?"    # Actual exit code of failing_command
}
```

> **This is a critical gotcha!** `local var=$(cmd)` always returns 0 because `local` itself succeeds.

### Pattern 3: Loop Variables Should Be Local

```bash
# BAD — i leaks to global scope
count_files() {
    for i in "$dir"/*; do
        ((count++))
    done
}

# GOOD
count_files() {
    local i count=0
    for i in "$dir"/*; do
        ((count++))
    done
    echo "$count"
}
```

---

## 27.5 Namerefs (Bash 4.3+)

Namerefs let a function write to a variable whose name is passed as an argument:

```bash
get_system_info() {
    local -n hostname_ref="$1"
    local -n kernel_ref="$2"
    
    hostname_ref=$(hostname)
    kernel_ref=$(uname -r)
}

get_system_info my_host my_kernel
echo "Host: $my_host"
echo "Kernel: $my_kernel"
```

This avoids both global variables and subshell overhead from `$(func)`.

---

## 27.6 Subshell Isolation

Variables set in a subshell are invisible to the parent:

```bash
x="original"

(x="subshell_value")       # Runs in a subshell
echo "$x"                  # original — subshell changes don't propagate

# Pipes also create subshells
echo "hello" | read word
echo "$word"               # (empty) — read ran in subshell

# Solution: use process substitution or lastpipe
shopt -s lastpipe
echo "hello" | read word
echo "$word"               # hello (with lastpipe)
```

---

## 27.7 Best Practices

1. **Always use `local`** for variables inside functions
2. **Separate `local` declaration from assignment** when you need the exit code
3. **Use `readonly`** for constants: `readonly MAX_SIZE=1024`
4. **Use UPPERCASE** for global/environment variables, **lowercase** for local
5. **Use namerefs** instead of `eval` for indirect variable access
6. **Avoid global state** — pass data via arguments and return via stdout

---

## Exercises

### Exercise 27.1: Scope Tracing
Without running it, predict the output of this script, then verify:

```bash
x=1
f() { local x=2; g; echo "f: $x"; }
g() { echo "g: $x"; x=3; }
f
echo "global: $x"
```

### Exercise 27.2: Safe Accumulator
Write a function `sum_numbers` that accepts any number of arguments, uses only local variables, and returns the sum via stdout.

---

## Summary

- All Bash variables are **global by default** — this is a major source of bugs
- Use `local` inside functions to prevent variable leakage
- Bash uses **dynamic scoping**: called functions see the caller's locals
- **Separate `local` from assignment** to preserve exit codes: `local var; var=$(cmd)`
- Namerefs (`local -n`) allow functions to write to caller's variables
- Subshells create isolated variable environments

---

**Next Chapter:** [Chapter 28: Libraries and Sourcing →](Chapter28-Libraries-Sourcing.md)
