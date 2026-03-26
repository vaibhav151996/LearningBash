# Chapter 15: Variables and Quoting Rules

## Learning Objectives

By the end of this chapter, you will be able to:

- Declare and use variables correctly in scripts
- Master the three quoting styles: single, double, and ANSI-C
- Understand when to quote and when not to
- Use `declare` and `readonly` for variable attributes
- Avoid the most common variable-related bugs

---

## 15.1 Variable Basics

### Assignment

```bash
# No spaces around the equals sign!
name="John"
age=30
filename="report_2026.txt"
empty_var=""

# WRONG — these are errors
name = "John"       # Bash runs "name" as a command with args "=" and "John"
age =30              # Same problem
```

### Accessing Variables

```bash
# Use $ to access a variable's value
echo $name           # John
echo "Hello, $name"  # Hello, John

# Use ${} for clarity and when adjacent to other characters
echo "${name}'s file"     # John's file
echo "File: ${filename}"  # File: report_2026.txt
echo "${name}sson"         # Johnsson

# Without braces, this fails:
echo "$namesson"           # (empty — $namesson doesn't exist)
```

### Variable Types (They're All Strings!)

In Bash, **all variables are strings** by default. Even numbers are stored as strings:

```bash
num=42
echo $num        # "42" — it's a string that happens to look like a number

# Arithmetic requires special syntax (covered in Chapter 16)
total=$((num + 8))
echo $total      # 50
```

---

## 15.2 The Three Quoting Styles

Quoting controls how the shell interprets special characters. This is one of the most important topics in Bash.

### Double Quotes `"..."` — Partial Protection

Double quotes preserve literal values of most characters but **allow** three types of expansion:

1. Variable expansion: `$var` and `${var}`
2. Command substitution: `$(command)` and `` `command` ``
3. Arithmetic expansion: `$((expression))`

```bash
name="World"
echo "Hello, $name"          # Hello, World
echo "Today is $(date +%A)"  # Today is Wednesday
echo "2 + 2 = $((2 + 2))"   # 2 + 2 = 4

# Special characters ARE protected:
echo "Stars: * * *"          # Stars: * * *   (no globbing!)
echo "Semicolons; here"     # Semicolons; here  (no command separation)
echo "Pipes | here"          # Pipes | here  (no piping!)
```

### Single Quotes `'...'` — Full Protection

Single quotes preserve **everything** literally. No expansion of any kind occurs:

```bash
echo '$name is literal'           # $name is literal
echo '$(date) is literal'        # $(date) is literal
echo 'Stars: * * *'              # Stars: * * *
echo 'Backslash: \n is literal'  # Backslash: \n is literal

# The ONLY character you can't include inside single quotes is a single quote
```

### ANSI-C Quotes `$'...'` — Escape Sequences

ANSI-C quoting allows C-style escape sequences:

```bash
echo $'Hello\tWorld'      # Hello    World  (tab)
echo $'Line1\nLine2'      # Line1
                           # Line2
echo $'It\'s alive'       # It's alive
echo $'\x41'              # A  (hex)
echo $'\u2764'            # ❤  (Unicode)
```

| Escape | Meaning |
|--------|---------|
| `\n` | Newline |
| `\t` | Tab |
| `\\` | Literal backslash |
| `\'` | Literal single quote |
| `\a` | Alert (bell) |
| `\xHH` | Hex byte |
| `\uHHHH` | Unicode character |

### Comparison Table

| Input | Double Quotes | Single Quotes | ANSI-C Quotes |
|-------|--------------|---------------|---------------|
| `$name` | Value of name | Literal `$name` | Literal `$name` |
| `$(date)` | Command output | Literal `$(date)` | Literal `$(date)` |
| `\n` | Literal `\n` | Literal `\n` | Newline character |
| `*` | Literal `*` | Literal `*` | Literal `*` |

### The Quoting Decision Tree

```
Do you need variable/command expansion?
├── YES → Use double quotes: "..."
│   Do you need escape sequences too?
│   └── YES → Use $'...' or combine: "text $var"$'\n'"more text"
│
└── NO → Use single quotes: '...'
    Does the string contain single quotes?
    ├── NO  → Use single quotes: 'string'
    └── YES → Use $'it\'s here' or "it's here"
```

---

## 15.3 When to Quote (and When Not To)

### The Golden Rule

> **Quote every variable expansion.** Use `"$var"`, not `$var`.

```bash
# WRONG — breaks on filenames with spaces
filename="my document.txt"
cat $filename        # cat "my" "document.txt" — TWO arguments!

# CORRECT
cat "$filename"      # cat "my document.txt" — ONE argument
```

### When Quoting Is Mandatory

```bash
# Always quote variable expansions
echo "$variable"
rm "$filename"
cd "$directory"
if [ "$var" = "value" ]; then ...

# Always quote command substitutions
result="$(some_command)"
echo "Output: $(some_command)"
```

### When You WANT Word Splitting (Rare)

