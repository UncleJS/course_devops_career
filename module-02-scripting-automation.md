# Module 02: Scripting & Automation

> Part of the [DevOps Career Course](./README.md) by UncleJS

[![CC BY-NC-SA 4.0](https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/) ![Module 02 of 15](https://img.shields.io/badge/module-02%20of%2015-grey) ![Level](https://img.shields.io/badge/level-Beginner%20%E2%86%92%20Intermediate-yellow) ![bash 5.2+](https://img.shields.io/badge/bash-5.2%2B-4EAA25?logo=gnubash&logoColor=white) ![Python 3.11+](https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white) ![Linux · macOS](https://img.shields.io/badge/platform-Linux%20%C2%B7%20macOS-lightgrey?logo=linux&logoColor=white)

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
- [Intermediate: JSON Processing with jq](#intermediate-json-processing-with-jq)
- [Intermediate: Cron Jobs & Scheduled Tasks](#intermediate-cron-jobs--scheduled-tasks)
- [Intermediate: Python for DevOps](#intermediate-python-for-devops)
- [Advanced: Production-Grade Bash Patterns](#advanced-production-grade-bash-patterns)
- [Advanced: Scripting for Infrastructure](#advanced-scripting-for-infrastructure)
- [Tools & Commands Reference](#tools--commands-reference)
- [Hands-On Labs](#hands-on-labs)
- [Further Reading](#further-reading)

---

## Overview

Scripting is the superpower that separates a manual operator from a DevOps engineer. Instead of running the same commands by hand every day, you write a script once and let it run automatically — forever.

This module covers Bash scripting from first principles, text processing with `awk` and `sed`, JSON processing with `jq`, scheduled automation with `cron`, Python for DevOps use cases, and production-grade scripting patterns including logging, secrets management, and idempotent execution.

[↑ Back to TOC](#table-of-contents)

---

## Learning Objectives

By the end of this module you will be able to:

- Write Bash scripts with variables, conditionals, loops, and functions
- Handle errors gracefully and use exit codes correctly
- Process and transform text data using `awk` and `sed`
- Parse, query, and transform JSON data using `jq`
- Schedule recurring tasks using `cron`
- Write Python scripts for common DevOps tasks (file ops, API calls, parsing JSON)
- Write production-grade Bash scripts with structured logging, locking, and retries
- Automate infrastructure tasks: health checks, deployment scripts, backup automation

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
set -o pipefail # Catch errors in pipes (e.g., false | true returns 1)

# Combined shorthand
set -euo pipefail

echo "Script starting..."

# Debug mode — print every command before running it
set -x          # Enable
set +x          # Disable
bash -x myscript.sh   # Run with debug output without modifying the script
```

### Script Structure Template

```bash
#!/bin/bash
set -euo pipefail

# ─── Constants ────────────────────────────────────────────
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "$0")"
readonly LOG_FILE="/var/log/${SCRIPT_NAME%.sh}.log"

# ─── Logging ──────────────────────────────────────────────
log()  { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [INFO]  $*" | tee -a "$LOG_FILE"; }
warn() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [WARN]  $*" | tee -a "$LOG_FILE" >&2; }
err()  { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [ERROR] $*" | tee -a "$LOG_FILE" >&2; }
die()  { err "$*"; exit 1; }

# ─── Cleanup ──────────────────────────────────────────────
cleanup() {
    local exit_code=$?
    log "Script exiting with code $exit_code"
    rm -f /tmp/${SCRIPT_NAME}.lock
}
trap cleanup EXIT

# ─── Main ─────────────────────────────────────────────────
main() {
    log "Script started"
    # ... your logic here ...
    log "Script completed"
}

main "$@"
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

# Curly braces for disambiguation
FILE="report"
echo "${FILE}_final.pdf"     # report_final.pdf
echo "$FILE_final.pdf"       # ERROR: looks for $FILE_final

# Command substitution — capture command output into a variable
TODAY=$(date +%Y-%m-%d)
HOSTNAME=$(hostname)
FILES=$(ls /etc/*.conf | wc -l)

echo "Today is $TODAY"
echo "This machine is $HOSTNAME"
echo "Config files: $FILES"

# Default values
DB_HOST="${DATABASE_HOST:-localhost}"     # Use localhost if not set
DB_PORT="${DATABASE_PORT:-5432}"          # Default to 5432
: "${API_KEY:?ERROR: API_KEY must be set}"  # Abort if not set

# Reading user input
read -p "Enter your name: " USERNAME
read -sp "Enter password: " PASSWORD      # -s = silent (no echo)
echo ""                                   # newline after silent input
echo "Welcome, $USERNAME!"

# Special variables
echo "Script name: $0"
echo "First argument: $1"
echo "Second argument: $2"
echo "All arguments: $@"
echo "All arguments (as one): $*"
echo "Number of arguments: $#"
echo "Last exit code: $?"
echo "Current PID: $$"
echo "Background job PID: $!"
```

### Arrays

```bash
# Indexed arrays
SERVERS=("web01" "web02" "web03")
echo "${SERVERS[0]}"          # web01
echo "${SERVERS[@]}"          # All elements
echo "${#SERVERS[@]}"         # Number of elements

# Append to array
SERVERS+=("web04")

# Iterate
for SERVER in "${SERVERS[@]}"; do
    echo "Checking $SERVER..."
done

# Slice
echo "${SERVERS[@]:1:2}"      # Elements 1 and 2 (web02 web03)

# Associative arrays (Bash 4+)
declare -A CONFIG
CONFIG["host"]="localhost"
CONFIG["port"]="5432"
echo "${CONFIG["host"]}"
for key in "${!CONFIG[@]}"; do
    echo "$key = ${CONFIG[$key]}"
done
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

# [[ ]] — modern conditional (preferred, more features)
NAME="Alice"
if [[ "$NAME" == A* ]]; then    # Pattern matching with *
    echo "Name starts with A"
fi

if [[ "$NAME" =~ ^[A-Z][a-z]+$ ]]; then  # Regex match
    echo "Name looks valid"
fi

# File tests
if [ -f "/etc/hosts" ]; then
    echo "File exists"
fi

if [ -d "/var/log" ]; then
    echo "Directory exists"
fi

if [ -r "/etc/hosts" ]; then echo "Readable"; fi
if [ -w "/tmp/test" ]; then echo "Writable"; fi
if [ -x "/usr/bin/curl" ]; then echo "Executable"; fi
if [ -s "/var/log/syslog" ]; then echo "Non-empty file"; fi
if [ -L "/etc/localtime" ]; then echo "Is a symlink"; fi

# String comparison
ENV="production"
if [ "$ENV" = "production" ]; then
    echo "Warning: Running in production!"
fi

# Combining conditions
if [ -f "/etc/nginx/nginx.conf" ] && [ -d "/var/log/nginx" ]; then
    echo "Nginx appears to be installed"
fi

if [[ "$ENV" == "prod" ]] || [[ "$ENV" == "production" ]]; then
    echo "This is prod"
fi

# case statement — cleaner than multiple if/elif
case "$ENV" in
    production|prod)
        echo "Production environment"
        ;;
    staging|stage)
        echo "Staging environment"
        ;;
    development|dev|local)
        echo "Development environment"
        ;;
    *)
        echo "Unknown environment: $ENV"
        exit 1
        ;;
esac

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
    echo "Processing: $FILE ($(wc -l < "$FILE") lines)"
done

# For loop — iterate over array
SERVICES=("nginx" "mysql" "redis")
for SERVICE in "${SERVICES[@]}"; do
    systemctl is-active --quiet "$SERVICE" \
        && echo "$SERVICE: running" \
        || echo "$SERVICE: stopped"
done

# C-style for loop
for ((i=1; i<=5; i++)); do
    echo "Iteration $i"
done

# Range
for i in {1..10}; do echo $i; done
for i in {0..100..10}; do echo $i; done  # Step by 10

# While loop
COUNTER=0
while [ $COUNTER -lt 5 ]; do
    echo "Counter: $COUNTER"
    ((COUNTER++))
done

# Wait for a service to be ready
wait_for_port() {
    local host=$1 port=$2 timeout=${3:-30}
    local start=$(date +%s)
    until nc -z "$host" "$port" 2>/dev/null; do
        [ $(( $(date +%s) - start )) -ge $timeout ] && return 1
        sleep 1
    done
    return 0
}
wait_for_port localhost 5432 60 && echo "DB ready" || echo "DB timeout"

# Read lines from a file
while IFS= read -r LINE; do
    echo "Host: $LINE"
done < servers.txt

# Read lines from command output
while IFS= read -r CONTAINER; do
    echo "Container: $CONTAINER"
done < <(docker ps --format '{{.Names}}')

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

# Function that returns a string (via stdout)
get_timestamp() {
    echo "$(date "+%Y-%m-%d %H:%M:%S")"
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

# Function with named parameters (via local + shift)
deploy_app() {
    local APP_NAME=$1
    local VERSION=$2
    local ENV=${3:-staging}   # Default to staging

    log "INFO" "Deploying $APP_NAME v$VERSION to $ENV"
    # ... deployment logic ...
}

deploy_app "myapi" "1.2.3"
deploy_app "frontend" "2.0.0" "production"

# Validate required arguments
require_args() {
    local count=$1
    shift
    if [ $# -lt $count ]; then
        echo "Usage: $(basename $0) arg1 arg2 ..."
        exit 1
    fi
}
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

ls /nonexistent/ 2>/dev/null || echo "Directory not found (exit $?)"

# Trap errors and cleanup
LOCKFILE="/tmp/myapp.lock"

cleanup() {
    local exit_code=$?
    [ -f "$LOCKFILE" ] && rm -f "$LOCKFILE"
    [ $exit_code -ne 0 ] && echo "Script failed with exit code $exit_code" >&2
}

error_handler() {
    local line_number=$1
    echo "ERROR: Script failed at line $line_number" >&2
}

trap cleanup EXIT
trap 'error_handler $LINENO' ERR

# Create lockfile to prevent concurrent runs
if [ -f "$LOCKFILE" ]; then
    echo "Script already running (PID $(cat $LOCKFILE)). Exiting." >&2
    exit 1
fi
echo $$ > "$LOCKFILE"

# Retry logic with exponential backoff
retry() {
    local attempts=$1
    local delay=$2
    shift 2
    local cmd=("$@")

    for ((i=1; i<=attempts; i++)); do
        if "${cmd[@]}"; then
            return 0
        fi
        if [ $i -lt $attempts ]; then
            echo "Attempt $i/$attempts failed. Retrying in ${delay}s..." >&2
            sleep $delay
            delay=$((delay * 2))   # Exponential backoff
        fi
    done
    echo "All $attempts attempts failed for: ${cmd[*]}" >&2
    return 1
}

retry 3 5 curl -sf https://example.com/healthz
retry 5 2 kubectl rollout status deployment/myapp

# Graceful error handling without set -e
if ! command -v kubectl &>/dev/null; then
    echo "kubectl not found — please install it first" >&2
    exit 127  # Standard "command not found" exit code
fi

# Check command exists
require_command() {
    command -v "$1" &>/dev/null || die "Required command '$1' not found"
}

require_command docker
require_command jq
require_command curl
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

# Edit in place with backup
sed -i.bak 's/localhost/production-db/g' config.ini  # Creates config.ini.bak

# Delete lines matching a pattern
sed '/^#/d' config.ini          # Remove comment lines
sed '/^$/d' config.ini          # Remove empty lines
sed '/^#/d; /^$/d' config.ini   # Remove both in one command

# Print only specific lines
sed -n '5,10p' file.txt         # Print lines 5–10
sed -n '/START/,/END/p' file.txt # Print between START and END markers

# Add a line after a match
sed '/\[database\]/a host = localhost' config.ini

# Add a line before a match
sed '/^server {/i # Auto-generated by deploy script' nginx.conf

# Replace a whole line matching a pattern
sed 's/^PORT=.*/PORT=8080/' .env

# Multi-line editing with -e
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt

# Practical DevOps examples
# Update version in a config file
sed -i "s/IMAGE_TAG=.*/IMAGE_TAG=$NEW_TAG/" .env

# Comment out a line
sed -i 's/^MaxClients/#MaxClients/' /etc/apache2/apache2.conf

# Uncomment a line
sed -i 's/^#MaxClients/MaxClients/' /etc/apache2/apache2.conf
```

### awk — Text Column Processor

`awk` processes text field by field — perfect for log parsing and reports.

```bash
# Print specific columns (fields are separated by whitespace by default)
awk '{print $1, $4}' access.log      # Print columns 1 and 4
awk '{print $NF}' file.txt           # Print last field
awk -F: '{print $1}' /etc/passwd     # Use : as delimiter, print username
awk -F, '{print $2}' data.csv        # CSV: print second column

# Filter and print
awk '$9 == "404" {print $1, $7}' access.log   # Show IPs with 404 errors
awk '$5 > 1000000 {print $9}' access.log      # Large requests (> 1 MB)
awk '/ERROR/ {print NR": "$0}' app.log        # Print ERROR lines with line numbers

# BEGIN and END blocks
awk 'BEGIN {print "Report:"} {sum += $5} END {print "Total bytes:", sum}' access.log

# Calculations
awk '{sum += $5} END {printf "Average: %.2f\n", sum/NR}' report.txt

# Count occurrences
awk '{counts[$9]++} END {for (code in counts) print code, counts[code]}' access.log

# Full log parsing example — top 10 IPs from nginx log
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head 10

# Parse nginx access log into a readable report
awk '
BEGIN { print "Status\tCount" }
{ status[$9]++ }
END {
    for (s in status) {
        printf "%s\t%d\n", s, status[s]
    }
}
' /var/log/nginx/access.log | sort -k2 -rn

# Extract specific fields from a structured file
awk -F= '/^(HOST|PORT|USER)/ {print $1"="$2}' config.ini

# Process only specific lines
awk 'NR>=10 && NR<=20' file.txt      # Lines 10 to 20
awk 'NF > 0' file.txt                # Skip empty lines
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: JSON Processing with jq

`jq` is the essential command-line JSON processor for DevOps. APIs, Kubernetes, Terraform, AWS CLI — all output JSON. `jq` lets you query, filter, and transform it like `awk` does for text.

### Installation

```bash
sudo apt install jq      # Ubuntu/Debian
sudo dnf install jq      # RHEL/Fedora
brew install jq          # macOS
```

### Basic Filtering

```bash
# Input JSON: {"name": "web01", "status": "healthy", "cpu": 42.5}

# Pretty-print JSON
echo '{"a":1}' | jq '.'

# Extract a field
echo '{"name":"web01","status":"healthy"}' | jq '.name'
# "web01"

# Extract nested field
echo '{"server":{"name":"web01","ip":"1.2.3.4"}}' | jq '.server.ip'
# "1.2.3.4"

# Extract without quotes (-r = raw output)
echo '{"name":"web01"}' | jq -r '.name'
# web01  (no quotes — use this when piping to other commands)

# Extract from a file
jq '.name' server.json

# Extract from curl output
curl -s https://api.github.com/repos/kubernetes/kubernetes | jq '.stargazers_count'
```

### Arrays

```bash
# Input: [{"name":"web01"},{"name":"web02"},{"name":"web03"}]

# Get all elements
jq '.[]' servers.json

# Get first element
jq '.[0]' servers.json

# Get a field from every element
jq '.[].name' servers.json
# "web01"
# "web02"
# "web03"

# Raw output of array field
jq -r '.[].name' servers.json
# web01
# web02
# web03

# Array length
jq 'length' servers.json

# Slice an array
jq '.[1:3]' servers.json         # Elements 1 and 2
jq '.[-1]' servers.json          # Last element
```

### Filtering & Selecting

```bash
# Input: [{"name":"web01","status":"healthy"},{"name":"web02","status":"down"}]

# Filter array by condition
jq '.[] | select(.status == "healthy")' servers.json

# Extract field after filtering
jq '.[] | select(.status == "down") | .name' servers.json
# "web02"

# Multiple conditions
jq '.[] | select(.status == "healthy" and .cpu > 80)' servers.json

# String contains
jq '.[] | select(.name | contains("web"))' servers.json
```

### Transforming & Building New JSON

```bash
# Create a new object from selected fields
jq '.[] | {name: .name, up: (.status == "healthy")}' servers.json
# {"name":"web01","up":true}
# {"name":"web02","up":false}

# Build an array of transformed objects
jq '[.[] | {name: .name, healthy: (.status == "healthy")}]' servers.json

# Add a field
jq '.[] | . + {checked_at: "2026-01-01"}' servers.json

# Delete a field
jq 'del(.password)' user.json

# Rename a field
jq '.[] | {server_name: .name, health: .status}' servers.json
```

### String Interpolation & Formatting

```bash
# String interpolation inside jq
jq -r '.[] | "Server \(.name) is \(.status)"' servers.json
# Server web01 is healthy
# Server web02 is down

# Format as CSV
jq -r '.[] | [.name, .status, (.cpu|tostring)] | @csv' servers.json
# "web01","healthy","42.5"
# "web02","down","0"

# Format as TSV
jq -r '.[] | [.name, .status] | @tsv' servers.json
```

### Real-World DevOps Examples

```bash
# Get all running Kubernetes pod names
kubectl get pods -o json | jq -r '.items[].metadata.name'

# Get pods with their status
kubectl get pods -o json | jq -r '.items[] | "\(.metadata.name)\t\(.status.phase)"'

# Find all pods NOT running
kubectl get pods -o json | jq -r '.items[] | select(.status.phase != "Running") | .metadata.name'

# AWS CLI — list all EC2 instance IDs and their state
aws ec2 describe-instances | jq -r '.Reservations[].Instances[] | "\(.InstanceId) \(.State.Name)"'

# AWS — find running instances only
aws ec2 describe-instances | \
  jq -r '.Reservations[].Instances[] | select(.State.Name == "running") | .InstanceId'

# Docker — list container names and image versions
docker inspect $(docker ps -q) | jq -r '.[] | "\(.Name) \(.Config.Image)"'

# GitHub API — list open PRs
curl -s "https://api.github.com/repos/owner/repo/pulls?state=open" | \
  jq -r '.[] | "\(.number)\t\(.title)\t\(.user.login)"'

# Parse Terraform output JSON
terraform output -json | jq -r '.db_endpoint.value'

# Process a JSON config file
jq '.database.host = "prod-db.example.com"' config.json > config.tmp && mv config.tmp config.json

# Combine multiple jq filters with pipes
curl -s https://api.example.com/services | \
  jq -r '[.[] | select(.healthy == true)] | length | "Healthy services: \(.)"'
```

### jq Variables and Advanced Features

```bash
# Store intermediate result in a variable
jq --arg env "production" '.[] | select(.environment == $env)' services.json

# Pass a value from shell into jq
TAG="v1.2.3"
jq --arg tag "$TAG" '.image = $tag' deployment.json

# Pass a JSON value
jq --argjson replicas 3 '.spec.replicas = $replicas' deployment.json

# Read a file into a variable
jq --slurpfile config config.json '.tag = $config[0].tag' deployment.json

# Multiple outputs separated by newlines → one-per-line processing
jq -c '.[]' servers.json | while read -r server; do
    name=$(echo "$server" | jq -r '.name')
    status=$(echo "$server" | jq -r '.status')
    echo "Checking $name: $status"
done
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
@daily          /usr/local/bin/backup.sh             # Same as 0 0 * * *
@weekly         /usr/local/bin/report.sh

# Redirect output to log file
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

# System-wide cron (requires root) — /etc/cron.d/myapp
0 3 * * *  www-data  /usr/local/bin/clean-cache.sh
```

### systemd Timers (Modern Alternative to Cron)

systemd timers are more powerful than cron: they integrate with `journalctl`, support monotonic (since-last-boot) schedules, and can be triggered on events.

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Database Backup

[Service]
Type=oneshot
User=backup
ExecStart=/usr/local/bin/backup.sh
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run database backup daily at 2 AM

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true        # Run immediately if missed (e.g. system was off)
RandomizedDelaySec=300 # Spread load: start within 5 min of trigger

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now backup.timer
systemctl list-timers              # Show all timers and next run times
journalctl -u backup.service       # View logs for the service
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

### YAML Processing (Common in DevOps)

```python
# pip install pyyaml
import yaml

# Read a YAML file (e.g., Kubernetes manifest, Compose file)
with open('deployment.yaml') as f:
    manifest = yaml.safe_load(f)

print(manifest['spec']['replicas'])
manifest['spec']['replicas'] = 5

# Write back to YAML
with open('deployment.yaml', 'w') as f:
    yaml.dump(manifest, f, default_flow_style=False)
```

[↑ Back to TOC](#table-of-contents)

---

## Advanced: Production-Grade Bash Patterns

### Structured Logging

```bash
#!/bin/bash
# Colored, leveled logging with timestamps

# Color codes
readonly RED='\033[0;31m'
readonly YELLOW='\033[1;33m'
readonly GREEN='\033[0;32m'
readonly BLUE='\033[0;34m'
readonly NC='\033[0m'  # No Color

LOG_FILE="${LOG_FILE:-/var/log/myapp/deploy.log}"
LOG_LEVEL="${LOG_LEVEL:-INFO}"  # DEBUG, INFO, WARN, ERROR

_log() {
    local level=$1 color=$2
    shift 2
    local msg="[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $*"
    # Write to log file (no color)
    echo "$msg" >> "$LOG_FILE" 2>/dev/null || true
    # Write to terminal (with color)
    echo -e "${color}${msg}${NC}"
}

log_debug() { [[ "$LOG_LEVEL" == "DEBUG" ]] && _log "DEBUG" "$BLUE" "$@" || true; }
log_info()  { _log "INFO " "$GREEN" "$@"; }
log_warn()  { _log "WARN " "$YELLOW" "$@" >&2; }
log_error() { _log "ERROR" "$RED" "$@" >&2; }
die()       { log_error "$@"; exit 1; }

# Usage
log_info "Starting deployment of $APP v$VERSION"
log_warn "High memory usage detected: ${MEMORY_PERCENT}%"
log_error "Failed to connect to database"
```

### Idempotent Scripts

An idempotent script can be run multiple times with the same result — safe to re-run if interrupted.

```bash
#!/bin/bash
set -euo pipefail

# Idempotent directory creation
ensure_dir() {
    local dir=$1
    local owner=${2:-root:root}
    local perms=${3:-755}
    if [ ! -d "$dir" ]; then
        mkdir -p "$dir"
        chown "$owner" "$dir"
        chmod "$perms" "$dir"
        log_info "Created directory: $dir"
    else
        log_debug "Directory already exists: $dir"
    fi
}

# Idempotent user creation
ensure_user() {
    local username=$1
    if ! id "$username" &>/dev/null; then
        useradd -r -m -s /bin/false "$username"
        log_info "Created user: $username"
    else
        log_debug "User already exists: $username"
    fi
}

# Idempotent package installation
ensure_package() {
    local pkg=$1
    if ! dpkg -l "$pkg" 2>/dev/null | grep -q "^ii"; then
        apt-get install -y "$pkg"
        log_info "Installed package: $pkg"
    else
        log_debug "Package already installed: $pkg"
    fi
}

# Idempotent file line insertion
ensure_line_in_file() {
    local line=$1 file=$2
    grep -qF "$line" "$file" 2>/dev/null || echo "$line" >> "$file"
}

# Usage
ensure_dir /opt/myapp myapp:myapp 750
ensure_user myapp
ensure_package nginx
ensure_line_in_file "vm.swappiness=10" /etc/sysctl.conf
```

### Secrets Handling

```bash
#!/bin/bash
# Never hard-code secrets — load from external sources

# Option 1: Environment variables (set by CI/CD or system)
DB_PASSWORD="${DB_PASSWORD:?ERROR: DB_PASSWORD not set}"

# Option 2: Files (e.g., Kubernetes secrets mounted as files)
load_secret_file() {
    local file=$1
    if [ ! -f "$file" ]; then
        die "Secret file not found: $file"
    fi
    if [ "$(stat -c %a "$file")" != "600" ]; then
        die "Secret file has wrong permissions: $file (expected 600)"
    fi
    cat "$file"
}
DB_PASSWORD=$(load_secret_file /run/secrets/db-password)

# Option 3: Vault CLI (HashiCorp Vault)
DB_PASSWORD=$(vault kv get -field=password secret/myapp/database)

# Option 4: AWS Secrets Manager
DB_PASSWORD=$(aws secretsmanager get-secret-value \
    --secret-id "myapp/db-password" \
    --query SecretString --output text | jq -r .password)

# Mask secrets in debug output
echo "Connecting to DB as ${DB_USER} ..." # OK
echo "Password: ${DB_PASSWORD}"           # NEVER DO THIS
```

### Parallel Execution

```bash
#!/bin/bash
# Run tasks in parallel with controlled concurrency

MAX_PARALLEL=5
SERVERS=("web01" "web02" "web03" "web04" "web05" "web06" "web07" "web08")

check_server() {
    local server=$1
    if ssh -o ConnectTimeout=5 "$server" "uptime" &>/dev/null; then
        echo "$server: OK"
    else
        echo "$server: FAILED" >&2
        return 1
    fi
}
export -f check_server

# Option 1: parallel (GNU Parallel)
printf '%s\n' "${SERVERS[@]}" | parallel -j "$MAX_PARALLEL" check_server {}

# Option 2: xargs -P
printf '%s\n' "${SERVERS[@]}" | xargs -P "$MAX_PARALLEL" -I{} bash -c 'check_server "$@"' _ {}

# Option 3: manual background jobs with limit
active=0
for server in "${SERVERS[@]}"; do
    check_server "$server" &
    ((active++))
    if [ $active -ge $MAX_PARALLEL ]; then
        wait -n        # Wait for any one job to finish (Bash 4.3+)
        ((active--))
    fi
done
wait    # Wait for remaining jobs
```

[↑ Back to TOC](#table-of-contents)

---

## Advanced: Scripting for Infrastructure

### Health Check Script

```bash
#!/bin/bash
set -euo pipefail

# Complete HTTP health check with alerting
SERVICES=(
    "https://api.example.com/health api-service"
    "https://app.example.com/ frontend"
    "https://metrics.example.com/-/healthy prometheus"
)

ALERT_EMAIL="ops@example.com"
TIMEOUT=10
FAILURES=()

check_http() {
    local url=$1 name=$2
    local response
    response=$(curl -sf -w "%{http_code}" -o /dev/null --max-time "$TIMEOUT" "$url" 2>/dev/null) || {
        FAILURES+=("$name ($url): connection failed")
        return 1
    }
    if [ "$response" != "200" ]; then
        FAILURES+=("$name ($url): HTTP $response")
        return 1
    fi
    echo "OK: $name"
}

for service in "${SERVICES[@]}"; do
    read -r url name <<< "$service"
    check_http "$url" "$name" || true
done

if [ ${#FAILURES[@]} -gt 0 ]; then
    echo "FAILURES detected:"
    printf '  - %s\n' "${FAILURES[@]}"
    # Send alert (requires mail or mailx)
    printf '%s\n' "${FAILURES[@]}" | mail -s "Health Check Failed" "$ALERT_EMAIL"
    exit 1
fi

echo "All services healthy"
```

### Deployment Script

```bash
#!/bin/bash
set -euo pipefail

# Deployment with rollback capability
APP="myapp"
IMAGE="ghcr.io/uncleJs/${APP}"
VERSION="${1:?Usage: deploy.sh <version>}"
DEPLOY_DIR="/opt/${APP}"
BACKUP_DIR="/opt/${APP}-backup"

log() { echo "[$(date '+%H:%M:%S')] $*"; }

rollback() {
    log "ROLLBACK triggered — restoring previous version"
    [ -d "$BACKUP_DIR" ] && rsync -a "$BACKUP_DIR/" "$DEPLOY_DIR/"
    systemctl restart "$APP" || true
    log "Rollback complete"
}
trap 'rollback' ERR

# 1. Pull new image
log "Pulling image: ${IMAGE}:${VERSION}"
docker pull "${IMAGE}:${VERSION}"

# 2. Backup current deployment
log "Backing up current deployment"
[ -d "$DEPLOY_DIR" ] && rsync -a "$DEPLOY_DIR/" "$BACKUP_DIR/"

# 3. Deploy
log "Deploying version $VERSION"
sed -i "s|IMAGE_TAG=.*|IMAGE_TAG=${VERSION}|" "$DEPLOY_DIR/.env"

# 4. Restart service
log "Restarting service"
systemctl restart "$APP"

# 5. Health check
log "Waiting for service to be healthy..."
for i in {1..30}; do
    if curl -sf http://localhost:3000/health &>/dev/null; then
        log "Deployment successful: $VERSION"
        exit 0
    fi
    sleep 2
done

log "Health check failed after 60 seconds"
exit 1   # Triggers rollback via ERR trap
```

### Log Rotation Script

```bash
#!/bin/bash
# Rotate application logs with compression and retention

LOG_DIR="/var/log/myapp"
RETAIN_DAYS=30
COMPRESS_DAYS=7

# Compress logs older than $COMPRESS_DAYS days
find "$LOG_DIR" -name "*.log" -mtime +"$COMPRESS_DAYS" ! -name "*.gz" \
    -exec gzip -9 {} \; -exec echo "Compressed: {}" \;

# Delete logs older than $RETAIN_DAYS days
find "$LOG_DIR" -name "*.gz" -mtime +"$RETAIN_DAYS" \
    -exec rm {} \; -exec echo "Deleted: {}" \;

# Report current log space usage
echo "Current log space: $(du -sh "$LOG_DIR" | cut -f1)"
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
| `${VAR:-default}` | Variable with default value |
| `${VAR:?error}` | Abort if variable not set |
| `trap` | Run commands on script exit or error |
| `local` | Scope variable to a function |
| `sed 's/old/new/g'` | Global text substitution |
| `sed -i` | In-place file editing |
| `awk '{print $1}'` | Print first column of each line |
| `awk -F: '{...}'` | Use custom field delimiter |
| `jq '.'` | Pretty-print JSON |
| `jq '.field'` | Extract a JSON field |
| `jq -r` | Raw output (no quotes) |
| `jq 'select(.key == val)'` | Filter JSON array |
| `jq '[.[] \| ...]'` | Transform array |
| `jq --arg k v` | Pass shell variable to jq |
| `crontab -e` | Edit scheduled jobs |
| `crontab -l` | List scheduled jobs |
| `systemctl list-timers` | List systemd timers |
| `python3 script.py` | Run a Python script |
| `subprocess.run()` | Run system commands from Python |
| `json.loads()` / `json.dumps()` | Parse/serialize JSON in Python |
| `yaml.safe_load()` | Parse YAML in Python |
| `xargs -P` | Parallel command execution |

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

### Lab 2.4 — jq JSON Processing

1. Fetch the GitHub API: `curl -s https://api.github.com/repos/kubernetes/kubernetes > k8s.json`
2. Extract the number of open issues: `jq '.open_issues_count' k8s.json`
3. Extract the repo name, description, and star count as a formatted string using `jq -r`
4. Fetch the releases API: `curl -s https://api.github.com/repos/kubernetes/kubernetes/releases`
5. List all release tag names: `jq -r '.[].tag_name'`
6. Find the latest release that is NOT a pre-release: `jq -r '[.[] | select(.prerelease == false)] | first | .tag_name'`

### Lab 2.5 — Python Health Check Script

Write `healthcheck.py` that:
1. Takes a list of URLs from a JSON config file
2. Makes an HTTP GET request to each URL
3. Prints `OK` or `FAIL` with the status code
4. Writes results to a JSON report file with timestamps

### Lab 2.6 — Backup Automation Script

Write `backup.sh` that:
1. Takes a source directory as argument `$1` and backup destination as `$2`
2. Creates a timestamped `.tar.gz` archive
3. Removes archives older than 7 days
4. Logs each action with timestamp to `/var/log/backup.log`
5. Exits with code `1` on any failure
6. Is idempotent — safe to run multiple times

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

- [Bash Manual (GNU)](https://www.gnu.org/software/bash/manual/)
- [ShellCheck](https://www.shellcheck.net/) — Online Bash script linter
- [jq Manual](https://jqlang.github.io/jq/manual/) — Official jq documentation
- [jq Play](https://jqplay.org/) — Interactive jq sandbox
- [Python for DevOps (O'Reilly)](https://www.oreilly.com/library/view/python-for-devops/9781492057680/)
- [Crontab Guru](https://crontab.guru/) — Visual cron expression builder
- [awk Tutorial](https://www.grymoire.com/Unix/Awk.html)
- [Bash Strict Mode](http://redsymbol.net/articles/unofficial-bash-strict-mode/)
- [Glossary: Bash](./glossary.md#b), [Cron](./glossary.md#c), [Environment Variable](./glossary.md#e)

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/). Non-commercial use only. Share alike with attribution. See [LICENSE.md](./LICENSE.md).*
