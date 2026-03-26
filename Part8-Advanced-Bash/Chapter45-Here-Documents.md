# Chapter 45: Advanced Here Documents and Strings

## Learning Objectives

By the end of this chapter, you will be able to:

- Use here documents for multi-line text and templates
- Apply here strings for concise input
- Generate configuration files and scripts dynamically
- Use here documents with indentation and variable control
- Build template engines with here documents

---

## 45.1 Here Document Syntax

```bash
# Basic here document
cat <<EOF
Hello, World!
This is a here document.
Line 3.
EOF

# Delimiter can be any word (EOF is conventional)
cat <<END_OF_MESSAGE
This uses END_OF_MESSAGE as delimiter.
END_OF_MESSAGE
```

---

## 45.2 Variable Expansion Control

```bash
name="Alice"

# WITH expansion (unquoted delimiter)
cat <<EOF
Hello, $name!
Today is $(date +%Y-%m-%d).
Home: $HOME
EOF
# Hello, Alice! Today is 2024-01-15. Home: /home/alice

# WITHOUT expansion (quoted delimiter)
cat <<'EOF'
Hello, $name!
Today is $(date +%Y-%m-%d).
These are literal dollar signs.
EOF
# Hello, $name! Today is $(date +%Y-%m-%d). ...

# Single variable expansion (selective quoting)
cat <<EOF
Hello, $name!
To use variables in scripts, write \$variable_name.
The cost is \$9.99.
EOF
# Hello, Alice! To use variables in scripts, write $variable_name. ...
```

---

## 45.3 Indented Here Documents (<<-)

`<<-` strips leading **tabs** (not spaces):

```bash
generate_config() {
	cat <<-EOF
	server {
	    listen 80;
	    server_name $1;
	    root /var/www/$1;
	}
	EOF
}
# Output has no leading indentation (tabs stripped)
```

> **Note:** You must use actual tab characters, not spaces. Many editors convert tabs to spaces — check your editor settings.

---

## 45.4 Redirecting Here Documents

```bash
# Write to a file
cat <<EOF > /tmp/config.conf
database_host = localhost
database_port = 5432
database_name = myapp
EOF

# Append to a file
cat <<EOF >> /tmp/config.conf
log_level = info
EOF

# Pipe to a command
mysql -u root <<EOF
CREATE DATABASE IF NOT EXISTS myapp;
GRANT ALL ON myapp.* TO 'appuser'@'localhost';
EOF

# Feed to a while loop
while read -r name age; do
    echo "$name is $age years old"
done <<EOF
Alice 30
Bob 25
Charlie 35
EOF
```

---

## 45.5 Here Strings

A single-line alternative to here documents:

```bash
# Here string
read -r first rest <<< "Hello World Bash"
echo "$first"    # Hello
echo "$rest"     # World Bash

# Feed to commands
grep "pattern" <<< "$variable"
bc <<< "scale=2; 22/7"         # 3.14
tr '[:lower:]' '[:upper:]' <<< "hello"    # HELLO

# With variable
message="Error: file not found"
awk '{print $2}' <<< "$message"    # file
```

---

## 45.6 Generating Scripts and Config Files

```bash
#!/bin/bash
# Generate a deployment script

server="$1"
app_name="$2"
deploy_dir="/opt/$app_name"

cat <<SCRIPT > deploy.sh
#!/bin/bash
# Auto-generated deploy script
# Generated: $(date)
# Target: $server

set -euo pipefail

echo "Deploying $app_name to $server..."
ssh $server "mkdir -p $deploy_dir"
rsync -avz ./dist/ $server:$deploy_dir/
ssh $server "systemctl restart $app_name"
echo "Deployment complete."
SCRIPT

chmod +x deploy.sh
echo "Created deploy.sh"
```

### Generate Systemd Unit File

```bash
create_service_file() {
    local name="$1" command="$2" user="${3:-root}"
    
    cat <<EOF > "/etc/systemd/system/${name}.service"
[Unit]
Description=$name service
After=network.target

[Service]
Type=simple
User=$user
ExecStart=$command
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
    
    systemctl daemon-reload
    echo "Created service: $name"
}

create_service_file "myapp" "/opt/myapp/bin/server" "appuser"
```

---

## 45.7 Embedded Documentation

```bash
#!/bin/bash
# Self-documenting script

show_help() {
    cat <<'HELP'
NAME
    backup.sh - Automated backup utility

SYNOPSIS
    backup.sh [OPTIONS] SOURCE DESTINATION

DESCRIPTION
    Creates compressed backups of the source directory
    to the destination.

OPTIONS
    -h          Show this help
    -v          Verbose output
    -n          Dry run
    -c METHOD   Compression: gzip (default), bzip2, xz
    -e PATTERN  Exclude pattern

EXAMPLES
    backup.sh /home/user /backup/
    backup.sh -c xz -v /data /mnt/backup/
    backup.sh -e '*.tmp' -e '*.log' /app /backup/

AUTHOR
    Written by Your Name.
HELP
}
```

---

## 45.8 Advanced Patterns

### Multi-line Variable Assignment

```bash
# Assign multi-line text to a variable
read -r -d '' SQL_QUERY <<'EOF' || true
SELECT u.name, u.email, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.active = true
GROUP BY u.id
ORDER BY order_count DESC
LIMIT 10;
EOF

echo "$SQL_QUERY" | mysql -u root mydb
```

> **Note:** `read -d ''` returns exit code 1 when it hits EOF, so `|| true` prevents `set -e` from killing the script.

### Nested Here Documents

```bash
# Here document inside a remote SSH session
ssh user@remote <<'OUTER'
cat <<'INNER' > /tmp/script.sh
#!/bin/bash
echo "This script was deployed remotely"
hostname
date
INNER
chmod +x /tmp/script.sh
/tmp/script.sh
OUTER
```

---

## Exercises

### Exercise 45.1: Config Generator
Write a script that generates an nginx virtual host configuration file using a here document. Accept domain name, root directory, and port as arguments.

### Exercise 45.2: Email Template
Write a function that generates an email body from a here document template with variables for recipient name, subject, and custom message body.

---

## Summary

- `<<EOF ... EOF` creates multi-line inline text (here document)
- `<<'EOF'` prevents variable/command expansion (literal text)
- `<<-EOF` strips leading tabs (for indented code)
- `<<<` is a here string — single-line input to commands
- Here documents can redirect to files, pipes, and commands
- Use here documents for: config generation, script generation, SQL queries, documentation
- `read -r -d '' var <<'EOF' || true` assigns multi-line text to a variable

---

**Next Chapter:** [Chapter 46: Subshells and Command Grouping →](Chapter46-Subshells.md)
