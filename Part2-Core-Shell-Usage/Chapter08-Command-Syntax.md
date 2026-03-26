# Chapter 8: Command Syntax and Parsing

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand how Bash parses and interprets command lines
- Distinguish between simple commands, compound commands, and pipelines
- Use command lists with `;`, `&&`, and `||`
- Understand word splitting and its implications
- Use quoting to control parsing behavior
- Chain commands effectively

---

## 8.1 The Anatomy of a Command

Every command you type follows a consistent structure:

```
command [options] [arguments]
```

```bash
ls -la /home/john
│   │   │
│   │   └── Argument: what to operate on
│   └────── Options: how to operate (long format + show hidden)
└────────── Command: what to do (list directory contents)
```

### Options vs. Arguments

**Options** (also called **flags** or **switches**) modify how a command behaves. They typically begin with `-` or `--`:

```bash
# Short options (single dash, single letter)
ls -l         # Long format
ls -a         # Show hidden files
ls -la        # Combined: -l and -a

# Long options (double dash, full word)
ls --all              # Same as -a
ls --human-readable   # Same as -h
grep --ignore-case    # Same as -i
grep --count          # Same as -c

# Options with values
head -n 5             # Short option with value
head --lines=5        # Long option with value
grep -e "pattern"     # -e takes "pattern" as its value
```

**Arguments** are the targets — files, directories, strings — that the command operates on:

```bash
cp source.txt dest.txt      # Two arguments
grep "error" /var/log/syslog # Two arguments: pattern and file
```

### The `--` End-of-Options Marker

The double dash `--` signals that everything after it is an argument, not an option. This is essential when arguments look like options:

```bash
# Problem: the file is named "-important.txt"
rm -important.txt            # Error! Bash thinks -i, -m, -p, etc. are flags

# Solution: use -- to end option processing
rm -- -important.txt         # Correct! "-important.txt" is treated as a filename

# Another example
grep -- "-v" file.txt        # Search for the literal string "-v"
```

---

## 8.2 How Bash Processes a Command Line

When you press Enter, Bash doesn't simply run what you typed. It goes through a multi-stage processing pipeline. Understanding this pipeline is key to mastering Bash.

### The Complete Processing Order

```
Input: echo "Today is $(date +%A)" > output.txt

1. TOKENIZATION (Lexical Analysis)
   Break input into words and operators
   Tokens: [echo] ["Today is $(date +%A)"] [>] [output.txt]

2. PARSING (Syntax Analysis)
   Group tokens into command structure
   Command: echo
   Args: ["Today is $(date +%A)"]
   Redirect: > output.txt

3. EXPANSIONS (in this strict order):
   a. Brace expansion:       {a,b,c} → a b c
   b. Tilde expansion:       ~ → /home/john
   c. Parameter expansion:   $VAR → value
   d. Command substitution:  $(date +%A) → "Wednesday"
   e. Arithmetic expansion:  $((1+2)) → 3
   f. Process substitution:  <(cmd) → /dev/fd/63
   g. Word splitting:        Split unquoted results on IFS
   h. Filename expansion:    *.txt → file1.txt file2.txt

4. QUOTE REMOVAL
   Remove quotes that served their purpose

5. REDIRECTION
   Set up I/O (open output.txt for writing)

6. COMMAND LOOKUP
   Find the command (builtin? function? alias? external program?)

7. EXECUTION
   Run the command with processed arguments
```

### Why the Expansion Order Matters

Each expansion stage processes the results of the previous stages:

```bash
# Brace expansion happens BEFORE variable expansion
name="john"
echo {$name,alice}
# Output: {john,alice}  — Wait, this doesn't look right!
# Actually: Brace expansion runs first and sees {$name,alice}
# Then variable expansion converts $name → john
# Result: john alice

# Compare:
files="*.txt"
echo $files       # Filename expansion happens: file1.txt file2.txt
echo "$files"     # Quoting prevents filename expansion: *.txt
```

---

## 8.3 Command Separators and Lists

### Semicolon `;` — Sequential Execution

Run commands one after another, regardless of success or failure:

```bash
# Both commands run regardless of outcomes
echo "First" ; echo "Second"

# Useful for one-liners
cd /tmp ; ls ; cd -

# Each command's exit status is independent
false ; echo "This still runs"    # Output: This still runs
```

### `&&` — Run Next Only If Previous Succeeded

The right side runs only if the left side exits with status 0 (success):

```bash
# Only echo if mkdir succeeds
mkdir newdir && echo "Directory created"

# Chain dependent operations
cd /tmp && touch testfile && echo "Done"

# If cd fails, nothing else runs
cd /nonexistent && echo "You won't see this"
```

