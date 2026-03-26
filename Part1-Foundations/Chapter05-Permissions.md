# Chapter 5: File Permissions and Ownership

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain the Linux permission model from first principles
- Read and interpret permission strings (`rwxr-xr--`)
- Set permissions using both symbolic and numeric (octal) notation
- Understand and use `chmod`, `chown`, and `chgrp`
- Explain special permissions: SUID, SGID, and the sticky bit
- Understand `umask` and default permissions
- Apply security-conscious permission settings

---

## 5.1 Why Permissions Exist

Linux is a **multi-user operating system**. Multiple people can use the same system simultaneously — logging in via SSH, running services, or working on a shared server. Permissions ensure that:

- Users can't read each other's private files
- Users can't modify system files
- Programs can't access resources they shouldn't
- The system remains stable and secure

Every file and directory on a Linux system has three security attributes:

1. **Owner** — The user who owns the file
2. **Group** — A group of users who share access
3. **Permissions** — What the owner, group, and others can do

---

## 5.2 Users and Groups

### Users

Every person (and many services) has a **user account** with:
- A **username** (e.g., `john`)
- A numeric **User ID (UID)** (e.g., `1000`)
- A **home directory** (e.g., `/home/john`)
- A **default group**

```bash
# See your username
whoami

# See your UID and group memberships
id

# Output:
# uid=1000(john) gid=1000(john) groups=1000(john),27(sudo),999(docker)
```

### Special Users

| User | UID | Purpose |
|------|-----|---------|
| `root` | 0 | Superuser — full system access |
| `www-data` | 33 | Web server (Apache/Nginx) |
| `nobody` | 65534 | Unprivileged user for processes that need minimal access |

### Groups

A **group** is a collection of users. Groups allow you to grant permissions to multiple users at once.

```bash
# See all groups you belong to
groups

# See all groups on the system
cat /etc/group

# Create a new group
sudo groupadd developers

# Add a user to a group
sudo usermod -aG developers john

# The -aG means "append to Group" — without -a, it would REPLACE all groups
```

---

## 5.3 Reading Permission Strings

When you run `ls -l`, the first column shows permissions:

```bash
ls -l
```

```
-rwxr-xr--  1 john developers 4096 Mar 26 10:30 script.sh
drwxr-x---  2 john developers 4096 Mar 26 10:30 private/
lrwxrwxrwx  1 john developers   11 Mar 26 10:30 link -> target
```

Let's decode `-rwxr-xr--`:

```
- rwx r-x r--
│ │   │   │
│ │   │   └── Others (everyone else): read only
│ │   └────── Group (developers): read + execute
│ └────────── Owner (john): read + write + execute
└──────────── File type: - = regular file
```

### The Permission Bits

```
Position:  1    2 3 4   5 6 7   8 9 10
String:    -    r w x   r - x   r - -
           │    └─┬─┘   └─┬─┘   └─┬─┘
           │     Owner   Group   Others
           │
           File Type
```

### File Type (Position 1)

| Character | Type |
|-----------|------|
| `-` | Regular file |
| `d` | Directory |
| `l` | Symbolic link |
| `c` | Character device |
| `b` | Block device |
| `p` | Named pipe |
| `s` | Socket |

### Permission Characters (Positions 2-10)

| Character | Permission | For Files | For Directories |
|-----------|-----------|-----------|-----------------|
| `r` | Read | Can view contents | Can list contents (`ls`) |
| `w` | Write | Can modify contents | Can create/delete files inside |
| `x` | Execute | Can run as a program | Can enter the directory (`cd`) |
| `-` | None | Permission denied | Permission denied |

### Permissions on Directories — A Common Source of Confusion

Directory permissions work differently from file permissions:

```
┌───────────────────────────────────────────────────────────────┐
│ Permission │ On Files              │ On Directories            │
├────────────┼───────────────────────┼───────────────────────────┤
│     r      │ Read file contents    │ List filenames (ls)       │
│     w      │ Modify file contents  │ Create/delete/rename files│
│     x      │ Execute as program    │ Enter directory (cd) and  │
│            │                       │ access files inside       │
└───────────────────────────────────────────────────────────────┘
```

**Critical insight:** To read a file, you need `r` on the file AND `x` on every directory in the path. Without `x` on a directory, you can't traverse through it, even if you have permission on the file itself.

```bash
# Example: To read /home/john/docs/report.txt, you need:
# x on /
# x on /home
# x on /home/john
# x on /home/john/docs
# r on /home/john/docs/report.txt
```

---

