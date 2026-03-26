# Chapter 9: Built-in Commands vs External Programs

## Learning Objectives

By the end of this chapter, you will be able to:

- Distinguish between shell built-ins, external programs, aliases, and functions
- Understand why the distinction matters for performance and behavior
- Use `type`, `command`, `builtin`, and `enable` to control command resolution
- Know the command lookup order Bash follows

---

## 9.1 The Command Lookup Order

When you type a command, Bash searches for it in this precise order:

```
1. ALIASES          → alias ll='ls -la'
2. FUNCTIONS        → my_function() { ... }
3. BUILT-INS        → cd, echo, export, test
4. HASH TABLE       → Cached paths of previously found programs
5. PATH SEARCH      → Search each directory in $PATH
```

Once a match is found, Bash stops looking. This means an alias can override a function, which can override a built-in.

```bash
# Demonstrate the lookup order
type -a echo
# echo is a shell builtin
# echo is /usr/bin/echo

# Bash uses the builtin first. Two versions exist!
```

---

## 9.2 Built-in Commands

Built-in commands are part of the Bash executable itself. They run inside the shell process — no new process is created.

### Why Built-ins Exist

Some commands MUST be built-in because they need to modify the shell's own state. For example:

- `cd` — Changes the shell's working directory. An external program couldn't do this because it would receive its own process with its own working directory.
- `export` — Modifies the shell's environment. An external `export` couldn't change the calling shell's environment.
- `exit` — Terminates the shell process itself.

```bash
# If cd were external, this would happen:
# 1. Shell forks a new process
# 2. New process changes ITS directory to /tmp
# 3. New process exits
# 4. Shell is still in the original directory!
# That's why cd MUST be built-in.
```

### Common Built-in Commands

| Category | Commands |
|----------|----------|
| Navigation | `cd`, `pwd`, `pushd`, `popd`, `dirs` |
| Variables | `export`, `unset`, `declare`, `local`, `readonly` |
| I/O | `echo`, `printf`, `read` |
| Control | `if`, `for`, `while`, `case`, `select`, `until` |
| Execution | `eval`, `exec`, `source` (`.`), `command`, `builtin` |
| Job Control | `jobs`, `fg`, `bg`, `wait`, `kill`, `disown` |
| Shell Config | `set`, `shopt`, `enable`, `hash`, `ulimit` |
| Misc | `test` (`[`), `true`, `false`, `type`, `alias`, `unalias` |
| Flow | `break`, `continue`, `return`, `exit`, `trap` |
| History | `history`, `fc` |

```bash
# Verify any command is a built-in
type cd          # cd is a shell builtin
type export      # export is a shell builtin
type echo        # echo is a shell builtin
```

### Performance Advantage

Built-ins are MUCH faster than external programs because:
- No new process is created (no `fork()` + `exec()` system calls)
- No disk access to load a program
- No process startup and teardown overhead

```bash
# echo is both a built-in and external program
type -a echo
# echo is a shell builtin      ← Used by default (faster)
# echo is /usr/bin/echo         ← External program (slower)

# In a loop, the difference is dramatic:
# Built-in echo (fast):
time for i in $(seq 1 10000); do echo "$i" > /dev/null; done

# External echo (slow):
time for i in $(seq 1 10000); do /usr/bin/echo "$i" > /dev/null; done
```

---

## 9.3 External Programs

External programs are executable files on disk. They're found by searching the `PATH` variable.

```bash
# External programs
type grep        # grep is /usr/bin/grep
type awk         # awk is /usr/bin/awk
type find        # find is /usr/bin/find
type python3     # python3 is /usr/bin/python3
```

When you run an external program:
1. Bash calls `fork()` to create a child process (a copy of the shell)
2. The child calls `exec()` to replace itself with the new program
3. The program runs
4. The program exits, returning a status code
5. Bash (the parent) collects the status code and continues

```
Shell (PID 1234)
    │
    ├── fork() → Child Process (PID 5678)
    │              │
    │              └── exec("grep", ...) → grep runs
    │                                        │
    │                                        └── exit(0)
    │
    └── wait() ← collects exit status 0
```

### The Hash Table

To avoid searching PATH repeatedly, Bash caches the locations of external programs in a **hash table**:

```bash
# See the hash table
hash

# Output:
# hits    command
#    3    /usr/bin/grep
#    7    /usr/bin/ls
#    1    /usr/bin/cat

# Clear the hash table (useful if you just installed a program)
hash -r

# Manually add an entry
hash -p /usr/local/bin/my_tool my_tool
```

