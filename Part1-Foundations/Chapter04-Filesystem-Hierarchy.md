# Chapter 4: The Linux Filesystem Hierarchy

## Learning Objectives

By the end of this chapter, you will be able to:

- Explain the purpose of every major directory in the Linux filesystem
- Understand the Filesystem Hierarchy Standard (FHS)
- Navigate the filesystem with purpose, knowing where to find configuration, logs, binaries, and data
- Distinguish between physical and virtual filesystems
- Understand mount points and how Linux unifies storage into one tree

---

## 4.1 The Single-Tree Design

One of the most fundamental differences between Linux and Windows is how the filesystem is structured.

**Windows** uses drive letters — `C:\`, `D:\`, `E:\` — each representing a separate partition or device. Files on a USB drive might be at `E:\photos\`, completely unrelated to `C:\Users\`.

**Linux** uses a single tree structure rooted at `/` (pronounced "root" or "slash"). Everything — every file, every device, every process — exists somewhere under this single root. There are no drive letters.

```
/  (root of everything)
├── bin/
├── boot/
├── dev/
├── etc/
├── home/
│   ├── john/
│   └── alice/
├── lib/
├── media/
├── mnt/
├── opt/
├── proc/
├── root/
├── run/
├── sbin/
├── srv/
├── sys/
├── tmp/
├── usr/
│   ├── bin/
│   ├── lib/
│   ├── local/
│   └── share/
└── var/
    ├── log/
    ├── cache/
    └── tmp/
```

When you plug in a USB drive on Linux, it doesn't appear as `D:\`. Instead, it gets **mounted** at a point in the tree — for example, `/media/john/usb_drive`. It becomes part of the same unified tree.

---

## 4.2 The Filesystem Hierarchy Standard (FHS)

The directory layout isn't random. It follows the **Filesystem Hierarchy Standard (FHS)**, maintained by the Linux Foundation. The FHS defines where files should go so that:

- Users know where to find things
- Programs know where to install things
- Different Linux distributions remain consistent

Let's explore every major directory.

---

## 4.3 Directory-by-Directory Guide

### `/` — The Root Directory

The root of the entire filesystem. Everything begins here. There is only one root directory on a Linux system.

```bash
ls /
```

**Do not confuse** `/` (the root directory) with `/root` (the root user's home directory).

---

### `/bin` — Essential User Binaries

Contains essential commands needed for the system to boot and operate in single-user mode. These are the commands available to **all users**.

```bash
ls /bin
# bash  cat  cp  echo  grep  ls  mkdir  mv  rm  sh  ...
```

On modern systems, `/bin` is often a symbolic link to `/usr/bin`. This merger simplifies things while maintaining compatibility.

**Key insight:** If a command is critical enough that the system can't boot without it, it lives here (or historically lived here).

---

### `/sbin` — System Binaries

Contains commands for **system administration** — typically only useful to the root user.

```bash
ls /sbin
# fdisk  fsck  ifconfig  iptables  mkfs  reboot  shutdown  ...
```

Like `/bin`, on modern systems `/sbin` is often a symlink to `/usr/sbin`.

**The difference:** `/bin` has user commands (`ls`, `cp`); `/sbin` has admin commands (`fdisk`, `iptables`).

---

### `/usr` — User Programs and Data

The largest directory on most systems. Contains the majority of installed software.

```
/usr/
├── bin/        ← Most user commands (non-essential ones)
├── sbin/       ← Non-essential system administration commands
├── lib/        ← Libraries for /usr/bin and /usr/sbin
├── lib64/      ← 64-bit libraries
├── local/      ← Locally installed software (by the admin)
│   ├── bin/
│   ├── lib/
│   └── share/
├── share/      ← Architecture-independent shared data
│   ├── doc/    ← Documentation
│   ├── man/    ← Manual pages
│   └── icons/  ← Icon files
├── include/    ← C/C++ header files
└── src/        ← Source code (kernel source, etc.)
```

**The "usr" name:** Contrary to popular belief, `/usr` doesn't stand for "user." It historically stood for "Unix System Resources." User files go in `/home`.

**`/usr/local/`** is special — it's where you install software manually (not through the package manager). This keeps custom software separate from distro-managed software:

```bash
# Package manager installs to /usr/bin/
sudo apt install vim     # → /usr/bin/vim

