# Appendix C: Practice Lab Setup Guide

> **Goal**: Set up a safe, isolated environment where you can practice every exercise in this textbook without risk to your main system.

---

## Option 1: Windows Subsystem for Linux (WSL) — Recommended for Windows Users

WSL lets you run a real Linux distribution directly on Windows 10/11 with near-native performance.

### Step 1: Enable WSL

Open PowerShell as Administrator:

```powershell
wsl --install
```

This installs WSL 2 with Ubuntu by default. Restart your computer when prompted.

### Step 2: First Launch

After restart, Ubuntu will open automatically and ask you to create a username and password. These are your Linux credentials (separate from Windows).

```
Enter new UNIX username: learner
New password: ********
```

### Step 3: Update the System

```bash
sudo apt update && sudo apt upgrade -y
```

### Step 4: Install Essential Tools

```bash
sudo apt install -y \
    bash \
    coreutils \
    grep \
    sed \
    gawk \
    curl \
    wget \
    git \
    tree \
    shellcheck \
    jq \
    man-db \
    vim \
    nano \
    htop \
    bc \
    net-tools \
    openssh-client
```

### Step 5: Verify Bash Version

```bash
bash --version
# Should be 5.x or higher
```

### Step 6: Create a Practice Directory

```bash
mkdir -p ~/bash-lab/{scripts,data,projects,logs}
cd ~/bash-lab
echo "Lab is ready!" > README.md
```

### Accessing Files

- **From Linux**: Your Windows files are at `/mnt/c/Users/YourName/`
- **From Windows**: Your Linux files are at `\\wsl$\Ubuntu\home\learner\`
- **VS Code**: Install the "WSL" extension, then run `code .` from your Linux terminal

### Installing Additional Distros

```powershell
# List available distributions
wsl --list --online

# Install a specific distro
wsl --install -d Debian
```

---

## Option 2: Virtual Machine (VM)

A VM gives you a fully isolated Linux system. Best for deep system administration practice.

### Using VirtualBox (Free)

1. **Download VirtualBox**: https://www.virtualbox.org/
2. **Download an ISO**:
   - Ubuntu Server: https://ubuntu.com/download/server
   - Debian: https://www.debian.org/download
   - Rocky Linux: https://rockylinux.org/download

### VM Settings for Learning

| Setting | Recommended Value |
|---------|------------------|
| RAM | 2 GB minimum |
| CPU | 2 cores |
| Disk | 20 GB (dynamically allocated) |
| Network | NAT (internet) + Host-Only (SSH from host) |

### After Installation

```bash
# Update system
sudo apt update && sudo apt upgrade -y    # Debian/Ubuntu
sudo dnf update -y                         # Rocky/Fedora

# Install guest additions for better experience
sudo apt install -y virtualbox-guest-utils  # Debian/Ubuntu

# Install practice tools (same as WSL Step 4)
```

### SSH Into Your VM

Set up Host-Only networking, then SSH from your host:

```bash
ssh learner@192.168.56.101
```

This gives you a terminal-only workflow—perfect for learning Bash.

---

## Option 3: Docker Containers

Docker is ideal for disposable environments. Break something? Destroy the container and start fresh.

### Install Docker

- **Windows/macOS**: Install Docker Desktop from https://www.docker.com/
- **Linux**: Follow https://docs.docker.com/engine/install/

### Quick Start

```bash
# Pull and run Ubuntu interactively
docker run -it --name bash-lab ubuntu:22.04 bash

# You're now inside a container
apt update && apt install -y bash coreutils grep sed gawk curl jq shellcheck
```

### Custom Lab Image

Create a `Dockerfile`:

```dockerfile
FROM ubuntu:22.04

# Avoid interactive prompts
ENV DEBIAN_FRONTEND=noninteractive