## 5.4 Changing Permissions with chmod

`chmod` (change mode) modifies file permissions. There are two notations.

### Symbolic Notation

Format: `chmod WHO+/-/=PERMISSION file`

**WHO:**
| Character | Meaning |
|-----------|---------|
| `u` | User (owner) |
| `g` | Group |
| `o` | Others |
| `a` | All (u + g + o) |

**OPERATOR:**
| Character | Meaning |
|-----------|---------|
| `+` | Add permission |
| `-` | Remove permission |
| `=` | Set exactly (remove all others) |

**PERMISSION:**
| Character | Meaning |
|-----------|---------|
| `r` | Read |
| `w` | Write |
| `x` | Execute |

```bash
# Add execute permission for the owner
chmod u+x script.sh

# Remove write permission for group and others
chmod go-w document.txt

# Set exact permissions: owner=rwx, group=rx, others=r
chmod u=rwx,g=rx,o=r program

# Add read permission for everyone
chmod a+r public_file.txt

# Remove all permissions for others
chmod o= private_file.txt

# Make a script executable by everyone
chmod +x script.sh     # Same as a+x
```

### Numeric (Octal) Notation

Each permission has a numeric value:

```
r = 4
w = 2
x = 1
- = 0
```

Add the values for each permission group:

```
rwx = 4 + 2 + 1 = 7
rw- = 4 + 2 + 0 = 6
r-x = 4 + 0 + 1 = 5
r-- = 4 + 0 + 0 = 4
--- = 0 + 0 + 0 = 0
```

Apply as a three-digit number: OWNER GROUP OTHERS

```bash
# rwxr-xr-x = 755
chmod 755 script.sh

# rw-r--r-- = 644
chmod 644 document.txt

# rw------- = 600
chmod 600 private_key

# rwxr-x--- = 750
chmod 750 program

# rwx------ = 700
chmod 700 secret_script.sh
```

### Common Permission Patterns

| Octal | Symbolic | Use Case |
|-------|----------|----------|
| `644` | `rw-r--r--` | Regular files (readable by all) |
| `755` | `rwxr-xr-x` | Scripts and programs |
| `600` | `rw-------` | Private files (SSH keys, passwords) |
| `700` | `rwx------` | Private directories/scripts |
| `750` | `rwxr-x---` | Group-shared programs |
| `775` | `rwxrwxr-x` | Group-writable directories |
| `444` | `r--r--r--` | Read-only for everyone |

### Recursive Permission Changes

```bash
# Change permissions on a directory and everything inside it
chmod -R 755 project/

# But be careful! You usually want different permissions for files vs directories.
# Common pattern:
# Directories: 755 (need x to enter)
# Files: 644 (don't need x unless they're scripts)

# Set all directories to 755
find project/ -type d -exec chmod 755 {} \;

# Set all files to 644
find project/ -type f -exec chmod 644 {} \;

# Set only .sh files to executable
find project/ -name "*.sh" -exec chmod 755 {} \;
```

---

## 5.5 Changing Ownership with chown and chgrp

### chown — Change Owner

```bash
# Change the owner of a file
sudo chown alice report.txt

# Change owner and group simultaneously
sudo chown alice:developers report.txt

# Change only the group (note the colon)
sudo chown :developers report.txt

# Recursive — change ownership of everything in a directory
sudo chown -R alice:developers project/
```

### chgrp — Change Group

```bash
# Change the group of a file
sudo chgrp developers report.txt

# Recursive
sudo chgrp -R developers project/
```

**Note:** Only root can change file ownership. Regular users can change the group only to a group they belong to.

---

## 5.6 Understanding umask

The `umask` controls the **default permissions** for newly created files and directories.

### How umask Works

The umask is a filter that **removes** permissions from the system default:
- Default for files: `666` (rw-rw-rw-)
- Default for directories: `777` (rwxrwxrwx)

The umask value is **subtracted** (conceptually) from these defaults:

```
Files:       666         Directories: 777
umask:      -022                     -022
             ───                      ───
Result:      644 (rw-r--r--)         755 (rwxr-xr-x)
```

```bash
# View your current umask
umask
# 0022

# View symbolically
umask -S
# u=rwx,g=rx,o=rx

# Test: create a file and directory
touch testfile
mkdir testdir
ls -la testfile testdir
# -rw-r--r-- ... testfile      (666 - 022 = 644)
# drwxr-xr-x ... testdir       (777 - 022 = 755)
```

### Changing the umask

