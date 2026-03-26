# Chapter 32: grep — Pattern Matching

## Learning Objectives

By the end of this chapter, you will be able to:

- Use `grep` to search for patterns in text
- Apply basic and extended regular expressions
- Use common grep options for real-world tasks
- Combine grep with other tools in pipelines
- Choose between grep variants (grep, egrep, fgrep, pcregrep)

---

## 32.1 Basic grep

```bash
grep "pattern" file          # Lines in file matching pattern
grep "error" server.log      # Find "error" in log file
grep -i "error" server.log   # Case-insensitive search
```

```
┌──────────────────────────────────────────────┐
│               grep workflow                   │
│                                              │
│  Input ──▶ Read line ──▶ Match? ─── Yes ──▶ Print │
│              ▲              │                │
│              │             No                │
│              │              │                │
│              └──── Next ◀───┘                │
└──────────────────────────────────────────────┘
```

---

## 32.2 Essential Options

```bash
# Case insensitive
grep -i "error" log.txt

# Invert match (lines that DON'T match)
grep -v "DEBUG" log.txt

# Count matches
grep -c "ERROR" log.txt              # 42

# Show line numbers
grep -n "TODO" source.py             # 15:# TODO: fix this

# Show only the matching part
grep -o '[0-9]\+' data.txt           # Just the numbers

# Recursive search in directories
grep -r "function" src/              # Search all files under src/
grep -rn "TODO" --include="*.py" .   # Only Python files

# Context: lines before/after match
grep -B 2 "error" log.txt           # 2 lines Before
grep -A 3 "error" log.txt           # 3 lines After
grep -C 2 "error" log.txt           # 2 lines Context (before + after)

# Quiet mode (just check if match exists)
grep -q "pattern" file && echo "Found" || echo "Not found"

# List filenames only
grep -rl "config" /etc/             # Files containing "config"
grep -rL "config" /etc/             # Files NOT containing "config"

# Fixed string (no regex interpretation)
grep -F "file.txt" log              # Treats . as literal dot
fgrep "file.txt" log                # Same thing
```

---

## 32.3 Regular Expressions with grep

### Basic Regular Expressions (BRE) — Default

```bash
grep "^root" /etc/passwd          # Lines starting with "root"
grep "bash$" /etc/passwd          # Lines ending with "bash"
grep "^$" file.txt                # Empty lines
grep "r..t" /etc/passwd           # r + any 2 chars + t
grep "go*d" file.txt              # gd, god, good, goood...
grep "[aeiou]" file.txt           # Lines with any vowel
grep "[0-9]\{3\}" file.txt        # Exactly 3 digits (BRE needs \{ \})
```

### Extended Regular Expressions (ERE) — grep -E or egrep

```bash
grep -E "error|warning|critical" log.txt    # Alternation
grep -E "^[0-9]{1,3}\.[0-9]{1,3}" log.txt  # IP-like pattern
grep -E "(ab)+" file.txt                     # One or more "ab"
grep -E "colou?r" file.txt                   # "color" or "colour"
grep -E "go+d" file.txt                      # god, good, goood (not gd)
```

### Regex Quick Reference

```
.        Any single character
^        Start of line
$        End of line
*        Zero or more of previous
+        One or more of previous (ERE)
?        Zero or one of previous (ERE)
[abc]    Any character in set
[^abc]   Any character NOT in set
[a-z]    Range
(ab)     Group (ERE)
|        Alternation (ERE)
{n}      Exactly n times (ERE)
{n,m}    Between n and m times (ERE)
\b       Word boundary
\w       Word character [a-zA-Z0-9_]
\d       Digit [0-9] (PCRE, use grep -P)
```

---

## 32.4 Practical Examples

```bash
# Find IP addresses in logs
grep -Eo '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' access.log

# Find email addresses
grep -Eio '[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}' contacts.txt

# Find lines with TODO/FIXME/HACK in source code
grep -rn -E "TODO|FIXME|HACK" --include="*.py" src/

# Find function definitions in Python
grep -n "def " mymodule.py

# Count errors per hour in a log
grep "ERROR" app.log | grep -oE "[0-9]{2}:[0-9]{2}" | sort | uniq -c

# Find processes by name
ps aux | grep "[n]ginx"    # The [n] trick avoids matching grep itself

# Show non-comment, non-empty lines in config
grep -v '^#' /etc/ssh/sshd_config | grep -v '^$'
# Or combined:
grep -Ev '^\s*(#|$)' /etc/ssh/sshd_config

# Find files larger than expected (in du output)
du -ah /var | grep -E "^[0-9]+G"     # Gigabyte-sized items
```

---

## 32.5 grep with Pipelines

```bash
# Top 10 IPs hitting your server
cat access.log | grep "POST" | awk '{print $1}' | sort | uniq -c | sort -rn | head -10

# Find which config files mention a specific setting
grep -rl "MaxRetries" /etc/ 2>/dev/null

# Count occurrences of each HTTP status code
grep -oE "HTTP/[0-9.]+ [0-9]+" access.log | awk '{print $2}' | sort | uniq -c | sort -rn

# Multi-line matching with context
grep -B5 -A5 "FATAL" application.log
```

---

## 32.6 grep -P (PCRE)

Perl-Compatible Regular Expressions (where available):

```bash
# Lookahead/lookbehind
grep -P '(?<=price: )\d+' catalog.txt     # Number after "price: "
grep -P '\d+(?= USD)' prices.txt          # Number before " USD"

# Non-greedy matching
grep -Po '".*?"' data.json                # Shortest quoted strings

# Named character classes
grep -P '\d{3}-\d{3}-\d{4}' phones.txt   # Phone numbers
```

---

## 32.7 Common Mistakes

### Mistake 1: Forgetting to Quote Patterns

```bash
grep hello world file.txt     # Searches for "hello" in TWO files: world and file.txt
grep "hello world" file.txt   # Searches for "hello world" in file.txt
```

### Mistake 2: Confusing BRE and ERE

```bash
grep "a|b" file          # Looks for literal "a|b"
grep -E "a|b" file       # Looks for "a" OR "b"
```

### Mistake 3: grep Matching Itself in ps

```bash
ps aux | grep nginx      # Also matches "grep nginx"!
ps aux | grep "[n]ginx"  # Trick: [n]ginx matches nginx but not the grep command
```

---

## Exercises

### Exercise 32.1: Log Analysis
Given a web server log, write grep commands to: (a) find all 404 errors, (b) extract unique IP addresses, (c) find requests for .php files, (d) count requests per HTTP method.

### Exercise 32.2: Code Search
Write a script that searches a source directory for: TODO comments, functions longer than 50 lines, hardcoded IP addresses, and print statements.

---

## Summary

- `grep` searches for patterns line by line
- Key options: `-i` (case), `-v` (invert), `-n` (line numbers), `-r` (recursive), `-c` (count), `-o` (only matching)
- Use `-E` for extended regex (alternation with `|`, `+`, `?`, `{n}`)
- Use `-F` for fixed/literal string matching (faster)
- Use context options (`-A`, `-B`, `-C`) to see surrounding lines
- Combine with pipes for powerful text analysis

---

**Next Chapter:** [Chapter 33: sed — Stream Editing →](Chapter33-Sed.md)