# Install all practice tools
RUN apt-get update && apt-get install -y \
    bash \
    coreutils \
    grep \
    sed \
    gawk \
    curl \
    wget \
    git \
    tree \
    shellcheck \
    jq \
    man-db \
    vim \
    nano \
    htop \
    bc \
    net-tools \
    openssh-client \
    iputils-ping \
    procps \
    && rm -rf /var/lib/apt/lists/*

# Create practice directory
RUN mkdir -p /home/learner/bash-lab/{scripts,data,projects,logs}

# Create a non-root user
RUN useradd -m -s /bin/bash learner
USER learner
WORKDIR /home/learner/bash-lab

CMD ["/bin/bash"]
```

Build and run:

```bash
docker build -t bash-lab .
docker run -it --name my-lab bash-lab
```

### Persistent Practice with Volumes

Mount a host directory so your scripts survive container restarts:

```bash
docker run -it \
    -v ~/bash-practice:/home/learner/bash-lab \
    --name my-lab \
    bash-lab
```

### Reset the Lab

```bash
# Destroy and recreate
docker rm my-lab
docker run -it --name my-lab bash-lab
```

---

## Option 4: Cloud Instances

For practicing system administration, networking, and cron jobs in a real server environment.

### Free Tier Options

| Provider | Free Offer |
|----------|-----------|
| AWS EC2 | t2.micro for 12 months |
| Google Cloud | e2-micro always free |
| Oracle Cloud | ARM instances always free |
| Azure | B1s for 12 months |

### AWS EC2 Quick Start

1. Sign up at https://aws.amazon.com/
2. Launch EC2 → Choose **Ubuntu 22.04 LTS**
3. Instance type: **t2.micro** (free tier)
4. Create or select a key pair
5. Connect:

```bash
ssh -i your-key.pem ubuntu@<public-ip>
```

### Google Cloud Quick Start

```bash
# Install gcloud CLI, then:
gcloud compute instances create bash-lab \
    --machine-type=e2-micro \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud \
    --zone=us-central1-a

gcloud compute ssh bash-lab
```

---

## Option 5: Online Terminals (No Installation)

For quick practice sessions without any setup:

| Service | URL | Notes |
|---------|-----|-------|
| JSLinux | https://bellard.org/jslinux/ | Linux in browser |
| Copy.sh | https://copy.sh/v86/ | x86 VM in browser |
| Replit | https://replit.com/ | Free Bash REPL |
| Google Cloud Shell | https://shell.cloud.google.com/ | Free, 60hr/week |

---

## Setting Up Your Practice Environment

Regardless of which option you chose, set up these standard directories:

```bash
# Create the lab structure
mkdir -p ~/bash-lab/{scripts,data,projects,logs,tmp}
cd ~/bash-lab

# Create a practice profile
cat > ~/.bash_lab_profile << 'EOF'
# Bash Lab Profile
export LAB_HOME="$HOME/bash-lab"
export PS1='[lab] \w\$ '

# Helpful aliases for learning
alias cls='clear'
alias scripts='cd $LAB_HOME/scripts'
alias projects='cd $LAB_HOME/projects'

# Safety net
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

echo "=== Bash Practice Lab ==="
echo "Lab directory: $LAB_HOME"
echo "Bash version: $BASH_VERSION"
echo ""
EOF

echo 'source ~/.bash_lab_profile' >> ~/.bashrc
source ~/.bash_lab_profile
```

## Sample Data Files

Create sample files that many exercises reference:

```bash
cd ~/bash-lab/data

# Sample /etc/passwd style file
cat > users.txt << 'EOF'
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
alice:x:1001:1001:Alice Smith:/home/alice:/bin/bash
bob:x:1002:1002:Bob Jones:/home/bob:/bin/bash
charlie:x:1003:1003:Charlie Brown:/home/charlie:/bin/zsh
diana:x:1004:1004:Diana Prince:/home/diana:/bin/bash
eve:x:1005:1005:Eve Wilson:/home/eve:/usr/sbin/nologin
EOF

# Sample CSV data
cat > sales.csv << 'EOF'
date,product,quantity,price,region
2024-01-15,Widget,100,9.99,North
2024-01-15,Gadget,50,24.99,South
2024-01-16,Widget,75,9.99,East
2024-01-16,Doohickey,200,4.99,North
2024-01-17,Gadget,30,24.99,West
2024-01-17,Widget,150,9.99,South
2024-01-18,Doohickey,80,4.99,East
2024-01-18,Thingamajig,25,49.99,North
EOF

# Sample log file
cat > access.log << 'EOF'
192.168.1.10 - - [15/Jan/2024:10:15:30 +0000] "GET /index.html HTTP/1.1" 200 1234
192.168.1.20 - - [15/Jan/2024:10:15:31 +0000] "GET /style.css HTTP/1.1" 200 567
192.168.1.10 - - [15/Jan/2024:10:15:32 +0000] "POST /api/login HTTP/1.1" 200 89
10.0.0.5 - - [15/Jan/2024:10:15:33 +0000] "GET /admin HTTP/1.1" 403 0
192.168.1.30 - - [15/Jan/2024:10:15:34 +0000] "GET /index.html HTTP/1.1" 200 1234
10.0.0.5 - - [15/Jan/2024:10:15:35 +0000] "GET /etc/passwd HTTP/1.1" 404 0
192.168.1.10 - - [15/Jan/2024:10:16:00 +0000] "GET /about.html HTTP/1.1" 200 2345
192.168.1.40 - - [15/Jan/2024:10:16:01 +0000] "GET /products HTTP/1.1" 200 5678
10.0.0.5 - - [15/Jan/2024:10:16:02 +0000] "POST /api/login HTTP/1.1" 401 45
10.0.0.5 - - [15/Jan/2024:10:16:03 +0000] "POST /api/login HTTP/1.1" 401 45
EOF

# Sample JSON
cat > config.json << 'EOF'
{
  "app": {
    "name": "MyApp",
    "version": "2.1.0",
    "port": 8080,
    "debug": false
  },
  "database": {
    "host": "localhost",
    "port": 5432,
    "name": "mydb"
  },
  "features": ["auth", "logging", "cache"]
}
EOF

# Simple word list
cat > words.txt << 'EOF'
apple
banana
cherry
date
elderberry
fig
grape
honeydew
kiwi
lemon
mango
nectarine
orange
papaya
quince
raspberry
strawberry
tangerine
watermelon
EOF

# Numbers file
seq 1 100 > numbers.txt

echo "Sample data files created in $(pwd)"
```

## Verifying Your Setup

Run this verification script to make sure everything is ready:

```bash
cat > ~/bash-lab/scripts/verify-lab.sh << 'SCRIPT'
#!/bin/bash
# Lab Environment Verification Script

passed=0
failed=0

check() {
    local description="$1"
    local command="$2"

    if eval "$command" &>/dev/null; then
        echo "  [PASS] $description"
        ((passed++))
    else
        echo "  [FAIL] $description"
        ((failed++))
    fi
}

echo "========================================"
echo "  Bash Lab Environment Verification"
echo "========================================"
echo ""

echo "--- Bash ---"
check "Bash installed" "command -v bash"
check "Bash version 4+" "[ ${BASH_VERSINFO[0]} -ge 4 ]"

echo ""
echo "--- Core Utilities ---"
check "grep" "command -v grep"
check "sed" "command -v sed"
check "awk" "command -v awk"
check "find" "command -v find"
check "sort" "command -v sort"
check "cut" "command -v cut"
check "tr" "command -v tr"
check "wc" "command -v wc"
check "tee" "command -v tee"
check "xargs" "command -v xargs"

echo ""
echo "--- Extra Tools ---"
check "curl" "command -v curl"
check "wget" "command -v wget"
check "git" "command -v git"
check "tree" "command -v tree"
check "jq" "command -v jq"
check "shellcheck" "command -v shellcheck"
check "bc" "command -v bc"

echo ""
echo "--- Editors ---"
check "vim or vi" "command -v vim || command -v vi"
check "nano" "command -v nano"

echo ""
echo "--- Lab Structure ---"
check "Lab home exists" "[ -d '$HOME/bash-lab' ]"
check "scripts/ dir" "[ -d '$HOME/bash-lab/scripts' ]"
check "data/ dir" "[ -d '$HOME/bash-lab/data' ]"
check "projects/ dir" "[ -d '$HOME/bash-lab/projects' ]"
check "logs/ dir" "[ -d '$HOME/bash-lab/logs' ]"

echo ""
echo "--- Sample Data ---"
check "users.txt" "[ -f '$HOME/bash-lab/data/users.txt' ]"
check "sales.csv" "[ -f '$HOME/bash-lab/data/sales.csv' ]"
check "access.log" "[ -f '$HOME/bash-lab/data/access.log' ]"
check "config.json" "[ -f '$HOME/bash-lab/data/config.json' ]"
check "words.txt" "[ -f '$HOME/bash-lab/data/words.txt' ]"
check "numbers.txt" "[ -f '$HOME/bash-lab/data/numbers.txt' ]"

echo ""
echo "========================================"
echo "  Results: $passed passed, $failed failed"
echo "========================================"

if [ "$failed" -eq 0 ]; then
    echo ""
    echo "  Your lab is fully set up! Start with Chapter 1."
    echo ""
else
    echo ""
    echo "  Some checks failed. Install missing tools with:"
    echo "  sudo apt install -y <package-name>"
    echo ""
fi
SCRIPT

chmod +x ~/bash-lab/scripts/verify-lab.sh
bash ~/bash-lab/scripts/verify-lab.sh
```

## Recommended VS Code Extensions

If you use VS Code as your editor (highly recommended), install these extensions:

| Extension | Purpose |
|-----------|---------|
| **WSL** | Connect VS Code to WSL |
| **ShellCheck** | Bash linting in the editor |
| **Bash IDE** | Syntax highlighting + completion |
| **Bash Debug** | Debugging with breakpoints |
| **Remote - SSH** | Edit files on remote/VM |

Install from the terminal:

```bash
code --install-extension ms-vscode-remote.remote-wsl
code --install-extension timonwong.shellcheck
code --install-extension mads-hartmann.bash-ide-vscode
code --install-extension rogalmic.bash-debug
```

## Tips for Effective Practice

1. **Type everything** — Don't copy-paste commands. Muscle memory matters.
2. **Experiment** — After each example, modify it. What happens if you change a flag?
3. **Break things** — Use your lab to intentionally cause errors and learn to fix them.
4. **Read error messages** — They tell you exactly what went wrong.
5. **Use `man` pages** — `man grep`, `man bash` — the definitive references.
6. **Keep a journal** — Note commands and patterns you discover.
7. **One chapter per day** — Consistency beats intensity.
8. **Do every exercise** — Reading isn't learning. Doing is learning.

---

**Your lab is ready. Open Chapter 1 and begin your journey!**
