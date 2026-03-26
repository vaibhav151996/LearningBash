# Chapter 38: Networking from the Shell

## Learning Objectives

By the end of this chapter, you will be able to:

- Use networking tools from the command line
- Transfer files with curl and wget
- Test connectivity and diagnose network issues
- Work with SSH for remote commands
- Use Bash's built-in network features

---

## 38.1 Testing Connectivity

```bash
# Ping — test if host is reachable
ping -c 4 google.com                # 4 packets
ping -c 1 -W 2 192.168.1.1         # 1 packet, 2-second timeout

# Check if a port is open
nc -zv host.example.com 80         # Netcat port check
timeout 3 bash -c "echo > /dev/tcp/host.com/80" 2>/dev/null && echo "Open"

# DNS lookup
nslookup example.com
dig example.com
host example.com

# Route tracing
traceroute example.com
tracepath example.com

# Show network interfaces
ip addr show
ifconfig                            # Legacy but common

# Show open connections
ss -tunlp                          # TCP/UDP, numeric, listening, processes
netstat -tunlp                     # Legacy equivalent
```

---

## 38.2 curl — Transfer Data

```bash
# GET request
curl https://api.example.com/users
curl -s https://api.example.com/users     # Silent (no progress bar)

# Save to file
curl -o output.html https://example.com
curl -O https://example.com/file.zip      # Keep original filename

# POST request
curl -X POST -d "name=Alice&age=30" https://api.example.com/users
curl -X POST -H "Content-Type: application/json" \
     -d '{"name":"Alice","age":30}' https://api.example.com/users

# Headers
curl -H "Authorization: Bearer $TOKEN" https://api.example.com/me
curl -I https://example.com              # Headers only (HEAD request)

# Follow redirects
curl -L https://example.com              # Follow 301/302 redirects

# With authentication
curl -u username:password https://api.example.com/secure

# Upload file
curl -F "file=@document.pdf" https://upload.example.com

# API scripting pattern
response=$(curl -sf "https://api.example.com/status")
if [[ $? -eq 0 ]]; then
    echo "$response" | jq '.status'
else
    echo "API request failed" >&2
fi

# Download with retry
curl --retry 3 --retry-delay 5 -O https://example.com/large-file.tar.gz
```

---

## 38.3 wget — Download Files

```bash
# Download file
wget https://example.com/file.zip

# Download to specific location
wget -O /tmp/data.csv https://example.com/data.csv

# Quiet mode
wget -q https://example.com/file.zip

# Resume interrupted download
wget -c https://example.com/large-file.iso

# Mirror a website
wget --mirror --convert-links --page-requisites https://example.com

# Download list of URLs
wget -i urls.txt
```

---

## 38.4 SSH — Secure Shell

```bash
# Connect to remote host
ssh user@remote-host.com
ssh -p 2222 user@host.com              # Custom port

# Run remote command
ssh user@host "ls -la /var/log"
ssh user@host "df -h && free -m"

# Run local script remotely
ssh user@host 'bash -s' < local_script.sh

# Copy files with scp
scp file.txt user@host:/remote/path/
scp user@host:/remote/file.txt ./local/
scp -r directory/ user@host:/remote/     # Recursive

# rsync — better for large or repeated transfers
rsync -avz ./local/ user@host:/remote/
rsync -avz --delete ./src/ user@host:/dest/   # Sync and delete removed files
rsync -avz --progress large_file user@host:/dest/

# SSH tunneling
ssh -L 8080:localhost:80 user@remote     # Local port forwarding
ssh -R 8080:localhost:3000 user@remote   # Remote port forwarding

# SSH key setup
ssh-keygen -t ed25519 -C "your@email.com"
ssh-copy-id user@remote-host
```

---

## 38.5 Bash Built-in Networking

Bash has built-in TCP/UDP support (not available in all builds):

```bash
# Check if a port is open
if echo > /dev/tcp/localhost/80 2>/dev/null; then
    echo "Port 80 is open"
else
    echo "Port 80 is closed"
fi

# Simple HTTP GET (using /dev/tcp)
exec 3<>/dev/tcp/example.com/80
echo -e "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n" >&3
cat <&3
exec 3>&-

# Wait for a service to be ready
wait_for_port() {
    local host="$1" port="$2" timeout="${3:-30}"
    local i=0
    while ! echo > /dev/tcp/"$host"/"$port" 2>/dev/null; do
        (( i++ >= timeout )) && return 1
        sleep 1
    done
    return 0
}

wait_for_port localhost 5432 30 && echo "Postgres is ready"
```

---

## 38.6 Practical Scripts

### Health Check Script

```bash
#!/bin/bash
check_url() {
    local url="$1" name="$2"
    local status
    status=$(curl -so /dev/null -w "%{http_code}" --max-time 5 "$url")
    if [[ "$status" == "200" ]]; then
        printf "%-20s ✓ OK (%s)\n" "$name" "$status"
    else
        printf "%-20s ✗ FAIL (%s)\n" "$name" "$status" >&2
    fi
}

check_url "https://google.com" "Google"
check_url "https://api.myapp.com/health" "API"
check_url "https://myapp.com" "Website"
```

### API Client

```bash
#!/bin/bash
API_URL="https://api.example.com"
TOKEN="${API_TOKEN:?Set API_TOKEN environment variable}"

api_get() {
    curl -sf -H "Authorization: Bearer $TOKEN" "$API_URL/$1"
}

api_post() {
    local endpoint="$1" data="$2"
    curl -sf -X POST \
        -H "Authorization: Bearer $TOKEN" \
        -H "Content-Type: application/json" \
        -d "$data" \
        "$API_URL/$endpoint"
}

# Usage
users=$(api_get "users" | jq -r '.[].name')
api_post "users" '{"name":"Alice","role":"admin"}'
```

---

## Exercises

### Exercise 38.1: Site Monitor
Write a script that checks a list of URLs every 60 seconds and sends an alert (write to a file) if any return non-200 status.

### Exercise 38.2: Deployment Script
Write a script that uses SSH and rsync to deploy a project directory to a remote server, restart a service, and verify the deployment.

---

## Summary

- `ping`, `nc`, `ss` for connectivity testing and diagnostics
- `curl` is the Swiss Army knife for HTTP requests and API interactions
- `wget` excels at file downloads and website mirroring
- `ssh` for remote commands; `scp`/`rsync` for file transfer
- Bash's `/dev/tcp` provides basic networking without external tools
- Always use timeouts and retries for network operations in scripts

---

**Next Chapter:** [Chapter 39: System Monitoring →](Chapter39-System-Monitoring.md)
