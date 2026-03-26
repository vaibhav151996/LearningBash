# Appendix B: Command Cheat Sheet

## File Operations

| Command | Description | Example |
|---------|-------------|---------|
| `ls` | List directory | `ls -lah` |
| `cd` | Change directory | `cd /var/log` |
| `pwd` | Print working directory | `pwd` |
| `cp` | Copy file/dir | `cp -r src/ dest/` |
| `mv` | Move/rename | `mv old.txt new.txt` |
| `rm` | Remove | `rm -rf directory/` |
| `mkdir` | Create directory | `mkdir -p a/b/c` |
| `touch` | Create/update timestamp | `touch file.txt` |
| `ln` | Create link | `ln -s target link` |
| `chmod` | Change permissions | `chmod 755 script.sh` |
| `chown` | Change owner | `chown user:group file` |
| `find` | Find files | `find . -name "*.sh"` |
| `locate` | Quick file search | `locate config.yml` |
| `file` | Determine file type | `file data.bin` |
| `stat` | File details | `stat file.txt` |

## Text Processing

| Command | Description | Example |
|---------|-------------|---------|
| `cat` | Display file | `cat file.txt` |
| `less` | Page through file | `less logfile` |
| `head` | First N lines | `head -20 file.txt` |
| `tail` | Last N lines / follow | `tail -f log.txt` |
| `grep` | Search patterns | `grep -rn "TODO" src/` |
| `sed` | Stream editor | `sed 's/old/new/g' file` |
| `awk` | Field processing | `awk '{print $1}' file` |
| `cut` | Extract fields | `cut -d: -f1 /etc/passwd` |
| `sort` | Sort lines | `sort -k2 -n data.txt` |
| `uniq` | Remove duplicates | `sort file \| uniq -c` |
| `tr` | Translate chars | `tr 'a-z' 'A-Z'` |
| `wc` | Count lines/words | `wc -l file.txt` |
| `paste` | Merge files | `paste f1.txt f2.txt` |
| `column` | Format columns | `column -t -s,` |
| `diff` | Compare files | `diff -u old new` |
| `tac` | Reverse lines | `tac file.txt` |
| `rev` | Reverse characters | `rev file.txt` |

## System Information

| Command | Description | Example |
|---------|-------------|---------|
| `uname` | System info | `uname -a` |
| `hostname` | Host name | `hostname` |
| `uptime` | System uptime | `uptime` |
| `whoami` | Current user | `whoami` |
| `id` | User/group IDs | `id` |
| `df` | Disk free space | `df -h` |
| `du` | Disk usage | `du -sh *` |
| `free` | Memory usage | `free -h` |
| `top` | Process monitor | `top` |
| `htop` | Enhanced top | `htop` |
| `ps` | Process list | `ps aux` |
| `lscpu` | CPU info | `lscpu` |
| `lsblk` | Block devices | `lsblk` |
| `mount` | Mount filesystems | `mount` |
| `dmesg` | Kernel messages | `dmesg \| tail` |

## Process Management

| Command | Description | Example |
|---------|-------------|---------|
| `ps` | List processes | `ps aux \| grep nginx` |
| `kill` | Send signal | `kill -TERM $pid` |
| `killall` | Kill by name | `killall nginx` |
| `pkill` | Kill by pattern | `pkill -f "python app"` |
| `pgrep` | Find by pattern | `pgrep -l nginx` |
| `nice` | Run with priority | `nice -n 10 cmd` |
| `nohup` | Keep after logout | `nohup cmd &` |
| `bg` | Background job | `bg %1` |
| `fg` | Foreground job | `fg %1` |
| `jobs` | List jobs | `jobs -l` |
| `wait` | Wait for jobs | `wait $pid` |

## Networking

| Command | Description | Example |
|---------|-------------|---------|
| `curl` | HTTP requests | `curl -sO url` |
| `wget` | Download files | `wget url` |
| `ssh` | Remote shell | `ssh user@host` |
| `scp` | Secure copy | `scp file user@host:path` |
| `rsync` | Sync files | `rsync -avz src/ host:dest/` |
| `ping` | Test connectivity | `ping -c4 host` |
| `ss` | Socket stats | `ss -tunlp` |
| `ip` | Network config | `ip addr show` |
| `dig` | DNS lookup | `dig example.com` |
| `nc` | Netcat | `nc -zv host port` |

## Archives & Compression

| Command | Description | Example |
|---------|-------------|---------|
| `tar` | Archive | `tar -czf archive.tar.gz dir/` |
| `tar` | Extract | `tar -xzf archive.tar.gz` |
| `tar` | List | `tar -tzf archive.tar.gz` |
| `gzip` | Compress | `gzip file` |
| `gunzip` | Decompress | `gunzip file.gz` |
| `zip` | Zip archive | `zip -r archive.zip dir/` |
| `unzip` | Extract zip | `unzip archive.zip` |

## User & Permissions

| Command | Description | Example |
|---------|-------------|---------|
| `useradd` | Create user | `useradd -m username` |
| `usermod` | Modify user | `usermod -aG group user` |
| `passwd` | Set password | `passwd username` |
| `su` | Switch user | `su - username` |
| `sudo` | Run as root | `sudo command` |
| `groups` | Show groups | `groups username` |

## Package Managers

```bash
# Debian/Ubuntu (apt)
sudo apt update && sudo apt install package
sudo apt remove package
apt search keyword

# Red Hat/CentOS (yum/dnf)
sudo dnf install package
sudo dnf remove package
dnf search keyword

# Arch (pacman)
sudo pacman -S package
sudo pacman -R package
pacman -Ss keyword
```

## Systemd

| Command | Description |
|---------|-------------|
| `systemctl start svc` | Start service |
| `systemctl stop svc` | Stop service |
| `systemctl restart svc` | Restart service |
| `systemctl enable svc` | Enable at boot |
| `systemctl status svc` | Service status |
| `journalctl -u svc` | Service logs |
| `journalctl -f` | Follow all logs |

## Git

| Command | Description |
|---------|-------------|
| `git init` | Initialize repo |
| `git clone url` | Clone repo |
| `git add .` | Stage all changes |
| `git commit -m "msg"` | Commit |
| `git push` | Push to remote |
| `git pull` | Pull from remote |
| `git status` | Show status |
| `git log --oneline` | Compact log |
| `git diff` | Show changes |
| `git branch` | List branches |
| `git checkout -b name` | New branch |

## Keyboard Shortcuts (Bash)

| Shortcut | Action |
|----------|--------|
| `Ctrl+C` | Cancel current command |
| `Ctrl+D` | Exit shell / EOF |
| `Ctrl+Z` | Suspend process |
| `Ctrl+L` | Clear screen |
| `Ctrl+R` | Reverse history search |
| `Ctrl+A` | Move to line start |
| `Ctrl+E` | Move to line end |
| `Ctrl+U` | Delete to line start |
| `Ctrl+K` | Delete to line end |
| `Ctrl+W` | Delete previous word |
| `Alt+.` | Last argument of previous command |
| `!!` | Repeat last command |
| `!$` | Last argument of last command |
| `Tab` | Auto-complete |