```bash
# Reading a list of items from a variable
options="-la --color=auto"
ls $options              # Intentionally split into separate arguments

# Iterating over words
words="one two three"
for word in $words; do   # Intentionally split
    echo "$word"
done
```

Even in these cases, using arrays is usually better (Chapter 42).

---

## 15.4 Variable Attributes with declare

`declare` lets you set attributes on variables:

```bash
# Integer attribute — enforces arithmetic
declare -i count=0
count=count+1        # Arithmetic happens automatically!
echo $count          # 1
count="hello"
echo $count          # 0 (non-numeric strings become 0)

# Read-only variable
declare -r PI=3.14159
PI=3.0               # Error: PI: readonly variable

# readonly is equivalent
readonly MAX_RETRIES=3

# Uppercase attribute (Bash 4+)
declare -u upper="hello"
echo $upper          # HELLO

# Lowercase attribute (Bash 4+)
declare -l lower="HELLO"
echo $lower          # hello

# Export attribute
declare -x MY_VAR="exported"
# Equivalent to: export MY_VAR="exported"

# List all variables with their attributes
declare -p
declare -p name      # Show attributes of specific variable
```

---

## 15.5 Indirect Variable References

Sometimes you want to use a variable whose name is stored in another variable:

```bash
# The variable name is in another variable
color="red"
red="FF0000"

# Indirect expansion with ${!var}
echo "${!color}"     # FF0000  (expanded $red because color="red")

# Practical use case: dynamic configuration
for setting in HOST PORT DATABASE; do
    varname="DB_${setting}"
    echo "${setting}: ${!varname}"
done
# With DB_HOST=localhost, DB_PORT=5432, DB_DATABASE=myapp:
# HOST: localhost
# PORT: 5432
# DATABASE: myapp
```

---

## 15.6 Common Quoting Patterns

### Mixing Quote Types

```bash
# Switch between quote types in the same argument
echo "She said, '"'"'hello'"'"'"
# Complex! Easier with $'...'
echo $'She said, \'hello\''

# Or mix double and single quotes
echo "It's a beautiful day"    # Double quotes handle the apostrophe

# Include a literal double quote in double quotes
echo "She said \"hello\""

# Include a literal dollar sign in double quotes
echo "Price: \$9.99"
```

### Quoting in Assignments

```bash
# Quotes in assignments prevent word splitting and globbing
path="/my/path with spaces"      # Double quotes needed
pattern='*.txt'                   # Single quotes to prevent premature expansion
message="Hello, $USER!"          # Double quotes for expansion

# Variable references in assignments don't NEED quotes
# (assignment is a special context — no word splitting)
x=$path                          # Works fine even without quotes
# But quoting is still recommended for consistency
x="$path"                        # Better practice
```

---

## 15.7 Quoting Special Characters

### The Backslash `\` — Escape a Single Character

```bash
echo "Price: \$10"       # Price: $10
echo "Newline: \\n"      # Newline: \n
echo "Quote: \""         # Quote: "
echo file\ name.txt      # file name.txt (spaces escaped)
```

Outside quotes, `\` preserves the literal value of the next character:

```bash
echo Hello\ World       # Hello World (space escaped)
echo \$HOME             # $HOME (dollar sign literal)
echo \*                 # * (no globbing)
```

---

## Common Mistakes

1. **Spaces around `=`** — `var = "value"` is wrong. No spaces!

2. **Unquoted variables** — Always use `"$var"`. The only time to omit quotes is when you specifically want word splitting.

3. **Confusing single and double quotes** — Use double quotes when you need expansion, single quotes when you don't.

4. **Not quoting in `[` commands** — `[ $var = "test" ]` breaks if `$var` is empty or contains spaces. Use `[ "$var" = "test" ]`.

5. **Forgetting `${var}` braces** — `echo "$varname"` is different from `echo "${var}name"`.

---

## Exercises

### Exercise 15.1: Quoting Practice
Predict the output of each:
```bash
greeting="Hello World"
echo $greeting
echo "$greeting"
echo '$greeting'
echo "'$greeting'"
echo "\"$greeting\""
```

### Exercise 15.2: Variable Expansion
What does each print?
```bash
animal="cat"
echo "${animal}s"
echo "$animals"
echo "${animal}erpillar"
```

### Exercise 15.3: Filename Safety
Create a file named `my important file.txt` (with spaces). Write commands to:
1. Display its contents
2. Copy it to `backup.txt`
3. Delete it

All must work correctly with the spaces.

---

## Summary

- Variables are assigned with `name=value` — **no spaces around `=`**
- Access with `$var` or `${var}`; braces resolve ambiguity
- **Double quotes** `"..."`: allow `$`, `$()`, `$(())` expansion
- **Single quotes** `'...'`: everything is literal
- **ANSI-C quotes** `$'...'`: C-style escape sequences
- **Always quote variable expansions**: `"$var"` is correct; `$var` is risky
- Use `declare -r` or `readonly` for constants
- `declare -i` enforces integer arithmetic
- All Bash variables are strings unless given special attributes

---

**Next Chapter:** [Chapter 16: Arithmetic Operations →](Chapter16-Arithmetic.md)
