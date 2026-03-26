# Chapter 3: Terminal Basics and Navigation

## Learning Objectives

By the end of this chapter, you will be able to:

- Navigate the Linux filesystem using the command line
- Understand absolute vs. relative paths
- Create, copy, move, rename, and delete files and directories
- Use essential commands fluently: `pwd`, `cd`, `ls`, `mkdir`, `cp`, `mv`, `rm`, `touch`, `cat`, `less`, `head`, `tail`
- Understand and use symbolic links and hard links
- Work with file metadata and timestamps

---

## 3.1 Your First Commands

When you open a terminal, you're placed in your **home directory**. Let's orient ourselves.

### pwd — Print Working Directory

```bash
pwd
```

Output:
```
/home/john
```

`pwd` tells you *where you are* in the filesystem. Think of the filesystem as a building — `pwd` tells you which room you're standing in.

### ls — List Directory Contents

```bash
ls
```

Output:
```
Desktop  Documents  Downloads  Music  Pictures  Videos
```

`ls` shows you *what's in the current directory*. Like opening your eyes and looking around the room.

### cd — Change Directory

```bash
cd Documents
pwd
```

Output:
```
/home/john/Documents
```

`cd` moves you to a different directory. You've walked to a different room.

These three commands — `pwd`, `ls`, `cd` — are your fundamental navigation tools. You'll use them hundreds of times a day.

---

## 3.2 Understanding Paths

A **path** is an address in the filesystem. There are two types.

### Absolute Paths

An absolute path starts from the **root** of the filesystem (`/`) and specifies the complete location:

```
/home/john/Documents/report.txt
│    │     │          │
│    │     │          └── File name
│    │     └───────────── Directory
│    └─────────────────── User's home directory
└──────────────────────── Root of the entire filesystem
```

Absolute paths always start with `/` and work from anywhere in the system. They are unambiguous.

```bash
# These work no matter where you currently are
cat /etc/hostname
ls /var/log
cd /home/john/Documents
```

### Relative Paths

A relative path starts from your **current directory**:

```bash
# If you're in /home/john:
cd Documents          # Goes to /home/john/Documents
cat Documents/report.txt  # Reads /home/john/Documents/report.txt
```

Relative paths do NOT start with `/`. They depend on where you currently are.

### Special Path Symbols

| Symbol | Meaning | Example |
|--------|---------|---------|
| `/` | Root directory (filesystem root) | `cd /` |
| `~` | Home directory (`/home/username`) | `cd ~` |
| `.` | Current directory | `ls .` (same as `ls`) |
| `..` | Parent directory (one level up) | `cd ..` |
| `-` | Previous directory (with `cd`) | `cd -` |

### Navigation Examples

```bash
# Start from home
cd ~
pwd                    # /home/john

# Go into a subdirectory
cd Documents
pwd                    # /home/john/Documents

# Go up one level
cd ..
pwd                    # /home/john

# Go up two levels
cd ../..
pwd                    # /home

# Go back to the previous directory
cd -
pwd                    # /home/john

# Go directly to an absolute path
cd /var/log
pwd                    # /var/log

# Go home (three equivalent ways)
cd ~
cd $HOME
cd                     # cd with no arguments goes home
```

### Path Resolution

When you type a relative path, Bash resolves it by combining your current directory with the relative path:

```
Current directory: /home/john
Relative path:    Documents/report.txt
Resolved path:    /home/john/Documents/report.txt

Current directory: /var/log
Relative path:    ../lib/apt
Resolved path:    /var/lib/apt
```

---

## 3.3 Listing Files in Detail

`ls` has many options that reveal critical information.

### Common ls Options

```bash
# Basic listing
ls

# Long format — shows permissions, owner, size, date
ls -l

# Show hidden files (files starting with .)
ls -a

# Combine: long format + hidden files
ls -la

# Human-readable file sizes (KB, MB, GB instead of bytes)
ls -lh

# Sort by modification time (newest first)
ls -lt

# Sort by size (largest first)
ls -lS

# Reverse the sort order
ls -lr

# Recursive — show all subdirectories too
ls -R

# Show one file per line
ls -1

# Classify — append / to directories, * to executables
ls -F
```

### Reading Long Format Output

```bash
ls -la
```

```
total 32
drwxr-xr-x  4 john users 4096 Mar 26 10:30 .
drwxr-xr-x 20 john users 4096 Mar 25 14:22 ..
-rw-r--r--  1 john users  220 Mar 20 09:00 .bashrc
drwxr-xr-x  2 john users 4096 Mar 26 10:30 Documents
-rw-r--r--  1 john users 1234 Mar 26 09:15 notes.txt
lrwxrwxrwx  1 john users   11 Mar 25 08:00 link -> /tmp/target
```

