# Chapter 35: Essential Text Tools

## Learning Objectives

By the end of this chapter, you will be able to:

- Use `cut`, `sort`, `uniq`, `tr`, `paste`, `join`, `column` and other text tools
- Choose the right tool for each text manipulation task
- Combine tools effectively in pipelines
- Handle CSV, TSV, and other delimited formats

---

## 35.1 cut — Extract Fields or Characters

```bash
# By delimiter and field
cut -d: -f1 /etc/passwd              # Username (field 1)
cut -d: -f1,3 /etc/passwd            # Username and UID
cut -d, -f2-4 data.csv               # Fields 2 through 4

# By character position
cut -c1-10 file.txt                  # First 10 characters
cut -c5- file.txt                    # Character 5 to end

# By bytes
cut -b1-4 file.txt                   # First 4 bytes
```

---

## 35.2 sort — Sort Lines

```bash
sort file.txt                        # Alphabetical sort
sort -n file.txt                     # Numeric sort
sort -r file.txt                     # Reverse sort
sort -u file.txt                     # Sort and remove duplicates
sort -t: -k3 -n /etc/passwd         # Sort by field 3 (UID), numeric
sort -t, -k2,2 -k3,3rn data.csv    # Sort by field 2 asc, then field 3 desc numeric
sort -h sizes.txt                    # Human-readable sizes (1K, 2M, 3G)
sort -R file.txt                     # Random/shuffle sort
sort -f file.txt                     # Case-insensitive
sort --stable -k1,1 file.txt        # Stable sort (preserve input order for ties)
```

### Sort Key Specification

```bash
# -k start,end
sort -t, -k2,2 file.csv       # Sort by field 2 only
sort -t, -k2,2 -k1,1 file.csv # Sort by field 2, then by field 1 for ties
sort -t: -k3,3n /etc/passwd   # Numeric sort by field 3
```

---

## 35.3 uniq — Remove Duplicates

> **Important**: `uniq` only removes **adjacent** duplicates. Sort first!

```bash
sort file.txt | uniq                 # Remove duplicates
sort file.txt | uniq -c              # Count occurrences
sort file.txt | uniq -d              # Show only duplicates
sort file.txt | uniq -u              # Show only unique lines
sort file.txt | uniq -c | sort -rn   # Frequency count, most common first
```

---

## 35.4 tr — Translate Characters

```bash
# Replace characters
echo "hello" | tr 'a-z' 'A-Z'           # HELLO (uppercase)
echo "HELLO" | tr 'A-Z' 'a-z'           # hello (lowercase)
echo "hello" | tr 'helo' 'HELO'         # HELLO

# Delete characters
echo "Hello 123 World" | tr -d '0-9'    # Hello  World
echo "Hello World" | tr -d ' '          # HelloWorld

# Squeeze repeated characters
echo "hello     world" | tr -s ' '      # "hello world"
echo "aabbbcccc" | tr -s 'a-z'          # "abc"

# Replace newlines with spaces
tr '\n' ' ' < file.txt

# Remove non-printable characters
tr -cd '[:print:]' < binary_file

# Convert spaces to tabs
tr ' ' '\t' < file.txt

# Character classes
echo "Hello 123!" | tr -d '[:digit:]'    # "Hello !"
echo "Hello 123!" | tr -d '[:alpha:]'    # " 123!"
```

---

## 35.5 paste — Merge Lines Side by Side

```bash
# Combine files as columns
paste names.txt scores.txt              # Tab-separated columns
paste -d, names.txt scores.txt          # Comma-separated

# Convert column to row
paste -s -d, file.txt                   # Join all lines with commas

# Create fixed-width columns from single stream
ls | paste - - - -                      # 4 columns
seq 12 | paste - - - -                  # 4 columns of numbers
```

---

## 35.6 join — Join Files on a Key

```bash
# File 1: id,name          File 2: id,salary
# 1,Alice                  1,75000
# 2,Bob                    2,82000

join -t, -1 1 -2 1 names.csv salaries.csv
# 1,Alice,75000
# 2,Bob,82000

# Files must be sorted on the join key!
```

---

## 35.7 column — Format into Columns

```bash
# Auto-format into aligned columns
mount | column -t

# With specific delimiter
echo "name:age:city" | column -t -s:

# CSV to pretty table
column -t -s, < data.csv
```

---

## 35.8 wc — Word Count

```bash
wc file.txt                # lines  words  bytes  filename
wc -l file.txt             # Line count only
wc -w file.txt             # Word count only
wc -c file.txt             # Byte count only
wc -m file.txt             # Character count only

# Count files
ls | wc -l                 # Number of files in directory
find . -name "*.py" | wc -l   # Number of Python files
```

---

## 35.9 Other Useful Tools

### comm — Compare Sorted Files

```bash
# Three columns: only-in-file1, only-in-file2, in-both
comm sorted1.txt sorted2.txt
comm -12 sorted1.txt sorted2.txt    # Only lines in BOTH files
comm -23 sorted1.txt sorted2.txt    # Only in file1
comm -13 sorted1.txt sorted2.txt    # Only in file2
```

### diff — Show Differences

```bash
diff file1.txt file2.txt            # Traditional diff
diff -u file1.txt file2.txt         # Unified diff (most readable)
diff -y file1.txt file2.txt         # Side by side
diff -r dir1/ dir2/                 # Recursive directory diff
```

### tac, rev — Reverse Content

```bash
tac file.txt          # Reverse line order (opposite of cat)
rev file.txt          # Reverse characters on each line
```

### fold, fmt — Wrap Text

```bash
fold -w 80 file.txt      # Wrap at 80 characters
fold -sw 80 file.txt     # Wrap at word boundaries
fmt -w 72 file.txt       # Reformat paragraphs to 72 chars
```

---

## 35.10 Tool Selection Guide

```
┌────────────────────────┬──────────────────────────┐
│ Task                   │ Tool                     │
├────────────────────────┼──────────────────────────┤
│ Extract columns        │ cut, awk                 │
│ Sort lines             │ sort                     │
│ Count duplicates       │ sort | uniq -c           │
│ Replace characters     │ tr                       │
│ Replace strings        │ sed                      │
│ Merge files side-by-side│ paste                   │
│ Join on key field      │ join                     │
│ Count lines/words      │ wc                       │
│ Compare files          │ diff, comm               │
│ Format as table        │ column -t                │
│ Reverse lines          │ tac                      │
│ Reverse characters     │ rev                      │
│ Wrap long lines        │ fold, fmt                │
│ Field-based processing │ awk                      │
└────────────────────────┴──────────────────────────┘
```

---

## Exercises

### Exercise 35.1: Data Pipeline
Given a CSV file with columns name,department,score, write a pipeline that: sorts by department, then score within department, adds a rank column, and formats as an aligned table.

### Exercise 35.2: File Comparison
Write a script that compares two directory listings and reports files that are only in directory A, only in directory B, and in both.

---

## Summary

- `cut` extracts fields or character positions from each line
- `sort` orders lines (numeric `-n`, reverse `-r`, by key `-k`, unique `-u`)
- `uniq` removes adjacent duplicates — always `sort` first
- `tr` translates, deletes, or squeezes characters
- `paste` merges files side by side; `join` merges on a shared key
- `column -t` formats output into aligned columns
- `wc` counts lines, words, and bytes
- Combine these tools in pipelines for powerful data processing

---

**Next Chapter:** [Chapter 36: Parsing and Structured Data →](Chapter36-Parsing-Data.md)
