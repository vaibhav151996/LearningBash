# Chapter 14: Your First Bash Script

## Learning Objectives

By the end of this chapter, you will be able to:

- Create and run a Bash script
- Understand the shebang line and why it matters
- Know the difference between `./script.sh`, `bash script.sh`, and `source script.sh`
- Apply proper file permissions to scripts
- Write well-structured, readable scripts

---

## 14.1 What Is a Script?

A **script** is a text file containing a sequence of commands that the shell executes automatically. Everything you've typed interactively can be put into a script and run repeatedly.

Why scripts?
- **Automation** — Run complex tasks with one command
- **Reproducibility** — Same result every time
- **Documentation** — The script IS the documentation of what was done
- **Sharing** — Give the script to others, they get the same behavior

---

## 14.2 Creating Your First Script

### Step 1: Create the File

```bash
# Open a new file in your editor
nano hello.sh        # or vim, or any text editor
```

### Step 2: Write the Script

```bash
#!/bin/bash
# My first Bash script
# Author: Your Name
# Date: 2026-03-26

echo "Hello, World!"
echo "Today is $(date)"
echo "You are logged in as: $USER"
echo "Your home directory is: $HOME"
```

### Step 3: Make It Executable

```bash
chmod +x hello.sh
```

### Step 4: Run It

```bash
./hello.sh
```

Output:
```
Hello, World!
Today is Wed Mar 26 10:30:00 UTC 2026
You are logged in as: john
Your home directory is: /home/john
```

---

## 14.3 The Shebang Line

The first line of a script is the **shebang** (also called hashbang):

```bash
#!/bin/bash
```

### How It Works

When you execute `./hello.sh`, the kernel reads the first two characters (`#!`). If they are `#!`, the kernel uses the rest of the line as the **interpreter** for the script:

```
#!/bin/bash
│└──┘└─────┘
│  │    └── Path to the interpreter program
│  └─────── The shebang characters
│
└────────── Must be the VERY FIRST characters of the file
```

The kernel effectively runs:
```bash
/bin/bash hello.sh
```

### Common Shebangs

```bash
#!/bin/bash              # Bash — most common for scripts
#!/bin/sh                # POSIX shell — most portable
#!/usr/bin/env bash      # Find bash in PATH — most flexible
#!/usr/bin/env python3   # Python 3 script
#!/usr/bin/env node      # Node.js script
#!/usr/bin/awk -f        # AWK script
```

### Why `#!/usr/bin/env bash` Is Often Preferred

`/bin/bash` hardcodes the path. On some systems (certain macOS versions, BSD), bash might be at `/usr/local/bin/bash`. Using `env` searches `PATH`:

```bash
#!/usr/bin/env bash
# This finds bash wherever it's installed
```

**Recommendation:** Use `#!/usr/bin/env bash` for portability, or `#!/bin/bash` when you know the exact location (most Linux systems).

### What Happens Without a Shebang?

If there's no shebang, the behavior depends on how you run the script:

```bash
./script.sh          # The parent shell (usually bash) runs it
bash script.sh       # Explicitly runs it with bash
sh script.sh         # Runs it with sh (may NOT be bash!)
```

**Always include a shebang.** It makes your script's requirements explicit.

---

## 14.4 Three Ways to Run a Script

### Method 1: `./script.sh` — Execute

```bash
chmod +x script.sh
./script.sh
```

- Creates a **new process** (child shell)
- Uses the interpreter from the shebang line
- Variables set in the script do NOT affect the parent shell
- Requires execute permission

### Method 2: `bash script.sh` — Explicit Interpreter

```bash
bash script.sh
```

