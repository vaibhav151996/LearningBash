# Chapter 36: Parsing and Structured Data

## Learning Objectives

By the end of this chapter, you will be able to:

- Parse CSV, JSON, XML, and INI data in Bash
- Use `jq` for JSON processing
- Handle structured configuration files
- Build robust parsers for common data formats
- Know when to use Bash vs a "real" language for parsing

---

## 36.1 Parsing CSV

### Simple CSV (no quotes, no embedded commas)

```bash
# Read CSV line by line
while IFS=, read -r name department salary; do
    echo "$name works in $department, earns $salary"
done < employees.csv

# Skip header
tail -n +2 employees.csv | while IFS=, read -r name dept salary; do
    echo "$name: $salary"
done

# Extract specific columns
cut -d, -f1,3 data.csv           # Columns 1 and 3
awk -F, '{ print $1, $3 }' data.csv
```

### Complex CSV (quoted fields, embedded commas)

Bash cannot easily handle: `"Smith, John",Engineering,75000`

For complex CSV, use proper tools:

```bash
# Python one-liner
python3 -c "
import csv, sys
for row in csv.reader(sys.stdin):
    print(row[0], row[2])
" < data.csv

# csvkit (install: pip install csvkit)
csvcut -c 1,3 data.csv
csvsort -c salary data.csv
csvgrep -c department -m Engineering data.csv
```

---

## 36.2 Parsing JSON with jq

`jq` is the standard tool for JSON processing in the shell:

```bash
# Pretty print
echo '{"name":"Alice","age":30}' | jq .

# Extract a field
echo '{"name":"Alice","age":30}' | jq '.name'        # "Alice"
echo '{"name":"Alice","age":30}' | jq -r '.name'     # Alice (raw, no quotes)

# Nested objects
echo '{"user":{"name":"Alice","email":"a@b.com"}}' | jq '.user.name'

# Arrays
echo '[1,2,3,4,5]' | jq '.[0]'              # 1 (first element)
echo '[1,2,3,4,5]' | jq '.[-1]'             # 5 (last element)
echo '[1,2,3,4,5]' | jq '.[2:4]'            # [3,4] (slice)
echo '[1,2,3,4,5]' | jq '.[]'               # Each element on separate line

# Array of objects
cat <<'EOF' | jq '.[] | .name'
[
  {"name": "Alice", "role": "admin"},
  {"name": "Bob", "role": "user"}
]
EOF
# "Alice"
# "Bob"
```

### jq Filters and Transformations

```bash
# Filter array elements
jq '.[] | select(.age > 25)' people.json

# Map/transform
jq '[.[] | {fullname: .name, dept: .department}]' employees.json

# Aggregate
jq '[.[].salary] | add' employees.json             # Sum
jq '[.[].salary] | add / length' employees.json    # Average
jq '[.[].salary] | max' employees.json              # Max

# Create CSV from JSON
jq -r '.[] | [.name, .email, .age] | @csv' users.json

# Create TSV
jq -r '.[] | [.name, .email] | @tsv' users.json

# Conditional
jq '.[] | if .score >= 90 then "A" elif .score >= 80 then "B" else "C" end' scores.json
```

### jq with API Responses

```bash
# Get weather data
curl -s "https://api.example.com/weather" | jq '{
    temp: .main.temp,
    humidity: .main.humidity,
    description: .weather[0].description
}'

# List GitHub repos
curl -s "https://api.github.com/users/octocat/repos" |
    jq -r '.[] | "\(.name)\t\(.stargazers_count)\t\(.language)"' |
    sort -t$'\t' -k2 -rn |
    column -t -s$'\t'
```

---

## 36.3 Parsing INI/Config Files

```bash
# config.ini
# [database]
# host = localhost
# port = 5432
# name = mydb
#
# [logging]
# level = info
# file = /var/log/app.log

# Read a specific value
get_ini_value() {
    local file="$1" section="$2" key="$3"
    sed -n "/^\[$section\]/,/^\[/p" "$file" |
        grep "^$key" |
        head -1 |
        cut -d= -f2- |
        sed 's/^[[:space:]]*//'
}

db_host=$(get_ini_value config.ini database host)
log_level=$(get_ini_value config.ini logging level)
echo "DB Host: $db_host"         # localhost
echo "Log Level: $log_level"     # info
```