# Manual installation goes to /usr/local/bin/
sudo cp my_tool /usr/local/bin/
```

---

### `/etc` — Configuration Files

The "nervous system" of Linux. Contains system-wide configuration files in **plain text**.

```bash
ls /etc
# bash.bashrc  crontab  fstab  hostname  hosts  network/
# nginx/  passwd  shadow  ssh/  sudoers  sysctl.conf  ...
```

Important files:

| File | Purpose |
|------|---------|
| `/etc/passwd` | User account information |
| `/etc/shadow` | Encrypted passwords |
| `/etc/group` | Group definitions |
| `/etc/hostname` | The system's hostname |
| `/etc/hosts` | Static host-to-IP mappings |
| `/etc/fstab` | Filesystem mount table (what to mount at boot) |
| `/etc/crontab` | System-wide cron jobs |
| `/etc/sudoers` | Sudo permissions |
| `/etc/ssh/sshd_config` | SSH server configuration |
| `/etc/bash.bashrc` | System-wide Bash configuration |
| `/etc/profile` | System-wide login shell configuration |
| `/etc/resolv.conf` | DNS resolver configuration |

**Key insight:** On Linux, configuration is done through text files, not registries or GUIs. This makes configuration scriptable, versionable, and easily backed up.

```bash
# Example: check the system hostname
cat /etc/hostname

# Example: see all user accounts
cat /etc/passwd

# Example: see DNS configuration
cat /etc/resolv.conf
```

**The name "etc":** Originally stood for "et cetera" — it was a catch-all. Now it exclusively holds configuration files.

---

### `/home` — User Home Directories

Each user gets their own directory under `/home`:

```
/home/
├── john/          ← John's files, settings, documents
│   ├── .bashrc    ← John's Bash configuration
│   ├── .ssh/      ← John's SSH keys
│   ├── Documents/
│   ├── Downloads/
│   └── projects/
├── alice/         ← Alice's files
└── bob/           ← Bob's files
```

The `~` shortcut (tilde) refers to your home directory:

```bash
echo ~           # /home/john
echo ~alice      # /home/alice  (another user's home)
```

**Your home directory is yours.** You have full control over it. System files in `/etc`, `/usr`, etc., typically require root (administrator) access to modify.

---

### `/root` — The Root User's Home

The superuser (administrator account, `root`) has its home directory at `/root`, NOT at `/home/root`.

```bash
# Regular users can't access this
ls /root
# ls: cannot open directory '/root': Permission denied

# Only root can
sudo ls /root
```

**Why not `/home/root`?** Because `/home` might be on a separate partition that isn't mounted during recovery. The root user needs a home directory even when the system is barely functional.

---

### `/var` — Variable Data

Contains files that change during normal system operation — logs, caches, mail, databases.

```
/var/
├── log/       ← System and application logs
│   ├── syslog
│   ├── auth.log
│   ├── kern.log
│   └── nginx/
├── cache/     ← Application caches
├── lib/       ← Variable state information (databases, etc.)
├── mail/      ← User mailboxes
├── spool/     ← Print queues, mail queues
├── tmp/       ← Temporary files that survive reboot
├── run/       ← Runtime data (PID files, sockets)
└── www/       ← Web server files (on some systems)
```

The most important subdirectory for a sysadmin is `/var/log`:

```bash
# View system log
sudo less /var/log/syslog

# View authentication log (who logged in?)
sudo less /var/log/auth.log

# Watch log in real-time
sudo tail -f /var/log/syslog
```

---

### `/tmp` — Temporary Files

A world-writable directory for temporary files. Any user can create files here.

```bash
# Create a temporary file
echo "scratch data" > /tmp/mytemp.txt

# Files here may be automatically deleted on reboot
```

**Security note:** Because `/tmp` is world-writable, it's a common attack vector. We'll discuss secure temp file handling in Chapter 48.

On many modern systems, `/tmp` is a `tmpfs` — a filesystem that lives in RAM, making it very fast but contents are lost on reboot.

---

### `/dev` — Device Files

In Linux, **everything is a file** — including hardware devices.

```bash
ls /dev
# sda  sda1  sda2  tty  null  zero  random  urandom  stdin  stdout  stderr
```

| Device | Purpose |
|--------|---------|
| `/dev/sda` | First hard disk |
| `/dev/sda1` | First partition of first disk |
| `/dev/nvme0n1` | First NVMe SSD |
| `/dev/tty` | Current terminal |
| `/dev/null` | The "black hole" — discards anything written to it |
| `/dev/zero` | Produces infinite zero bytes |
| `/dev/random` | Random number generator (blocking) |
| `/dev/urandom` | Random number generator (non-blocking) |
| `/dev/stdin` | Standard input (fd 0) |
| `/dev/stdout` | Standard output (fd 1) |
| `/dev/stderr` | Standard error (fd 2) |

Special devices you'll use often:

```bash
# Discard output (send to the "black hole")
command_that_is_noisy > /dev/null

# Discard all output including errors
command 2>/dev/null

# Create a file of zeros (useful for creating disk images)
dd if=/dev/zero of=blank.img bs=1M count=100

# Generate random data
head -c 32 /dev/urandom | base64
```

---

### `/proc` — Process Information (Virtual Filesystem)

`/proc` is a **virtual filesystem** — it doesn't exist on disk. The kernel generates its contents on the fly, providing a window into the running system.

```bash
# Every running process has a directory here
ls /proc
# 1/  2/  3/  ... 1234/  ... cpuinfo  meminfo  version

