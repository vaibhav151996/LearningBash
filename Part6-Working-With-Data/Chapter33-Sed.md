# Chapter 33: sed — Stream Editing

## Learning Objectives

By the end of this chapter, you will be able to:

- Use `sed` for find-and-replace operations
- Apply sed commands for insertion, deletion, and transformation
- Use addresses to target specific lines
- Combine multiple sed operations
- Edit files in-place safely

---

## 33.1 What Is sed?

`sed` (Stream EDitor) reads input line by line, applies transformations, and writes to stdout. It **does not modify the original file** unless you use `-i`.

```bash
sed 'command' input.txt          # Process file, output to stdout
echo "hello world" | sed 's/hello/hi/'   # Process from pipe
```

---

## 33.2 Substitution (s command)

The most common sed operation:

```bash
sed 's/old/new/' file.txt        # Replace first occurrence per line
sed 's/old/new/g' file.txt       # Replace ALL occurrences per line
sed 's/old/new/gi' file.txt      # Case-insensitive, all occurrences
sed 's/old/new/2' file.txt       # Replace only 2nd occurrence per line
```

### Delimiter Choice

```bash
# Default delimiter is /
sed 's/path/to/old/path/to/new/g'   # BROKEN — too many slashes!

# Use any delimiter:
sed 's|/path/to/old|/path/to/new|g'   # | as delimiter
sed 's#/old/path#/new/path#g'          # # as delimiter
sed 's@pattern@replacement@g'          # @ as delimiter
```

### Capture Groups

```bash
# Swap first and last name
echo "John Smith" | sed 's/\(.*\) \(.*\)/\2, \1/'
# Smith, John

# Extended regex (-E) for cleaner groups
echo "John Smith" | sed -E 's/(.*) (.*)/\2, \1/'
# Smith, John

# Add prefix to numbers
echo "42" | sed -E 's/([0-9]+)/Number: \1/'
# Number: 42

# Extract parts
echo "2024-01-15" | sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/'
# 15/01/2024
```

---

## 33.3 Addressing — Targeting Specific Lines

```bash
# By line number
sed '3s/old/new/' file.txt           # Only line 3
sed '1,5s/old/new/' file.txt         # Lines 1 through 5
sed '$s/old/new/' file.txt           # Last line only

# By pattern
sed '/error/s/old/new/' file.txt     # Lines containing "error"
sed '/^#/d' file.txt                 # Delete comment lines
sed '/start/,/end/s/old/new/' file   # Range: from "start" to "end"

# Negation
sed '/^#/!s/foo/bar/g' file.txt      # Replace foo with bar except in comments
```

---

## 33.4 Common sed Commands

```bash
# Delete lines (d)
sed '5d' file.txt                    # Delete line 5
sed '1,3d' file.txt                  # Delete lines 1-3
sed '/pattern/d' file.txt            # Delete lines matching pattern
sed '/^$/d' file.txt                 # Delete empty lines
sed '/^#/d' file.txt                 # Delete comment lines

# Print specific lines (p) — use with -n
sed -n '5p' file.txt                 # Print only line 5
sed -n '10,20p' file.txt             # Print lines 10-20
sed -n '/error/p' file.txt           # Print lines matching "error"

# Insert / Append text
sed '3i\New line above 3' file.txt   # Insert before line 3
sed '3a\New line below 3' file.txt   # Append after line 3
sed '1i\#!/bin/bash' script.sh       # Add shebang at top

# Change entire line (c)
sed '3c\Replacement line' file.txt   # Replace line 3 entirely

# Transliterate (y) — character-by-character replacement
sed 'y/abc/ABC/' file.txt            # a→A, b→B, c→C
```

---

## 33.5 In-Place Editing

```bash
# Edit file directly (GNU sed)
sed -i 's/old/new/g' file.txt

# Create backup before editing
sed -i.bak 's/old/new/g' file.txt    # Creates file.txt.bak

# macOS sed requires -i '' (empty extension)
sed -i '' 's/old/new/g' file.txt     # macOS
```

> **Best Practice:** Always use `-i.bak` or test with stdout first before using `-i`.

---

## 33.6 Multiple Commands

```bash
# Semicolons
sed 's/foo/bar/g; s/baz/qux/g' file.txt

# -e flag
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt

# Sed script file
cat > transforms.sed <<'EOF'
s/foo/bar/g
s/baz/qux/g
/^#/d
/^$/d
EOF
sed -f transforms.sed file.txt
```

---

## 33.7 Practical Examples

```bash
# Remove trailing whitespace
sed 's/[[:space:]]*$//' file.txt

# Remove leading whitespace
sed 's/^[[:space:]]*//' file.txt

# Add line numbers
sed = file.txt | sed 'N; s/\n/\t/'

# Convert DOS line endings to Unix
sed 's/\r$//' file.txt

# Extract text between markers
sed -n '/BEGIN/,/END/p' file.txt

# Comment out lines containing "debug"
sed '/debug/s/^/# /' script.sh

# Uncomment lines
sed 's/^# //' config.txt

# Replace only in specific section
sed '/\[database\]/,/\[/s/host=.*/host=newserver/' config.ini

# Double-space a file
sed 'G' file.txt

# Remove all HTML tags
sed 's/<[^>]*>//g' page.html

# Insert a blank line before lines matching pattern
sed '/^Section/i\\' document.txt
```

---

## 33.8 Common Mistakes

### Mistake 1: Forgetting /g for Global Replacement

```bash
echo "aaa" | sed 's/a/b/'     # baa  (only first match!)
echo "aaa" | sed 's/a/b/g'    # bbb  (all matches)
```

### Mistake 2: Greedy Matching

```bash
# Sed regex is always greedy
echo '<b>bold</b> and <i>italic</i>' | sed 's/<.*>//'
# " and "  — removed too much!

# Use negated character class instead
echo '<b>bold</b> and <i>italic</i>' | sed 's/<[^>]*>//g'
# "bold and italic"
```

---

## Exercises

### Exercise 33.1: Config File Editor
Write a script that uses sed to update key=value entries in a config file. Accept the key, new value, and filename as arguments.

### Exercise 33.2: Log Sanitizer
Write a sed script that anonymizes a log file by replacing IP addresses with "X.X.X.X", email addresses with "user@redacted", and credit card numbers (16 digits) with "XXXX-XXXX-XXXX-XXXX".

---

## Summary

- `sed` processes text line by line with pattern-based transformations
- `s/old/new/g` is the core substitution command
- Use addresses (line numbers, `/pattern/`, ranges) to target specific lines
- Key commands: `s` (substitute), `d` (delete), `p` (print), `i`/`a` (insert/append)
- `-i` edits in-place; **always use `-i.bak`** for safety
- Use `-E` for extended regex (cleaner group syntax)
- Choose alternate delimiters (`|`, `#`, `@`) when patterns contain `/`

---

**Next Chapter:** [Chapter 34: awk — Data Extraction →](Chapter34-Awk.md)
