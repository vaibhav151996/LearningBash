# Chapter 2: The Shell — Your Command-Line Interface

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain what a shell is and what role it plays in the operating system
- Describe how Bash differs from other shells
- Understand the relationship between the terminal emulator and the shell
- Explain how the shell processes a command from start to finish
- Identify your current shell and switch between shells

---

## 2.1 What Is a Shell?

A **shell** is a program that accepts commands from you (the user) and translates them into instructions that the operating system's kernel can execute.

Think of it as a translator:

```
┌──────────┐     ┌───────────┐     ┌────────────┐     ┌──────────┐
│   You    │────>│   Shell   │────>│   Kernel   │────>│ Hardware │
│ (Human)  │     │(Translator│     │ (Manager)  │     │(CPU, RAM │
│          │<────│  & Manager│<────│            │<────│  Disk)   │
└──────────┘     └───────────┘     └────────────┘     └──────────┘
```

When you type `ls -la /home`, here's what happens:

1. You type text at a prompt
2. The **shell** parses your input, finds the `ls` program, and asks the **kernel** to run it
3. The **kernel** runs `ls`, which reads the `/home` directory from the **disk**
4. The results flow back through the kernel to the shell
5. The shell displays the output on your screen

The word "shell" is a metaphor: it's the outer **shell** around the kernel — the layer you interact with.

### Shells Are Programs, Not Magic

This is a crucial insight: **the shell is just a regular program.** It's an executable file sitting on your disk, just like any other program. It reads input, processes it, and produces output. There's nothing magical about it.

You can run a shell inside a shell. You can write your own shell. You can replace the default shell with a different one.

```bash
# What program is your shell?
echo $SHELL

# Where is it on disk?
which bash

# It's just a file!
ls -la /bin/bash
```

---

## 2.2 A Brief History of Shells

Understanding the shell family tree helps you understand why Bash works the way it does.

```
1971  ┌─────────────────────┐
      │    Thompson Shell   │    First Unix shell (Ken Thompson)
      │     (/bin/sh)       │    Very primitive — no scripting
      └──────────┬──────────┘
                 │
1977  ┌──────────┴──────────┐
      │    Bourne Shell     │    First programmable shell (Steve Bourne)
      │     (/bin/sh)       │    Variables, loops, conditionals
      └──────────┬──────────┘
                 │
         ┌───────┼────────┐
         │       │        │
1978  ┌──┴──┐   │    ┌───┴───┐
      │ csh │   │    │  ksh  │    C Shell (Bill Joy) — C-like syntax
      │     │   │    │       │    Korn Shell (David Korn) — enhanced sh
      └──┬──┘   │    └───┬───┘
         │      │        │
1989  ┌──┴──┐   │        │
      │tcsh │   │        │
      └─────┘   │        │
                │        │
1989     ┌──────┴────────┴──┐
         │       Bash       │    "Bourne Again Shell" (Brian Fox)
         │  (/bin/bash)     │    Free replacement for sh
         └──────────────────┘    Combines best of sh, csh, ksh
                 │
         ┌───────┼────────┐
         │       │        │
1990  ┌──┴──┐ ┌─┴──┐ ┌──┴───┐
      │ zsh │ │fish│ │ dash │
      └─────┘ └────┘ └──────┘
```

### The Bourne Shell (sh)

Created by Steve Bourne at Bell Labs in 1977. It introduced:
- Variables
- Control flow (if/else, loops)
- Functions
- I/O redirection

It became the standard Unix shell. The path `/bin/sh` traditionally points to the Bourne shell.

### The C Shell (csh) and tcsh

Created by Bill Joy (co-founder of Sun Microsystems) in 1978. It used C-like syntax and added:
- Command history (press Up to recall previous commands)
- Aliases
- Job control

However, its scripting language had many quirks, and a famous paper titled *"Csh Programming Considered Harmful"* documented its problems.

### The Korn Shell (ksh)

Created by David Korn at Bell Labs in 1983. It aimed to combine the scripting strength of sh with the interactive features of csh. It added:
- Command-line editing
- Built-in arithmetic
- Arrays
- Improved string handling

### Bash — The Bourne Again Shell

In 1989, **Brian Fox** wrote Bash for the GNU Project. Its goals:
- Be a free replacement for the Bourne shell
- Incorporate the best features from csh and ksh
- Maintain backward compatibility with sh scripts

Bash became the default shell on most Linux distributions and macOS (until macOS switched to zsh in 2019). It is the shell we'll focus on throughout this entire book.

### Modern Alternatives

