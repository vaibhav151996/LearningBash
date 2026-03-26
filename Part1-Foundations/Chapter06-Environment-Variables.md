# Chapter 6: Environment Variables and PATH

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain what environment variables are and why they exist
- View, set, and modify environment variables
- Understand and manipulate the PATH variable
- Distinguish between shell variables and environment variables
- Configure persistent environment settings
- Understand variable scope and inheritance

---

## 6.1 What Are Environment Variables?

An **environment variable** is a named value stored in the shell's memory that can be passed to programs. Think of them as system-wide settings that programs can read.

When a program starts, it receives a copy of the environment — a collection of key-value pairs. Programs use these to customize their behavior without needing configuration files or command-line arguments.

```
┌─────────────────────────────────────────┐
│         Shell Environment               │
│                                         │
│  HOME=/home/john                        │
│  USER=john                              │
│  PATH=/usr/bin:/bin:/usr/local/bin       │
│  SHELL=/bin/bash                         │
│  LANG=en_US.UTF-8                       │
│  EDITOR=vim                              │
│  TERM=xterm-256color                    │
│                                         │
│  When you run a program, it gets a      │
│  COPY of these variables.               │
└─────────────────────────────────────────┘
```

---

## 6.2 Viewing Environment Variables

### View All Environment Variables

```bash
# Show all environment variables
env

# Alternative — shows both environment and shell-local variables
set

# Show all exported variables (the actual environment)
export -p

# Sorted output — easier to read
env | sort
```

### View a Specific Variable

```bash
# Use echo with $
echo $HOME
# /home/john

echo $USER
# john

echo $SHELL
# /bin/bash

# Safer syntax with braces (helps disambiguate)
echo ${HOME}
echo "My home directory is: ${HOME}"
```

### Important Built-in Variables

| Variable | Purpose | Example Value |
|----------|---------|---------------|
| `HOME` | User's home directory | `/home/john` |
| `USER` | Current username | `john` |
| `LOGNAME` | Login name | `john` |
| `SHELL` | User's default shell | `/bin/bash` |
| `PATH` | Command search path | `/usr/bin:/bin` |
| `PWD` | Current working directory | `/home/john` |
| `OLDPWD` | Previous directory | `/tmp` |
| `HOSTNAME` | Machine hostname | `myserver` |
| `LANG` | Locale setting | `en_US.UTF-8` |
| `TERM` | Terminal type | `xterm-256color` |
| `EDITOR` | Default text editor | `vim` |
| `VISUAL` | Default visual editor | `vim` |
| `PAGER` | Default pager program | `less` |
| `UID` | User's numeric ID | `1000` |
| `RANDOM` | Random number (0-32767) | `17324` |
| `SECONDS` | Seconds since shell started | `3842` |
| `HISTSIZE` | History lines to keep in memory | `1000` |
| `HISTFILE` | History file location | `~/.bash_history` |
| `PS1` | Prompt string | `\u@\h:\w\$ ` |
| `IFS` | Internal Field Separator | (space, tab, newline) |

---

## 6.3 Shell Variables vs. Environment Variables

This distinction is critical and often misunderstood.

### Shell Variables

A **shell variable** exists only in the current shell. It is NOT passed to child processes (programs you run).

```bash
# Create a shell variable (no export)
greeting="Hello, World"
echo $greeting
# Hello, World

# It does NOT get passed to child processes
bash -c 'echo $greeting'
# (empty — child process doesn't see it)
```

### Environment Variables

An **environment variable** is **exported** — meaning it IS passed to child processes.

```bash
# Export a variable to the environment
export greeting="Hello, World"
echo $greeting
# Hello, World

# Now child processes CAN see it
bash -c 'echo $greeting'
# Hello, World
```

### The Key Difference

```
┌──────── Parent Shell ────────┐
│                               │
│  Shell variable: x=10         │  ← only visible here
│  Env variable:   export y=20  │  ← visible here AND in children
│                               │
│  ┌──── Child Process ─────┐  │
│  │                         │  │
│  │  echo $x  → (empty)    │  │  ← x is NOT inherited
│  │  echo $y  → 20         │  │  ← y IS inherited
│  │                         │  │
│  └─────────────────────────┘  │
│                               │
└───────────────────────────────┘
```

### Creating and Exporting Variables

```bash
# Method 1: Create variable, then export
MY_VAR="some value"
export MY_VAR

# Method 2: Create and export in one step
export MY_VAR="some value"

# Method 3: Set variable for a single command only
MY_VAR="some value" ./my_program
# MY_VAR is set ONLY for my_program, not in the current shell

# Check if a variable is exported
export -p | grep MY_VAR
```

### Unsetting Variables

```bash
# Remove a variable
unset MY_VAR
echo $MY_VAR
# (empty)
```

---

## 6.4 The PATH Variable — How the Shell Finds Commands

`PATH` is the most important environment variable. It tells the shell **where to look for executable programs**.

### How Command Lookup Works