```bash
# More restrictive — remove all group and other permissions
umask 077
touch private_file
ls -l private_file
# -rw------- ... private_file  (666 - 077 = 600)

# Less restrictive — allow group writing
umask 002
touch shared_file
ls -l shared_file
# -rw-rw-r-- ... shared_file   (666 - 002 = 664)
```

### Common umask Values

| umask | File Result | Dir Result | Use Case |
|-------|-------------|------------|----------|
| `022` | `644` | `755` | Default on most systems |
| `002` | `664` | `775` | Group collaboration |
| `077` | `600` | `700` | Maximum privacy |
| `027` | `640` | `750` | Privacy + group read |

To make a umask permanent, add it to your `~/.bashrc`:

```bash
echo "umask 027" >> ~/.bashrc
```

---

## 5.7 Special Permissions

Beyond the standard rwx, Linux has three special permissions.

### SUID (Set User ID) — Octal: 4000

When set on an **executable file**, it runs with the permissions of the **file's owner**, not the user who runs it.

```bash
# The passwd command has SUID set
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ... /usr/bin/passwd
#    ^
#    s instead of x = SUID is set
```

Why? The `passwd` command needs to write to `/etc/shadow`, which only root can modify. SUID lets any user run `passwd`, and it temporarily runs as root to update the password file.

```bash
# Set SUID on a file
chmod u+s program
chmod 4755 program

# Remove SUID
chmod u-s program
```

**Security warning:** SUID is a significant security concern. Never set SUID on scripts — only on compiled programs that are designed for it.

### SGID (Set Group ID) — Octal: 2000

**On executable files:** Runs with the permissions of the file's group.

**On directories:** Files created inside inherit the directory's group instead of the creator's primary group. This is very useful for shared project directories.

```bash
# Create a shared directory with SGID
sudo mkdir /shared/project
sudo chgrp developers /shared/project
sudo chmod 2775 /shared/project

# Now any file created inside will belong to the "developers" group
touch /shared/project/newfile.txt
ls -l /shared/project/newfile.txt
# -rw-rw-r-- 1 john developers ... newfile.txt
#                    ^^^^^^^^^^
#                    Inherited from directory, not john's primary group
```

```bash
# Set SGID
chmod g+s directory/
chmod 2755 directory/

# Check for SGID (s in group execute position)
ls -ld directory/
# drwxr-sr-x ... directory/
#       ^
```

### Sticky Bit — Octal: 1000

When set on a **directory**, only the owner of a file can delete or rename it within that directory — even if other users have write permission on the directory.

The classic example is `/tmp`:

```bash
ls -ld /tmp
# drwxrwxrwt ... /tmp
#          ^
#          t = sticky bit is set
```

Everyone can create files in `/tmp`, but you can only delete your own files.

```bash
# Set the sticky bit
chmod +t shared_directory/
chmod 1777 shared_directory/
```

### Special Permissions Summary

```
Permission    Octal   Symbol    On Files               On Directories
──────────    ─────   ──────    ──────────────────────  ────────────────────────
SUID          4000    s in u+x  Runs as file owner     (no effect)
SGID          2000    s in g+x  Runs as file group     New files inherit group
Sticky Bit    1000    t in o+x  (no effect)            Only owner can delete
```

Full four-digit octal example:
```bash
chmod 4755 script    # SUID + rwxr-xr-x
chmod 2775 dir       # SGID + rwxrwxr-x
chmod 1777 dir       # Sticky + rwxrwxrwx  (like /tmp)
```

---

## 5.8 Practical Permission Scenarios

### Scenario 1: Making a Script Executable

```bash
# You wrote a script but can't run it
./myscript.sh
# bash: ./myscript.sh: Permission denied

# Check permissions
ls -l myscript.sh
# -rw-r--r-- 1 john john 512 Mar 26 10:30 myscript.sh
# No execute (x) permission!

# Fix it
chmod +x myscript.sh
# or more specifically
chmod 755 myscript.sh

# Now it works
./myscript.sh
```

### Scenario 2: Protecting SSH Keys

```bash
# SSH requires strict permissions on key files
ls -l ~/.ssh/
# -rw------- 1 john john 1675 id_rsa         (private key: 600)
# -rw-r--r-- 1 john john  411 id_rsa.pub     (public key: 644)
# -rw------- 1 john john   88 config         (config: 600)
# drwx------ 2 john john 4096 .ssh/          (directory: 700)

# Fix SSH key permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
chmod 600 ~/.ssh/config
```

### Scenario 3: Shared Team Directory

