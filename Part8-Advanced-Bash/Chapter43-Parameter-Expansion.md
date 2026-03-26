# Chapter 43: Advanced Parameter Expansion

## Learning Objectives

By the end of this chapter, you will be able to:

- Master all forms of Bash parameter expansion
- Use string manipulation without external commands
- Apply pattern matching in variable expansions
- Implement default values, error messages, and substitutions
- Replace common `sed`, `cut`, and `basename`/`dirname` calls

---

## 43.1 Default Values

```bash
# Use default if unset or empty
name="${1:-anonymous}"            # anonymous if $1 is unset or empty

# Use default if unset only (empty string is kept)
name="${1-anonymous}"             # anonymous if $1 is unset, but "" if empty

# Assign default if unset or empty
: "${TIMEOUT:=30}"               # Sets TIMEOUT to 30 if not already set
: "${VERBOSE:=false}"

# Error if unset or empty
dir="${1:?Usage: $0 <directory>}"  # Prints error message and exits

# Use alternative value if SET
result="${var:+value_if_set}"     # "value_if_set" if var is set and non-empty
```

```
┌────────────┬──────────────────────┬────────────────────┐
│ Expansion  │ var is unset/empty   │ var is set          │
├────────────┼──────────────────────┼────────────────────┤
│ ${var:-D}  │ returns D            │ returns $var        │
│ ${var:=D}  │ sets var=D, returns D│ returns $var        │
│ ${var:?E}  │ prints E, exits      │ returns $var        │
│ ${var:+V}  │ returns ""           │ returns V           │
└────────────┴──────────────────────┴────────────────────┘
```

---

## 43.2 String Length

```bash
str="Hello, World!"
echo "${#str}"           # 13

# Array element count
arr=(a b c d)
echo "${#arr[@]}"        # 4

# Length of array element
echo "${#arr[0]}"        # 1 (length of "a")
```

---

## 43.3 Substring Extraction

```bash
str="Hello, World!"

echo "${str:0:5}"        # Hello    (offset 0, length 5)
echo "${str:7}"          # World!   (offset 7 to end)
echo "${str: -6}"        # orld!    (last 6 chars — note the space before -)
echo "${str:7:5}"        # World    (offset 7, length 5)
```

---

## 43.4 Pattern Removal (Trimming)

```bash
# Remove from front (prefix)
file="/home/user/documents/report.tar.gz"

echo "${file#*/}"        # home/user/documents/report.tar.gz  (shortest match)
echo "${file##*/}"       # report.tar.gz                      (longest match — like basename)

# Remove from back (suffix)
echo "${file%.*}"        # /home/user/documents/report.tar    (shortest match)
echo "${file%%.*}"       # /home/user/documents/report        (longest match)

# Practical: get filename components
filepath="/path/to/script.sh"
echo "${filepath##*/}"   # script.sh     (filename, like basename)
echo "${filepath%/*}"    # /path/to       (directory, like dirname)

filename="report.tar.gz"
echo "${filename%%.*}"   # report         (name without ANY extension)
echo "${filename%.*}"    # report.tar     (name without LAST extension)
echo "${filename##*.}"   # gz             (last extension only)
```

```
Memory aid:
  # removes from front  (# is on the left of $ on keyboard)
  % removes from back   (% is on the right of $ on keyboard)
  Single = shortest match
  Double = longest match
```

---

## 43.5 Search and Replace

```bash
str="hello world hello bash hello"

echo "${str/hello/HI}"       # HI world hello bash hello    (first match)
echo "${str//hello/HI}"      # HI world HI bash HI          (all matches)
echo "${str/#hello/HI}"      # HI world hello bash hello    (match at start)
echo "${str/%hello/HI}"      # hello world hello bash HI    (match at end)

# Delete pattern (replace with nothing)
echo "${str//hello/}"        #  world  bash                  (delete all "hello")
```

---

## 43.6 Case Conversion (Bash 4+)

```bash
str="Hello World"

echo "${str^^}"          # HELLO WORLD    (all uppercase)
echo "${str,,}"          # hello world    (all lowercase)
echo "${str^}"           # Hello World    (first char uppercase)
echo "${str,}"           # hello World    (first char lowercase)

# Selective conversion
echo "${str^^[aeiou]}"   # HEllO WOrld    (uppercase vowels only)
```

---

## 43.7 Indirect Expansion

```bash
# Access variable by name stored in another variable
var_name="greeting"
greeting="Hello, World!"

echo "${!var_name}"      # Hello, World!

# List variables matching prefix
APP_NAME="MyApp"
APP_VERSION="1.0"
APP_PORT=8080

echo "${!APP_*}"         # APP_NAME APP_PORT APP_VERSION
echo "${!APP_@}"         # Same, each as separate word
```

---

## 43.8 Real-World Examples

```bash
# Strip extension and add new one
input="data.csv"
output="${input%.csv}.json"          # data.json

# Process all .log files into .log.gz
for f in *.log; do
    gzip -c "$f" > "${f}.gz"
done

# Extract components from a URL
url="https://api.example.com:8080/v2/users?page=1"
protocol="${url%%://*}"              # https
rest="${url#*://}"                   # api.example.com:8080/v2/users?page=1
host="${rest%%/*}"                   # api.example.com:8080
path="/${rest#*/}"                   # /v2/users?page=1

# Rename files: replace spaces with underscores
for f in *.txt; do
    mv "$f" "${f// /_}"
done

# Remove color codes from string
clean="${colored_string//\033\[[0-9;]*m/}"

# Simple template expansion
template="Hello, NAME! Welcome to CITY."
result="${template//NAME/$user_name}"
result="${result//CITY/$city}"
```

---

## 43.9 Comparison: Parameter Expansion vs External Tools

```bash
# Instead of basename:
filepath="/path/to/file.txt"
basename "$filepath"          # file.txt (fork + exec)
echo "${filepath##*/}"        # file.txt (pure bash, no fork!)

# Instead of dirname:
dirname "$filepath"           # /path/to
echo "${filepath%/*}"         # /path/to

# Instead of sed for simple replacements:
echo "$str" | sed 's/old/new/g'    # Forks sed
echo "${str//old/new}"              # Pure bash

# Instead of cut for extension:
echo "$file" | cut -d. -f2-        # Forks cut
echo "${file#*.}"                   # Pure bash

# Instead of tr for case:
echo "$str" | tr 'a-z' 'A-Z'      # Forks tr
echo "${str^^}"                     # Pure bash
```

> **Performance:** In loops processing thousands of items, parameter expansion is **orders of magnitude faster** than forking external commands.

---

## Exercises

### Exercise 43.1: Path Parser
Write a function `parse_path` that takes a file path and outputs: directory, filename, basename (no extension), and extension — using only parameter expansion (no `basename`, `dirname`, `cut`).

### Exercise 43.2: Template Engine
Write a simple template engine that reads a template file with `{{VARIABLE}}` placeholders and replaces them with environment variable values using parameter expansion.

---

## Summary

- `${var:-default}` — default value; `${var:=default}` — assign default
- `${#var}` — string length
- `${var:offset:length}` — substring extraction
- `${var#pattern}` / `${var##pattern}` — remove prefix (shortest/longest)
- `${var%pattern}` / `${var%%pattern}` — remove suffix (shortest/longest)
- `${var/old/new}` — replace first; `${var//old/new}` — replace all
- `${var^^}` / `${var,,}` — case conversion
- `${!prefix*}` — list variable names matching prefix
- Parameter expansion is much faster than forking external commands

---

**Next Chapter:** [Chapter 44: Process Substitution and Coprocesses →](Chapter44-Process-Substitution.md)