# Information about process 1234
ls /proc/1234/
# cmdline  cwd  environ  exe  fd/  maps  status  ...

# System information
cat /proc/cpuinfo       # CPU details
cat /proc/meminfo       # Memory information
cat /proc/version       # Kernel version
cat /proc/uptime        # System uptime in seconds
cat /proc/loadavg       # System load averages
cat /proc/filesystems   # Supported filesystems
```

Process-specific information:

```bash
# What command started process 1234?
cat /proc/1234/cmdline

# What is its current working directory?
ls -l /proc/1234/cwd

# What files does it have open?
ls /proc/1234/fd/

# What is its status?
cat /proc/1234/status
```

**Key insight:** When you run `top` or `ps` or `free`, they're reading from `/proc`. You could write your own system monitoring tools just by reading these files.

---

### `/sys` — System and Hardware Information (Virtual Filesystem)

Like `/proc`, `/sys` is a virtual filesystem. It provides a structured interface to kernel data, especially hardware-related information.

```bash
# CPU information
ls /sys/devices/system/cpu/

# Block devices
ls /sys/block/

# Network interfaces
ls /sys/class/net/

# Power management
cat /sys/class/power_supply/BAT0/capacity   # Battery percentage (laptops)
```

`/proc` is older and somewhat disorganized; `/sys` is the modern, well-structured replacement for hardware information.

---

### `/boot` — Boot Loader Files

Contains files needed to boot the system.

```bash
ls /boot
# grub/  vmlinuz-5.15.0-91-generic  initrd.img-5.15.0-91-generic  config-5.15.0-91-generic
```

| File | Purpose |
|------|---------|
| `vmlinuz-*` | The Linux kernel (compressed) |
| `initrd.img-*` or `initramfs-*` | Initial RAM disk for boot |
| `grub/` | GRUB bootloader configuration |
| `config-*` | Kernel configuration |

**Warning:** Modifying files in `/boot` incorrectly can make your system unbootable.

---

### `/lib` and `/lib64` — Shared Libraries

Contains shared libraries needed by programs in `/bin` and `/sbin`. Think of them as the Linux equivalent of Windows DLL files.

```bash
ls /lib/x86_64-linux-gnu/
# libc.so.6  libm.so.6  libpthread.so.0  ...
```

On modern systems, `/lib` is often a symlink to `/usr/lib`.

---

### `/opt` — Optional Software

For self-contained third-party software packages that don't follow the standard `/usr` layout.

```bash
# Examples of software that might install here
ls /opt
# google/  slack/  visual-studio-code/  zoom/
```

Each program gets its own directory: `/opt/vendor/program/`.

---

### `/media` and `/mnt` — Mount Points

- `/media/` — Where removable media (USB drives, CDs) are automatically mounted
- `/mnt/` — Traditionally used for temporarily mounting filesystems manually

```bash
# A USB drive might appear at:
ls /media/john/MyUSBDrive/

# Manually mounting a filesystem:
sudo mount /dev/sdb1 /mnt
ls /mnt
sudo umount /mnt
```

---

### `/srv` — Service Data

Data served by the system — web pages, FTP files, etc.

```bash
# Web server files might be at:
ls /srv/www/
# or /srv/http/
```

Many distributions use `/var/www` instead.

---

### `/run` — Runtime Data

A `tmpfs` filesystem for data needed since the last boot — PID files, sockets, lock files.

```bash
ls /run
# lock/  log/  nginx.pid  sshd.pid  user/
```

---

## 4.4 Understanding Mount Points

A **mount point** is a directory where a filesystem (from a disk partition, USB drive, network share, etc.) is attached to the single directory tree.

```bash
# View mounted filesystems
mount | column -t
# or
df -h

# Example output:
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1        50G   15G   33G  32% /
# /dev/sda2       200G   80G  110G  43% /home
# tmpfs           3.9G  1.2M  3.9G   1% /tmp
# /dev/sdb1        16G  5.0G   11G  32% /media/john/usb
```

In this example:
- `/dev/sda1` is the system disk, mounted at `/`
- `/dev/sda2` is a separate partition, mounted at `/home`
- `/tmp` is a RAM-based filesystem
- `/dev/sdb1` is a USB drive, mounted at `/media/john/usb`

They all appear as one seamless filesystem tree.

```
/                    ← /dev/sda1 (root partition)
├── home/            ← /dev/sda2 (home partition)  ← mount boundary
│   └── john/
├── tmp/             ← tmpfs (RAM)                 ← mount boundary
└── media/
    └── john/
        └── usb/     ← /dev/sdb1 (USB drive)      ← mount boundary