- **Zsh (Z Shell)** — Extensible, powerful tab completion, themes (Oh My Zsh). Default on macOS since Catalina.
- **Fish (Friendly Interactive Shell)** — User-friendly, auto-suggestions, web-based configuration. Not POSIX-compatible.
- **Dash** — Minimal, POSIX-compliant, very fast. Used as `/bin/sh` on Debian/Ubuntu for speed.

**Why learn Bash specifically?**
- It's the default shell on most Linux systems
- It's the standard for shell scripting in industry
- Nearly every Linux server you'll encounter uses it
- Other shells share most of its syntax and concepts
- Learning Bash first makes learning other shells easy

---

## 2.3 Terminal vs. Shell — Understanding the Difference

People often use "terminal" and "shell" interchangeably. They are different things.

### The Terminal Emulator

A **terminal emulator** is the **window program** that displays text and handles keyboard input. It's the "screen" where you see the shell working. Examples:

- **GNOME Terminal** — Default on Ubuntu/GNOME desktops
- **Konsole** — Default on KDE desktops
- **iTerm2** — Popular on macOS
- **Windows Terminal** — Microsoft's modern terminal
- **Alacritty** — GPU-accelerated terminal
- **tmux / screen** — Terminal multiplexers (terminals within terminals)

### The Shell

The **shell** is the **program running inside the terminal** that actually interprets your commands. The terminal is just the display; the shell is the brain.

```
┌─────────────────────────────────────────────────┐
│  GNOME Terminal (Terminal Emulator)              │
│  ┌───────────────────────────────────────────┐   │
│  │  bash (Shell Process)                     │   │
│  │                                           │   │
│  │  user@laptop:~$ ls -la                    │   │
│  │  total 32                                 │   │
│  │  drwxr-xr-x 4 user user 4096 Mar 26 ...  │   │
│  │  user@laptop:~$ █                         │   │
│  │                                           │   │
│  └───────────────────────────────────────────┘   │
│                                                   │
│  [Font settings] [Color settings] [Tab bar]       │
└─────────────────────────────────────────────────┘
```

An analogy:
- The **terminal** is like a TV screen
- The **shell** is like the TV show
- You can show different shows (shells) on the same TV (terminal)
- You can show the same show (shell) on different TVs (terminals)

### The Console

Historically, the **console** was the physical terminal — a screen and keyboard directly attached to the computer. On modern Linux, the **virtual console** refers to the text-mode screens you can access with `Ctrl+Alt+F1` through `Ctrl+Alt+F6` (without any GUI).

---

## 2.4 How the Shell Processes a Command

When you type a command and press Enter, the shell goes through a precise sequence of steps. Understanding this process is fundamental to mastering Bash.

### The Shell Processing Pipeline

```
Input: echo "Hello, $USER" > greeting.txt

Step 1: TOKENIZATION
  Split input into tokens: [echo] ["Hello, $USER"] [>] [greeting.txt]

Step 2: EXPANSION
  Expand variables: $USER → "john"
  Result: [echo] ["Hello, john"] [>] [greeting.txt]

Step 3: QUOTE REMOVAL
  Remove the quote characters (quotes did their job during expansion)
  Result: [echo] [Hello, john] [>] [greeting.txt]

Step 4: REDIRECTION SETUP
  Recognize > as output redirection
  Open greeting.txt for writing

Step 5: COMMAND LOOKUP
  Is "echo" a built-in command? YES → use built-in
  (If not built-in, search PATH for an executable named "echo")

Step 6: EXECUTION
  Run the echo built-in with argument "Hello, john"
  Output goes to greeting.txt (due to redirection)

Step 7: RETURN STATUS
  Store the exit status (0 for success) in $?
  Display the next prompt
```

Here's the complete order of operations:

1. **Read** — Read a line of input
2. **Tokenize** — Break input into words and operators
3. **Parse** — Build a command structure (pipes, redirections, lists)
4. **Expand** — Perform expansions in this order:
   - Brace expansion: `{a,b,c}`
   - Tilde expansion: `~` → `/home/user`
   - Parameter/variable expansion: `$var`, `${var}`
   - Command substitution: `$(command)` or `` `command` ``
   - Arithmetic expansion: `$((1 + 2))`
   - Word splitting: split results on IFS characters
   - Filename expansion (globbing): `*.txt`
5. **Quote removal** — Remove quotes that aren't part of the result
6. **Redirect** — Set up I/O redirections
7. **Execute** — Run the command
8. **Wait** — Wait for the command to finish, collect exit status

We'll cover each of these in great detail in later chapters. For now, just understand that the shell doesn't simply "run what you type" — it processes it through many transformation stages first.