```bash
# Create a shared project directory
sudo mkdir -p /projects/webapp
sudo groupadd webapp-team
sudo usermod -aG webapp-team john
sudo usermod -aG webapp-team alice

# Set ownership and permissions
sudo chown root:webapp-team /projects/webapp
sudo chmod 2775 /projects/webapp
# 2   = SGID (files inherit group)
# 775 = owner:rwx, group:rwx, others:r-x

# Now john and alice can both create and modify files
# All files will belong to the webapp-team group
```

### Scenario 4: Web Server Files

```bash
# Typical web server file permissions
# Web files owned by deployer, readable by web server
sudo chown -R deploy:www-data /var/www/mysite/
find /var/www/mysite/ -type d -exec chmod 750 {} \;
find /var/www/mysite/ -type f -exec chmod 640 {} \;

# Upload directory — web server needs to write
chmod 770 /var/www/mysite/uploads/
```

---

## 5.9 Access Control Lists (ACLs) — Beyond Basic Permissions

When the owner/group/others model isn't flexible enough, ACLs provide fine-grained control.

```bash
# View ACL for a file
getfacl document.txt

# Grant read access to a specific user
setfacl -m u:alice:r document.txt

# Grant read/write to a specific group
setfacl -m g:editors:rw document.txt

# Remove a specific ACL entry
setfacl -x u:alice document.txt

# Remove all ACLs
setfacl -b document.txt
```

A `+` at the end of the permission string indicates ACLs are set:

```bash
ls -l document.txt
# -rw-r--r--+ 1 john john 1234 document.txt
#           ^
#           ACL is present
```

ACLs are an advanced topic but invaluable when you need to give specific permissions to specific users beyond the basic three categories.

---

## Common Mistakes

1. **Setting everything to 777** — This is the "just make it work" approach and a serious security risk. Never do this in production. Always use the minimum permissions necessary.

2. **Forgetting directory execute permissions** — To access files in a directory, you need `x` on the directory. `r` on a directory only lets you list file names, not access them.

3. **Using `chmod -R 755` on everything** — This makes all files executable, which is wrong and potentially dangerous. Use `find` to set different permissions for files and directories.

4. **Not understanding umask** — If new files have unexpected permissions, check your `umask`. It's the hidden hand controlling defaults.

5. **Using SUID carelessly** — Never set SUID on scripts. SUID on the wrong binary can give attackers root access.

6. **Forgetting the `sudo` requirement** — You can't change ownership without root access. Use `sudo chown`.

---

## Exercises

### Exercise 5.1: Permission Reading
For each permission string, write what the owner, group, and others can do:
1. `-rwxr--r--`
2. `-rw-rw----`
3. `drwxr-x---`
4. `-r-x------`
5. `-rwsr-xr-x`

### Exercise 5.2: Permission Setting
Write the `chmod` command (both symbolic and octal) for each scenario:
1. Owner can read/write; everyone else can only read
2. Owner can do everything; group can read and execute; others get nothing
3. Everyone can read and execute; only owner can write
4. Only the owner can read and write; no one else has any access

### Exercise 5.3: Shared Directory
Create a directory `/tmp/teamwork` that:
1. Is owned by group `users`
2. Allows all members of `users` to create files
3. Ensures new files inherit the `users` group
4. Prevents users from deleting each other's files

What permissions did you set?

### Exercise 5.4: Troubleshooting
You get "Permission denied" when running `./script.sh`. Walk through the complete troubleshooting process:
1. What permissions does the script need?
2. What permissions do the parent directories need?
3. How do you check each one?
4. How do you fix each one?

### Exercise 5.5: Security Audit
Run these commands and evaluate whether the permissions are appropriate:
```bash
find /home -perm -002 -type f    # World-writable files
find / -perm -4000 -type f 2>/dev/null  # SUID files
```
Why might world-writable files be dangerous? Why should you audit SUID files?

---

## Summary

- Linux uses a **three-tier permission model**: user (owner), group, others
- Each tier has three permissions: **read (r=4)**, **write (w=2)**, **execute (x=1)**
- **Permissions work differently on directories** — `x` means "can enter," `w` means "can create/delete files"
- Use **`chmod`** to change permissions (symbolic: `u+x`, octal: `755`)
- Use **`chown`** and **`chgrp`** to change ownership
- **`umask`** controls default permissions for new files
- **Special permissions**: SUID (run as owner), SGID (inherit group), sticky bit (only owner can delete)
- **Security principle**: Always use the minimum permissions necessary

---

**Next Chapter:** [Chapter 6: Environment Variables and PATH →](Chapter06-Environment-Variables.md)
