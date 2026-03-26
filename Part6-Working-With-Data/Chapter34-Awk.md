# Chapter 34: awk — Data Extraction and Reporting

## Learning Objectives

By the end of this chapter, you will be able to:

- Use `awk` for field-based text processing
- Write awk programs with patterns and actions
- Use built-in variables (NR, NF, FS, etc.)
- Perform calculations and aggregations
- Build reports from structured data

---

## 34.1 What Is awk?

`awk` is a pattern-action language designed for processing structured text. It automatically splits each line into fields:

```
Input line:    "Alice  Engineering  75000"
               $1       $2           $3
               (field1) (field2)     (field3)
               $0 = entire line
```

```bash
# Basic syntax
awk 'pattern { action }' file

# Print specific fields
awk '{ print $1 }' data.txt          # First field of each line
awk '{ print $1, $3 }' data.txt      # Fields 1 and 3
awk '{ print $NF }' data.txt         # Last field
```

---

## 34.2 Built-in Variables

| Variable | Meaning |
|----------|---------|
| `$0` | Entire current line |
| `$1`, `$2`... | Fields 1, 2, etc. |
| `NR` | Current line number (across all files) |
| `NF` | Number of fields in current line |
| `FS` | Field separator (default: whitespace) |
| `OFS` | Output field separator (default: space) |
| `RS` | Record separator (default: newline) |
| `ORS` | Output record separator (default: newline) |
| `FILENAME` | Current input filename |
| `FNR` | Line number in current file |

```bash
# Print line numbers
awk '{ print NR, $0 }' file.txt

# Print lines with more than 3 fields
awk 'NF > 3' file.txt

# Print last field of each line
awk '{ print $NF }' file.txt

# Custom field separator
awk -F: '{ print $1, $3 }' /etc/passwd     # user and UID
awk -F, '{ print $1, $2 }' data.csv        # CSV fields
```

---

## 34.3 Patterns and Actions

```bash
# Pattern only (default action: print)
awk '/error/' log.txt                  # Print lines containing "error"
awk 'NR == 5' file.txt                 # Print line 5
awk 'NR >= 10 && NR <= 20' file.txt    # Lines 10-20

# Action only (default pattern: match all)
awk '{ print $1 }' file.txt           # Print first field of every line

# Pattern + Action
awk '/error/ { print $1, $4 }' log.txt

# Multiple rules
awk '
    /ERROR/   { errors++ }
    /WARNING/ { warnings++ }
    END       { print "Errors:", errors, "Warnings:", warnings }
' log.txt

# BEGIN and END blocks
awk '
    BEGIN { print "=== Report ===" }
    { print NR, $0 }
    END   { print "Total:", NR, "lines" }
' file.txt
```

---

## 34.4 Operators and Expressions

```bash
# Comparison
awk '$3 > 50000 { print $1, $3 }' salaries.txt    # Salary > 50000
awk '$2 == "Engineering"' employees.txt

# String matching
awk '$1 ~ /^A/' names.txt             # First field starts with A
awk '$0 !~ /DEBUG/' log.txt            # Lines not containing DEBUG

# Arithmetic
awk '{ total += $3 } END { print "Sum:", total }' data.txt
awk '{ total += $3; count++ } END { print "Avg:", total/count }' data.txt

# String concatenation
awk '{ fullname = $1 " " $2; print fullname }' names.txt

# Ternary operator
awk '{ status = ($3 > 50) ? "pass" : "fail"; print $1, status }' scores.txt
```

---

## 34.5 Printf — Formatted Output

```bash
# printf for formatted output (like C)
awk '{ printf "%-20s %10.2f\n", $1, $3 }' data.txt

# Format specifiers:
# %s    string
# %d    integer
# %f    float
# %10s  right-align, width 10
# %-10s left-align, width 10
# %.2f  float with 2 decimal places

# Formatted table
awk -F: 'BEGIN { printf "%-20s %6s %s\n", "USER", "UID", "SHELL" }
         { printf "%-20s %6d %s\n", $1, $3, $7 }' /etc/passwd
```

---

## 34.6 Arrays

awk has associative arrays (like hash maps):

```bash
# Count occurrences
awk '{ count[$1]++ } END { for (k in count) print k, count[k] }' access.log

# Sum by category
awk -F, '{ total[$2] += $3 }
    END { for (cat in total) print cat, total[cat] }' sales.csv

# Unique values
awk '!seen[$1]++' file.txt             # Remove duplicates (preserves order!)
```