---

## 2.5 Interactive vs. Non-Interactive Shells

Bash operates in two modes, and they behave differently.

### Interactive Shell

An **interactive shell** is what you see when you open a terminal and type commands:

```bash
user@laptop:~$ echo "I'm interactive!"
I'm interactive!
user@laptop:~$
```

Characteristics:
- Displays a **prompt** (like `$` or `user@host:~$`)
- Waits for your input
- Reads startup files (`.bashrc`, `.bash_profile`)
- Supports **job control** (Ctrl+Z, `bg`, `fg`)
- Keeps **command history**
- Has **tab completion**

### Non-Interactive Shell

A **non-interactive shell** runs commands from a script file without user interaction:

```bash
#!/bin/bash
echo "I'm non-interactive!"
echo "I run commands from a script file."
```

Characteristics:
- No prompt displayed
- Reads commands from a file or pipe
- Does NOT read `.bashrc` (reads `.bash_env` if set)
- No job control
- No command history
- Runs to completion or until an error

### Why This Matters

Some features work differently in interactive vs. non-interactive mode. When your script behaves differently from what you tested in the terminal, this distinction is often the reason.

```bash
# Check if the current shell is interactive
if [[ $- == *i* ]]; then
    echo "Interactive shell"
else
    echo "Non-interactive shell"
fi
```

---

## 2.6 Login vs. Non-Login Shells

This is another important distinction that affects which configuration files get read at startup.

### Login Shell

A **login shell** is the first shell started when you log into a system. It's started when:
- You log in at a text console
- You connect via SSH
- You explicitly start one with `bash --login`

A login shell reads these files in order:
1. `/etc/profile` (system-wide)
2. Then the first one found of:
   - `~/.bash_profile`
   - `~/.bash_login`
   - `~/.profile`

### Non-Login Shell

A **non-login shell** is started when you:
- Open a new terminal window in a GUI
- Run `bash` inside an existing shell
- Run a Bash script

A non-login shell reads:
1. `/etc/bash.bashrc` (system-wide, on some systems)
2. `~/.bashrc`

### The Startup File Flow

```
┌─────────────────────────────────────────────────────┐
│                  Login Shell?                       │
│                                                     │
│   YES                                  NO           │
│    │                                    │           │
│    ▼                                    ▼           │
│ /etc/profile                      /etc/bash.bashrc  │
│    │                                    │           │
│    ▼                                    ▼           │
│ ~/.bash_profile                     ~/.bashrc       │
│  (or ~/.bash_login)                                 │
│  (or ~/.profile)                                    │
│    │                                                │
│    │  Common practice:                              │
│    │  ~/.bash_profile sources ~/.bashrc             │
│    │  so settings work in both modes                │
│    ▼                                                │
│ ┌──────────────────────────────────┐                │
│ │  # In ~/.bash_profile:           │                │
│ │  if [ -f ~/.bashrc ]; then      │                │
│ │      source ~/.bashrc           │                │
│ │  fi                              │                │
│ └──────────────────────────────────┘                │
└─────────────────────────────────────────────────────┘
```

### Practical Advice

The simplest approach (and what most distributions set up by default):

1. Put all your customizations in `~/.bashrc`
2. Have `~/.bash_profile` source `~/.bashrc`
3. Now your settings work in both login and non-login shells

```bash
# ~/.bash_profile — keep it simple
# Source .bashrc if it exists
if [ -f "$HOME/.bashrc" ]; then
    source "$HOME/.bashrc"
fi
```

---

## 2.7 Identifying and Changing Your Shell

### What Shell Am I Using?

```bash
# Method 1: Check the SHELL environment variable
echo $SHELL
# Output: /bin/bash

# Method 2: Check the current process
echo $0
# Output: -bash  (the dash means it's a login shell)

# Method 3: Check /etc/passwd
grep $USER /etc/passwd
# Output: john:x:1000:1000:John:/home/john:/bin/bash
#                                           ^^^^^^^^^ default shell

# Method 4: Check the running process
ps -p $$
#   PID TTY          TIME CMD
#  1234 pts/0    00:00:00 bash
```

### Switching Shells Temporarily

```bash
# Start a new zsh session inside your current bash
zsh

# Start a new bash session (subshell)
bash

# Exit back to the previous shell
exit
```

### Changing Your Default Shell Permanently

```bash
# List available shells
cat /etc/shells
# /bin/sh
# /bin/bash
# /bin/zsh
# /usr/bin/fish

# Change your default shell to zsh (example)
chsh -s /bin/zsh

# You'll need to log out and back in for this to take effect
```