When you type a command like `ls`, the shell doesn't magically know where the `ls` program is. It searches through the directories listed in `PATH`, in order, until it finds a match.

```bash
echo $PATH
# /usr/local/bin:/usr/bin:/bin:/usr/local/sbin:/usr/sbin:/sbin
```

The directories are separated by colons (`:`). For the command `ls`:

```
Search order:
1. /usr/local/bin/ls  → Does it exist? No  → Keep looking
2. /usr/bin/ls        → Does it exist? YES → Run this one
3. (stops searching)
```

```bash
# Where did the shell find a command?
which ls
# /usr/bin/ls

# Show ALL locations (if the command exists in multiple PATH directories)
which -a python3
# /usr/local/bin/python3
# /usr/bin/python3

# More detailed — also shows builtins and aliases
type -a echo
# echo is a shell builtin
# echo is /usr/bin/echo
```

### Why PATH Order Matters

The shell uses the **first match**. If `/usr/local/bin` is before `/usr/bin` in your PATH, and both contain a program called `python3`, the one in `/usr/local/bin` will be used.

This is by design — `/usr/local/bin` is for custom-installed software that should take precedence over system defaults.

### Modifying PATH

```bash
# Add a directory to the BEGINNING of PATH (highest priority)
export PATH="/my/custom/bin:$PATH"

# Add a directory to the END of PATH (lowest priority)
export PATH="$PATH:/my/custom/bin"

# Add your own bin directory
export PATH="$HOME/bin:$PATH"

# View the result
echo $PATH | tr ':' '\n'    # Show each directory on its own line
```

### A Common Mistake with PATH

```bash
# WRONG — This REPLACES PATH instead of adding to it
export PATH="/my/custom/bin"
# Now ls, cat, grep, etc. all stop working!
# You just lost /usr/bin, /bin, etc.

# To recover, use the full path to set PATH back:
/usr/bin/export PATH="/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
# Or simply open a new terminal
```

**Always include `$PATH`** when modifying PATH:
```bash
# CORRECT — adds to existing PATH
export PATH="/my/custom/bin:$PATH"
```

### Making PATH Changes Permanent

Add the `export` line to your `~/.bashrc`:

```bash
# Add to ~/.bashrc
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc

# Reload for current session
source ~/.bashrc
```

---

## 6.5 Variable Naming Rules

```bash
# Valid variable names: letters, numbers, underscores
MY_VAR="valid"
_count=0
path2config="/etc/config"
DATABASE_URL="postgres://localhost/db"

# By convention, environment variables use UPPERCASE
export DATABASE_URL="postgres://localhost/db"
export MAX_RETRIES=3

# Shell-local variables often use lowercase
counter=0
filename="report.txt"

# INVALID names
# 2name="value"       # Cannot start with a number
# my-var="value"      # Cannot use hyphens
# my var="value"      # Cannot use spaces
# my.var="value"      # Cannot use dots
```

### Important: No Spaces Around `=`

```bash
# CORRECT — no spaces around =
name="John"

# WRONG — spaces cause errors
name = "John"    # Bash tries to run "name" as a command with arguments "=" and "John"
```

This is one of the most common beginner mistakes in Bash. There must be **NO SPACES** around the `=` in variable assignment.

---

## 6.6 Persistent Environment Configuration

### Where to Put Environment Settings

| File | When It's Read | Use For |
|------|---------------|---------|
| `/etc/environment` | Every login | System-wide simple settings |
| `/etc/profile` | Login shells (system-wide) | System-wide scripts/PATH |
| `/etc/profile.d/*.sh` | Sourced by /etc/profile | Modular system config |
| `~/.bash_profile` | Login shells (per user) | User login setup |
| `~/.bashrc` | Non-login interactive shells | Aliases, functions, prompt |
| `~/.profile` | Login shells (generic/non-bash) | Portable user config |

### Practical Setup

**~/.bashrc** — Your primary configuration file:

```bash
# ~/.bashrc — sourced for interactive non-login shells

# Custom PATH
export PATH="$HOME/bin:$HOME/.local/bin:$PATH"

# Default editor
export EDITOR="vim"
export VISUAL="vim"

# Custom pager settings
export PAGER="less"
export LESS="-R"    # Allow color in less

# Application-specific settings
export HISTSIZE=10000
export HISTFILESIZE=20000
export HISTCONTROL=ignoreboth  # Ignore duplicates and spaces

# Custom aliases (covered later)
alias ll='ls -alF'
alias la='ls -A'
alias grep='grep --color=auto'
```

**~/.bash_profile** — Sources .bashrc so everything works consistently:

```bash
# ~/.bash_profile — sourced for login shells

# Source .bashrc if it exists
if [ -f "$HOME/.bashrc" ]; then
    source "$HOME/.bashrc"
fi

# Login-specific settings (things that should only run once)
# e.g., starting ssh-agent
```

### Applying Changes

After editing configuration files:

```bash
# Reload .bashrc in the current session
source ~/.bashrc
# or
. ~/.bashrc

# For .bash_profile, it's often easier to just log out and back in
```