- Creates a **new process** (child shell)
- Ignores the shebang (you're specifying the interpreter)
- Does NOT require execute permission
- Useful for testing or when you can't set permissions

### Method 3: `source script.sh` or `. script.sh` — Source

```bash
source script.sh
# or equivalently:
. script.sh
```

- Runs in the **current shell** (no new process)
- Changes to variables, directory, etc. AFFECT the current shell
- Does NOT require execute permission
- Used for loading configuration and functions

### When to Use Each

| Method | New Process? | Affects Current Shell? | Use Case |
|--------|-------------|----------------------|----------|
| `./script.sh` | Yes | No | Normal execution |
| `bash script.sh` | Yes | No | When no execute permission |
| `source script.sh` | No | Yes | Loading config, setting variables |

```bash
# Demonstration of source vs execute
# File: setname.sh
#!/bin/bash
NAME="John"

# Using execute:
./setname.sh
echo $NAME          # (empty) — NAME was set in a child process

# Using source:
source setname.sh
echo $NAME          # John — NAME was set in this shell
```

This is why `~/.bashrc` is **sourced** (not executed) — its settings need to affect your current shell.

---

## 14.5 Script Structure Best Practices

A well-structured script follows this template:

```bash
#!/usr/bin/env bash
#
# Script Name: backup_home.sh
# Description: Creates a compressed backup of the user's home directory
# Author: John Doe <john@example.com>
# Date: 2026-03-26
# Version: 1.0
# Usage: ./backup_home.sh [destination_directory]
#
# Dependencies: tar, gzip
# License: MIT

# ----- Configuration -----
BACKUP_DIR="${1:-/tmp/backups}"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="home_backup_${DATE}.tar.gz"

# ----- Functions -----
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

die() {
    echo "ERROR: $*" >&2
    exit 1
}

# ----- Validation -----
if [ ! -d "$BACKUP_DIR" ]; then
    mkdir -p "$BACKUP_DIR" || die "Cannot create backup directory: $BACKUP_DIR"
fi

# ----- Main Logic -----
log "Starting backup of $HOME"
log "Destination: ${BACKUP_DIR}/${BACKUP_FILE}"

tar czf "${BACKUP_DIR}/${BACKUP_FILE}" -C / "home/${USER}" 2>/dev/null \
    || die "Backup failed"

log "Backup complete: $(du -h "${BACKUP_DIR}/${BACKUP_FILE}" | cut -f1)"

exit 0
```

### Key Principles

1. **Header comment** — What, who, when, how
2. **Configuration section** — Variables at the top, easy to change
3. **Functions** — Defined before use
4. **Validation** — Check prerequisites before doing work
5. **Main logic** — The actual work
6. **Explicit exit** — End with a clear exit code

---

## 14.6 Where to Put Your Scripts

```bash
# Personal scripts: ~/bin/ (add to PATH)
mkdir -p ~/bin
mv myscript.sh ~/bin/
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Now you can run it from anywhere:
myscript.sh

# System-wide custom scripts: /usr/local/bin/
sudo cp myscript.sh /usr/local/bin/

# Project-specific scripts: in the project directory
# project/scripts/deploy.sh
```

---

## Common Mistakes

1. **Forgetting `chmod +x`** — The most common error. Your script needs execute permission.

2. **Forgetting the shebang** — Without it, the script might run with a different shell than expected.

3. **Using Windows line endings** — If you edit on Windows, files may have `\r\n` line endings, causing `#!/bin/bash\r` (not found). Fix with:
   ```bash
   dos2unix script.sh
   # or
   sed -i 's/\r$//' script.sh
   ```

4. **Running `./script.sh` when you meant `source script.sh`** — Variables set in the script won't be visible in your shell.

5. **Not using `./`** — `script.sh` alone won't work unless `.` is in your PATH (it shouldn't be, for security).

---

## Exercises

### Exercise 14.1: Hello World
Create a script that prints:
- A welcome message
- The current date and time
- Your username and hostname
- Your current directory

### Exercise 14.2: System Info Script
Create `sysinfo.sh` that displays:
- OS name and version (`cat /etc/os-release`)
- Kernel version (`uname -r`)
- CPU info (`nproc` or `grep "model name" /proc/cpuinfo`)
- Total memory (`free -h`)
- Disk usage (`df -h /`)

### Exercise 14.3: Source vs Execute
Create a script that sets a variable `GREETING="Hello from script"`. Run it with `./`, then with `source`. Explain the difference in behavior.

---

## Summary

- A **script** is a text file with shell commands executed in sequence
- The **shebang** (`#!/bin/bash`) tells the system which interpreter to use
- Scripts need **execute permission** (`chmod +x script.sh`)
- `./script.sh` runs in a child process; `source script.sh` runs in the current shell
- Put personal scripts in `~/bin/` and add it to your `PATH`
- Follow a consistent structure: header, config, functions, validation, main logic

---

**Next Chapter:** [Chapter 15: Variables and Quoting Rules →](Chapter15-Variables-Quoting.md)