**For this book, make sure Bash is your shell.** If it isn't:

```bash
chsh -s /bin/bash
```

---

## 2.8 The Prompt

The prompt is the text that appears when the shell is waiting for your input. Understanding it tells you important information at a glance.

### Anatomy of the Default Prompt

```
user@hostname:~/projects$
│    │        │          │
│    │        │          └── $ means regular user (# means root)
│    │        └───────────── Current directory (~ means home)
│    └────────────────────── Hostname of the computer
└─────────────────────────── Your username
```

### The PS1 Variable

The prompt is controlled by the `PS1` variable. You can customize it:

```bash
# See your current prompt string
echo "$PS1"

# A simple prompt
PS1="$ "

# Username and directory
PS1="\u@\h:\w\$ "

# Colorized prompt with git branch (common in modern setups)
PS1="\[\033[32m\]\u@\h\[\033[00m\]:\[\033[34m\]\w\[\033[00m\]\$ "
```

### Common Prompt Escape Sequences

| Escape | Meaning |
|--------|---------|
| `\u` | Username |
| `\h` | Hostname (short) |
| `\H` | Hostname (full) |
| `\w` | Current working directory (full path, `~` for home) |
| `\W` | Current directory name only (basename) |
| `\d` | Date (e.g., "Tue Mar 26") |
| `\t` | Time (24-hour, HH:MM:SS) |
| `\n` | Newline |
| `\$` | `$` for regular user, `#` for root |
| `\!` | History number of this command |
| `\#` | Command number in this session |

### Other Prompt Variables

| Variable | When It's Used |
|----------|---------------|
| `PS1` | Primary prompt (the main one you see) |
| `PS2` | Continuation prompt (when a command spans multiple lines). Default: `> ` |
| `PS3` | Prompt for `select` statement |
| `PS4` | Debug prompt (used with `set -x`). Default: `+ ` |

```bash
# PS2 example — type an incomplete command
$ echo "hello
>       # <-- This ">" is PS2, waiting for the closing quote
> world"
hello
world
```

---

## Common Mistakes

1. **Confusing the terminal with the shell** — The terminal is the window; the shell is the program inside it. If you change your terminal emulator (GNOME Terminal → Alacritty), the shell (Bash) stays the same.

2. **Not understanding which startup files are read** — If your environment variable settings aren't working, you're probably editing the wrong file. Use `.bashrc` for everything and source it from `.bash_profile`.

3. **Running `sh` scripts with Bash syntax** — `sh` is not Bash. If your script starts with `#!/bin/sh`, don't use Bash-specific features. We'll cover this more in later chapters.

4. **Ignoring the expansion order** — When something doesn't expand the way you expect, remember the precise order: braces → tildes → parameters → commands → arithmetic → word splitting → globbing.

---

## Exercises

### Exercise 2.1: Identify Your Environment
Run these commands and record the output:
```bash
echo $SHELL
echo $BASH_VERSION
echo $0
ps -p $$
```
Explain what each command tells you.

### Exercise 2.2: Startup Files
Find and read your `~/.bashrc` file:
```bash
cat ~/.bashrc
```
List three settings or configurations you find in it. What does each one do?

### Exercise 2.3: Customize Your Prompt
Temporarily change your prompt to show:
- The current time
- Your username
- The current directory
- A newline before the `$` sign

Write the `PS1` value you used.

### Exercise 2.4: Explore Shell Types
1. Open a terminal and determine if it's a login or non-login shell.
2. Run `bash --login` and check again.
3. Run a script and check inside the script.

Hint: Use `shopt login_shell` to check.

### Exercise 2.5: PS2 in Action
Type an incomplete command (like `echo "hello` without the closing quote). Observe the PS2 prompt. Press Ctrl+C to cancel. Now try an incomplete for loop:
```bash
for i in 1 2 3
```
What prompt do you see?

---

## Summary

- A **shell** is a command interpreter — a program that reads your commands and asks the kernel to execute them
- **Bash** (Bourne Again Shell) is the most widely used shell on Linux
- The **terminal emulator** is the window; the **shell** is the program inside it
- Shells can be **interactive** (you type commands) or **non-interactive** (running a script)
- Shells can be **login** (first shell at login) or **non-login** (subsequent shells)
- The shell processes input through a precise pipeline: tokenize → expand → redirect → execute
- The **prompt** is controlled by `PS1` and can be customized
- **Startup files** (`.bashrc`, `.bash_profile`) control shell configuration

---

**Next Chapter:** [Chapter 3: Terminal Basics and Navigation →](Chapter03-Terminal-Basics.md)
