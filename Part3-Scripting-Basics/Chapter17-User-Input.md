# Chapter 17: Reading User Input

## Learning Objectives

By the end of this chapter, you will be able to:

- Read interactive input from users with `read`
- Implement prompts, timeouts, and default values
- Validate user input
- Read input securely (passwords, sensitive data)
- Read from files line by line

---

## 17.1 The read Command

`read` reads a line from standard input and assigns it to one or more variables.

```bash
# Basic usage
echo "What is your name?"
read name
echo "Hello, $name!"

# Prompt on the same line with -p
read -p "Enter your name: " name
echo "Hello, $name!"

# Read multiple variables
read -p "Enter first and last name: " first last
echo "First: $first, Last: $last"

# If more words than variables, the last variable gets the rest
echo "one two three four" | read a b c
# a=one, b=two, c="three four"
```

---

## 17.2 Useful read Options

```bash
# -p: Prompt string
read -p "Username: " username

# -s: Silent (don't show input — for passwords)
read -sp "Password: " password
echo    # Add newline after hidden input
echo "You entered ${#password} characters"

# -t: Timeout (seconds)
read -t 10 -p "Enter value (10s timeout): " value
if [ $? -ne 0 ]; then
    echo "Timed out!"
fi

# -n: Read exactly N characters (no Enter needed)
read -n 1 -p "Continue? (y/n) " answer
echo
if [ "$answer" = "y" ]; then
    echo "Continuing..."
fi

# -r: Raw input (don't interpret backslashes)
read -r line    # ALWAYS use -r unless you specifically want backslash processing

# -a: Read into an array
read -ra words -p "Enter words: "
echo "First word: ${words[0]}"
echo "Count: ${#words[@]}"

# -d: Change delimiter
read -d ':' field1 field2 <<< "hello:world:"

# -e: Use readline (with tab completion and history)
read -e -p "Enter path: " filepath
```

### The -r Flag Is Critical

```bash
# WITHOUT -r: backslashes are interpreted
echo "hello\nworld" | read line
# line = "hellonworld" (backslash-n was interpreted)

# WITH -r: backslashes are literal
echo "hello\nworld" | read -r line
# line = "hello\nworld" (preserved as-is)
```

**Always use `read -r`** unless you have a specific reason not to.

---

## 17.3 Default Values

```bash
# Provide a default if the user presses Enter without typing
read -p "Server [localhost]: " server
server="${server:-localhost}"
echo "Using server: $server"

# Pattern: prompt with default
read_with_default() {
    local prompt="$1"
    local default="$2"
    local varname="$3"
    
    read -rp "${prompt} [${default}]: " value
    eval "$varname='${value:-$default}'"
}

read_with_default "Hostname" "localhost" hostname
read_with_default "Port" "8080" port
echo "Connecting to $hostname:$port"
```

---

## 17.4 Input Validation

```bash
#!/bin/bash
# Validate numeric input
while true; do
    read -rp "Enter a number (1-100): " num
    
    # Check if it's a number
    if ! [[ "$num" =~ ^[0-9]+$ ]]; then
        echo "Error: Not a number" >&2
        continue
    fi
    
    # Check range
    if (( num < 1 || num > 100 )); then
        echo "Error: Out of range (1-100)" >&2
        continue
    fi
    
    break
done
echo "You entered: $num"

# Validate yes/no
ask_yes_no() {
    while true; do
        read -rp "$1 (yes/no): " answer
        case "${answer,,}" in    # ${answer,,} converts to lowercase
            yes|y) return 0 ;;
            no|n)  return 1 ;;
            *)     echo "Please answer yes or no." ;;
        esac
    done
}

if ask_yes_no "Do you want to continue?"; then
    echo "Proceeding..."
else
    echo "Aborted."
fi
```

---

## 17.5 Reading Files Line by Line

The most important use of `read` in scripts:

```bash
# Read a file line by line
while IFS= read -r line; do
    echo "Line: $line"
done < data.txt

# IFS= prevents stripping leading/trailing whitespace
# -r prevents backslash interpretation
# This combination is the CORRECT way to read files line by line
```

### Why `IFS= read -r`?

```bash
# File content: "  hello  world  "

# WRONG: loses whitespace
while read line; do echo "[$line]"; done < file.txt
# [hello  world]  ← leading/trailing spaces stripped

# CORRECT: preserves whitespace
while IFS= read -r line; do echo "[$line]"; done < file.txt
# [  hello  world  ]  ← everything preserved
```

### Reading Specific Fields

```bash
# Parse /etc/passwd (colon-separated)
while IFS=: read -r user _ uid gid _ home shell; do
    echo "User: $user, UID: $uid, Shell: $shell"
done < /etc/passwd

# Parse CSV
while IFS=, read -r name age city; do
    echo "$name is $age years old, lives in $city"
done < people.csv
```

---

## 17.6 The Pipe-Read Pitfall

```bash
# BUG: read in a pipeline runs in a subshell!
echo "hello" | read -r word
echo "$word"    # EMPTY! The subshell's variable is gone.

# FIX 1: Use a here-string
read -r word <<< "hello"
echo "$word"    # hello

# FIX 2: Use process substitution
while IFS= read -r line; do
    echo "$line"
done < <(some_command)

# FIX 3: Use lastpipe (Bash 4.2+)
shopt -s lastpipe
echo "hello" | read -r word
echo "$word"    # hello
```

---

## Common Mistakes

1. **Forgetting `-r`** — Always use `read -r` unless you want backslash processing.
2. **Forgetting `IFS=`** — When reading lines, `IFS=` prevents whitespace stripping.
3. **Reading in a pipeline** — `cmd | read var` loses the variable. Use `<<<` or process substitution.
4. **Not validating input** — Never trust user input. Always validate before using.

---

## Exercises

### Exercise 17.1: Interactive Script
Write a script that asks for the user's name, age, and favorite color, then prints a formatted summary.

### Exercise 17.2: Password Prompt
Write a script that asks for a password twice and verifies they match.

### Exercise 17.3: File Processing
Write a script that reads a file line by line and numbers each line (like `cat -n`).

---

## Summary

- `read -rp "prompt: " var` reads user input into a variable
- **Always use `-r`** to prevent backslash interpretation
- Use `-s` for passwords, `-t` for timeouts, `-n` for character counts
- Default values: `var="${var:-default}"`
- Read files with: `while IFS= read -r line; do ... done < file`
- Pipe-read pitfall: use `<<<` or process substitution instead

---

**Next Chapter:** [Chapter 18: Positional Parameters and Special Variables →](Chapter18-Parameters.md)
