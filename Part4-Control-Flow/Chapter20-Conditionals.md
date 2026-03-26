# Chapter 20: Conditional Statements — if, elif, else

## Learning Objectives

By the end of this chapter, you will be able to:

- Write `if`/`elif`/`else` statements
- Use `test`, `[ ]`, and `[[ ]]` for conditional checks
- Perform string, numeric, and file comparisons
- Understand the difference between `[ ]` and `[[ ]]`
- Combine conditions with logical operators

---

## 20.1 The if Statement

```bash
if COMMAND; then
    # Commands run if COMMAND exits with status 0
fi
```

`if` evaluates the **exit code** of a command — not a "boolean expression." Any command that returns 0 is "true."

```bash
# The simplest if
if true; then
    echo "This always runs"
fi

# Using a real command
if grep -q "root" /etc/passwd; then
    echo "root user exists"
fi

# Check if a directory exists
if [ -d "/var/log" ]; then
    echo "/var/log exists"
fi
```

### if/else

```bash
if [ -f "$filename" ]; then
    echo "File exists"
else
    echo "File not found"
fi
```

### if/elif/else

```bash
if [ "$score" -ge 90 ]; then
    grade="A"
elif [ "$score" -ge 80 ]; then
    grade="B"
elif [ "$score" -ge 70 ]; then
    grade="C"
elif [ "$score" -ge 60 ]; then
    grade="D"
else
    grade="F"
fi
echo "Grade: $grade"
```

---

## 20.2 Test Commands: `[ ]` vs `[[ ]]`

### `[ ]` — POSIX test

`[` is actually a command (an alias for `test`). The `]` is required as its last argument.

```bash
# These are identical:
test -f /etc/passwd
[ -f /etc/passwd ]

# Spaces are REQUIRED around [ and ] — they're part of the command syntax
[ -f /etc/passwd ]     # CORRECT
[-f /etc/passwd]       # WRONG — bash tries to run "[-f" command
```

### `[[ ]]` — Bash Extended Test

`[[ ]]` is a Bash keyword with additional features:

```bash
# Pattern matching
[[ "$name" == J* ]]          # True if name starts with J

# Regex matching
[[ "$email" =~ ^[a-z]+@[a-z]+\.[a-z]+$ ]]

# No word splitting — safe without quotes (but still quote anyway)
[[ $var == "hello" ]]        # Works even if var is empty

# Logical operators inside
[[ -f "$file" && -r "$file" ]]    # AND
[[ "$a" == "x" || "$a" == "y" ]]  # OR
```

### Comparison: `[ ]` vs `[[ ]]`

| Feature | `[ ]` | `[[ ]]` |
|---------|-------|---------|
| POSIX compatible | Yes | No (Bash-only) |
| Word splitting | Yes (must quote) | No |
| Glob patterns | No | `==` and `!=` do pattern matching |
| Regex | No | `=~` operator |
| `&&` and `||` inside | No (use `-a`, `-o`) | Yes |
| Empty variable safe | No (need quotes) | Yes |

**Recommendation:** Use `[[ ]]` in Bash scripts. Use `[ ]` only if POSIX compatibility is required.

---

## 20.3 String Comparisons

```bash
# Equality
[[ "$str" == "hello" ]]       # Equal (Bash)
[ "$str" = "hello" ]          # Equal (POSIX — single =)

# Inequality
[[ "$str" != "hello" ]]

# Empty string
[[ -z "$str" ]]               # True if empty (zero length)
[[ -n "$str" ]]               # True if not empty

# Pattern matching (only in [[ ]])
[[ "$file" == *.txt ]]        # Glob pattern
[[ "$input" == [Yy]* ]]       # Starts with Y or y

# Regex matching (only in [[ ]])
[[ "$email" =~ ^[[:alnum:]]+@[[:alnum:]]+\.[a-z]{2,}$ ]]

# Lexicographic comparison
[[ "$a" < "$b" ]]             # a sorts before b
[[ "$a" > "$b" ]]             # a sorts after b
```

---

## 20.4 Numeric Comparisons

Inside `[ ]` and `[[ ]]`, use the following operators for numbers:

```bash
[[ "$a" -eq "$b" ]]    # Equal
[[ "$a" -ne "$b" ]]    # Not equal
[[ "$a" -lt "$b" ]]    # Less than
[[ "$a" -le "$b" ]]    # Less than or equal
[[ "$a" -gt "$b" ]]    # Greater than
[[ "$a" -ge "$b" ]]    # Greater than or equal
```

Or use arithmetic evaluation `(( ))`:

```bash
# Inside (( )), use familiar operators
(( a == b ))
(( a != b ))
(( a < b ))
(( a <= b ))
(( a > b ))
(( a >= b ))

# Example
age=25
if (( age >= 18 )); then
    echo "Adult"
fi
```

