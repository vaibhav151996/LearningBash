# Chapter 42: Arrays

## Learning Objectives

By the end of this chapter, you will be able to:

- Create and manipulate indexed arrays
- Use associative arrays (hash maps)
- Iterate over arrays safely
- Apply common array patterns and idioms
- Avoid array-related pitfalls

---

## 42.1 Indexed Arrays

```bash
# Declaration
fruits=("apple" "banana" "cherry")
declare -a numbers=(1 2 3 4 5)

# Assignment
fruits[0]="apple"
fruits[1]="banana"
fruits[3]="date"      # Index 2 is unset (sparse array!)

# Append
fruits+=("elderberry")
fruits+=("fig" "grape")

# Access
echo "${fruits[0]}"           # apple
echo "${fruits[-1]}"          # Last element (Bash 4.3+)
echo "${fruits[@]}"           # All elements (space-separated)
echo "${#fruits[@]}"          # Number of elements
echo "${!fruits[@]}"          # All indices
```

---

## 42.2 Array Operations

```bash
arr=("one" "two" "three" "four" "five")

# Slice
echo "${arr[@]:1:3}"          # two three four (offset 1, count 3)
echo "${arr[@]:2}"            # three four five (offset 2 to end)

# Copy
copy=("${arr[@]}")

# Delete element
unset 'arr[2]'                # Removes index 2 (array becomes sparse!)

# Replace element
arr[1]="TWO"

# Length of element
echo "${#arr[0]}"             # Length of first element's string

# Search/filter (with loop)
for item in "${arr[@]}"; do
    [[ "$item" == *"o"* ]] && echo "$item"
done
```

---

## 42.3 Iterating Safely

```bash
# CORRECT — quote "${arr[@]}" to preserve elements with spaces
files=("my file.txt" "another doc.pdf" "script.sh")

for f in "${files[@]}"; do
    echo "Processing: $f"
done
# Processing: my file.txt
# Processing: another doc.pdf
# Processing: script.sh

# WRONG — unquoted splits on spaces
for f in ${files[@]}; do
    echo "Processing: $f"
done
# Processing: my
# Processing: file.txt        ← BROKEN!
# ...
```

> **Golden Rule:** Always use `"${array[@]}"` with double quotes when expanding arrays.

---

## 42.4 Associative Arrays (Bash 4+)

```bash
# Must declare with -A
declare -A config

# Assignment
config[host]="localhost"
config[port]="5432"
config[database]="myapp"

# Or inline
declare -A colors=(
    [red]="#FF0000"
    [green]="#00FF00"
    [blue]="#0000FF"
)

# Access
echo "${config[host]}"           # localhost
echo "${config[@]}"              # All values
echo "${!config[@]}"             # All keys
echo "${#config[@]}"             # Number of entries

# Check if key exists
if [[ -v config[host] ]]; then
    echo "host is set"
fi

# Delete entry
unset 'config[port]'

# Iterate
for key in "${!config[@]}"; do
    echo "$key = ${config[$key]}"
done
```

---

## 42.5 Practical Patterns

### Building Arrays from Commands

```bash
# From command output
mapfile -t lines < file.txt           # Read file into array (one line per element)
readarray -t lines < file.txt         # Same as mapfile

# From command
mapfile -t processes < <(ps -eo comm --no-headers)

# From glob
shopt -s nullglob
scripts=(*.sh)                         # Array of .sh files

# From splitting a string
IFS=',' read -ra fields <<< "a,b,c,d"
echo "${fields[1]}"                    # b
```

### Passing Arrays to Functions

```bash
# Pass all elements
process_items() {
    local items=("$@")
    for item in "${items[@]}"; do
        echo "Processing: $item"
    done
}
my_array=("one" "two" "three")
process_items "${my_array[@]}"

# Return an array (via nameref, Bash 4.3+)
get_items() {
    local -n result="$1"
    result=("item1" "item2" "item3")
}
get_items my_result
echo "${my_result[@]}"
```

### Counting Occurrences

```bash
declare -A count
while read -r word; do
    ((count[$word]++))
done < <(tr ' ' '\n' < document.txt)

for word in "${!count[@]}"; do
    printf "%4d %s\n" "${count[$word]}" "$word"
done | sort -rn | head -10
```

---

## 42.6 Common Mistakes

### Mistake 1: Forgetting declare -A for Associative Arrays

```bash
# BAD — creates indexed array, keys treated as arithmetic
config[host]="localhost"      # host evaluates to 0!

# GOOD
declare -A config
config[host]="localhost"
```

### Mistake 2: Sparse Arrays After unset

```bash
arr=(a b c d e)
unset 'arr[2]'
echo "${arr[@]}"         # a b d e
echo "${!arr[@]}"        # 0 1 3 4  ← index 2 is gone, not renumbered!
echo "${arr[2]}"         # (empty)
```

### Mistake 3: Using @ vs * Without Quotes

```bash
"${arr[@]}"    # Each element as separate word (CORRECT for loops)
"${arr[*]}"    # All elements as single string (useful for joining)

# Join with custom separator
IFS=','
echo "${arr[*]}"    # one,two,three
unset IFS
```

---

## Exercises

### Exercise 42.1: Array Utilities
Write functions: `array_contains`, `array_index_of`, `array_reverse`, `array_unique`, and `array_join` (join with delimiter).

### Exercise 42.2: Frequency Counter
Read a log file and use an associative array to count occurrences of each HTTP status code. Display results sorted by count.

---

## Summary

- Indexed arrays: `arr=("a" "b" "c")`, access with `${arr[0]}`
- Associative arrays: `declare -A map`, access with `${map[key]}`
- Always quote: `"${arr[@]}"` preserves elements with spaces
- `${!arr[@]}` gives indices/keys, `${#arr[@]}` gives count
- `mapfile -t` reads lines from a file/command into an array
- Arrays are sparse after `unset` — indices don't renumber
- Use `"${arr[*]}"` with custom IFS to join elements

---

**Next Chapter:** [Chapter 43: Advanced Parameter Expansion →](Chapter43-Parameter-Expansion.md)
