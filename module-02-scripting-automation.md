# Module 02: Scripting & Automation

> Part of the [DevOps Career Course](./README.md) by UncleJS

---

## Table of Contents

- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [Beginner: Bash Scripting Basics](#beginner-bash-scripting-basics)
- [Beginner: Variables & Input](#beginner-variables--input)
- [Beginner: Conditionals](#beginner-conditionals)
- [Beginner: Loops](#beginner-loops)
- [Beginner: Functions](#beginner-functions)
- [Intermediate: Error Handling & Exit Codes](#intermediate-error-handling--exit-codes)
- [Intermediate: Text Processing with awk & sed](#intermediate-text-processing-with-awk--sed)
- [Intermediate: Cron Jobs & Scheduled Tasks](#intermediate-cron-jobs--scheduled-tasks)
- [Intermediate: Python for DevOps](#intermediate-python-for-devops)
- [Tools & Commands Reference](#tools--commands-reference)
- [Hands-On Labs](#hands-on-labs)
- [Further Reading](#further-reading)

---

## Overview

Scripting is the superpower that separates a manual operator from a DevOps engineer. Instead of running the same commands by hand every day, you write a script once and let it run automatically — forever.

This module covers Bash scripting from first principles, text processing with `awk` and `sed`, scheduled automation with `cron`, and an introduction to Python for DevOps use cases.

[↑ Back to TOC](#table-of-contents)

---

## Learning Objectives

By the end of this module you will be able to:

- Write Bash scripts with variables, conditionals, loops, and functions
- Handle errors gracefully and use exit codes correctly
- Process and transform text data using `awk` and `sed`
- Schedule recurring tasks using `cron`
- Write Python scripts for common DevOps tasks (file ops, API calls, parsing JSON)
- Automate repetitive operational tasks with confidence

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Bash Scripting Basics

Every Bash script starts with a **shebang** line that tells the system which interpreter to use.

```bash
#!/bin/bash

# This is a comment
echo "Hello, DevOps!"
```

### Making a Script Executable

```bash
chmod +x myscript.sh    # Add execute permission
./myscript.sh           # Run the script
bash myscript.sh        # Run without execute permission
```

### Script Best Practices

```bash
#!/bin/bash
set -e          # Exit immediately if any command fails
set -u          # Treat unset variables as errors
set -o pipefail # Catch errors in pipes

echo "Script starting..."
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Variables & Input

```bash
#!/bin/bash

# Assigning variables (no spaces around =)
NAME="Alice"
AGE=30
PI=3.14

# Using variables
echo "Hello, $NAME"
echo "You are ${AGE} years old"

# Command substitution — capture command output into a variable
TODAY=$(date +%Y-%m-%d)
HOSTNAME=$(hostname)
FILES=$(ls /etc/*.conf | wc -l)

echo "Today is $TODAY"
echo "This machine is $HOSTNAME"
echo "Config files: $FILES"

# Reading user input
read -p "Enter your name: " USERNAME
echo "Welcome, $USERNAME!"

# Special variables
echo "Script name: $0"
echo "First argument: $1"
echo "Second argument: $2"
echo "All arguments: $@"
echo "Number of arguments: $#"
echo "Last exit code: $?"
echo "Current PID: $$"
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Conditionals

```bash
#!/bin/bash

# if / elif / else
AGE=25
if [ $AGE -ge 18 ]; then
    echo "Adult"
elif [ $AGE -ge 13 ]; then
    echo "Teenager"
else
    echo "Child"
fi

# File tests
if [ -f "/etc/hosts" ]; then
    echo "File exists"
fi

if [ -d "/var/log" ]; then
    echo "Directory exists"
fi

if [ ! -f "/tmp/lockfile" ]; then
    echo "No lockfile, proceeding..."
fi

# String comparison
ENV="production"
if [ "$ENV" = "production" ]; then
    echo "Warning: Running in production!"
fi

# Combining conditions
if [ -f "/etc/nginx/nginx.conf" ] && [ -d "/var/log/nginx" ]; then
    echo "Nginx appears to be installed"
fi

# Common test operators
# -eq  equal (numbers)         -ne  not equal
# -lt  less than               -gt  greater than
# -le  less or equal           -ge  greater or equal
# -f   file exists             -d   directory exists
# -z   string is empty         -n   string is not empty
# =    strings equal           !=   strings not equal
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Loops

```bash
#!/bin/bash

# For loop — iterate over a list
for SERVER in web01 web02 web03; do
    echo "Pinging $SERVER..."
    ping -c 1 $SERVER > /dev/null && echo "$SERVER is up" || echo "$SERVER is down"
done

# For loop — iterate over files
for FILE in /var/log/*.log; do
    echo "Processing: $FILE"
    wc -l "$FILE"
done

# C-style for loop
for ((i=1; i<=5; i++)); do
    echo "Iteration $i"
done

# While loop
COUNTER=0
while [ $COUNTER -lt 5 ]; do
    echo "Counter: $COUNTER"
    ((COUNTER++))
done

# Read lines from a file
while IFS= read -r LINE; do
    echo "Host: $LINE"
done < servers.txt

# Loop with break and continue
for i in {1..10}; do
    if [ $i -eq 3 ]; then continue; fi   # Skip 3
    if [ $i -eq 7 ]; then break; fi      # Stop at 7
    echo $i
done
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Functions

```bash
#!/bin/bash

# Define a function
greet() {
    local NAME=$1   # 'local' keeps variable scoped to the function
    echo "Hello, $NAME!"
}

# Call the function
greet "Alice"
greet "Bob"

# Function with return value (via exit code)
is_port_open() {
    local HOST=$1
    local PORT=$2
    nc -z -w2 "$HOST" "$PORT" 2>/dev/null
    return $?   # 0 = success, non-zero = failure
}

if is_port_open "localhost" 80; then
    echo "Port 80 is open"
else
    echo "Port 80 is closed"
fi

# Function that returns a string
get_timestamp() {
    echo $(date "+%Y-%m-%d %H:%M:%S")
}

TIMESTAMP=$(get_timestamp)
echo "Started at: $TIMESTAMP"

# Logging function
log() {
    local LEVEL=$1
    local MSG=$2
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$LEVEL] $MSG"
}

log "INFO" "Script started"
log "ERROR" "Something went wrong"
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Error Handling & Exit Codes

```bash
#!/bin/bash
set -euo pipefail

# Exit codes: 0 = success, non-zero = failure
# Check exit code of last command
ls /tmp/
echo "Exit code: $?"   # 0 — success

ls /nonexistent/
echo "Exit code: $?"   # 2 — failure

# Trap errors and cleanup
cleanup() {
    echo "Cleaning up temporary files..."
    rm -f /tmp/myapp.lock
}

trap cleanup EXIT          # Run cleanup() on any exit
trap 'echo "Error on line $LINENO"' ERR  # Report line on error

# Create lockfile to prevent concurrent runs
LOCKFILE="/tmp/myapp.lock"
if [ -f "$LOCKFILE" ]; then
    echo "Script already running. Exiting."
    exit 1
fi
touch "$LOCKFILE"

# Retry logic
retry() {
    local ATTEMPTS=$1
    local DELAY=$2
    shift 2
    local CMD="$@"

    for ((i=1; i<=ATTEMPTS; i++)); do
        if $CMD; then
            return 0
        fi
        echo "Attempt $i/$ATTEMPTS failed. Retrying in ${DELAY}s..."
        sleep $DELAY
    done
    echo "All $ATTEMPTS attempts failed."
    return 1
}

retry 3 5 curl -f https://example.com/healthz
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Text Processing with awk & sed

### sed — Stream Editor

`sed` is used for substitution and in-place file editing.

```bash
# Basic substitution: replace first occurrence per line
sed 's/old/new/' file.txt

# Replace all occurrences per line (global flag)
sed 's/old/new/g' file.txt

# Edit file in place
sed -i 's/localhost/production-db/g' config.ini

# Delete lines matching a pattern
sed '/^#/d' config.ini          # Remove comment lines
sed '/^$/d' config.ini          # Remove empty lines

# Print only specific lines
sed -n '5,10p' file.txt         # Print lines 5–10
sed -n '/START/,/END/p' file.txt # Print between START and END markers

# Add a line after a match
sed '/\[database\]/a host = localhost' config.ini
```

### awk — Text Column Processor

`awk` processes text field by field — perfect for log parsing and reports.

```bash
# Print specific columns (fields are separated by whitespace by default)
awk '{print $1, $4}' access.log      # Print columns 1 and 4
awk '{print $NF}' file.txt           # Print last field
awk -F: '{print $1}' /etc/passwd     # Use : as delimiter, print username

# Filter and print
awk '$9 == "404" {print $1, $7}' access.log   # Show IPs with 404 errors

# Calculations
awk '{sum += $5} END {print "Total:", sum}' report.txt

# Count occurrences
awk '{counts[$9]++} END {for (code in counts) print code, counts[code]}' access.log

# Full log parsing example — top 10 IPs from nginx log
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head 10
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Cron Jobs & Scheduled Tasks

`cron` runs commands on a schedule. Edit your crontab with `crontab -e`.

### Crontab Format

```
# ┌──────── minute (0–59)
# │ ┌────── hour (0–23)
# │ │ ┌──── day of month (1–31)
# │ │ │ ┌── month (1–12)
# │ │ │ │ ┌ day of week (0–7, 0 and 7 = Sunday)
# │ │ │ │ │
# * * * * *   command to run
```

### Common Cron Patterns

```bash
# Edit crontab
crontab -e

# List current crontab
crontab -l

# Examples:
0 * * * *       /usr/local/bin/check-disk.sh         # Every hour
0 2 * * *       /usr/local/bin/backup.sh             # Daily at 2:00 AM
0 2 * * 0       /usr/local/bin/weekly-report.sh      # Every Sunday at 2:00 AM
*/5 * * * *     /usr/local/bin/health-check.sh       # Every 5 minutes
0 0 1 * *       /usr/local/bin/monthly-cleanup.sh    # 1st of every month
@reboot         /usr/local/bin/start-services.sh     # On system startup

# Redirect output to log file
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

# System-wide cron (requires root) — /etc/cron.d/myapp
0 3 * * *  www-data  /usr/local/bin/clean-cache.sh
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Python for DevOps

Python is widely used in DevOps for automation scripts, API integrations, and tooling. Here are the most relevant patterns.

### File Operations

```python
#!/usr/bin/env python3
import os
import shutil
from pathlib import Path

# Read a file
with open('/etc/hosts', 'r') as f:
    content = f.read()
    print(content)

# Write a file
with open('/tmp/output.txt', 'w') as f:
    f.write("Hello, DevOps!\n")

# Append to a file
with open('/tmp/output.txt', 'a') as f:
    f.write("Second line\n")

# List files in a directory
for path in Path('/var/log').glob('*.log'):
    print(path.name, path.stat().st_size)

# Create directories
os.makedirs('/tmp/myapp/data', exist_ok=True)

# Copy and move
shutil.copy('/tmp/source.txt', '/tmp/dest.txt')
shutil.move('/tmp/old.txt', '/tmp/new.txt')
```

### Running System Commands

```python
import subprocess

# Run a command and capture output
result = subprocess.run(['ls', '-la', '/tmp'], capture_output=True, text=True)
print(result.stdout)
print(result.returncode)  # 0 = success

# Run with shell=True (use carefully)
result = subprocess.run('df -h | grep /dev/sda', shell=True, capture_output=True, text=True)
print(result.stdout)

# Run and raise exception on failure
try:
    subprocess.run(['systemctl', 'start', 'nginx'], check=True)
    print("nginx started successfully")
except subprocess.CalledProcessError as e:
    print(f"Failed to start nginx: {e}")
```

### Working with JSON & APIs

```python
import json
import urllib.request

# Parse JSON
data = '{"name": "web01", "status": "healthy"}'
parsed = json.loads(data)
print(parsed['name'])

# Load JSON from file
with open('config.json') as f:
    config = json.load(f)

# Write JSON to file
config = {"env": "production", "replicas": 3}
with open('config.json', 'w') as f:
    json.dump(config, f, indent=2)

# HTTP GET request (built-in, no extra libraries)
with urllib.request.urlopen('https://httpbin.org/get') as response:
    data = json.loads(response.read())
    print(data['origin'])

# Using the requests library (install: pip install requests)
import requests

response = requests.get('https://api.github.com/repos/kubernetes/kubernetes')
data = response.json()
print(f"Stars: {data['stargazers_count']}")

# POST request with JSON body
payload = {'key': 'value'}
response = requests.post('https://httpbin.org/post', json=payload)
print(response.status_code)
```

### Environment Variables

```python
import os

# Read environment variable
db_host = os.getenv('DATABASE_HOST', 'localhost')  # with default
api_key = os.environ['API_KEY']  # raises KeyError if not set

# Check if variable is set
if 'DEBUG' in os.environ:
    print("Debug mode enabled")
```

[↑ Back to TOC](#table-of-contents)

---

## Tools & Commands Reference

| Tool/Command | Purpose |
|---|---|
| `#!/bin/bash` | Shebang — declare Bash as interpreter |
| `set -euo pipefail` | Strict error handling in scripts |
| `chmod +x` | Make script executable |
| `$1`, `$@`, `$#` | Script arguments |
| `$?` | Last command exit code |
| `$(...)` | Command substitution |
| `trap` | Run commands on script exit or error |
| `sed 's/old/new/g'` | Global text substitution |
| `awk '{print $1}'` | Print first column of each line |
| `crontab -e` | Edit scheduled jobs |
| `crontab -l` | List scheduled jobs |
| `python3 script.py` | Run a Python script |
| `subprocess.run()` | Run system commands from Python |
| `json.loads()` / `json.dumps()` | Parse/serialize JSON in Python |

[↑ Back to TOC](#table-of-contents)

---

## Hands-On Labs

### Lab 2.1 — Your First Bash Script

1. Create a file called `system-info.sh`
2. Write a script that prints: hostname, current user, date, disk usage, and memory usage
3. Make it executable and run it
4. Add a `log()` function that prefixes output with a timestamp

### Lab 2.2 — Disk Alert Script

Write a script called `disk-alert.sh` that:
1. Checks disk usage on `/` using `df`
2. If usage exceeds 80%, prints a warning message
3. If usage exceeds 90%, prints a critical message and exits with code `2`
4. Otherwise prints "Disk usage is healthy"
5. Schedule it to run every 5 minutes using `cron`

### Lab 2.3 — Log Parser with awk & sed

1. Create a sample log file with mixed content including ERROR and INFO lines
2. Use `grep` and `awk` to extract only ERROR lines and print the timestamp and message
3. Use `sed` to replace all IP addresses (pattern `[0-9.]+`) with `REDACTED`
4. Count how many unique error types appear

### Lab 2.4 — Python Health Check Script

Write `healthcheck.py` that:
1. Takes a list of URLs from a JSON config file
2. Makes an HTTP GET request to each URL
3. Prints `OK` or `FAIL` with the status code
4. Writes results to a JSON report file with timestamps

### Lab 2.5 — Backup Automation Script

Write `backup.sh` that:
1. Takes a source directory as argument `$1` and backup destination as `$2`
2. Creates a timestamped `.tar.gz` archive
3. Removes archives older than 7 days
4. Logs each action with timestamp to `/var/log/backup.log`
5. Exits with code `1` on any failure

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

- [Bash Manual (GNU)](https://www.gnu.org/software/bash/manual/)
- [ShellCheck](https://www.shellcheck.net/) — Online Bash script linter
- [Python for DevOps (O'Reilly)](https://www.oreilly.com/library/view/python-for-devops/9781492057680/)
- [Crontab Guru](https://crontab.guru/) — Visual cron expression builder
- [awk Tutorial](https://www.grymoire.com/Unix/Awk.html)
- [Glossary: Bash](./glossary.md#b), [Cron](./glossary.md#c), [Environment Variable](./glossary.md#e)

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)*