### `||` — Run Next Only If Previous Failed

The right side runs only if the left side exits with a non-zero status (failure):

```bash
# Provide a fallback
cd /nonexistent || echo "Directory doesn't exist"

# Try one thing, fall back to another
which python3 || which python || echo "No Python found!"

# Error handling pattern
mkdir newdir || { echo "Failed to create directory"; exit 1; }
```

### Combining `&&` and `||`

```bash
# Simple success/failure branching (NOT a full if/else — see caveat below)
ping -c 1 google.com && echo "Online" || echo "Offline"

# Common pattern: try and report
command && echo "Success" || echo "Failure"
```

**Caveat:** `cmd1 && cmd2 || cmd3` is NOT the same as if/then/else. If `cmd2` fails, `cmd3` will ALSO run. For proper branching, use `if` statements (Chapter 20).

### Newlines as Separators

A newline acts like `;` — it separates commands:

```bash
# These are equivalent:
echo "Hello" ; echo "World"

echo "Hello"
echo "World"
```

---

## 8.4 Command Grouping

### Braces `{ }` — Group in Current Shell

Commands inside braces run in the **current shell** (same process, same variables):

```bash
# Group commands to apply a single redirection
{ echo "Header"; cat data.txt; echo "Footer"; } > output.txt

# Use with || for error handling
mkdir newdir || { echo "Error: cannot create dir" >&2; exit 1; }
```

**Syntax requirements:**
- Opening `{` must be followed by a space
- Closing `}` must be preceded by `;` or newline
- Closing `}` must be on its own or after `;`

```bash
# CORRECT
{ echo "hello"; echo "world"; }

# WRONG — missing space after { and ; before }
{echo "hello"; echo "world"}
```

### Parentheses `( )` — Group in Subshell

Commands inside parentheses run in a **subshell** (a new process). Changes to variables and directory don't affect the parent:

```bash
# Changes are isolated
x=10
( x=20; echo "Inside: $x" )    # Inside: 20
echo "Outside: $x"              # Outside: 10

# Useful for temporary directory changes
( cd /tmp && do_something )
# You're still in your original directory
pwd    # Not /tmp

# Combine output from multiple commands
( cat header.txt; process_data; cat footer.txt ) | send_email
```

---

## 8.5 Brace Expansion

Brace expansion generates lists of strings. It's performed BEFORE any other expansion.

### Comma-Separated Lists

```bash
echo {apple,banana,cherry}
# apple banana cherry

echo file.{txt,md,html}
# file.txt file.md file.html

# Practical uses:
mkdir -p project/{src,tests,docs,config}

# Create multiple files
touch report.{2024,2025,2026}.txt

# Copy with a backup extension
cp config.ini{,.bak}
# Expands to: cp config.ini config.ini.bak

# Rename using brace expansion
mv file.{txt,md}
# Expands to: mv file.txt file.md
```

### Sequence Expressions

```bash
# Numeric ranges
echo {1..10}
# 1 2 3 4 5 6 7 8 9 10

echo {01..10}
# 01 02 03 04 05 06 07 08 09 10  (zero-padded)

echo {1..20..2}
# 1 3 5 7 9 11 13 15 17 19  (step by 2)

# Letter ranges
echo {a..z}
# a b c d e f g h i j k l m n o p q r s t u v w x y z

echo {A..Z}
# A B C D E F G H I J K L M N O P Q R S T U V W X Y Z

# Nested braces
echo {A,B}{1,2}
# A1 A2 B1 B2

echo {a..c}{1..3}
# a1 a2 a3 b1 b2 b3 c1 c2 c3
```

### Practical Brace Expansion Examples

```bash
# Create a year/month directory structure
mkdir -p logs/{2024,2025,2026}/{01..12}

# Download sequential files
# wget https://example.com/image_{001..100}.jpg

# Compare two files quickly
diff <(cat file1) <(cat file2)    # (process substitution — Ch. 44)
```

---

## 8.6 Tilde Expansion

The tilde `~` is expanded to directory paths:

```bash
echo ~              # /home/john  (your home)
echo ~/Documents    # /home/john/Documents
echo ~root          # /root  (root's home)
echo ~alice         # /home/alice  (another user's home)
echo ~+             # $PWD  (current directory)
echo ~-             # $OLDPWD  (previous directory)
```

**Important:** Tilde expansion only works at the beginning of a word and when unquoted:

```bash
echo ~           # /home/john
echo "~"         # ~ (literally — quotes prevent expansion)
echo ~/file      # /home/john/file
echo ~john/file  # /home/john/file
```

---

## 8.7 Word Splitting

After expansions, Bash splits unquoted results into separate words using the **Internal Field Separator (IFS)**.