```

### The fstab File

`/etc/fstab` defines which filesystems are mounted at boot:

```bash
cat /etc/fstab
# <filesystem>  <mount point>  <type>  <options>        <dump>  <pass>
# /dev/sda1     /              ext4    errors=remount-ro 0       1
# /dev/sda2     /home          ext4    defaults          0       2
# tmpfs         /tmp           tmpfs   defaults          0       0
```

---

## 4.5 Virtual Filesystems — Where Files Aren't Really Files

Three important virtual filesystems don't correspond to data on disk:

| Virtual FS | Mount Point | Purpose |
|-----------|------------|---------|
| `procfs` | `/proc` | Process and kernel information |
| `sysfs` | `/sys` | Device and driver information |
| `tmpfs` | `/tmp`, `/run` | RAM-based temporary storage |

These exist only in memory. They're generated by the kernel on demand. When you read `/proc/cpuinfo`, there's no file on disk — the kernel generates the content at the moment you read it.

This is the power of "everything is a file" — even system information looks like a file, so you can read it with standard tools (`cat`, `grep`, `awk`).

---

## 4.6 Filesystem Quick Reference

```
Directory    Purpose                              Remember As
─────────    ────────────────────────────────      ──────────────────
/            Root of everything                   "The top"
/bin         Essential user commands              "Basic binaries"
/sbin        System administration commands       "System binaries"
/usr         User programs and shared data        "Unix System Resources"
/usr/local   Locally installed software           "Your custom installs"
/etc         Configuration files                  "Editable Text Config"
/home        User home directories                "Your stuff"
/root        Root user's home                     "Admin's home"
/var         Variable data (logs, caches)         "Things that change"
/var/log     Log files                            "What happened?"
/tmp         Temporary files (cleared on reboot)  "Scratch space"
/dev         Device files                         "Hardware as files"
/proc        Process information (virtual)        "Running system info"
/sys         Hardware information (virtual)       "Hardware details"
/boot        Boot loader files                    "Startup files"
/lib         Shared libraries                     "DLLs of Linux"
/opt         Optional third-party software        "Extra packages"
/media       Removable media mount points         "USB drives, CDs"
/mnt         Temporary manual mounts              "Mount it here"
/srv         Service data (web, FTP)              "Server content"
/run         Runtime data (PIDs, sockets)         "Since last boot"
```

---

## Common Mistakes

1. **Storing personal files in system directories** — Keep your files in `/home/yourname/`. Don't put scripts in `/usr/bin/` (use `/usr/local/bin/` for custom scripts).

2. **Editing system files without backups** — Before modifying anything in `/etc/`, make a backup:
   ```bash
   sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
   ```

3. **Confusing `/` with `/root`** — `/` is the root of the filesystem. `/root` is the home directory of the root user. Very different things.

4. **Ignoring `/var/log`** — When something goes wrong, the answer is almost always in the logs. Check `/var/log/syslog`, `/var/log/auth.log`, and application-specific logs.

5. **Not understanding virtual filesystems** — Files in `/proc` and `/sys` aren't real files. They're generated on the fly. You can't copy `/proc` to back up your system.

---

## Exercises

### Exercise 4.1: Filesystem Tour
Navigate to each of these directories and list their contents. Write one sentence describing what you find:
1. `/etc`
2. `/var/log`
3. `/proc`
4. `/dev`
5. `/usr/bin`

### Exercise 4.2: Finding Configuration
Without using Google, find the files that contain:
1. Your system's hostname
2. The list of user accounts
3. The DNS server configuration
4. The filesystem mount table

### Exercise 4.3: System Information from /proc
Using only `cat` and files in `/proc`, answer:
1. How many CPUs/cores does your system have?
2. How much total RAM do you have?
3. How long has the system been running?
4. What version of the Linux kernel are you running?

### Exercise 4.4: Device Files
1. Write "hello" to `/dev/null`. What happens?
2. Read 10 bytes from `/dev/urandom`. What do you see?
3. What is the difference between `/dev/random` and `/dev/urandom`?

### Exercise 4.5: Filesystem Map
Draw a diagram (on paper or ASCII art) of the top-level directories on YOUR system. Next to each, write what you found inside. Compare with the FHS standard.

---

## Summary

- Linux uses a **single directory tree** rooted at `/` — no drive letters
- The **Filesystem Hierarchy Standard** (FHS) defines where files belong
- **`/etc`** holds configuration files; **`/var/log`** holds logs; **`/home`** holds user data
- **`/proc`** and **`/sys`** are virtual filesystems generated by the kernel
- **`/dev`** represents hardware devices as files — "everything is a file"
- **Mount points** attach different storage devices into the unified tree
- **`/usr/local`** is where you install custom software
- Understanding the filesystem hierarchy is essential for system administration

---

**Next Chapter:** [Chapter 5: File Permissions and Ownership →](Chapter05-Permissions.md)