---

## 6.7 Variable Scope and Inheritance

Understanding scope prevents subtle bugs.

### Export Creates One-Way Inheritance

Environment variables flow DOWN to child processes, never UP to the parent.

```bash
# In the parent shell
export COLOR="blue"

# Start a child shell
bash

# In the child shell
echo $COLOR        # blue — inherited from parent
COLOR="red"        # Change it in the child
echo $COLOR        # red — changed in child
exit

# Back in the parent shell
echo $COLOR        # blue — parent is unchanged!
```

```
Parent Shell (COLOR=blue)
    │
    │ export → child gets a COPY
    │
    ├── Child Process (COLOR=blue)
    │       │
    │       │ change COLOR=red
    │       │
    │       └── COLOR=red (only in child)
    │
    └── Parent still has COLOR=blue
```

### Subshell Variables

Parentheses create a subshell — changes inside don't affect the parent:

```bash
x=10
(
    x=20
    echo "Inside subshell: $x"    # 20
)
echo "Outside subshell: $x"       # 10 — unchanged!
```

### Setting Variables for a Single Command

```bash
# Set a variable only for the duration of one command
LANG=C sort unsorted.txt
# LANG is "C" only for the sort command
echo $LANG    # Back to original value (e.g., en_US.UTF-8)

# Multiple variables for one command
CC=gcc CXX=g++ make
```

---

## 6.8 Special Shell Variables

These are set automatically by Bash:

| Variable | Meaning |
|----------|---------|
| `$0` | Name of the current script/shell |
| `$1, $2, ...` | Positional parameters (arguments) |
| `$#` | Number of arguments |
| `$?` | Exit status of the last command |
| `$$` | PID of the current shell |
| `$!` | PID of the last background process |
| `$@` | All arguments (individually quoted) |
| `$*` | All arguments (as a single string) |
| `$_` | Last argument of previous command |
| `$-` | Current shell option flags |
| `$BASHPID` | PID of the current Bash process (differs from `$$` in subshells) |

We'll use these extensively in Part 3 (Scripting).

```bash
# Quick demonstration
echo "Shell name: $0"
echo "Process ID: $$"
echo "Last exit status: $?"
echo "Current options: $-"
```

---

## Common Mistakes

1. **Spaces around `=` in assignments** — `VAR = "value"` is WRONG. Use `VAR="value"`.

2. **Overwriting PATH** — `export PATH="/my/dir"` loses all standard directories. Always use `export PATH="/my/dir:$PATH"`.

3. **Forgetting to export** — If a child process can't see your variable, you probably forgot `export`.

4. **Editing the wrong config file** — Variables in `.bashrc` won't apply in login shells unless `.bash_profile` sources `.bashrc`.

5. **Assuming child changes propagate to parent** — They don't. Environment inheritance is one-way (parent → child only).

6. **Using undefined variables** — Bash treats undefined variables as empty strings, which can cause silent bugs. Use `set -u` to catch these (Chapter 47).

---

## Exercises

### Exercise 6.1: Environment Exploration
1. Display all your environment variables sorted alphabetically
2. Find the values of `HOME`, `USER`, `SHELL`, `LANG`, and `TERM`
3. How many environment variables are set? (hint: `env | wc -l`)

### Exercise 6.2: PATH Investigation
1. Display your PATH, one directory per line
2. How many directories are in your PATH?
3. Where is `bash` located? Where is `python3` (if installed)?
4. Add `$HOME/scripts` to the beginning of your PATH
5. Create a script in `$HOME/scripts` and verify that you can run it by name

### Exercise 6.3: Variable Scope
Create this script and predict the output before running it:
```bash
#!/bin/bash
export PARENT_VAR="I'm from the parent"
LOCAL_VAR="I'm local"

echo "In script: PARENT_VAR=$PARENT_VAR"
echo "In script: LOCAL_VAR=$LOCAL_VAR"

bash -c 'echo "In child: PARENT_VAR=$PARENT_VAR"'
bash -c 'echo "In child: LOCAL_VAR=$LOCAL_VAR"'
```

### Exercise 6.4: Custom Environment
Configure your environment with:
1. `EDITOR` set to your preferred editor
2. A custom `PS1` prompt
3. An alias `ll` for `ls -la`
4. `HISTSIZE` set to 5000
Make these persist across sessions.

---

## Summary

- **Environment variables** are key-value pairs available to processes
- **Shell variables** (`VAR="val"`) are local; **environment variables** (`export VAR="val"`) are inherited by child processes
- **PATH** tells the shell where to find executables — order matters
- Always include `$PATH` when modifying it: `export PATH="/new/dir:$PATH"`
- No spaces around `=` in variable assignments
- Use `~/.bashrc` for persistent settings; have `~/.bash_profile` source it
- Variable inheritance is **one-way**: parent → child only
- Use `env`, `echo $VAR`, and `export -p` to inspect variables

---

**Next Chapter:** [Chapter 7: Getting Help — man, info, and Beyond →](Chapter07-Getting-Help.md)
