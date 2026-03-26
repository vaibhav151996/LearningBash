# Chapter 51: Performance and Optimization

## Learning Objectives

By the end of this chapter, you will be able to:

- Identify performance bottlenecks in Bash scripts
- Optimize scripts by reducing process forks
- Use built-in features instead of external commands
- Benchmark and profile scripts
- Know when Bash is the wrong tool

---

## 51.1 The #1 Performance Rule

**Minimize process forks.** Every external command (grep, sed, awk, cut, etc.) creates a new process. In tight loops, this is the dominant cost.

```bash
# SLOW — 3 forks per iteration (1000 × 3 = 3000 forks)
for i in $(seq 1 1000); do
    result=$(echo "$i" | grep -o '[0-9]' | wc -l)
done

# FAST — 0 forks per iteration (pure Bash)
for i in {1..1000}; do
    result=${#i}
done
```

---

## 51.2 Built-in vs External Commands

```bash
# Instead of external 'basename':
name=$(basename "$path")         # Fork
name="${path##*/}"               # No fork (parameter expansion)

# Instead of external 'dirname':
dir=$(dirname "$path")           # Fork
dir="${path%/*}"                 # No fork

# Instead of 'tr' for case conversion:
upper=$(echo "$str" | tr 'a-z' 'A-Z')   # Fork
upper="${str^^}"                          # No fork (Bash 4+)

# Instead of 'wc -c' for string length:
len=$(echo -n "$str" | wc -c)   # Fork
len=${#str}                      # No fork

# Instead of 'cut' for field extraction:
field=$(echo "$line" | cut -d: -f1)   # Fork
IFS=: read -r field _ <<< "$line"     # No fork

# Instead of 'expr' for arithmetic:
result=$(expr 5 + 3)            # Fork
result=$((5 + 3))               # No fork

# Instead of 'seq':
for i in $(seq 1 100); do       # Fork
for i in {1..100}; do           # No fork
for ((i=1; i<=100; i++)); do    # No fork
```

---

## 51.3 Batch vs Per-Line Processing

```bash
# SLOW — call grep once per pattern
while read -r pattern; do
    grep "$pattern" huge_file.txt
done < patterns.txt

# FAST — single grep call with multiple patterns
grep -f patterns.txt huge_file.txt

# SLOW — call sed per file
for file in *.txt; do
    sed -i 's/old/new/g' "$file"
done

# FAST — single sed call
sed -i 's/old/new/g' *.txt

# SLOW — process line by line in Bash
total=0
while read -r line; do
    value=$(echo "$line" | awk '{print $3}')
    total=$((total + value))
done < data.txt

# FAST — let awk do it all
total=$(awk '{sum+=$3} END{print sum}' data.txt)
```

> **Rule:** If you're processing data line by line in a Bash loop, you're probably doing it wrong. Use awk, sed, or grep to process the entire file at once.

---

## 51.4 I/O Optimization

```bash
# SLOW — many small writes
for i in {1..10000}; do
    echo "$i" >> output.txt      # Open, write, close per iteration
done

# FAST — single redirection block
{
    for i in {1..10000}; do
        echo "$i"
    done
} > output.txt                   # One open/close for the whole block

# FAST — printf is faster than echo for bulk output
printf '%s\n' {1..10000} > output.txt
```

### Reading Files Efficiently

```bash
# SLOW — for loop with cat
for line in $(cat file.txt); do     # Word splitting breaks lines!
    echo "$line"
done

# CORRECT but still Bash-loop overhead
while IFS= read -r line; do
    echo "$line"
done < file.txt

# FASTEST for simple transforms — use awk/sed
awk '{print toupper($0)}' file.txt
```

---

## 51.5 Benchmarking

```bash
# time command
time ./script.sh
# real    0m5.123s    ← wall clock
# user    0m3.456s    ← CPU in user mode
# sys     0m1.234s    ← CPU in kernel mode

# Inline timing
start=$SECONDS
expensive_operation
elapsed=$((SECONDS - start))
echo "Took ${elapsed}s"

# More precise with date
start=$(date +%s%N)
expensive_operation
end=$(date +%s%N)
elapsed=$(( (end - start) / 1000000 ))
echo "Took ${elapsed}ms"

# Compare two approaches
echo "Approach 1:"
time for i in {1..1000}; do basename "/path/to/file.txt" > /dev/null; done

echo "Approach 2:"
time for i in {1..1000}; do x="/path/to/file.txt"; echo "${x##*/}" > /dev/null; done
```

---

## 51.6 Parallel Processing

```bash
# Sequential — slow
for file in *.csv; do
    process_file "$file"
done

# Parallel with & and wait
for file in *.csv; do
    process_file "$file" &
done
wait

# Controlled parallelism (max N processes)
max_jobs=4
for file in *.csv; do
    process_file "$file" &
    # Wait if we have too many jobs
    while (( $(jobs -r | wc -l) >= max_jobs )); do
        sleep 0.1
    done
done
wait

# GNU parallel (best option for complex parallelism)
find . -name "*.csv" | parallel -j4 process_file {}

# xargs parallel
find . -name "*.csv" -print0 | xargs -0 -P4 -I{} process_file {}
```

---

## 51.7 String Operations Performance

```bash
# String concatenation in a loop
# SLOW — string grows, copies entire string each time
result=""
for i in {1..10000}; do
    result="$result $i"          # O(n²) — gets slower each iteration
done

# FASTER — use printf to an array, then join
parts=()
for i in {1..10000}; do
    parts+=("$i")
done
result="${parts[*]}"

# FASTEST — use printf directly
result=$(printf '%s ' {1..10000})
```

---

## 51.8 When Bash Is the Wrong Tool

```
Use Bash when:
├── Gluing existing commands together
├── Automation and deployment scripts
├── Simple file operations and text processing
├── Prototyping and one-off tasks
└── < 100 lines of logic

Consider Python/Go/etc when:
├── Processing large data (> 100MB)
├── Complex data structures needed
├── Floating-point math required
├── Network programming / APIs
├── JSON/XML parsing
├── Concurrent/async operations
├── Script exceeds ~300 lines
└── Performance is critical
```

---

## 51.9 Optimization Checklist

```
1. □ Profile first — find the actual bottleneck
2. □ Replace external commands with builtins in loops
3. □ Replace line-by-line processing with awk/sed
4. □ Batch I/O: use { } > file instead of >> in loops
5. □ Use GNU parallel for CPU-bound tasks
6. □ Avoid unnecessary subshells and pipes
7. □ Use [[ ]] instead of [ ] (slightly faster, no fork)
8. □ Pre-compute values used repeatedly in loops
9. □ Consider if Bash is the right tool at all
```

---

## Exercises

### Exercise 51.1: Optimize a Script
Given a slow script that processes a log file line by line using multiple external commands per line, rewrite it to run 10x faster.

### Exercise 51.2: Benchmark Battle
Write two versions of a word-frequency counter: one using a Bash while-read loop, one using awk. Benchmark both on a large file and compare.

---

## Summary

- **Minimize forks** — the single biggest optimization for Bash scripts
- Use parameter expansion (`${var#...}`) instead of `basename`, `cut`, `tr`, `sed`
- Process files with `awk`/`sed` in one pass — not line-by-line in Bash loops
- Batch I/O with `{ } > file` instead of `>>` in loops
- Use `parallel` or `xargs -P` for parallel processing
- Benchmark with `time` before and after optimizing
- If performance is critical and the logic is complex, use Python or Go

---

**Next Chapter:** [Project 1: File Organizer →](../Part10-Projects/Project01-File-Organizer.md)