Let's decode each column:

```
-rw-r--r--  1  john  users  1234  Mar 26 09:15  notes.txt
│           │  │     │      │     │              │
│           │  │     │      │     │              └── Filename
│           │  │     │      │     └───────────────── Last modified date
│           │  │     │      └─────────────────────── Size in bytes
│           │  │     └────────────────────────────── Group owner
│           │  └──────────────────────────────────── File owner
│           └─────────────────────────────────────── Number of hard links
└─────────────────────────────────────────────────── File type & permissions
```

The first character indicates the file type:

| Character | Meaning |
|-----------|---------|
| `-` | Regular file |
| `d` | Directory |
| `l` | Symbolic link |
| `c` | Character device |
| `b` | Block device |
| `s` | Socket |
| `p` | Named pipe (FIFO) |

We'll explore permissions in detail in Chapter 5.

---

## 3.4 Creating Files and Directories

### touch — Create Empty Files or Update Timestamps

```bash
# Create a new empty file
touch newfile.txt

# Create multiple files at once
touch file1.txt file2.txt file3.txt

# If the file already exists, touch updates its modification time
touch existingfile.txt
```

### mkdir — Create Directories

```bash
# Create a single directory
mkdir projects

# Create nested directories (parent directories created automatically)
mkdir -p projects/webapp/src/components

# Without -p, this would fail if projects/ doesn't exist
# mkdir projects/webapp/src/components  # Error!

# Create multiple directories
mkdir dir1 dir2 dir3
```

The `-p` flag is essential. It means "create **p**arent directories as needed" and also silently succeeds if the directory already exists.

### Creating a Project Structure

```bash
# Create a typical project layout in one command
mkdir -p myproject/{src,tests,docs,config}

# This creates:
# myproject/
# ├── config/
# ├── docs/
# ├── src/
# └── tests/

# Verify with tree (if installed) or ls -R
ls -R myproject/
```

The `{src,tests,docs,config}` syntax is **brace expansion** — Bash automatically expands it into multiple arguments. We'll cover this in detail in Chapter 8.

---

## 3.5 Viewing File Contents

### cat — Concatenate and Display

```bash
# Display a file's contents
cat notes.txt

# Display multiple files concatenated together
cat header.txt body.txt footer.txt

# Display with line numbers
cat -n script.sh

# Show non-printing characters (tabs as ^I, line endings as $)
cat -A config.txt
```

**Warning:** Don't `cat` binary files — they'll fill your terminal with garbage characters. Use `file` to check a file's type first:

```bash
file image.png      # image.png: PNG image data, 800 x 600, 8-bit/color RGB
file notes.txt      # notes.txt: ASCII text
```

### less — Page Through Files

For files too long to fit on one screen, use `less`:

```bash
less /var/log/syslog
```

`less` controls:

| Key | Action |
|-----|--------|
| `Space` / `Page Down` | Next page |
| `b` / `Page Up` | Previous page |
| `g` | Go to beginning |
| `G` | Go to end |
| `/pattern` | Search forward for "pattern" |
| `?pattern` | Search backward for "pattern" |
| `n` | Next search result |
| `N` | Previous search result |
| `q` | Quit |
| `h` | Help |

**The phrase to remember:** "Less is more" — `less` replaced an older pager called `more`, and `less` can do everything `more` does, plus scroll backward.

### head and tail — View the Beginning or End

```bash
# First 10 lines of a file (default)
head /var/log/syslog

# First 5 lines
head -n 5 /var/log/syslog
head -5 /var/log/syslog        # Shorthand

# Last 10 lines
tail /var/log/syslog

# Last 20 lines
tail -n 20 /var/log/syslog

# Follow a file in real-time (watch for new lines)
tail -f /var/log/syslog
# Press Ctrl+C to stop following
```

`tail -f` is one of the most useful commands for monitoring log files. It keeps watching the file and displays new lines as they're added.

---

## 3.6 Copying, Moving, and Renaming

### cp — Copy Files and Directories

```bash
# Copy a file
cp source.txt destination.txt

# Copy a file into a directory
cp report.txt Documents/

# Copy multiple files into a directory
cp file1.txt file2.txt file3.txt Documents/

# Copy a directory and all its contents (recursive)
cp -r projects/ projects_backup/

# Copy preserving permissions, timestamps, and ownership
cp -a source_dir/ destination_dir/

# Interactive — ask before overwriting
cp -i source.txt destination.txt

# Verbose — show what's being copied
cp -v source.txt destination.txt
```

