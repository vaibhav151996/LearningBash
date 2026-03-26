# Chapter 10: Wildcards and Globbing

## Learning Objectives

By the end of this chapter, you will be able to:

- Use all standard glob patterns: `*`, `?`, `[...]`
- Understand how globbing differs from regular expressions
- Use extended globbing patterns with `shopt -s extglob`
- Control globbing behavior with shell options
- Avoid common pitfalls with filename expansion

---

## 10.1 What Is Globbing?

**Globbing** (also called **filename expansion** or **pathname expansion**) is the process by which the shell expands wildcard patterns into matching filenames. The shell does the expansion — the command never sees the wildcards.

```bash
# You type:
ls *.txt

# Bash sees *.txt and expands it BEFORE running ls
# If the directory contains file1.txt, notes.txt, and readme.txt:
# Bash actually runs:
ls file1.txt notes.txt readme.txt
```

The command receives the expanded list as individual arguments. It has no idea wildcards were involved.

---

## 10.2 Standard Glob Patterns

### `*` — Match Any String (Including Empty)

```bash
ls *.txt          # All files ending in .txt
ls data*          # All files starting with "data"
ls *report*       # All files containing "report"
ls *              # All non-hidden files
```

**Important:** `*` does NOT match:
- Hidden files (those starting with `.`) — use `.*` or `ls -a`
- The `/` path separator — `*` stays within one directory level

```bash
ls .*              # Only hidden files
ls .??*            # Hidden files with 3+ character names (excludes . and ..)
```

### `?` — Match Exactly One Character

```bash
ls file?.txt       # file1.txt, fileA.txt, but NOT file10.txt
ls ???             # All 3-character filenames
ls ?????.*         # 5-character names with any extension
```

### `[...]` — Match One Character from a Set

```bash
ls file[123].txt            # file1.txt, file2.txt, file3.txt
ls file[a-z].txt            # filea.txt through filez.txt
ls file[0-9][0-9].txt       # file00.txt through file99.txt
ls [A-Z]*.txt               # Files starting with uppercase letter
```

### `[!...]` or `[^...]` — Match One Character NOT in the Set

```bash
ls file[!0-9].txt           # Files not ending with a digit
ls [!.]*.txt                # Non-hidden .txt files
ls file[^abc].txt           # file not followed by a, b, or c
```

### Character Classes

POSIX character classes work inside `[...]`:

```bash
ls [[:upper:]]*          # Files starting with uppercase
ls [[:digit:]]*          # Files starting with a digit
ls *[[:space:]]*         # Files containing spaces (dangerous!)
ls [[:alpha:]]*          # Files starting with a letter
```

| Class | Matches |
|-------|---------|
| `[[:alpha:]]` | Letters (a-z, A-Z) |
| `[[:digit:]]` | Digits (0-9) |
| `[[:alnum:]]` | Letters and digits |
| `[[:upper:]]` | Uppercase letters |
| `[[:lower:]]` | Lowercase letters |
| `[[:space:]]` | Whitespace characters |
| `[[:punct:]]` | Punctuation |

---

## 10.3 Extended Globbing

Bash has extended glob patterns that provide more power. They must be enabled:

```bash
# Enable extended globbing
shopt -s extglob
```

| Pattern | Matches |
|---------|---------|
| `?(pattern)` | Zero or one occurrence |
| `*(pattern)` | Zero or more occurrences |
| `+(pattern)` | One or more occurrences |
| `@(pattern)` | Exactly one occurrence |
| `!(pattern)` | Anything EXCEPT the pattern |

```bash
shopt -s extglob

# Match files that are NOT .txt
ls !(*.txt)

# Match .jpg OR .png files
ls *.@(jpg|png)

# Match files not ending in .bak or .tmp
ls !(*.bak|*.tmp)

# Match files with optional numeric suffix
ls report?(s).txt          # report.txt or reports.txt

# Match one or more digits
ls file+([0-9]).txt        # file1.txt, file123.txt, NOT file.txt
```

---

## 10.4 Globbing Options

### `dotglob` — Include Hidden Files in `*`