---

## 36.4 Parsing Key-Value Files

```bash
# environment file (.env format)
# APP_NAME=MyApp
# DB_HOST=localhost
# DB_PORT=5432

# Safe loading (without eval)
load_env() {
    local file="$1"
    while IFS='=' read -r key value; do
        # Skip comments and empty lines
        [[ "$key" =~ ^[[:space:]]*# ]] && continue
        [[ -z "$key" ]] && continue
        # Remove surrounding whitespace and quotes
        key=$(echo "$key" | xargs)
        value=$(echo "$value" | sed "s/^['\"]//;s/['\"]$//")
        export "$key=$value"
    done < "$file"
}

load_env .env
echo "$APP_NAME"    # MyApp
```

---

## 36.5 Parsing XML (Basic)

For simple XML, use command-line tools. For complex XML, use a proper parser.

```bash
# xmllint (from libxml2)
xmllint --xpath '//title/text()' books.xml

# xmlstarlet
xmlstarlet sel -t -v "//book/title" books.xml

# Simple tag extraction with grep/sed (fragile — for simple cases only)
grep -oP '(?<=<title>).*?(?=</title>)' books.xml
```

> **Warning:** Don't parse XML with regex for anything serious. XML is not a regular language.

---

## 36.6 Processing Log Files

```bash
# Apache/Nginx access log format:
# 192.168.1.1 - - [20/Jan/2024:10:15:30 +0000] "GET /page HTTP/1.1" 200 1234

# Top IPs
awk '{ print $1 }' access.log | sort | uniq -c | sort -rn | head -10

# Requests per hour
awk '{ split($4, a, ":"); print a[2] }' access.log | sort | uniq -c

# Status code distribution
awk '{ print $9 }' access.log | sort | uniq -c | sort -rn

# 404 errors
awk '$9 == 404 { print $7 }' access.log | sort | uniq -c | sort -rn

# Total bytes served
awk '{ sum += $10 } END { printf "Total: %.2f GB\n", sum/1073741824 }' access.log
```

---

## 36.7 When to Stop Using Bash

Use a "real" language when you need:

```
┌──────────────────────────────────┬─────────────────────────┐
│ Scenario                         │ Better Tool             │
├──────────────────────────────────┼─────────────────────────┤
│ Complex CSV with quoting         │ Python csv module       │
│ Deep JSON transformations        │ Python, jq              │
│ XML/HTML parsing                 │ Python lxml, xmlstarlet │
│ Database queries                 │ SQL, Python             │
│ Binary data                      │ Python, C               │
│ Data > 1GB                       │ Python pandas, SQL      │
│ Complex error handling in parser │ Python, Go              │
└──────────────────────────────────┴─────────────────────────┘
```

Rule of thumb: If your parsing code exceeds ~20 lines of awk or sed, consider Python.

---

## Exercises

### Exercise 36.1: JSON Report
Given a JSON file of employees, use `jq` to generate: (a) average salary by department, (b) list of employees earning above average, (c) a summary CSV.

### Exercise 36.2: Log Dashboard
Write a script that parses a web server log and outputs a dashboard showing: total requests, unique visitors, top 5 pages, top 5 IPs, error rate percentage.

---

## Summary

- Simple CSV: use `cut`, `awk`, or `IFS=, read`
- Complex CSV: use `csvkit` or Python's `csv` module
- JSON: use `jq` — it's the gold standard for shell JSON processing
- INI files: parse with `sed` section extraction
- Key-value files: parse with `IFS='=' read`
- Log files: combine `awk`, `sort`, `uniq` for analytics
- Know your limits: switch to Python/etc. for complex or large-scale parsing

---

**Next Chapter:** [Chapter 37: Process Management →](../Part7-System-Interaction/Chapter37-Process-Management.md)