### mv — Move or Rename Files

```bash
# Rename a file
mv oldname.txt newname.txt

# Move a file to a different directory
mv report.txt Documents/

# Move and rename simultaneously
mv report.txt Documents/final_report.txt

# Move a directory
mv projects/ /home/john/work/

# Interactive — ask before overwriting
mv -i source.txt destination.txt

# Move multiple files into a directory
mv file1.txt file2.txt file3.txt Documents/
```

**Important:** In Linux, renaming and moving are the *same operation*. There is no separate "rename" command — `mv` does both.

### Renaming Multiple Files

For bulk renaming, you can use the `rename` command (if available) or a loop:

```bash
# Rename all .txt files to .md (using a loop — we'll learn loops in Ch. 22)
for f in *.txt; do
    mv "$f" "${f%.txt}.md"
done
```

---

## 3.7 Deleting Files and Directories

### rm — Remove Files

```bash
# Remove a file
rm unwanted.txt

# Remove without confirmation (DANGEROUS — no recycle bin!)
rm -f secret.txt

# Remove a directory and all its contents
rm -r old_project/

# Remove forcefully and recursively (VERY DANGEROUS)
rm -rf old_project/

# Interactive — ask before each deletion
rm -i *.tmp

# Verbose — show what's being deleted
rm -v *.log
```

### The rm -rf Warning

```
╔══════════════════════════════════════════════════════════════╗
║  ⚠  CRITICAL WARNING                                       ║
║                                                              ║
║  rm -rf PERMANENTLY DELETES files. There is NO recycle bin, ║
║  NO undo, NO confirmation (with -f), NO recovery.           ║
║                                                              ║
║  NEVER run: rm -rf /      (deletes everything)              ║
║  NEVER run: rm -rf ~      (deletes your home directory)     ║
║  NEVER run: rm -rf *      (in the wrong directory)          ║
║                                                              ║
║  ALWAYS double-check your path before pressing Enter.       ║
║  ALWAYS use pwd first to confirm where you are.             ║
╚══════════════════════════════════════════════════════════════╝
```

### rmdir — Remove Empty Directories

```bash
# Remove an empty directory (fails if not empty — this is a safety feature)
rmdir empty_directory/

# Remove nested empty directories
rmdir -p a/b/c/    # Removes c/, then b/, then a/ — all must be empty
```

### Safe Deletion Practices

```bash
# 1. Always check where you are first
pwd

# 2. List what you're about to delete
ls old_project/

# 3. Use interactive mode for important deletions
rm -ri old_project/

# 4. Consider moving to a "trash" directory instead of deleting
mkdir -p ~/.trash
mv unwanted_file.txt ~/.trash/
```

---

## 3.8 Finding Files

### find — Search the Filesystem

`find` is extremely powerful. Here are the essential uses:

```bash
# Find files by name in the current directory and subdirectories
find . -name "report.txt"

# Find case-insensitive
find . -iname "readme.md"

# Find all .txt files
find . -name "*.txt"

# Find by type (f=file, d=directory, l=symlink)
find /home -type f -name "*.sh"
find /var -type d -name "log"

# Find files modified in the last 24 hours
find . -mtime -1

# Find files larger than 100MB
find / -size +100M

# Find files and execute a command on each
find . -name "*.tmp" -exec rm {} \;

# Find files and print with details
find . -name "*.log" -ls

# Find empty files
find . -empty

# Find files by permissions
find . -perm 644
```

### which and type — Find Commands

```bash
# Where is a command located?
which bash              # /usr/bin/bash
which python3           # /usr/bin/python3

# More detailed information about a command
type ls                 # ls is aliased to 'ls --color=auto'
type cd                 # cd is a shell builtin
type bash               # bash is /usr/bin/bash
type -a echo            # Shows ALL locations (builtin + external)
```

### locate — Fast File Search (if available)

```bash
# Search a pre-built database (very fast)
locate report.txt

# Update the database (run as root)
sudo updatedb
```

`locate` is much faster than `find` because it searches a pre-built index instead of scanning the filesystem. But the index may not be current.

---

## 3.9 Links — Hard Links and Symbolic Links

Links are a way to have one file accessible from multiple locations.

### Hard Links

A hard link creates a second name for the same file data on disk. Both names are equal — there's no "original" and "copy."