**Warning:** Don't use `==`, `<`, `>` for numbers inside `[ ]` — they do string comparison!

```bash
# BUG: string comparison
[ "9" \> "10" ]     # TRUE! "9" > "1" lexicographically!

# CORRECT: numeric comparison
[ 9 -gt 10 ]        # FALSE — proper numeric comparison
```

---

## 20.5 File Test Operators

```bash
# File existence and type
[[ -e "$path" ]]      # Exists (any type)
[[ -f "$path" ]]      # Exists and is a regular file
[[ -d "$path" ]]      # Exists and is a directory
[[ -L "$path" ]]      # Exists and is a symbolic link
[[ -p "$path" ]]      # Exists and is a named pipe

# File permissions
[[ -r "$path" ]]      # Readable
[[ -w "$path" ]]      # Writable
[[ -x "$path" ]]      # Executable

# File properties
[[ -s "$path" ]]      # Exists and is not empty (size > 0)
[[ -O "$path" ]]      # Owned by current user
[[ -G "$path" ]]      # Owned by current group

# File comparisons
[[ "$file1" -nt "$file2" ]]   # file1 is newer than file2
[[ "$file1" -ot "$file2" ]]   # file1 is older than file2
[[ "$file1" -ef "$file2" ]]   # Same file (same inode)
```

### Practical Example

```bash
#!/bin/bash
config_file="/etc/myapp/config.ini"

if [[ ! -f "$config_file" ]]; then
    echo "Config file not found: $config_file" >&2
    exit 1
elif [[ ! -r "$config_file" ]]; then
    echo "Config file not readable: $config_file" >&2
    exit 1
elif [[ ! -s "$config_file" ]]; then
    echo "Config file is empty: $config_file" >&2
    exit 1
fi

echo "Config file OK, loading..."
source "$config_file"
```

---

## 20.6 Combining Conditions

### With `[[ ]]`

```bash
# AND
if [[ -f "$file" && -r "$file" ]]; then
    echo "File exists and is readable"
fi

# OR
if [[ "$answer" == "y" || "$answer" == "Y" ]]; then
    echo "Confirmed"
fi

# NOT
if [[ ! -d "$dir" ]]; then
    echo "Not a directory"
fi

# Complex combinations
if [[ -f "$file" && ( -r "$file" || "$USER" == "root" ) ]]; then
    echo "Can access file"
fi
```

### With `[ ]`

```bash
# AND and OR with separate [ ] commands
if [ -f "$file" ] && [ -r "$file" ]; then
    echo "File exists and is readable"
fi

# Old-style (deprecated): -a and -o
if [ -f "$file" -a -r "$file" ]; then      # -a is AND (avoid this)
    echo "File exists and is readable"
fi
```

---

## 20.7 One-Line Conditionals

```bash
# Short form for simple checks
[[ -f "$file" ]] && echo "File exists"
[[ -f "$file" ]] || echo "File missing"
[[ -d "$dir" ]] && cd "$dir" || { echo "Cannot cd" >&2; exit 1; }
```

---

## Common Mistakes

1. **Missing spaces in `[ ]`** — `[$var = "test"]` is wrong. Needs spaces: `[ "$var" = "test" ]`.
2. **Using `==` in `[ ]`** — POSIX `test` uses `=` for string comparison. `==` works in Bash but isn't portable.
3. **Unquoted variables in `[ ]`** — `[ $var = "test" ]` breaks if var is empty. Use `[ "$var" = "test" ]`.
4. **Using `<` `>` for numbers** — Those are string comparisons. Use `-lt`, `-gt`, etc.
5. **Confusing `-a`/`-o` with `&&`/`||`** — Use `&&` and `||` between separate `[ ]` commands.

---

## Exercises

### Exercise 20.1: File Checker
Write a script that takes a filename and reports whether it's a file, directory, or symlink, and what permissions you have.

### Exercise 20.2: Number Classifier
Write a script that takes a number and outputs whether it's positive, negative, or zero, and whether it's even or odd.

### Exercise 20.3: Age Validator
Write a script that asks for an age and validates it (must be numeric, 0-150).

---

## Summary

- `if` tests the **exit code** of a command (0 = true, non-zero = false)
- `[[ ]]` is preferred in Bash (pattern matching, regex, no word splitting)
- String: `==`, `!=`, `-z`, `-n`, `<`, `>`
- Numbers: `-eq`, `-ne`, `-lt`, `-le`, `-gt`, `-ge` (or use `(( ))`)
- Files: `-f`, `-d`, `-r`, `-w`, `-x`, `-s`, `-e`
- Combine: `&&` (AND), `||` (OR), `!` (NOT) — inside `[[ ]]` or between tests

---

**Next Chapter:** [Chapter 21: Case Statements and Pattern Matching →](Chapter21-Case-Statements.md)
