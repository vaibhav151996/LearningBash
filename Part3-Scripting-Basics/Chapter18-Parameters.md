# Chapter 18: Positional Parameters and Special Variables

## Learning Objectives

By the end of this chapter, you will be able to:

- Pass arguments to scripts and functions
- Use positional parameters `$1`, `$2`, ..., `$#`, `$@`, `$*`
- Understand the difference between `"$@"` and `"$*"`
- Use `shift` to process arguments
- Implement argument parsing with `getopts`

---

## 18.1 Positional Parameters

When you run a script with arguments, they're accessible as numbered variables:

```bash
./script.sh hello world 42
```

```bash
#!/bin/bash
echo "Script name: $0"       # ./script.sh
echo "First arg:   $1"       # hello
echo "Second arg:  $2"       # world
echo "Third arg:   $3"       # 42
echo "Arg count:   $#"       # 3
echo "All args:    $@"       # hello world 42
echo "All args:    $*"       # hello world 42
```

### Arguments Beyond $9

For arguments 10 and beyond, use braces:

```bash
echo "${10}"     # Tenth argument
echo "${11}"     # Eleventh argument
# Without braces: $10 is interpreted as ${1}0
```

---

## 18.2 Special Variables

| Variable | Meaning |
|----------|---------|
| `$0` | The script's name/path |
| `$1`-`$9` | Positional parameters 1-9 |
| `${10}+` | Positional parameters 10+ |
| `$#` | Number of arguments |
| `$@` | All arguments (as separate words when quoted) |
| `$*` | All arguments (as single string when quoted) |
| `$?` | Exit status of last command |
| `$$` | PID of current shell/script |
| `$!` | PID of last background command |
| `$-` | Current shell options |
| `$_` | Last argument of previous command |

---

## 18.3 `"$@"` vs `"$*"` — The Critical Difference

```bash
# Script called with: ./test.sh "hello world" foo "bar baz"

# "$@" preserves each argument as separate words
for arg in "$@"; do
    echo "Arg: [$arg]"
done
# Arg: [hello world]
# Arg: [foo]
# Arg: [bar baz]

# "$*" joins all arguments into ONE string (separated by first char of IFS)
for arg in "$*"; do
    echo "Arg: [$arg]"
done
# Arg: [hello world foo bar baz]
```

**Rule: Almost always use `"$@"`**, not `"$*"`. It preserves argument boundaries.

```bash
# Pass all arguments to another command (correctly)
wrapper() {
    some_command "$@"     # CORRECT — preserves argument structure
}

# WRONG:
wrapper() {
    some_command $@       # Breaks on arguments with spaces
    some_command "$*"     # Merges all arguments into one
}
```

---

## 18.4 The shift Command

`shift` removes the first argument, shifting all others down by one:

```bash
#!/bin/bash
echo "Before shift: $1 $2 $3    (\$# = $#)"
shift
echo "After shift:  $1 $2       (\$# = $#)"
shift
echo "After shift:  $1          (\$# = $#)"
```

```
$ ./test.sh A B C
Before shift: A B C    ($# = 3)
After shift:  B C      ($# = 2)
After shift:  C        ($# = 1)
```

### Processing Arguments with shift

```bash
#!/bin/bash
# Simple argument processor
while [ $# -gt 0 ]; do
    case "$1" in
        -v|--verbose)
            VERBOSE=true
            ;;
        -o|--output)
            OUTPUT="$2"
            shift        # Extra shift to skip the value
            ;;
        -h|--help)
            echo "Usage: $0 [-v] [-o file] [args...]"
            exit 0
            ;;
        --)
            shift
            break        # Stop processing options
            ;;
        -*)
            echo "Unknown option: $1" >&2
            exit 1
            ;;
        *)
            break        # Non-option argument — stop processing
            ;;
    esac
    shift
done

# Remaining arguments are in $@
echo "Verbose: ${VERBOSE:-false}"
echo "Output: ${OUTPUT:-stdout}"
echo "Remaining args: $@"
```

---

## 18.5 Argument Parsing with getopts

`getopts` is the built-in tool for parsing short options:

```bash
#!/bin/bash
# getopts handles: -v, -o filename, -h

verbose=false
output=""

while getopts ":vo:h" opt; do
    case $opt in
        v) verbose=true ;;
        o) output="$OPTARG" ;;
        h)
            echo "Usage: $0 [-v] [-o file] [args...]"
            exit 0
            ;;
        \?)
            echo "Invalid option: -$OPTARG" >&2
            exit 1
            ;;
        :)
            echo "Option -$OPTARG requires an argument" >&2
            exit 1
            ;;
    esac
done

# Shift past processed options
shift $((OPTIND - 1))

echo "Verbose: $verbose"
echo "Output: $output"
echo "Remaining args: $@"
```

The option string `":vo:h"`:
- Leading `:` enables silent error handling
- `v` — flag (no argument)
- `o:` — option with required argument (trailing colon)
- `h` — flag (no argument)

```bash
./script.sh -v -o report.txt file1.txt file2.txt
# Verbose: true
# Output: report.txt
# Remaining args: file1.txt file2.txt

./script.sh -vo report.txt    # Combined short options
# Same result
```

---

## 18.6 Checking for Required Arguments

```bash
#!/bin/bash
# Require at least one argument
if [ $# -eq 0 ]; then
    echo "Usage: $0 <filename> [options...]" >&2
    exit 1
fi

filename="$1"

# Check file exists
if [ ! -f "$filename" ]; then
    echo "Error: File not found: $filename" >&2
    exit 1
fi

echo "Processing $filename..."
```

---

## 18.7 Default Values for Parameters

```bash
# ${var:-default} — use default if var is unset or empty
output="${1:-/tmp/output.txt}"

# ${var:=default} — assign default if var is unset or empty
: "${EDITOR:=vim}"    # Set EDITOR to vim if not already set

# ${var:?message} — error if var is unset or empty
input="${1:?Error: filename required}"
# If $1 is empty: "bash: 1: Error: filename required"
```

---

## Common Mistakes

1. **Using `$*` instead of `"$@"`** — Almost always wrong. `"$@"` preserves argument boundaries.

2. **Not quoting `"$@"`** — Unquoted `$@` undergoes word splitting. Always use `"$@"`.

3. **Accessing `$10` without braces** — `$10` is `${1}0`. Use `${10}`.

4. **Not checking `$#`** — Always validate that required arguments were provided.

---

## Exercises

### Exercise 18.1: Argument Printer
Write a script that prints each argument on its own line with its position number.

### Exercise 18.2: Simple Calculator
Write a script that takes three arguments: number, operator (+,-,*,/), number, and prints the result.

### Exercise 18.3: Option Parsing
Write a script with getopts that accepts `-n name`, `-a age`, and `-v` (verbose), and prints a profile.

---

## Summary

- `$1`-`$9`, `${10}+` are positional parameters (arguments)
- `$#` is the argument count; `$0` is the script name
- **`"$@"`** preserves each argument separately — use this almost always
- `"$*"` joins all arguments into one string
- `shift` removes the first argument and renumbers the rest
- `getopts` provides structured option parsing for short flags
- Always validate arguments with `$#` checks and default values

---

**Next Chapter:** [Chapter 19: Comments, Style, and Readability →](Chapter19-Comments-Style.md)
