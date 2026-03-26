# Chapter 7: Getting Help — man, info, and Beyond

## Learning Objectives

By the end of this chapter, you will be able to:

- Use `man` pages effectively to look up any command
- Navigate man page sections
- Use `info`, `help`, `--help`, and `apropos` for different types of documentation
- Find answers independently — the most important skill for a Linux user
- Understand how to read command syntax notation

---

## 7.1 The man Command — Your Primary Reference

`man` (manual) is the built-in documentation system on every Linux system. It contains precise, authoritative documentation for nearly every command, system call, and configuration file.

```bash
# Read the manual for any command
man ls
man chmod
man bash     # The complete Bash manual — enormous and comprehensive
```

### Navigating man Pages

Man pages use the `less` pager, so the same navigation keys work:

| Key | Action |
|-----|--------|
| `Space` | Next page |
| `b` | Previous page |
| `g` | Go to beginning |
| `G` | Go to end |
| `/pattern` | Search forward |
| `?pattern` | Search backward |
| `n` | Next search result |
| `N` | Previous search result |
| `q` | Quit |

### The Structure of a man Page

Every man page follows a standard structure:

```
NAME
    ls - list directory contents

SYNOPSIS
    ls [OPTION]... [FILE]...

DESCRIPTION
    List information about the FILEs (the current directory by default).
    Sort entries alphabetically if none of -cftuvSUX nor --sort is specified.

    -a, --all
        do not ignore entries starting with .

    -l
        use a long listing format

    ...

EXAMPLES
    ls -la /home
        List all files including hidden ones in long format

SEE ALSO
    dir(1), vdir(1), dircolors(1)

AUTHOR
    Written by Richard M. Stallman and David MacKenzie.
```

Key sections:

| Section | Contents |
|---------|----------|
| NAME | Command name and one-line description |
| SYNOPSIS | Usage syntax (how to call the command) |
| DESCRIPTION | Detailed explanation of what the command does |
| OPTIONS | All available flags and options |
| EXAMPLES | Usage examples (not always present, unfortunately) |
| FILES | Configuration files the command uses |
| EXIT STATUS | What exit codes the command returns |
| SEE ALSO | Related commands and man pages |
| BUGS | Known issues |
| AUTHOR | Who wrote it |

---

## 7.2 Reading Command Syntax Notation

Man page synopses use a standard notation that can look cryptic at first. Learn to read it:

```
ls [OPTION]... [FILE]...
```

| Notation | Meaning |
|----------|---------|
| `UPPERCASE` | Placeholder — you replace with an actual value |
| `[brackets]` | Optional — you can omit this |
| `...` | Repeatable — you can provide multiple |
| `|` (pipe) | OR — choose one |
| `{braces}` | Required choice — pick one from the list |
| No brackets | Required — you must provide this |

### Examples

```
cp [OPTION]... SOURCE... DEST
```
- `[OPTION]...` — Zero or more optional flags
- `SOURCE...` — One or more source files (required)
- `DEST` — One destination (required)

```
grep [OPTIONS] PATTERN [FILE...]
```
- `[OPTIONS]` — Optional flags
- `PATTERN` — Search pattern (required)
- `[FILE...]` — Zero or more files (optional; reads stdin if omitted)

```
chmod [OPTION]... MODE[,MODE]... FILE...
```
- `MODE[,MODE]...` — One or more modes, comma-separated
- `FILE...` — One or more files (required)

---

## 7.3 Man Page Sections

Man pages are organized into **numbered sections** because the same name might refer to different things.

| Section | Contents | Example |
|---------|----------|---------|
| 1 | User commands | `man 1 ls` |
| 2 | System calls (kernel functions) | `man 2 open` |
| 3 | Library functions (C library) | `man 3 printf` |
| 4 | Special files (/dev) | `man 4 null` |
| 5 | File formats and conventions | `man 5 passwd` |
| 6 | Games and screensavers | `man 6 sl` |
| 7 | Miscellaneous (conventions, protocols) | `man 7 regex` |
| 8 | System administration commands | `man 8 mount` |

### Why Sections Matter

Some names exist in multiple sections:

```bash
# printf — the command (Section 1)
man 1 printf

# printf — the C library function (Section 3)
man 3 printf

# passwd — the command to change passwords (Section 1)
man 1 passwd

# passwd — the password file format (Section 5)
man 5 passwd
```

Without a section number, `man` returns the first match (usually Section 1):

```bash
man printf     # Shows Section 1 (the command)
man 3 printf   # Shows Section 3 (the C function)
```

The man page header shows the section: `PRINTF(1)` means Section 1.

---

## 7.4 Searching for man Pages

### apropos — Search by Keyword

When you don't know the command name, search by what it does:

```bash
# Find commands related to "password"
apropos password
# chage (1)           - change user password expiry information
# chpasswd (8)        - update passwords in batch mode
# passwd (1)          - change user password
# passwd (5)          - the password file
# ...

# Find commands related to "network"
apropos network

# Find commands related to "compress"
apropos compress

# apropos is equivalent to:
man -k password
```

### whatis — One-Line Description

```bash
whatis ls
# ls (1)               - list directory contents

whatis chmod
# chmod (1)            - change file mode bits

# Equivalent to:
man -f ls
```

### Searching Within a man Page

When the man page is open, use `/` to search:

```bash
man bash
# Then press / and type: variable
# Press n for next match, N for previous
```

---

## 7.5 The help Command — For Bash Built-ins

Some commands are **built into Bash** itself and don't have separate man pages (or have minimal ones). Use `help` for these:

```bash
# Get help on built-in commands
help cd
help echo
help export
help type
help read
help test

# List ALL built-in commands
help

# Compare:
man cd        # Might not exist or show minimal info
help cd       # Shows the built-in documentation
```

### How to Know If a Command Is Built-in

```bash
type cd
# cd is a shell builtin

type ls
# ls is aliased to 'ls --color=auto'

type grep
# grep is /usr/bin/grep

type echo
# echo is a shell builtin
# (echo is BOTH a builtin and an external command)
```

Built-in commands → use `help`
External commands → use `man`

---

## 7.6 The --help Flag

Almost every command supports `--help` for a quick usage summary:

```bash
ls --help
grep --help
chmod --help

# Some commands use -h instead
head -h
```

`--help` output is typically shorter and more practical than the full man page. Use it when you need a quick reminder of the options.

---

## 7.7 The info Command

GNU programs often have `info` pages that are more detailed than man pages, with hyperlinks and a hierarchical structure.

```bash
info coreutils    # Documentation for all core GNU utilities
info bash         # Very detailed Bash documentation
info grep         # grep documentation with examples
```

`info` navigation:

| Key | Action |
|-----|--------|
| `Space` | Next page |
| `Backspace` | Previous page |
| `n` | Next node (section) |
| `p` | Previous node |
| `u` | Up one level |
| `Enter` | Follow a link (when cursor is on `*` menu item) |
| `l` | Go back |
| `q` | Quit |

**Honestly:** Most people use `man` and `--help` rather than `info`. But `info` pages for some GNU tools contain the best documentation available.

---

## 7.8 Online Resources

### Official Documentation

- **Bash Reference Manual**: https://www.gnu.org/software/bash/manual/
- **Linux man pages online**: https://man7.org/linux/man-pages/
- **TLDR pages**: Simplified, practical man pages

### TLDR — Community-Driven Examples

The `tldr` tool provides simplified, example-driven help:

```bash
# Install (if not available)
sudo apt install tldr
# or
npm install -g tldr

# Usage
tldr tar
# tar
# Archiving utility.
# Often combined with a compression method, such as gzip or bzip2.
#
# Create an archive from files:
#   tar cf target.tar file1 file2 file3
#
# Extract an archive:
#   tar xf source.tar
#
# Create a gzipped archive:
#   tar czf target.tar.gz file1 file2 file3
```

`tldr` gives you the 20% of options you use 80% of the time. It's excellent for quick reference.

### explainshell.com

Paste any shell command and get each part explained:

```
Input: tar -czf backup.tar.gz --exclude='*.log' /var/www

Output:
- tar: an archiving utility
- -c: create a new archive
- -z: filter the archive through gzip
- -f backup.tar.gz: use archive file backup.tar.gz
- --exclude='*.log': exclude files matching *.log
- /var/www: source directory
```

---

## 7.9 Building Your Research Skills

The ability to find answers independently is the single most valuable skill in Linux. Here's a systematic approach:

### The Help Hierarchy

```
1. --help flag          Quick reminder of options
       │
2. man page             Detailed, authoritative documentation
       │
3. help command          For Bash built-ins
       │
4. info page            Extended GNU documentation
       │
5. tldr                 Practical examples
       │
6. Online search        Broader community knowledge
```

### How to Read a Man Page Efficiently

Don't read the entire man page. Use this approach:

1. **Read NAME** — Make sure it's the right command
2. **Read SYNOPSIS** — Understand the basic usage pattern
3. **Search for your flag** — Press `/` and search for the option you need
4. **Read EXAMPLES** — If present, these are gold
5. **Check SEE ALSO** — There might be a better command

```bash
# Example: You want to know how to copy directories with cp
man cp
# Press /recursive
# Find: -R, -r, --recursive
#        copy directories recursively
```

### How to Search When You Don't Know the Command