### Default IFS

The default IFS contains three characters: space, tab, and newline.

```bash
# Word splitting in action
files="one two three"
for f in $files; do         # $files is UNQUOTED — word splitting occurs
    echo "File: $f"
done
# File: one
# File: two
# File: three

# With quotes — NO word splitting
for f in "$files"; do       # $files is QUOTED — treated as one word
    echo "File: $f"
done
# File: one two three
```

### Custom IFS

You can change IFS for specialized parsing:

```bash
# Parse a colon-separated string (like PATH)
IFS=':' read -ra dirs <<< "$PATH"
for dir in "${dirs[@]}"; do
    echo "$dir"
done

# Parse CSV (comma-separated)
line="John,Doe,42"
IFS=',' read -r first last age <<< "$line"
echo "$first $last is $age years old"
# John Doe is 42 years old
```

### Why Word Splitting Matters

```bash
# Common bug: unquoted variable with spaces
filename="my important file.txt"

# WRONG — word splitting creates THREE arguments
rm $filename    # rm "my" "important" "file.txt" — deletes wrong files!

# CORRECT — quotes prevent word splitting
rm "$filename"  # rm "my important file.txt" — deletes the right file
```

**Rule: Always quote your variables** unless you specifically want word splitting.

---

## 8.8 Filename Expansion (Globbing)

After word splitting, Bash expands special characters into matching filenames.

| Pattern | Matches |
|---------|---------|
| `*` | Any string (including empty) |
| `?` | Any single character |
| `[abc]` | Any one of: a, b, c |
| `[a-z]` | Any one character in range a-z |
| `[!abc]` or `[^abc]` | Any character NOT in the set |

```bash
# Match all .txt files
ls *.txt

# Match single character
ls file?.txt        # file1.txt, fileA.txt, but not file10.txt

# Character classes
ls file[0-9].txt    # file0.txt through file9.txt
ls file[!0-9].txt   # fileA.txt, fileb.txt, but not file1.txt

# Combine patterns
ls *.{txt,md,sh}    # All .txt, .md, and .sh files
```

**Important:** Globbing only matches filenames that exist. If no files match, the pattern is left as-is (by default):

```bash
ls *.xyz
# ls: cannot access '*.xyz': No such file or directory
# The literal string "*.xyz" was passed to ls because nothing matched
```

---

## Common Mistakes

1. **Unquoted variables** — `$var` is susceptible to word splitting and globbing. Use `"$var"` unless you want those behaviors.

2. **Not understanding expansion order** — Brace expansion happens before variable expansion. `{$a,$b}` works differently than you might expect.

3. **Treating `&&`/`||` as if/then/else** — `cmd && success || failure` will run `failure` if `success` fails too. Use proper `if` statements for complex logic.

4. **Forgetting `--` before filenames** — Files starting with `-` will be interpreted as options without `--`.

5. **Missing spaces in grouping syntax** — `{ cmd; }` requires spaces after `{` and `;` before `}`.

---

## Exercises

### Exercise 8.1: Brace Expansion
Write one-line commands using brace expansion to:
1. Create directories `day1`, `day2`, ..., `day7`
2. Create `backup_jan.tar`, `backup_feb.tar`, ..., `backup_dec.tar` (touch)
3. Create a nested structure: `project/{frontend,backend}/{src,tests}`

### Exercise 8.2: Command Lists
What is the output/behavior of each line?
1. `true && echo "yes" || echo "no"`
2. `false && echo "yes" || echo "no"`
3. `false || true && echo "reached"`
4. `cd /nonexistent 2>/dev/null && echo "found" || echo "missing"`

### Exercise 8.3: Word Splitting
Predict the output:
```bash
var="one   two   three"
echo $var
echo "$var"
```
Why are they different?

### Exercise 8.4: Globbing
In a directory with files `a.txt`, `b.txt`, `ab.txt`, `abc.txt`, and `123.txt`, what does each pattern match?
1. `?.txt`
2. `[ab].txt`
3. `??.txt`
4. `[0-9]*.txt`
5. `*b*.txt`

---

## Summary

- Commands follow: `command [options] [arguments]`
- `;` runs commands sequentially; `&&` runs on success; `||` runs on failure
- Bash processes through: tokenize → expand → word-split → glob → execute
- **Brace expansion** generates lists: `{a,b,c}`, `{1..10}`
- **Word splitting** breaks unquoted expansions on IFS characters
- **Globbing** expands patterns into matching filenames
- **Always quote variables** to prevent word splitting and globbing bugs
- Use `--` to separate options from arguments

---

**Next Chapter:** [Chapter 9: Built-in Commands vs External Programs →](Chapter09-Builtins-vs-External.md)