```bash
# Create a hard link
ln original.txt hardlink.txt

# Both files share the same data
echo "Hello" > original.txt
cat hardlink.txt          # Output: Hello

# Modifying through either name changes the same data
echo "Modified" >> hardlink.txt
cat original.txt          # Output: Hello\nModified

# Check — they share the same inode number
ls -li original.txt hardlink.txt
# 1234567 -rw-r--r-- 2 john users 14 Mar 26 10:30 hardlink.txt
# 1234567 -rw-r--r-- 2 john users 14 Mar 26 10:30 original.txt
#                    ^
#                    Link count is 2
```

Hard link limitations:
- Cannot link to directories (prevents filesystem loops)
- Cannot cross filesystem boundaries (must be on the same partition)

### Symbolic Links (Symlinks)

A symbolic link is a pointer to another file's path. Think of it as a shortcut.

```bash
# Create a symbolic link
ln -s /var/log/syslog ~/syslog_link

# The link points to the target
ls -l ~/syslog_link
# lrwxrwxrwx 1 john users 15 Mar 26 10:30 syslog_link -> /var/log/syslog

# Reading the link reads the target
cat ~/syslog_link    # Shows contents of /var/log/syslog

# Create a symlink to a directory
ln -s /etc ~/etc_link
ls ~/etc_link        # Lists contents of /etc
```

Symlink characteristics:
- Can link to directories
- Can cross filesystem boundaries
- If you delete the target, the link becomes "broken" (dangling)
- The link itself has a small size (just the path string)

### Comparison

```
Hard Link                          Symbolic Link
┌──────────┐                      ┌───────────┐
│hardlink.txt│──┐                 │ symlink   │──→ path: "/data/file.txt"
└──────────┘  │                   └───────────┘           │
              ├──→ [inode 1234]                           ▼
┌──────────┐  │    [file data]    ┌───────────┐    ┌──────────┐
│original.txt│──┘                 │/data/     │    │[inode]   │
└──────────┘                      │ file.txt  │───→│[file data]│
                                  └───────────┘    └──────────┘
```

| Feature | Hard Link | Symbolic Link |
|---------|-----------|---------------|
| Syntax | `ln file link` | `ln -s target link` |
| Cross filesystems | No | Yes |
| Link to directories | No | Yes |
| Target deleted | Data persists | Link breaks |
| Link count | Increases | No change |
| Inode | Same as target | Different |

---

## 3.10 Working with File Metadata

### stat — Detailed File Information

```bash
stat notes.txt
```

Output:
```
  File: notes.txt
  Size: 1234          Blocks: 8          IO Block: 4096   regular file
Device: 802h/2050d    Inode: 654321      Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1000/   john)   Gid: ( 1000/  users)
Access: 2026-03-26 10:30:15.000000000 -0400
Modify: 2026-03-26 09:15:42.000000000 -0400
Change: 2026-03-26 09:15:42.000000000 -0400
 Birth: 2026-03-20 08:00:00.000000000 -0400
```

Three timestamps:
- **Access (atime)** — Last time the file was read
- **Modify (mtime)** — Last time the file's contents changed
- **Change (ctime)** — Last time the file's metadata changed (permissions, ownership)

### du — Disk Usage

```bash
# Size of a file or directory
du notes.txt                    # Size in blocks
du -h notes.txt                 # Human-readable
du -sh Documents/               # Summary for a directory
du -sh */                       # Size of each subdirectory
du -sh * | sort -rh | head -10  # Top 10 largest items
```

### df — Disk Free Space

```bash
# Show filesystem disk space usage
df -h

# Output:
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        50G   15G   33G  32% /
# tmpfs           3.9G     0  3.9G   0% /dev/shm
```

---

## 3.11 Essential Keyboard Shortcuts

These shortcuts work in Bash and will dramatically speed up your work.

### Navigation Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+A` | Move cursor to beginning of line |
| `Ctrl+E` | Move cursor to end of line |
| `Alt+F` | Move cursor forward one word |
| `Alt+B` | Move cursor backward one word |
| `Ctrl+F` | Move cursor forward one character |
| `Ctrl+B` | Move cursor backward one character |

### Editing Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+K` | Delete from cursor to end of line |
| `Ctrl+U` | Delete from cursor to beginning of line |
| `Ctrl+W` | Delete the word before the cursor |
| `Alt+D` | Delete the word after the cursor |
| `Ctrl+Y` | Paste the last deleted text ("yank") |
| `Ctrl+T` | Swap the two characters before the cursor |