```bash
# 1. Search by keyword
apropos "disk usage"
# du (1) - estimate file space usage

# 2. Search man pages for a phrase
man -K "create directory"    # Capital K — searches FULL TEXT (slow)

# 3. Use Google/Stack Overflow with specific terms
# "linux command to find large files"
# "bash how to count lines in a file"
```

---

## 7.10 Getting Help on Bash Itself

The Bash manual is one of the most important references you'll use.

```bash
# The complete Bash manual
man bash

# This is HUGE — use searches:
# /EXPANSION        → Variable expansion rules
# /REDIRECTION      → I/O redirection
# /CONDITIONAL      → Conditional expressions
# /ARITHMETIC       → Arithmetic evaluation
# /PARAMETER        → Special parameters ($?, $$, etc.)
# /QUOTING          → Quoting rules
# /PROMPTING        → PS1, PS2, etc.
```

### Bash Built-in Help Categories

```bash
# All built-in commands
help

# Help on a specific topic
help variables    # Shell variables
help expansion    # Expansion rules
help test         # Test/conditional expressions
help for          # for loop syntax
help while        # while loop syntax
help if           # if statement syntax
```

---

## Common Mistakes

1. **Not reading man pages** — Many people immediately search online, but the man page often has the answer faster and more accurately.

2. **Being intimidated by man pages** — They're dense, but you don't need to read them cover to cover. Search for what you need with `/`.

3. **Forgetting about `help`** — For built-in commands like `cd`, `export`, `read`, `type`, `test`, the man page for that specific command might not exist. Use `help` instead.

4. **Not checking `SEE ALSO`** — The related commands section often leads you to the right tool. Looking at `man sort`? It references `uniq`, which might be what you actually need.

5. **Ignoring the SYNOPSIS** — The synopsis tells you the exact syntax. `[OPTION]` means optional. No brackets means required.

---

## Exercises

### Exercise 7.1: Man Page Navigation
1. Open `man grep` and find the option for case-insensitive search
2. Open `man find` and find how to search by file size
3. Open `man chmod` and find the example section (if it has one)
4. Open `man bash` and search for "QUOTING" — read the three types of quoting

### Exercise 7.2: Finding Commands
Use `apropos` to find commands that:
1. Compare files
2. Sort text
3. Count words
4. Compress files
5. Monitor processes

### Exercise 7.3: Built-in vs External
Determine whether each of these commands is a built-in or external program:
1. `echo`
2. `cd`
3. `grep`
4. `export`
5. `test`
6. `[`
7. `awk`
8. `type`

Use `type` to check each one.

### Exercise 7.4: Reading Synopses
Interpret these command synopses. What is required? What is optional?
1. `cp [OPTION]... [-T] SOURCE DEST`
2. `grep [OPTIONS] PATTERN [FILE...]`
3. `find [-H] [-L] [-P] [-D debugopts] [-Olevel] [starting-point...] [expression]`

### Exercise 7.5: Self-Directed Research
Without looking at any future chapters in this book, use only `man`, `help`, and `--help` to figure out:
1. How to reverse the output of `sort`
2. How to number the lines when using `cat`
3. How to create a symbolic link with `ln`
4. How to show disk usage in human-readable format with `du`

---

## Summary

- **`man command`** — The primary reference for any command; navigate with `/`, `n`, `q`
- **`help command`** — Documentation for Bash built-in commands
- **`command --help`** — Quick usage summary
- **`apropos keyword`** — Search for commands by what they do
- **`type command`** — Determine if a command is built-in, alias, or external
- **`info command`** — Extended GNU documentation
- **Man page sections**: 1=commands, 5=file formats, 8=admin commands
- **Learn to read synopses**: `[brackets]` = optional, `CAPS` = placeholder, `...` = repeatable
- **Search within man pages** using `/pattern` — don't read cover to cover
- **Learning to find answers yourself** is the most important skill in Linux

---

## Part 1 Summary: Foundations Complete

You now understand:

1. **What Linux is** — A free, open-source operating system kernel
2. **What a shell is** — A command interpreter (Bash) running inside a terminal
3. **Terminal navigation** — `pwd`, `cd`, `ls`, creating and managing files
4. **The filesystem** — Every directory has a purpose, governed by the FHS
5. **Permissions** — Owner/group/others with rwx, modified by `chmod`/`chown`
6. **Environment variables** — Key-value pairs that configure the system; PATH finds commands
7. **Getting help** — `man`, `help`, `--help`, `apropos`

These foundations will support everything that follows. In Part 2, we'll dive deeper into how the shell actually works — command parsing, pipes, redirections, and job control.

---

**Next Chapter:** [Part 2, Chapter 8: Command Syntax and Parsing →](../Part2-Core-Shell-Usage/Chapter08-Command-Syntax.md)