```bash
shopt -s dotglob
ls *                      # Now includes .bashrc, .profile, etc.
shopt -u dotglob          # Disable
```

### `nullglob` — Expand to Nothing When No Match

```bash
# Default behavior: pattern stays as-is if nothing matches
echo *.xyz                # *.xyz (literal string — nothing matched)

# With nullglob: pattern expands to nothing
shopt -s nullglob
echo *.xyz                # (nothing — no output)
shopt -u nullglob
```

This is critical in scripts with loops:

```bash
# BUG without nullglob:
for f in *.csv; do
    process "$f"
done
# If no CSV files exist, this processes the literal string "*.csv"!

# FIX:
shopt -s nullglob
for f in *.csv; do
    process "$f"
done
# If no CSV files exist, the loop simply doesn't execute
```

### `failglob` — Error When No Match

```bash
shopt -s failglob
ls *.xyz
# bash: no match: *.xyz
```

### `globstar` — Recursive Matching with `**`

```bash
shopt -s globstar

# Match ALL .txt files in ALL subdirectories
ls **/*.txt

# Compare with find:
find . -name "*.txt"      # Similar result

# Match everything recursively
ls **
```

### `nocaseglob` — Case-Insensitive Matching

```bash
shopt -s nocaseglob
ls *.txt                  # Matches .txt, .TXT, .Txt, etc.
shopt -u nocaseglob
```

---

## 10.5 Globbing vs. Regular Expressions

Beginners often confuse glob patterns with regular expressions. They are **completely different** systems.

| Feature | Globbing | Regular Expressions |
|---------|----------|-------------------|
| `*` | Any string | Zero or more of previous |
| `?` | One character | Zero or one of previous |
| `.` | Literal dot | Any character |
| Used by | Shell (filenames) | grep, sed, awk |
| Scope | Filename matching | Text pattern matching |

```bash
# GLOBBING (shell expansion)
ls *.txt           # * = any string of characters

# REGULAR EXPRESSION (grep)
grep "a.*b" file   # .* = any character (.) repeated zero or more times (*)
```

They are used in completely different contexts. Don't mix them up.

---

## 10.6 Practical Examples

```bash
# Delete all backup files
rm *.bak

# Copy all images to a directory
cp *.{jpg,png,gif} images/

# Find all shell scripts
ls **/*.sh

# List files modified today (combining with find)
find . -name "*.log" -mtime 0

# Move all numbered files
mv file[0-9]*.dat archive/

# Delete everything EXCEPT .txt files
shopt -s extglob
rm !(*.txt)
```

---

## Common Mistakes

1. **Glob in quotes doesn't expand** — `"*.txt"` is the literal string `*.txt`, not a pattern. Globs must be unquoted.

2. **Expecting `*` to match hidden files** — It doesn't by default. Use `.*` or `shopt -s dotglob`.

3. **Forgetting `nullglob` in scripts** — Without it, unmatched patterns pass as literal strings to commands.

4. **Confusing glob and regex** — `*` means different things in globbing vs. regex.

5. **Using `rm *` in the wrong directory** — Always `pwd` first!

---

## Exercises

### Exercise 10.1: Pattern Writing
Write glob patterns to match:
1. All files ending in `.log` or `.txt`
2. Files named `data` followed by exactly two digits
3. All files NOT starting with a lowercase letter
4. All hidden files (starting with `.`)

### Exercise 10.2: Extended Globs
Enable `extglob` and write patterns to:
1. List everything except `.bak` files
2. Match files ending in `.jpg`, `.jpeg`, or `.png`
3. Match filenames with one or more digits

---

## Summary

- **Globbing** is filename expansion performed by the shell, not the command
- `*` matches any string; `?` matches one character; `[...]` matches a character set
- **Extended globbing** (`shopt -s extglob`) adds `!()`, `@()`, `+()`, `?()`, `*()`
- Use `nullglob` in scripts to handle no-match cases safely
- Use `globstar` for recursive matching with `**`
- Globbing and regular expressions are **different systems** — don't confuse them

---

**Next Chapter:** [Chapter 11: Pipes and Redirections →](Chapter11-Pipes-Redirections.md)