### Control Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+C` | Cancel/kill the current command |
| `Ctrl+D` | Exit the shell (or signal end-of-input) |
| `Ctrl+L` | Clear the screen (same as `clear`) |
| `Ctrl+Z` | Suspend the current process |
| `Ctrl+R` | Search command history (reverse search) |
| `Tab` | Auto-complete file names and commands |
| `Tab Tab` | Show all possible completions |

### Tab Completion — Your Best Friend

Tab completion saves enormous amounts of typing and prevents errors:

```bash
# Type the beginning of a file name, then press Tab
cat no[TAB]           # Completes to: cat notes.txt

# If multiple matches, Tab Tab shows options
ls Do[TAB][TAB]
# Documents/  Downloads/

# Works for commands too
sys[TAB][TAB]
# sysctl     syslog     systemctl  systemd   ...

# Works for paths
cd /etc/sys[TAB]
# cd /etc/sysctl.d/
```

**Rule of thumb:** If you're typing a path or filename without pressing Tab, you're doing it wrong. Tab completion is faster AND prevents typos.

---

## Common Mistakes

1. **Forgetting that `rm` is permanent** — There is no trash can in the terminal by default. Always double-check before deleting.

2. **Using spaces in filenames without quoting** — `rm my file.txt` deletes TWO files: `my` and `file.txt`. Use `rm "my file.txt"` or `rm my\ file.txt`.

3. **Confusing `-r` and `-R`** — For `cp` and `rm`, `-r` means recursive. For `ls`, it's `-R`. Some commands accept both. When in doubt, check `man command`.

4. **Not using Tab completion** — Beginners type complete paths manually and make mistakes. Use Tab after every few characters.

5. **Using `cd` with `..` incorrectly** — `cd..` (no space) is an error. It's `cd ..` (with a space). `cd` is a command; `..` is its argument.

6. **Confusion between `.` and `..`** — `.` is the current directory, `..` is the parent. `cp file.txt .` copies to the current directory. `cp file.txt ..` copies to the parent.

---

## Exercises

### Exercise 3.1: Navigation Practice
Starting from your home directory, perform these operations and verify each with `pwd`:
1. Navigate to `/var/log`
2. Go up one directory
3. Go to `/tmp`
4. Go back to the previous directory (`cd -`)
5. Go home without typing the full path

### Exercise 3.2: File Operations
1. Create a directory structure: `~/practice/level1/level2/level3`
2. Create files `a.txt`, `b.txt`, `c.txt` in `level1`
3. Copy `a.txt` to `level2`
4. Move `b.txt` to `level3`
5. Rename `c.txt` to `renamed.txt`
6. Delete `level3` and everything inside it

### Exercise 3.3: Exploration
1. How many files and directories are in `/etc`? (Use `ls -1 /etc | wc -l`)
2. What is the largest file in `/var/log`? (Use `ls -lS`)
3. Find all `.conf` files in `/etc` (Use `find`)
4. What type of file is `/bin/bash`? (Use `file`)

### Exercise 3.4: Links
1. Create a file called `original.txt` with some content
2. Create a hard link called `hardlink.txt`
3. Create a symbolic link called `symlink.txt`
4. Delete `original.txt`
5. Can you still read `hardlink.txt`? Why?
6. Can you still read `symlink.txt`? Why not?

### Exercise 3.5: Keyboard Shortcuts
Practice each of these until they're muscle memory:
1. Type a long command, then jump to the beginning (Ctrl+A) and end (Ctrl+E)
2. Delete a word at a time (Ctrl+W)
3. Clear the screen (Ctrl+L)
4. Use Ctrl+R to search your command history

---

## Summary

- `pwd` shows your current directory; `cd` changes it; `ls` lists contents
- **Absolute paths** start with `/`; **relative paths** are relative to current directory
- `.` is current directory; `..` is parent; `~` is home; `-` is previous
- `mkdir -p` creates parent directories; `touch` creates empty files
- `cat` displays files; `less` pages through them; `head`/`tail` show beginning/end
- `cp` copies; `mv` moves/renames; `rm` deletes (permanently!)
- `find` searches the filesystem; `which`/`type` find commands
- **Hard links** share data; **symbolic links** are pointers
- **Tab completion** and keyboard shortcuts are essential tools
- Always verify with `pwd` and `ls` before destructive operations

---

**Next Chapter:** [Chapter 4: The Linux Filesystem Hierarchy →](Chapter04-Filesystem-Hierarchy.md)
