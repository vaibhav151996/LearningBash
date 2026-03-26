# Chapter 31: The Unix Text Philosophy

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand the Unix philosophy of text processing
- Explain why "everything is a file" matters
- Use pipes to build processing pipelines
- Understand text streams and filters
- Appreciate the power of composing small tools

---

## 31.1 The Unix Philosophy

Doug McIlroy summarized it:

> *"Write programs that do one thing and do it well. Write programs to work together. Write programs to handle text streams, because that is a universal interface."*

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Input   │────▶│ Filter 1 │────▶│ Filter 2 │────▶│  Output  │
│  (text)  │     │ (grep)   │     │ (sort)   │     │  (text)  │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
```

Each program:
- Reads text from **stdin**
- Processes / transforms it
- Writes results to **stdout**
- Reports errors to **stderr**

---

## 31.2 Everything Is a File

In Unix, nearly everything is represented as a file:

| Resource | File Path |
|----------|-----------|
| Regular files | `/home/user/data.txt` |
| Directories | `/home/user/` |
| Devices | `/dev/sda`, `/dev/null` |
| Processes | `/proc/1234/status` |
| Network | `/dev/tcp/host/port` (Bash) |
| Random data | `/dev/urandom` |
| Nothing | `/dev/null` |

This means the same tools (cat, grep, less) work on all of them.

---

## 31.3 Text Streams

A text stream is a sequence of lines separated by newlines (`\n`):

```bash
# Generate a stream
seq 1 5                    # Numbers 1-5, one per line
echo -e "a\nb\nc"         # Three lines
cat /etc/passwd            # File contents as stream

# Streams can be infinitely long
yes "hello"                # Infinite "hello" lines
/dev/urandom               # Infinite random bytes
```

### Line-Oriented Processing

Most Unix tools process **one line at a time**:

```bash
# Each tool in the pipeline processes lines:
cat access.log |    # Read lines from file
grep "ERROR" |      # Keep only lines containing "ERROR"
cut -d' ' -f1 |    # Extract first field (IP address)
sort |              # Sort alphabetically
uniq -c |           # Count duplicates
sort -rn |          # Sort by count, descending
head -10            # Show top 10
```

---

## 31.4 The Filter Model

A **filter** reads stdin, transforms it, writes stdout:

```bash
# Built-in filters:
cat          # Pass through (identity filter)
grep         # Select matching lines
sort         # Sort lines
uniq         # Remove adjacent duplicate lines
head / tail  # First/last N lines
cut          # Extract columns
tr           # Translate/delete characters
wc           # Count lines/words/bytes
sed          # Stream editor (transform text)
awk          # Pattern-action text processing
tee          # Duplicate stream (stdout + file)
```

### Building Pipelines

```bash
# Pipeline: who's logged in, sorted alphabetically
who | cut -d' ' -f1 | sort -u

# Pipeline: count file types in current directory
find . -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn

# Pipeline: top 5 largest files
du -ah . | sort -rh | head -5

# Pipeline: unique words in a file
tr ' ' '\n' < book.txt | tr '[:upper:]' '[:lower:]' | sort -u
```

---

## 31.5 Key Text Processing Principles

### Principle 1: One Record Per Line

```bash
# Good text format (one record per line):
alice:admin:2024-01-15
bob:user:2024-02-20

# Process easily:
grep ":admin:" users.txt | cut -d: -f1
```

### Principle 2: Fields Separated by Delimiters

```bash
# Common delimiters: colon, tab, comma, space
# /etc/passwd uses colons:
root:x:0:0:root:/root:/bin/bash

# CSV uses commas:
Alice,Engineering,Senior

# TSV uses tabs (safest — rare in data):
Alice	Engineering	Senior
```

### Principle 3: Compose, Don't Monolith

```bash
# BAD: One complex command trying to do everything
awk 'BEGIN{...} /pattern/{...complex logic...} END{...}' file

# GOOD: Pipeline of simple steps
grep "pattern" file |
    cut -d',' -f2,5 |
    sort -t',' -k2 -n |
    head -20
```

---

## 31.6 Essential Stream Manipulations

```bash
# Combine files vertically (append)
cat file1.txt file2.txt > combined.txt

# Combine files horizontally (side by side)
paste file1.txt file2.txt

# Split a stream
tee backup.txt < input.txt | process_further

# Reverse lines
tac file.txt

# Reverse characters on each line
rev file.txt

# Number lines
nl file.txt
cat -n file.txt

# Remove duplicate lines (must be sorted first)
sort file.txt | uniq

# Sort and deduplicate in one step
sort -u file.txt
```

---

## Exercises

### Exercise 31.1: Pipeline Challenge
Given `/etc/passwd`, write a one-liner pipeline to: find all users with `/bin/bash` as their shell, extract just usernames, sort them, and count how many there are.

### Exercise 31.2: Word Frequency
Write a pipeline that reads a text file and produces a word frequency list sorted by count (most common first).

---

## Summary

- Unix philosophy: small tools, text streams, composition via pipes
- "Everything is a file" — uniform interface for all resources
- Text streams are line-oriented sequences of data
- Filters read stdin, transform, write stdout
- Build complex processing from simple pipeline stages
- Use standard delimiters (colon, tab, comma) for structured text

---

**Next Chapter:** [Chapter 32: grep — Pattern Matching →](Chapter32-Grep.md)