---

## 9.4 Aliases

An alias is a shorthand for a longer command. Aliases are resolved FIRST in the lookup order.

```bash
# View all aliases
alias

# Create an alias
alias ll='ls -la'
alias grep='grep --color=auto'
alias rm='rm -i'        # Ask before deleting (safety net)
alias ..='cd ..'
alias ...='cd ../..'

# Remove an alias
unalias ll

# An alias only works in interactive shells, NOT in scripts (by default)
```

### Temporarily Bypassing an Alias

```bash
# If rm is aliased to 'rm -i', you can bypass it:

# Method 1: Use backslash
\rm file.txt

# Method 2: Use the full path
/bin/rm file.txt

# Method 3: Use 'command' built-in
command rm file.txt

# Method 4: Use quotes
'rm' file.txt
"rm" file.txt
```

---

## 9.5 Shell Functions

Functions (covered in detail in Chapter 26) are looked up AFTER aliases but BEFORE built-ins:

```bash
# A function can override a built-in
cd() {
    builtin cd "$@" && ls    # cd AND show directory contents
}

# Now every cd automatically runs ls afterward
cd /tmp    # Changes directory AND lists contents
```

---

## 9.6 Controlling Command Resolution

### `type` — Identify How a Command Would Be Resolved

```bash
type ls          # ls is aliased to 'ls --color=auto'
type cd          # cd is a shell builtin
type grep        # grep is /usr/bin/grep
type -a echo     # Shows ALL versions (builtin + external)
type -t echo     # Just the type: builtin, file, alias, function, or keyword
```

### `command` — Bypass Aliases and Functions

`command` skips aliases and functions, running either the built-in or external program:

```bash
# If you have a cd function, this bypasses it
command cd /tmp

# Useful in functions to avoid infinite recursion
cd() {
    command cd "$@" && echo "Changed to: $PWD"
}
```

### `builtin` — Force Using the Built-in Version

```bash
# Force the built-in echo, even if there's an alias or function
builtin echo "hello"

# Useful inside functions that override built-ins
cd() {
    builtin cd "$@"    # Use the real cd, not this function (which would recurse)
    ls
}
```

### `enable` — Enable/Disable Built-ins

```bash
# Disable the built-in echo (forces use of /usr/bin/echo)
enable -n echo

# Re-enable it
enable echo

# List all built-ins
enable -a
```

---

## 9.7 When to Use Which

| Situation | Use |
|-----------|-----|
| Need to modify shell state (cd, export) | Built-in (automatic) |
| Need maximum performance in loops | Built-in (prefer `printf` over external tools) |
| Need consistent behavior across systems | External program (less variation) |
| Want portable scripts | External programs + POSIX built-ins |
| Want a convenient shorthand | Alias (interactive) or function |
| Override a built-in safely | Function + `builtin` keyword |

---

## Common Mistakes

1. **Assuming `echo` behaves the same everywhere** — The built-in `echo` and `/usr/bin/echo` have different options. The built-in varies between shells. Use `printf` for portable output.

2. **Writing functions that recurse infinitely** — If you define `cd() { cd "$@"; ls; }`, the `cd` inside calls your function, not the built-in. Use `builtin cd` or `command cd`.

3. **Relying on aliases in scripts** — Aliases are disabled in non-interactive shells by default. Use functions instead.

4. **Not clearing the hash after installing software** — If `hash` has cached an old path, run `hash -r`.

---

## Exercises

### Exercise 9.1: Classification
Classify each command as built-in, external, or both:
`cd`, `ls`, `echo`, `printf`, `grep`, `test`, `[`, `awk`, `export`, `kill`

### Exercise 9.2: Performance Test
Write a timing test comparing built-in `echo` vs `/usr/bin/echo` in a loop of 5000 iterations.

### Exercise 9.3: Function Override
Write a function called `rm` that asks for confirmation before deleting (hint: use `read` and `builtin`/`command`).

---

## Summary

- Bash looks up commands in order: **aliases → functions → built-ins → hash → PATH**
- **Built-ins** run inside the shell (fast, can modify shell state)
- **External programs** run as child processes (new fork+exec each time)
- Use `type` to identify what a command is; `type -a` to see all versions
- Use `command` to bypass aliases/functions; `builtin` to force built-in usage
- **Always quote variables** — this applies universally, regardless of command type

---

**Next Chapter:** [Chapter 10: Wildcards and Globbing →](Chapter10-Wildcards-Globbing.md)