> **Classic One-Liner:** `awk '!seen[$0]++'` removes duplicate lines while preserving order — something `sort -u` cannot do.

---

## 34.7 Control Flow in awk

```bash
# If-else
awk '{
    if ($3 > 90) grade = "A"
    else if ($3 > 80) grade = "B"
    else if ($3 > 70) grade = "C"
    else grade = "F"
    print $1, $3, grade
}' scores.txt

# For loop
awk '{
    for (i = 1; i <= NF; i++)
        printf "%s ", toupper($i)
    print ""
}' file.txt

# While loop
awk '{
    i = 1
    while (i <= NF) {
        if ($i ~ /^[0-9]+$/) sum += $i
        i++
    }
} END { print "Sum of all numbers:", sum }' data.txt
```

---

## 34.8 Built-in Functions

```bash
# String functions
awk '{ print length($0) }' file.txt                  # Line length
awk '{ print toupper($1) }' file.txt                  # Uppercase
awk '{ print tolower($0) }' file.txt                  # Lowercase
awk '{ print substr($0, 1, 10) }' file.txt            # Substring
awk '{ gsub(/old/, "new"); print }' file.txt           # Global replace
awk '{ n = split($0, arr, ":"); print arr[1] }' file   # Split string
awk 'match($0, /[0-9]+/) { print substr($0, RSTART, RLENGTH) }'  # Regex match

# Math functions
awk '{ print sqrt($1) }' numbers.txt
awk '{ print int($1) }' decimals.txt
awk 'BEGIN { srand(); print rand() }'       # Random number 0-1
```

---

## 34.9 Practical Examples

```bash
# CSV to formatted table
awk -F, 'NR==1 { for(i=1;i<=NF;i++) header[i]=$i; next }
    { for(i=1;i<=NF;i++) printf "%-15s: %s\n", header[i], $i; print "---" }
' data.csv

# Sum a column
awk -F, '{ sum += $3 } END { print "Total:", sum }' sales.csv

# Top N entries
awk -F, 'NR > 1 { print $3, $1 }' data.csv | sort -rn | head -5

# Frequency histogram
awk '{ len=length($0); bucket=int(len/10)*10; hist[bucket]++ }
    END { for (b in hist) printf "%3d-%3d: %s\n", b, b+9, 
          sprintf("%*s", hist[b], ""); gsub(/ /, "#", $0); print }' file.txt

# Multi-file processing
awk 'FNR==1 { print "=== " FILENAME " ===" } { print }' file1.txt file2.txt

# Calculate disk usage percentages
df | awk 'NR>1 && $5+0 > 80 { printf "WARNING: %s is %s full\n", $6, $5 }'

# Parse /etc/passwd for shells
awk -F: '{ shells[$7]++ }
    END { for (s in shells) printf "%4d %s\n", shells[s], s }
' /etc/passwd | sort -rn
```

---

## 34.10 awk vs sed vs grep

```
┌──────────┬────────────────────────────────────┐
│ Tool     │ Best For                           │
├──────────┼────────────────────────────────────┤
│ grep     │ Finding lines matching pattern     │
│ sed      │ Simple text substitutions          │
│ awk      │ Field-based data processing        │
│          │ Calculations and aggregations      │
│          │ Formatted reports                  │
└──────────┴────────────────────────────────────┘
```

---

## Exercises

### Exercise 34.1: Employee Report
Given a CSV with columns name,department,salary, write awk commands to:
(a) Average salary by department
(b) Highest-paid employee per department
(c) Formatted table with totals

### Exercise 34.2: Log Analyzer
Write an awk script that processes an access log and reports: requests per hour, top 10 URLs, and total bytes transferred.

---

## Summary

- awk automatically splits lines into fields (`$1`, `$2`, ..., `$NF`)
- Use `-F` to set field separator: `awk -F: '{...}' /etc/passwd`
- Structure: `pattern { action }` — pattern selects lines, action processes them
- `BEGIN` runs before input, `END` runs after all input
- Associative arrays enable counting, grouping, and deduplication
- `printf` provides formatted output
- awk excels at columnar data processing and aggregation

---

**Next Chapter:** [Chapter 35: Essential Text Tools →](Chapter35-Text-Tools.md)
