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

```mermaid
flowchart LR
    INVOKE(["Shell invocation<br/>./script.sh"])
    SHEBANG["Shebang read<br/>#!/bin/bash"]
    INTERP["Interpreter launched<br/>/bin/bash"]
    EXEC["Script executes<br/>line by line"]
    EXIT(["Exit code returned<br/>0 = success"])

    INVOKE --> SHEBANG
    SHEBANG --> INTERP
    INTERP --> EXEC
    EXEC --> EXIT
```

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

The shebang (`#!/bin/bash`) is not just a comment. When the kernel executes a file, it reads the first two bytes looking for `#!`. If found, it treats the rest of the line as the path to the interpreter and passes the script as an argument. This is why the same file can behave differently depending on whether you call `./script.sh` (uses the shebang) versus `bash script.sh` (explicitly uses Bash regardless of shebang). Getting this right matters when you write portable scripts that must work across environments where Bash lives in different locations — `#!/usr/bin/env bash` is more portable than a hardcoded path.

`set -euo pipefail` is non-negotiable in production scripts. Without `set -e`, a failing command is silently ignored and the script continues — this is how automated deployments delete production databases while cheerfully reporting success. Without `set -u`, referencing an unset variable expands to an empty string, which causes bugs that are nearly impossible to reproduce in testing. Without `set -o pipefail`, the exit code of a pipeline is the exit code of the last command, so `false | true` returns 0 even though the first command failed. Together, these three options make scripts fail loudly and early, which is always better than failing silently and late.

The distinction between a shell variable and an environment variable matters in automation. A variable you define with `NAME=value` is only visible to the current shell and its functions. A variable exported with `export NAME=value` is passed to child processes — programs and scripts you launch from within your script. This is why environment variables are the standard way to pass configuration into containers, CI/CD runners, and external tools. Unexported variables are invisible to everything you call, which is a common source of confusion when scripts work interactively but fail in cron or CI.

```mermaid
flowchart TD
    PIPEFAIL["set -euo pipefail"]
    PIPEFAIL --> E["set -e<br/>Exit on any command failure"]
    PIPEFAIL --> U["set -u<br/>Error on unset variable reference"]
    PIPEFAIL --> PF["set -o pipefail<br/>Pipe exit = first non-zero exit"]

    E --> E1["false<br/>→ script exits immediately"]
    U --> U1["echo $UNSET_VAR<br/>→ unbound variable error"]
    PF --> PF1["false | true<br/>→ returns exit code 1"]
```

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

Exit codes are the contract between your script and everything that calls it. A zero exit code means success; any non-zero value means failure. CI/CD pipelines, monitoring systems, and orchestration tools all rely on exit codes to decide whether to proceed, retry, or alert. A script that always exits zero — even when it fails — is actively dangerous in automated pipelines because it silently masks failures while downstream systems proceed on the assumption that everything worked.

`trap` is the mechanism for registering cleanup handlers that run when a script exits, regardless of how it exits. `trap cleanup EXIT` ensures your cleanup function runs whether the script completes normally, exits early due to `set -e`, or receives a signal. `trap on_error ERR` fires specifically when a command fails, giving you access to the line number and command that triggered the failure. Using both together — a general cleanup on `EXIT` and a specific error reporter on `ERR` — is the pattern that production scripts should follow. The lockfile pattern (`/tmp/script.lock`) prevents concurrent execution and should always be cleaned up in the `EXIT` trap.

Lockfiles solve a critical concurrency problem in scheduled automation. Cron does not know whether the previous invocation of your script is still running. Without a lockfile, a slow script that takes 90 seconds will overlap with the next invocation when running every minute. Two simultaneous deployments, two simultaneous database backups, two simultaneous queue workers — these produce undefined and often destructive behavior. The lockfile pattern uses `mkdir` (atomic on Linux filesystems) or `flock` to ensure only one instance runs at a time. Always write the current PID into the lockfile so you can detect and clean up stale locks from crashed processes.

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

The `crond` daemon reads crontab files from `/var/spool/cron/` (per-user) and `/etc/cron.d/` (system-wide) and spawns the specified commands at the scheduled times. A detail that catches many engineers off guard: cron runs each job in a minimal environment. The `PATH` is typically just `/usr/bin:/bin`, none of your shell profile files are sourced, and your working directory is your home directory. Scripts that work perfectly when run interactively often fail in cron because they depend on environment variables or `PATH` entries that your `~/.bashrc` normally provides. Always use absolute paths for commands and explicitly set any environment variables your script needs.

By default, any output from a cron job (stdout and stderr) is mailed to the user who owns the crontab. On most servers there is no mail delivery configured, so this output is silently discarded or accumulates in a local mail spool nobody reads. In practice, you should always redirect output explicitly: redirect stdout and stderr to a log file, or to `/dev/null` if you truly do not care about it. Including timestamps in your log output is essential — cron jobs run without context, and a log line without a timestamp is nearly useless for debugging failures that happened at 3 AM.

Systemd timers are a modern alternative to cron that addresses several of its limitations. Timer units are associated with service units, so you get the full power of systemd service management: proper logging via journald, dependency ordering, resource limits, and restart policies. Timers can trigger relative to when the last run completed (not just a wall-clock schedule), which prevents the overlap problem that lockfiles solve in cron. For new infrastructure on systemd-based systems, prefer systemd timers for critical scheduled work. Keep cron for simple, low-stakes jobs where the overhead of writing two unit files is not justified.

```mermaid
flowchart LR
    SCHED["Crontab entry<br/>* * * * * command"]
    SCHED --> MIN["Minute<br/>0-59"]
    MIN --> HOUR["Hour<br/>0-23"]
    HOUR --> DOM["Day of Month<br/>1-31"]
    DOM --> MON["Month<br/>1-12"]
    MON --> DOW["Day of Week<br/>0-7"]
    DOW --> CMD(["Command executed<br/>in minimal environment"])
```

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

Idempotency is the most important property a production script can have. An idempotent script produces the same result whether it runs once or a hundred times. This property is what makes automated infrastructure safe to rerun after failures, safe to include in retry loops, and safe to apply in configuration management tools that converge state. The "check before act" principle implements idempotency at the operation level: before creating a directory, check if it exists; before adding a line to a config file, check if it is already there; before installing a package, check the installed version. Every operation in your script should be safe to repeat.

Parallel execution multiplies script throughput but introduces accounting complexity. Bash's `wait` builtin combined with process arrays allows you to fan out work and collect results. The pattern is: launch all background jobs, capture their PIDs, then iterate over the PIDs waiting for each one and checking its exit code. Without this accounting, a background job failure is silently ignored — the job fails, the parent script continues, and you discover the problem in production. Job control with `wait -n` (wait for any single job to complete) and `jobs -l` (list with PIDs) gives you fine-grained control over parallel execution without external dependencies.

Structured logging in production scripts is not optional. Raw `echo` output with no timestamps or severity levels is useless in a system where logs from multiple scripts and services are aggregated together. A minimal logging library — four functions for debug, info, warn, and error — adds ten lines to your script and saves hours of debugging. Emit to both a log file and stderr using `tee`, include timestamps in ISO 8601 format, and use consistent field ordering so log aggregators can parse your output reliably. When a 3 AM cron job fails, structured logs are the difference between a five-minute fix and a two-hour investigation.

```mermaid
flowchart LR
    START(["Script invoked"])
    CHECK{"State already<br/>at target?"}
    ACT["Perform operation"]
    VERIFY["Verify result"]
    DONE(["Exit 0"])

    START --> CHECK
    CHECK -->|"yes"| DONE
    CHECK -->|"no"| ACT
    ACT --> VERIFY
    VERIFY -->|"success"| DONE
    VERIFY -->|"failure"| ACT
```

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

```mermaid
flowchart LR
    PULL["Pull new image"]
    BACKUP["Backup current deployment"]
    DEPLOY["Write new version"]
    RESTART["Restart service"]
    HEALTH{"Health check<br/>passes?"}
    SUCCESS(["Exit 0: success"])
    ROLLBACK["Rollback to backup"]
    FAIL(["Exit 1: failed"])

    PULL --> BACKUP
    BACKUP --> DEPLOY
    DEPLOY --> RESTART
    RESTART --> HEALTH
    HEALTH -->|"yes"| SUCCESS
    HEALTH -->|"no after retries"| ROLLBACK
    ROLLBACK --> FAIL
```

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

## Testing Bash Scripts with bats-core

Bash scripts are often the least-tested part of a DevOps codebase. They handle deployments, backups, and infrastructure provisioning — operations with serious consequences — yet they are frequently written once and never tested. `bats-core` (Bash Automated Testing System) brings proper testing to shell scripts with a syntax that feels familiar if you have used Mocha or RSpec.

### Installing bats-core

```bash
# Via git (recommended — always get latest)
git clone https://github.com/bats-core/bats-core.git
cd bats-core
sudo ./install.sh /usr/local

# Via package manager (may be older version)
sudo apt install bats        # Ubuntu
sudo brew install bats-core  # macOS

# Verify
bats --version
```

### Basic Test Structure

```bash
#!/usr/bin/env bats
# test/deploy.bats

# Runs before each test
setup() {
  # Create temp dir for test artifacts
  TEST_TMPDIR="$(mktemp -d)"
  export TEST_TMPDIR
}

# Runs after each test
teardown() {
  rm -rf "$TEST_TMPDIR"
}

@test "deploy.sh creates the output directory" {
  run bash deploy.sh --env staging --output-dir "$TEST_TMPDIR/deploy"
  [ "$status" -eq 0 ]
  [ -d "$TEST_TMPDIR/deploy" ]
}

@test "deploy.sh fails with no arguments" {
  run bash deploy.sh
  [ "$status" -ne 0 ]
  [[ "$output" == *"Usage:"* ]]
}

@test "deploy.sh validates environment argument" {
  run bash deploy.sh --env production-invalid
  [ "$status" -eq 1 ]
  [[ "$output" == *"Invalid environment"* ]]
}
```

### Mocking External Commands

```bash
#!/usr/bin/env bats
# Mock kubectl so tests don't need a real cluster

setup() {
  # Create a directory for mock commands
  MOCK_BIN="$(mktemp -d)"
  export PATH="$MOCK_BIN:$PATH"

  # Mock kubectl to succeed
  cat > "$MOCK_BIN/kubectl" << 'EOF'
#!/bin/bash
echo "mock kubectl: $*" >&2
exit 0
EOF
  chmod +x "$MOCK_BIN/kubectl"
}

teardown() {
  rm -rf "$MOCK_BIN"
}

@test "rollout.sh calls kubectl apply" {
  run bash rollout.sh --manifest k8s/deployment.yaml
  [ "$status" -eq 0 ]
  [[ "$output" == *"kubectl apply"* ]]
}

@test "rollout.sh handles kubectl failure" {
  # Override mock to fail
  cat > "$MOCK_BIN/kubectl" << 'EOF'
#!/bin/bash
exit 1
EOF
  run bash rollout.sh --manifest k8s/deployment.yaml
  [ "$status" -ne 0 ]
}
```

### Running Tests in CI

```yaml
# .github/workflows/test-scripts.yml
name: Test Shell Scripts
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install bats-core
        run: |
          git clone https://github.com/bats-core/bats-core.git /tmp/bats-core
          sudo /tmp/bats-core/install.sh /usr/local
      - name: Run tests
        run: bats test/
```

[↑ Back to TOC](#table-of-contents)

---

## Python CLI Tools with Click and Typer

For DevOps tooling beyond simple bash scripts, Python CLIs are the right choice. `argparse` from the standard library works but leads to verbose, boilerplate-heavy code. `Click` and `Typer` are modern alternatives that make building well-structured CLI tools fast and readable.

### Click

Click uses decorators to define commands, options, and arguments. It handles help text, type validation, and error messages automatically.

```python
#!/usr/bin/env python3
# drift-checker.py — Check environment configuration drift
import click
import yaml
import sys
from pathlib import Path


@click.group()
@click.option("--verbose", "-v", is_flag=True, help="Enable verbose output")
@click.pass_context
def cli(ctx, verbose):
    """Environment drift checker — compare actual vs expected config."""
    ctx.ensure_object(dict)
    ctx.obj["verbose"] = verbose


@cli.command()
@click.argument("config_file", type=click.Path(exists=True, path_type=Path))
@click.option("--env", "-e", required=True, help="Environment name (dev/staging/prod)")
@click.option("--output", "-o", type=click.Choice(["text", "json"]), default="text")
def check(config_file, env, output):
    """Check for drift between expected and actual configuration."""
    with open(config_file) as f:
        expected = yaml.safe_load(f)

    env_config = expected.get(env)
    if not env_config:
        click.echo(f"Error: environment '{env}' not found in config", err=True)
        sys.exit(1)

    drift_found = False
    for key, expected_value in env_config.items():
        # In a real tool: fetch actual value from AWS SSM, Consul, etc.
        click.echo(f"  Checking {key}: expected={expected_value}")

    if not drift_found:
        click.secho("No drift detected.", fg="green")
    else:
        click.secho("Drift detected!", fg="red", bold=True)
        sys.exit(2)


@cli.command()
@click.argument("config_file", type=click.Path(exists=True, path_type=Path))
def list_envs(config_file):
    """List available environments in config file."""
    with open(config_file) as f:
        config = yaml.safe_load(f)
    for env in config:
        click.echo(env)


if __name__ == "__main__":
    cli()
```

```bash
# Usage
python drift-checker.py check config.yaml --env production --output json
python drift-checker.py list-envs config.yaml
python drift-checker.py --help
```

### Typer — Click with Type Annotations

Typer builds on Click but uses Python type annotations to define CLI parameters — no decorators needed for simple cases.

```python
#!/usr/bin/env python3
# release-tool.py — Automate release workflows
from typing import Optional
from enum import Enum
import typer

app = typer.Typer(help="Release automation tool")


class Environment(str, Enum):
    dev = "dev"
    staging = "staging"
    prod = "prod"


@app.command()
def deploy(
    service: str = typer.Argument(..., help="Service name to deploy"),
    version: str = typer.Option(..., "--version", "-v", help="Version tag to deploy"),
    env: Environment = typer.Option(Environment.dev, help="Target environment"),
    dry_run: bool = typer.Option(False, "--dry-run", help="Show what would happen without doing it"),
):
    """Deploy a service version to an environment."""
    typer.echo(f"Deploying {service}:{version} to {env.value}")
    if dry_run:
        typer.secho("DRY RUN — no changes made", fg=typer.colors.YELLOW)
        return
    # Real deploy logic here
    typer.secho(f"Deploy complete!", fg=typer.colors.GREEN)


@app.command()
def rollback(
    service: str = typer.Argument(..., help="Service name to roll back"),
    env: Environment = typer.Option(Environment.dev),
    steps: int = typer.Option(1, help="How many versions to roll back"),
):
    """Roll back a service to a previous version."""
    typer.echo(f"Rolling back {service} in {env.value} by {steps} version(s)")


if __name__ == "__main__":
    app()
```

[↑ Back to TOC](#table-of-contents)

---

## Testing Python DevOps Scripts with pytest

```python
# tests/test_deploy.py
import subprocess
import pytest
from unittest.mock import patch, MagicMock, call
from pathlib import Path
import boto3

# Assume we are testing this deploy module
# from deploy import deploy_service, get_current_version, update_ssm_parameter


class TestDeployService:
    """Tests for the deploy_service function."""

    @patch("deploy.subprocess.run")
    def test_deploy_calls_kubectl_apply(self, mock_run):
        """deploy_service should call kubectl apply with the manifest."""
        mock_run.return_value = MagicMock(returncode=0, stdout=b"deployment applied")

        from deploy import deploy_service
        deploy_service("myapp", "v1.2.3", "staging")

        mock_run.assert_called_once()
        call_args = mock_run.call_args[0][0]
        assert "kubectl" in call_args
        assert "apply" in call_args

    @patch("deploy.subprocess.run")
    def test_deploy_raises_on_kubectl_failure(self, mock_run):
        """deploy_service should raise an exception when kubectl fails."""
        mock_run.return_value = MagicMock(returncode=1, stderr=b"error: connection refused")

        from deploy import deploy_service
        with pytest.raises(RuntimeError, match="kubectl apply failed"):
            deploy_service("myapp", "v1.2.3", "staging")

    @patch("boto3.client")
    def test_update_ssm_parameter(self, mock_boto3_client):
        """update_ssm_parameter should call SSM put_parameter with correct args."""
        mock_ssm = MagicMock()
        mock_boto3_client.return_value = mock_ssm
        mock_ssm.put_parameter.return_value = {"Version": 1}

        from deploy import update_ssm_parameter
        update_ssm_parameter("/myapp/staging/version", "v1.2.3")

        mock_ssm.put_parameter.assert_called_once_with(
            Name="/myapp/staging/version",
            Value="v1.2.3",
            Type="String",
            Overwrite=True,
        )


class TestGetCurrentVersion:
    """Tests for reading current deployed version."""

    def test_reads_version_from_file(self, tmp_path):
        """Should read version string from version file."""
        version_file = tmp_path / "VERSION"
        version_file.write_text("v1.2.2\n")

        from deploy import get_current_version
        assert get_current_version(version_file) == "v1.2.2"

    def test_returns_none_for_missing_file(self, tmp_path):
        """Should return None if version file does not exist."""
        from deploy import get_current_version
        assert get_current_version(tmp_path / "VERSION") is None


@pytest.mark.parametrize("env,expected_namespace", [
    ("dev", "myapp-dev"),
    ("staging", "myapp-staging"),
    ("prod", "myapp-prod"),
])
def test_namespace_for_environment(env, expected_namespace):
    """Namespace should follow the myapp-{env} convention."""
    from deploy import get_namespace
    assert get_namespace("myapp", env) == expected_namespace
```

```bash
# Run tests with coverage
pytest tests/ -v --cov=deploy --cov-report=term-missing

# Run only fast unit tests (skip integration tests marked slow)
pytest tests/ -v -m "not slow"
```

[↑ Back to TOC](#table-of-contents)

---

## Python subprocess Deep Dive

```python
import subprocess
import logging
import shlex
from typing import Optional

logger = logging.getLogger(__name__)


def run_command(
    cmd: list[str] | str,
    timeout: int = 60,
    capture_output: bool = True,
    check: bool = True,
    cwd: Optional[str] = None,
    env: Optional[dict] = None,
) -> subprocess.CompletedProcess:
    """
    Run a shell command with proper error handling and logging.

    Args:
        cmd: Command as list (preferred) or string (shell=True risk!)
        timeout: Seconds before TimeoutExpired is raised
        capture_output: Capture stdout/stderr (True) or let them stream
        check: Raise CalledProcessError on non-zero exit code
        cwd: Working directory
        env: Environment variables (None = inherit from parent)

    Returns:
        CompletedProcess with .returncode, .stdout, .stderr

    Security note: NEVER pass shell=True with user-controlled input.
    """
    if isinstance(cmd, str):
        # Parse string safely — avoids shell injection
        cmd = shlex.split(cmd)

    cmd_str = shlex.join(cmd)
    logger.debug(f"Running: {cmd_str}")

    try:
        result = subprocess.run(
            cmd,
            capture_output=capture_output,
            text=True,
            timeout=timeout,
            check=check,
            cwd=cwd,
            env=env,
        )
        if result.stdout:
            logger.debug(f"stdout: {result.stdout.strip()}")
        return result

    except subprocess.CalledProcessError as e:
        logger.error(f"Command failed (exit {e.returncode}): {cmd_str}")
        if e.stdout:
            logger.error(f"stdout: {e.stdout.strip()}")
        if e.stderr:
            logger.error(f"stderr: {e.stderr.strip()}")
        raise
    except subprocess.TimeoutExpired:
        logger.error(f"Command timed out after {timeout}s: {cmd_str}")
        raise


def stream_command(cmd: list[str]) -> int:
    """Run a command and stream output line by line in real time."""
    process = subprocess.Popen(
        cmd,
        stdout=subprocess.PIPE,
        stderr=subprocess.STDOUT,
        text=True,
        bufsize=1,  # Line buffered
    )

    for line in process.stdout:
        print(line, end="", flush=True)

    process.wait()
    return process.returncode


# Piping between commands (equivalent of: kubectl get pods | grep Running)
def get_running_pods(namespace: str) -> list[str]:
    get_pods = subprocess.Popen(
        ["kubectl", "get", "pods", "-n", namespace, "--no-headers"],
        stdout=subprocess.PIPE,
        text=True,
    )
    grep = subprocess.Popen(
        ["grep", "Running"],
        stdin=get_pods.stdout,
        stdout=subprocess.PIPE,
        text=True,
    )
    get_pods.stdout.close()  # Allow get_pods to receive SIGPIPE if grep exits
    output, _ = grep.communicate()
    return [line.split()[0] for line in output.splitlines()]
```

[↑ Back to TOC](#table-of-contents)

---

## YAML & TOML Manipulation in Python

```python
#!/usr/bin/env python3
"""
bump-image-tag.py — Update image tag in Helm values.yaml without
destroying comments or formatting, using ruamel.yaml.
"""
import sys
from pathlib import Path
from ruamel.yaml import YAML


def bump_image_tag(values_file: Path, image_key: str, new_tag: str) -> None:
    """
    Update an image tag in a Helm values.yaml file.

    Preserves comments, anchor references, and key ordering.
    PyYAML would strip all comments — ruamel.yaml does not.
    """
    yaml = YAML()
    yaml.preserve_quotes = True
    yaml.width = 4096  # Prevent line wrapping

    with open(values_file) as f:
        data = yaml.load(f)

    # Navigate nested keys: "image.tag" -> data["image"]["tag"]
    keys = image_key.split(".")
    node = data
    for key in keys[:-1]:
        node = node[key]

    old_tag = node[keys[-1]]
    node[keys[-1]] = new_tag
    print(f"Updated {image_key}: {old_tag} -> {new_tag}")

    with open(values_file, "w") as f:
        yaml.dump(data, f)


if __name__ == "__main__":
    if len(sys.argv) != 4:
        print(f"Usage: {sys.argv[0]} <values.yaml> <image.key> <new-tag>")
        sys.exit(1)

    bump_image_tag(Path(sys.argv[1]), sys.argv[2], sys.argv[3])
```

```bash
# Usage
python bump-image-tag.py charts/myapp/values.yaml image.tag v1.2.3

# TOML manipulation (Python 3.11+ has tomllib in stdlib for reading)
python3 -c "
import tomllib, tomli_w
with open('pyproject.toml', 'rb') as f:
    data = tomllib.load(f)
data['project']['version'] = '1.2.3'
with open('pyproject.toml', 'wb') as f:
    tomli_w.dump(data, f)
print('Updated version to 1.2.3')
"
```

[↑ Back to TOC](#table-of-contents)

---

## Makefile as a DevOps Task Runner

```makefile
# Makefile — DevOps project task runner
# Run 'make help' to see all targets

.PHONY: help build test lint push deploy clean

# Default target
.DEFAULT_GOAL := help

# Variables with defaults (override on CLI: make deploy ENV=production)
ENV          ?= staging
IMAGE_NAME   ?= myapp
REGISTRY     ?= 123456789012.dkr.ecr.us-east-1.amazonaws.com
VERSION      ?= $(shell git describe --tags --always --dirty)
IMAGE_TAG    := $(REGISTRY)/$(IMAGE_NAME):$(VERSION)
KUBE_NS      ?= $(IMAGE_NAME)-$(ENV)

## help: Show this help message
help:
	@echo "Usage: make [target] [VAR=value]"
	@echo ""
	@grep -E '^## [a-zA-Z_-]+:' $(MAKEFILE_LIST) | \
		sed 's/^## //' | \
		awk -F: '{printf "  %-20s %s\n", $$1, $$2}'

## build: Build Docker image
build:
	docker build \
		--build-arg VERSION=$(VERSION) \
		--tag $(IMAGE_TAG) \
		--tag $(REGISTRY)/$(IMAGE_NAME):latest \
		.

## test: Run unit tests
test:
	pytest tests/ -v --cov=src --cov-report=term-missing

## lint: Run linters (ruff, shellcheck, hadolint)
lint:
	ruff check src/ tests/
	shellcheck scripts/*.sh
	hadolint Dockerfile

## push: Push image to registry (requires: build)
push: build
	aws ecr get-login-password | docker login --username AWS --password-stdin $(REGISTRY)
	docker push $(IMAGE_TAG)
	docker push $(REGISTRY)/$(IMAGE_NAME):latest

## deploy: Deploy to Kubernetes (requires: push)
deploy: push
	@echo "Deploying $(IMAGE_TAG) to $(ENV)..."
	helm upgrade --install $(IMAGE_NAME) charts/$(IMAGE_NAME)/ \
		--namespace $(KUBE_NS) \
		--create-namespace \
		--set image.tag=$(VERSION) \
		--set image.repository=$(REGISTRY)/$(IMAGE_NAME) \
		--values charts/$(IMAGE_NAME)/values-$(ENV).yaml \
		--wait --timeout=5m

## rollback: Roll back last deployment
rollback:
	helm rollback $(IMAGE_NAME) --namespace $(KUBE_NS)

## clean: Remove local build artifacts
clean:
	docker rmi $(IMAGE_TAG) $(REGISTRY)/$(IMAGE_NAME):latest 2>/dev/null || true
	find . -type d -name __pycache__ -exec rm -rf {} + 2>/dev/null || true
	find . -name "*.pyc" -delete 2>/dev/null || true
```

[↑ Back to TOC](#table-of-contents)

---

## Regex in Shell and Python

```bash
# Extract all IP addresses from a log file
grep -oE '[0-9]{1,3}(\.[0-9]{1,3}){3}' /var/log/nginx/access.log | sort -u

# Extract HTTP status codes and count occurrences
grep -oE '"[A-Z]+ [^ ]+ HTTP/[0-9.]+" [0-9]+' /var/log/nginx/access.log | \
  grep -oE ' [0-9]{3} ' | sort | uniq -c | sort -rn

# Match semver strings (1.2.3 or v1.2.3)
echo "Release v2.14.0 is live" | grep -oP 'v?[0-9]+\.[0-9]+\.[0-9]+'

# Extract named groups with grep -P (Perl-compatible)
echo "ERROR 2026-01-15 09:23:10 Payment failed for user 12345" | \
  grep -oP '(?P<level>ERROR|WARN|INFO) (?P<ts>[0-9-]+ [0-9:]+)'

# Validate an AWS ARN format
echo "arn:aws:iam::123456789012:role/my-role" | \
  grep -qP '^arn:aws:[a-z0-9]+:([a-z0-9-]*)?:[0-9]{12}:.+$' && echo "valid" || echo "invalid"
```

```python
import re

# Parse nginx access log line
LOG_PATTERN = re.compile(
    r'(?P<ip>\d+\.\d+\.\d+\.\d+) - - \[(?P<time>[^\]]+)\] '
    r'"(?P<method>\w+) (?P<path>[^\s]+) HTTP/[0-9.]+" '
    r'(?P<status>\d{3}) (?P<bytes>\d+)'
)

line = '192.168.1.1 - - [15/Jan/2026:09:23:10 +0000] "GET /api/health HTTP/1.1" 200 42'
m = LOG_PATTERN.match(line)
if m:
    print(m.group("ip"), m.group("status"), m.group("path"))
    # 192.168.1.1 200 /api/health

# Extract and validate semver
SEMVER = re.compile(r'^v?(?P<major>\d+)\.(?P<minor>\d+)\.(?P<patch>\d+)(?:-(?P<pre>[a-zA-Z0-9.]+))?$')
versions = ["v1.2.3", "2.0.0-rc.1", "invalid", "1.0"]
for v in versions:
    m = SEMVER.match(v)
    if m:
        print(f"Valid: major={m.group('major')} minor={m.group('minor')} patch={m.group('patch')}")
    else:
        print(f"Invalid semver: {v}")
```

[↑ Back to TOC](#table-of-contents)

---

## Signal Handling in Bash

```bash
#!/bin/bash
# long-running-job.sh — Demonstrates production-grade signal handling
set -euo pipefail

PIDFILE="/var/run/myjob.pid"
TMPDIR_WORK=""
CHILD_PID=""

cleanup() {
  local exit_code=$?
  echo "Cleanup triggered (exit code: $exit_code)"

  # Kill child process if still running
  if [[ -n "$CHILD_PID" ]] && kill -0 "$CHILD_PID" 2>/dev/null; then
    echo "Stopping child process $CHILD_PID..."
    kill -TERM "$CHILD_PID"
    wait "$CHILD_PID" 2>/dev/null || true
  fi

  # Remove temp directory
  if [[ -n "$TMPDIR_WORK" ]] && [[ -d "$TMPDIR_WORK" ]]; then
    rm -rf "$TMPDIR_WORK"
    echo "Removed temp dir $TMPDIR_WORK"
  fi

  # Remove PID file
  rm -f "$PIDFILE"
  echo "Job exited cleanly"
}

# Register cleanup for all exit paths
trap cleanup EXIT

# Handle specific signals
trap 'echo "Received SIGTERM — shutting down gracefully..."; exit 0' SIGTERM
trap 'echo "Received SIGINT (Ctrl+C) — aborting..."; exit 130' SIGINT
trap 'echo "Received SIGHUP — reloading config (not implemented)..."' SIGHUP

# Write PID file
echo $$ > "$PIDFILE"

# Create working temp dir
TMPDIR_WORK="$(mktemp -d)"
echo "Working in $TMPDIR_WORK"

# Start a long-running background task
some_long_running_process &
CHILD_PID=$!
echo "Started child process $CHILD_PID"

# Wait for child, propagating signals
wait "$CHILD_PID"
echo "Job completed successfully"
```

[↑ Back to TOC](#table-of-contents)

---

## Parallel Automation

```python
#!/usr/bin/env python3
"""parallel-deploy.py — Deploy to multiple servers in parallel."""
import asyncio
import asyncssh
from typing import NamedTuple


class DeployResult(NamedTuple):
    host: str
    success: bool
    output: str
    error: str = ""


async def deploy_to_host(host: str, command: str, semaphore: asyncio.Semaphore) -> DeployResult:
    """Deploy to a single host via SSH, respecting concurrency limit."""
    async with semaphore:
        try:
            async with asyncssh.connect(
                host,
                username="deploy",
                known_hosts=None,  # In production: use proper known_hosts
                connect_timeout=10,
            ) as conn:
                result = await conn.run(command, check=True, timeout=120)
                return DeployResult(host=host, success=True, output=result.stdout)
        except Exception as e:
            return DeployResult(host=host, success=False, output="", error=str(e))


async def parallel_deploy(hosts: list[str], command: str, max_concurrent: int = 10):
    """Deploy to all hosts in parallel, max max_concurrent at once."""
    semaphore = asyncio.Semaphore(max_concurrent)
    tasks = [deploy_to_host(host, command, semaphore) for host in hosts]
    results = await asyncio.gather(*tasks)

    successful = [r for r in results if r.success]
    failed = [r for r in results if not r.success]

    print(f"Deployed to {len(successful)}/{len(hosts)} hosts")
    for r in failed:
        print(f"  FAILED {r.host}: {r.error}")

    return results


# Run
hosts = [f"app-{i:02d}.internal" for i in range(1, 21)]  # 20 app servers
asyncio.run(parallel_deploy(hosts, "systemctl restart myapp", max_concurrent=5))
```

```bash
# GNU parallel — deploy restart command to 20 servers (5 at a time)
parallel --jobs 5 --tag \
  'ssh deploy@{} "systemctl restart myapp && echo OK"' \
  ::: app-{01..20}.internal

# xargs -P for parallel execution
echo -e "server1\nserver2\nserver3" | \
  xargs -P 3 -I{} ssh deploy@{} "df -h / | tail -1"
```

[↑ Back to TOC](#table-of-contents)

---

## Python for AWS Automation (boto3)

```python
#!/usr/bin/env python3
"""aws-inventory.py — Inventory EC2 instances across all regions."""
import boto3
from botocore.exceptions import ClientError, EndpointResolutionError


def get_all_regions() -> list[str]:
    """Get all enabled AWS regions."""
    ec2 = boto3.client("ec2", region_name="us-east-1")
    regions = ec2.describe_regions(Filters=[{"Name": "opt-in-status", "Values": ["opt-in-not-required", "opted-in"]}])
    return [r["RegionName"] for r in regions["Regions"]]


def get_instances_in_region(region: str) -> list[dict]:
    """Get all running EC2 instances in a region using pagination."""
    ec2 = boto3.client("ec2", region_name=region)
    instances = []

    # Use paginator to handle > 1000 instances
    paginator = ec2.get_paginator("describe_instances")
    for page in paginator.paginate(
        Filters=[{"Name": "instance-state-name", "Values": ["running"]}]
    ):
        for reservation in page["Reservations"]:
            for instance in reservation["Instances"]:
                name = next(
                    (t["Value"] for t in instance.get("Tags", []) if t["Key"] == "Name"),
                    "unnamed"
                )
                instances.append({
                    "region": region,
                    "id": instance["InstanceId"],
                    "type": instance["InstanceType"],
                    "name": name,
                    "private_ip": instance.get("PrivateIpAddress", ""),
                    "launch_time": instance["LaunchTime"].isoformat(),
                })
    return instances


def find_untagged_s3_buckets() -> list[str]:
    """Find S3 buckets without a required 'Environment' tag."""
    s3 = boto3.client("s3")
    untagged = []

    for bucket in s3.list_buckets()["Buckets"]:
        name = bucket["Name"]
        try:
            tags = s3.get_bucket_tagging(Bucket=name)
            tag_keys = [t["Key"] for t in tags.get("TagSet", [])]
            if "Environment" not in tag_keys:
                untagged.append(name)
        except ClientError as e:
            if e.response["Error"]["Code"] == "NoSuchTagSet":
                untagged.append(name)  # No tags at all
            else:
                raise

    return untagged


if __name__ == "__main__":
    print("Scanning all regions for running EC2 instances...")
    all_instances = []
    for region in get_all_regions():
        try:
            instances = get_instances_in_region(region)
            all_instances.extend(instances)
            if instances:
                print(f"  {region}: {len(instances)} instances")
        except Exception as e:
            print(f"  {region}: ERROR — {e}")

    print(f"\nTotal running instances: {len(all_instances)}")

    print("\nFinding untagged S3 buckets...")
    untagged = find_untagged_s3_buckets()
    print(f"Untagged buckets (missing 'Environment' tag): {len(untagged)}")
    for b in untagged:
        print(f"  - {b}")
```

[↑ Back to TOC](#table-of-contents)

---

## Templating Beyond Jinja2

```bash
# envsubst — simple environment variable substitution
# Template file: k8s/deployment.tmpl.yaml
cat k8s/deployment.tmpl.yaml
# image: ${IMAGE_REGISTRY}/${SERVICE_NAME}:${IMAGE_TAG}
# replicas: ${REPLICA_COUNT}

# Generate the final manifest
export IMAGE_REGISTRY="123456789012.dkr.ecr.us-east-1.amazonaws.com"
export SERVICE_NAME="myapp"
export IMAGE_TAG="v1.2.3"
export REPLICA_COUNT="3"

envsubst < k8s/deployment.tmpl.yaml > k8s/deployment.yaml

# Limit substitution to specific variables (avoid substituting $PATH, $HOME etc.)
envsubst '${IMAGE_REGISTRY} ${SERVICE_NAME} ${IMAGE_TAG} ${REPLICA_COUNT}' \
  < k8s/deployment.tmpl.yaml > k8s/deployment.yaml
```

```bash
# gomplate — more powerful template engine
# Install
curl -sL https://github.com/hairyhenderson/gomplate/releases/latest/download/gomplate_linux-amd64 \
  -o /usr/local/bin/gomplate && chmod +x /usr/local/bin/gomplate

# Template with YAML data source
cat values.yaml
# service: myapp
# environments:
#   staging: {replicas: 2, memory: "512Mi"}
#   prod: {replicas: 10, memory: "2Gi"}

cat deployment.tmpl.yaml
# replicas: {{ (datasource "vals").environments.prod.replicas }}
# resources:
#   limits:
#     memory: {{ (datasource "vals").environments.prod.memory }}

gomplate -d vals=values.yaml -f deployment.tmpl.yaml -o deployment.yaml
```

[↑ Back to TOC](#table-of-contents)

---

## Common Mistakes & Pitfalls

- **Forgetting `set -euo pipefail` at the top of every bash script.** Without `-e`, the script continues after errors. Without `-u`, unset variables silently expand to empty strings (leading to `rm -rf /` disasters). Without `-o pipefail`, a failed command in a pipeline is masked by the exit code of the last command.

- **Unquoted variables causing word splitting.** `for file in $FILES` splits on spaces — a filename with spaces becomes two separate items. Always quote: `for file in "$FILES"`. Use arrays for lists of values: `files=("file one.txt" "file two.txt"); for f in "${files[@]}"; do`.

- **Using `ls` output in scripts.** `for f in $(ls *.txt)` breaks on filenames with spaces and other special characters. Use globbing directly: `for f in *.txt; do`.

- **Not testing with `shellcheck`.** Install `shellcheck` and run it on all your bash scripts in CI. It catches the vast majority of common bash bugs automatically.

- **Parsing JSON with grep/sed/awk.** Use `jq` for JSON. `echo "$response" | grep '"status"' | cut -d'"' -f4` breaks on whitespace variations, nested structures, and edge cases. `echo "$response" | jq -r '.status'` is correct.

- **Hardcoding credentials in scripts.** Even in "private" repositories. Use environment variables, AWS IAM roles, Vault, or cloud secret managers. Check git history — secrets committed and later deleted are still in the history.

- **Not handling the case where a required environment variable is unset.** Use `${VAR:?Variable VAR is required}` to fail with a clear message if the variable is not set.

- **Mutable `latest` tags in deployment scripts.** `docker pull myapp:latest` pulls whatever is currently tagged latest on the registry — which may have changed since you last tested. Pin to an explicit version or digest.

- **Not using virtual environments for Python automation.** Running automation scripts with the system Python pollutes the system packages, causes version conflicts, and makes the environment non-reproducible.

- **Using `subprocess.run(cmd, shell=True)` with any user-provided or external input.** This allows shell injection. Always pass commands as lists: `subprocess.run(["kubectl", "apply", "-f", manifest], ...)`.

- **Ignoring exit codes from external commands.** In Python, `subprocess.run(cmd)` does not raise on failure by default. Always use `check=True` or explicitly check `result.returncode`.

- **Scripts that are not idempotent.** A deployment script that fails halfway through and cannot be safely re-run causes outages. Design scripts to be idempotent — running them twice should produce the same result as running them once.

- **Not logging what a script is doing.** Scripts that run silently in CI make debugging impossible. Log key actions, decisions, and external calls. Include timestamps.

- **Global state in Python automation.** Module-level variables, mutable defaults in function signatures, and class-level state make scripts harder to test and reason about. Prefer explicit parameter passing.

- **Not using `--dry-run` flags.** Any script that makes changes should have a `--dry-run` option that shows what it would do without doing it. This is invaluable for reviewing changes before they happen in production.

- **Bash arithmetic on strings.** `$(( $VAR + 1 ))` silently produces 1 if `$VAR` is empty or non-numeric. Validate inputs before arithmetic operations.

[↑ Back to TOC](#table-of-contents)

---

## Interview Prep

**Q: What does `set -euo pipefail` do, and why should you use it in every bash script?**

**A:** `-e` causes the script to exit immediately when any command returns a non-zero exit code (without this, errors are silently ignored and execution continues). `-u` treats unset variables as errors instead of expanding them to empty strings (prevents `rm -rf "$UNSET_DIR/"` disasters). `-o pipefail` makes a pipeline fail if any command in it fails, not just the last one — without this, `broken_command | grep something` returns 0 because grep succeeded. Together, they turn bash from a permissive language where errors get ignored into one where errors halt execution immediately.

**Q: How do you safely pass arguments to a subprocess in Python to avoid shell injection?**

**A:** Always pass commands as lists to `subprocess.run`, never as strings with `shell=True`. With `shell=True`, the command is passed to `/bin/sh -c` and shell metacharacters like `;`, `|`, and `$()` are interpreted — if any part of the command comes from user input or external data, this allows arbitrary command execution. With a list, each element is a separate argument passed directly to `execve()` — no shell interpretation, no injection risk. Example: `subprocess.run(["kubectl", "apply", "-f", user_provided_path], check=True)` is safe regardless of what `user_provided_path` contains.

**Q: What is the difference between `subprocess.run`, `subprocess.Popen`, and `subprocess.check_output`?**

**A:** `subprocess.run` is the modern high-level interface (Python 3.5+) — it runs the process, waits for it to complete, and returns a `CompletedProcess` object. Use it for most cases. `subprocess.Popen` is the low-level interface for streaming output in real time, bidirectional communication (sending stdin while reading stdout), or managing process lifecycle manually. `subprocess.check_output` is a legacy convenience function that returns stdout as bytes and raises `CalledProcessError` on failure — equivalent to `subprocess.run(..., capture_output=True, check=True).stdout`, but `subprocess.run` is preferred for new code.

**Q: How do you test a bash script that makes SSH connections or calls AWS APIs?**

**A:** Mock the external commands by prepending a directory containing fake executables to `$PATH` in your test setup. For SSH: create a mock `ssh` script in a temp directory that echoes its arguments and returns 0. For AWS CLI: create a mock `aws` script that reads the subcommand and returns pre-defined JSON responses. `bats-core` makes this pattern clean with `setup()` and `teardown()` functions. For Python scripts using boto3, use `unittest.mock.patch` to replace the boto3 client with a `MagicMock` that returns controlled responses.

**Q: What is `trap` in bash, and how do you use it for cleanup?**

**A:** `trap` registers a command to run when the shell receives a signal or exits. `trap cleanup EXIT` runs the `cleanup` function whenever the script exits, whether normally, via `exit`, or via an error with `set -e`. This guarantees cleanup (removing temp files, stopping background processes, deleting PID files) even if the script is interrupted. `trap 'echo interrupted; exit 130' SIGINT` handles Ctrl+C gracefully. Note: `SIGKILL` (signal 9) cannot be trapped — it is sent directly by the kernel.

**Q: How do you bump an image tag in a Helm values.yaml from a CI pipeline without losing comments?**

**A:** Use `ruamel.yaml` (not PyYAML) which preserves comments, anchor references, and key ordering when loading and dumping YAML. Load the file, navigate to the `image.tag` key, update it, and write it back. PyYAML strips all comments on dump, which is unacceptable for human-maintained config files. Alternatively, use `yq` (the Go implementation) which is a binary you can use from shell scripts: `yq e '.image.tag = "v1.2.3"' -i values.yaml`.

**Q: When would you use `asyncio` for DevOps automation, and when is `ThreadPoolExecutor` better?**

**A:** Use `asyncio` for I/O-bound concurrency — making many API calls, SSH connections, or HTTP requests simultaneously. It is more efficient than threads because it uses cooperative multitasking with a single thread, so 1000 concurrent API calls don't require 1000 threads. Use `ThreadPoolExecutor` when you need to parallelize code that calls blocking C extensions or legacy libraries that are not async-aware. Use `ProcessPoolExecutor` for CPU-bound work (data processing, compression) that needs to bypass the GIL. For most DevOps tasks (deploying to N servers, checking N services), `asyncio` with `asyncssh` or `aiohttp` is the right choice.

**Q: How do you prevent a Makefile from continuing if one target fails?**

**A:** By default, Make stops when any recipe line returns a non-zero exit code. To explicitly ensure a target fails on error, use shell commands that propagate errors (`set -e` in multi-line recipes). For multi-line recipes, add `.ONESHELL:` at the top of the Makefile and add `set -euo pipefail` as the first line of the recipe. Without `.ONESHELL:`, each line in a recipe runs in a separate shell, so `set -e` in line 1 does not affect line 2. Prefix a recipe line with `-` to continue even if it fails: `- rm build/` (useful for cleanup targets where the file may not exist).

**Q: What is the difference between `envsubst` and a full template engine like gomplate?**

**A:** `envsubst` performs simple find-and-replace of `$VAR` and `${VAR}` patterns with environment variable values. It has no logic, no conditionals, no loops, and no data sources. It is perfect for simple cases. `gomplate` is a full template engine that supports conditionals (`if/else`), loops (`range`), multiple data sources (YAML files, AWS SSM, Vault, environment variables), and a rich function library. Use `envsubst` when you need to substitute a handful of variables in CI. Use `gomplate` when you need to generate configuration from structured data with logic.

**Q: How do you handle rate limits when making parallel API calls in Python?**

**A:** Use `asyncio.Semaphore` to limit the number of concurrent requests — `semaphore = asyncio.Semaphore(10)` limits to 10 simultaneous calls. Add exponential backoff with jitter for retries on 429 responses: wait `min(2^attempt * 100ms + random_jitter, 30s)`. The `tenacity` library provides production-grade retry decorators with configurable strategies. For AWS APIs specifically, the boto3 config's `retries` parameter handles `ThrottlingException` automatically — configure `max_attempts` and `mode='adaptive'` to let boto3 automatically back off under rate limit pressure.

**Q: What does idempotency mean for a deployment script, and why does it matter?**

**A:** An idempotent script produces the same result whether it runs once or ten times on the same system. For deployment scripts, this means: if the script deploys version v1.2.3 and you run it again, it results in version v1.2.3 deployed — it does not fail because the resource already exists. Idempotency matters because deployments fail halfway through regularly (network issues, timeouts, rate limits). If you cannot safely re-run a failed script, recovery requires manual investigation and intervention. Implement idempotency by checking current state before making changes: "is this resource at the desired state already? If so, skip."

**Q: How do you structure a Python automation project for maintainability?**

**A:** Organize code into modules by function (deploy.py, inventory.py, notifications.py), keep business logic separate from CLI parsing (makes testing easier), use dataclasses or Pydantic models for structured data instead of raw dicts, keep configuration in environment variables or files (never hardcoded), write tests from the start with pytest (mock external calls), pin dependencies in `requirements.txt` or `pyproject.toml`, and use a virtual environment. For team projects, add `ruff` for linting and formatting, `mypy` for type checking, and `pre-commit` hooks to enforce standards before every commit.

[↑ Back to TOC](#table-of-contents)

---

## A Day in the Life

It is Thursday afternoon and the CTO has just sent a message to the engineering channel: "Why are we deploying to production manually? Can we automate this?" You have been meaning to tackle this for months. Now it is your job.

The current process is embarrassing in retrospect: someone SSHs into a Jenkins server, finds the right job, clicks "Build with Parameters," fills in a form, watches a log scroll by for 10 minutes, then manually updates a Confluence page with the deployed version. Three different people have "the knowledge" of how to do it correctly. Last quarter, someone got the parameter wrong and deployed the wrong service version to production — it took 40 minutes to notice and roll back.

You start by understanding what actually needs to happen: query the GitHub API for the latest release tag of the service, update the Helm `values.yaml` with the new image tag, commit the change to the GitOps repository, and open a pull request. If the PR passes CI checks and gets approved, it merges automatically, which triggers Argo CD to deploy to production. The Confluence page should update itself.

You open your editor and start with the GitHub API query. The `gh` CLI is already installed, so you write a Python wrapper using `PyGithub` to fetch the latest release. You test it locally: `python release_tool.py get-latest myorg/myapp` returns `v1.8.4`. That part takes 20 minutes.

The `values.yaml` bumper is next. You remember the last time someone used a naive `sed` command to update the image tag — it matched the wrong line because two services had similar names. You use `ruamel.yaml` this time, navigating to the exact key path. You write a pytest test that creates a temp `values.yaml`, calls the function, and asserts the output is correct. Green on first run. You feel briefly smug.

The GitOps commit is trickier. You need to clone the GitOps repo, make the change, commit it with a meaningful message (following Conventional Commits format: `feat(myapp): bump image to v1.8.4`), push to a branch, and open a PR using the GitHub API. You wire up `subprocess` calls to `git` rather than using GitPython, because GitPython has surprised you before. Each external call goes through your `run_command` wrapper that logs everything and raises clearly on failure.

Halfway through, the `git push` call fails — it tries to use HTTPS and prompts for credentials. You load the `git-ssh` skill and convert the remote URL to SSH. Fifteen minutes lost, lesson learned.

You add the Confluence update last — it is the least critical and you want to get the core automation working first. A simple REST call to the Confluence API to update a page with the deployed version and timestamp.

By 6pm you have a working script, 42 tests, and a GitHub Actions workflow that runs the deployment on demand with approval gates. You demo it to the team: click one button in the GitHub Actions UI, enter the service name, watch it run, get a Slack notification with the PR link, approve the PR, watch Argo CD sync. Total hands-on time: under two minutes.

The CTO responds with a single emoji. You know which one.

[↑ Back to TOC](#table-of-contents)

---

## Advanced Bash Scripting Patterns

### Associative Arrays

Bash 4+ supports associative arrays (hash maps). They are invaluable when you need to look up values by key without repeatedly scanning lists.

```bash
#!/usr/bin/env bash

# Declare an associative array
declare -A service_ports
service_ports[nginx]=80
service_ports[postgres]=5432
service_ports[redis]=6379
service_ports[rabbitmq]=5672

# Look up a value
echo "nginx port: ${service_ports[nginx]}"

# Iterate over keys
for service in "${!service_ports[@]}"; do
  echo "$service listens on ${service_ports[$service]}"
done

# Check if a key exists
if [[ -v service_ports[nginx] ]]; then
  echo "nginx is in the map"
fi

# Real-world use: health check all known services
declare -A service_hosts=(
  [api-gateway]=api.internal
  [user-service]=users.internal
  [payment-service]=payments.internal
)

for svc in "${!service_hosts[@]}"; do
  host="${service_hosts[$svc]}"
  if timeout 2 bash -c ">/dev/tcp/$host/80" 2>/dev/null; then
    echo "[OK]   $svc ($host)"
  else
    echo "[FAIL] $svc ($host)"
  fi
done
```

### Here-Documents and Here-Strings

```bash
# Here-document: multi-line string passed to a command
cat > /etc/nginx/conf.d/myapp.conf << 'EOF'
server {
    listen 80;
    server_name myapp.example.com;
    location / {
        proxy_pass http://127.0.0.1:3000;
    }
}
EOF
# Note: 'EOF' (quoted) prevents variable expansion inside the heredoc
# EOF (unquoted) allows variable expansion

# Here-string: single string passed to stdin
# Useful when a command only accepts stdin but you have a variable
checksum=$(md5sum <<< "$my_config_string" | awk '{print $1}')

# Redirect into a heredoc with variable substitution
cat << EOF > /tmp/report.txt
Report generated: $(date)
Host: $(hostname)
Uptime: $(uptime -p)
EOF
```

### Process Substitution

```bash
# Process substitution feeds the output of a command as if it were a file
# This avoids creating temporary files

# diff two commands' output directly
diff <(kubectl get nodes --context=prod) <(kubectl get nodes --context=staging)

# Read from two sorted files simultaneously
join <(sort file1.txt) <(sort file2.txt)

# Pass multiple log files to a single awk program
awk '{sum += $5} END {print sum}' \
  <(grep "GET /api" access.log.1) \
  <(grep "GET /api" access.log)

# tee to multiple destinations simultaneously
myapp 2>&1 | tee >(logger -t myapp) >(gzip > /var/log/myapp-$(date +%Y%m%d).gz)
```

### Named Pipes (FIFOs)

```bash
# Named pipes let two processes communicate without a file
mkfifo /tmp/mypipe

# Producer writes to the pipe
myapp_producer > /tmp/mypipe &

# Consumer reads from the pipe
myapp_consumer < /tmp/mypipe

# Real-world example: split a log stream to two processors simultaneously
mkfifo /tmp/logpipe
tail -F /var/log/app.log > /tmp/logpipe &
tee /tmp/logpipe < /dev/stdin | \
  grep ERROR | send_to_slack &
  grep WARN  | aggregate_metrics &

# Clean up
rm /tmp/mypipe
```

[↑ Back to TOC](#table-of-contents)

---

## Python for DevOps: Production Patterns

### Structured CLI with Click

```python
#!/usr/bin/env python3
"""
deploy.py — service deployment CLI
"""
import click
import subprocess
import sys
from pathlib import Path

@click.group()
@click.option('--dry-run', is_flag=True, help='Show what would happen, do nothing')
@click.option('--verbose', '-v', is_flag=True)
@click.pass_context
def cli(ctx, dry_run, verbose):
    """Service deployment tool."""
    ctx.ensure_object(dict)
    ctx.obj['dry_run'] = dry_run
    ctx.obj['verbose'] = verbose


@cli.command()
@click.argument('service')
@click.argument('version')
@click.option('--namespace', '-n', default='production', show_default=True)
@click.option('--replicas', type=int, default=None)
@click.pass_context
def deploy(ctx, service, version, namespace, replicas):
    """Deploy SERVICE at VERSION to a Kubernetes namespace."""
    dry_run = ctx.obj['dry_run']
    
    click.echo(f"Deploying {service}:{version} to {namespace}")
    
    cmd = [
        'kubectl', 'set', 'image',
        f'deployment/{service}',
        f'{service}={service}:{version}',
        '-n', namespace
    ]
    
    if dry_run:
        click.echo(f"[DRY RUN] Would run: {' '.join(cmd)}")
        return
    
    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode != 0:
        click.echo(f"Error: {result.stderr}", err=True)
        sys.exit(1)
    
    click.echo(f"Successfully deployed {service}:{version}")


@cli.command()
@click.argument('service')
@click.option('--namespace', '-n', default='production', show_default=True)
def rollback(service, namespace):
    """Roll back SERVICE to the previous version."""
    result = subprocess.run(
        ['kubectl', 'rollout', 'undo', f'deployment/{service}', '-n', namespace],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        click.echo(f"Rollback failed: {result.stderr}", err=True)
        sys.exit(1)
    click.echo(f"Rolled back {service}")


if __name__ == '__main__':
    cli()
```

### Robust HTTP Client with retries

```python
import httpx
import tenacity
import structlog

log = structlog.get_logger()

@tenacity.retry(
    wait=tenacity.wait_exponential(multiplier=1, min=1, max=10),
    stop=tenacity.stop_after_attempt(5),
    retry=tenacity.retry_if_exception_type(httpx.TransientError),
    before_sleep=tenacity.before_sleep_log(log, tenacity.log.logging.WARNING),
)
def fetch_with_retry(url: str, **kwargs) -> httpx.Response:
    """Fetch a URL with exponential backoff retry."""
    with httpx.Client(timeout=30.0) as client:
        response = client.get(url, **kwargs)
        response.raise_for_status()
        return response


def get_deployment_status(cluster_api: str, namespace: str, deployment: str) -> dict:
    url = f"{cluster_api}/apis/apps/v1/namespaces/{namespace}/deployments/{deployment}"
    try:
        response = fetch_with_retry(url)
        return response.json()
    except tenacity.RetryError as e:
        log.error("api_unreachable", url=url, error=str(e))
        raise
```

### Parallel Operations with concurrent.futures

```python
import concurrent.futures
from typing import List, Tuple
import subprocess


def run_command(args: Tuple[str, list]) -> Tuple[str, bool, str]:
    """Run a command and return (name, success, output)."""
    name, cmd = args
    result = subprocess.run(cmd, capture_output=True, text=True, timeout=60)
    success = result.returncode == 0
    output = result.stdout if success else result.stderr
    return name, success, output.strip()


def parallel_health_checks(services: List[dict]) -> dict:
    """
    Run health checks on multiple services concurrently.
    services = [{"name": "api", "url": "http://api.internal/health"}, ...]
    """
    commands = [
        (svc["name"], ["curl", "-sf", "-o", "/dev/null", "-w", "%{http_code}", svc["url"]])
        for svc in services
    ]
    
    results = {}
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        for name, success, output in executor.map(run_command, commands):
            results[name] = {"healthy": success and output == "200", "status_code": output}
    
    return results


# Usage
services = [
    {"name": "api-gateway",       "url": "http://api.internal/health"},
    {"name": "user-service",      "url": "http://users.internal/health"},
    {"name": "payment-service",   "url": "http://payments.internal/health"},
]
status = parallel_health_checks(services)
for name, info in status.items():
    icon = "OK" if info["healthy"] else "FAIL"
    print(f"[{icon}] {name}: HTTP {info['status_code']}")
```

[↑ Back to TOC](#table-of-contents)

---

## YAML and JSON Processing in Scripts

### Python: pyyaml and json

```python
import yaml
import json
from pathlib import Path

# Read a YAML config file
config_path = Path("/etc/myapp/config.yaml")
with config_path.open() as f:
    config = yaml.safe_load(f)    # safe_load prevents code execution exploits

# Always use safe_load — never yaml.load() without a Loader argument
# yaml.load(f) with untrusted input is a security vulnerability

# Modify and write back
config["database"]["pool_size"] = 20
config["feature_flags"]["new_checkout"] = True

with config_path.open("w") as f:
    yaml.dump(config, f, default_flow_style=False, sort_keys=False)


# JSON processing
import json, sys

# Pretty-print JSON from stdin
data = json.load(sys.stdin)
print(json.dumps(data, indent=2, sort_keys=True))

# Extract a nested field safely
def deep_get(d: dict, *keys, default=None):
    """Safely get a deeply nested value."""
    for key in keys:
        if not isinstance(d, dict):
            return default
        d = d.get(key, default)
    return d

# Example: extract image from kubectl get deployment output
deployment = json.loads(subprocess.check_output(
    ["kubectl", "get", "deployment", "myapp", "-o", "json"]
))
image = deep_get(deployment, "spec", "template", "spec", "containers", default=[{}])
print([c.get("image") for c in image])
```

### jq for JSON Processing in Shell

```bash
# jq is the shell-native JSON processor — faster than Python for pipeline work

# Extract a single field
kubectl get nodes -o json | jq '.items[].metadata.name'

# Filter and format
kubectl get pods -o json | jq -r '
  .items[]
  | select(.status.phase != "Running")
  | "\(.metadata.name) \(.status.phase)"
'

# Create a new JSON object from parts
kubectl get pods -o json | jq '[
  .items[]
  | {
    name: .metadata.name,
    image: .spec.containers[0].image,
    status: .status.phase,
    node: .spec.nodeName
  }
]'

# Update a field in a JSON file
jq '.replicas = 5' deployment.json > deployment-updated.json

# Merge two JSON objects
jq -s '.[0] * .[1]' base.json override.json

# Convert JSON array to CSV
jq -r '.[] | [.name, .status, .version] | @csv' services.json

# Sum values across an array
jq '[.items[].spec.containers[0].resources.requests.cpu 
     | rtrimstr("m") | tonumber] | add' pods.json
```

### yq for YAML Processing in Shell

```bash
# yq (mikefarah/yq) is jq for YAML

# Read a value
yq '.database.host' config.yaml

# Set a value (in place with -i)
yq -i '.replicas = 5' deployment.yaml
yq -i '.spec.template.spec.containers[0].image = "myapp:v2.3.1"' deployment.yaml

# Merge two YAML files
yq eval-all 'select(fileIndex == 0) * select(fileIndex == 1)' base.yaml override.yaml

# Convert YAML to JSON
yq -o json config.yaml

# Convert JSON to YAML
yq -P '.' config.json

# Extract all image references from a Kubernetes manifest directory
yq '.spec.template.spec.containers[].image' k8s/*.yaml 2>/dev/null | sort -u
```

[↑ Back to TOC](#table-of-contents)

---

## Secrets Management in Scripts

This is one of the most common areas where engineers make mistakes. Hard-coded credentials in scripts have caused more security incidents than almost any other single practice.

### What NOT to Do

```bash
# NEVER do this — credentials visible in process list and shell history
mysql -u root -pmysecretpassword -e "SELECT 1"

# NEVER do this — password stored in plain text in the script
DB_PASSWORD="mysecretpassword"
psql "postgresql://user:${DB_PASSWORD}@host/db"

# NEVER do this — .env files committed to git
echo "DB_PASSWORD=mysecretpassword" > .env
git add .env
git commit -m "add config"    # now the password is in git history forever
```

### Correct Patterns

```bash
# Pattern 1: Read from environment (set by systemd/Kubernetes/CI, not by the script)
# The secret is in the environment, not in the script or its history
DB_PASSWORD="${DB_PASSWORD:?'DB_PASSWORD must be set'}"

# Pattern 2: Read from a file that the application owns and is mode 400
DB_PASSWORD="$(cat /run/secrets/db-password)"

# Pattern 3: HashiCorp Vault CLI
DB_PASSWORD="$(vault kv get -field=password secret/myapp/db)"

# Pattern 4: AWS Secrets Manager (in an IAM-authenticated environment)
DB_PASSWORD="$(aws secretsmanager get-secret-value \
  --secret-id myapp/db/password \
  --query SecretString \
  --output text | jq -r '.password')"

# Pattern 5: Kubernetes — the container reads from a mounted secret volume
# The script just reads a file:
DB_PASSWORD="$(cat /var/run/secrets/db-password/password)"

# Always: mask secrets from logs when using set -x
{
  set +x
  export DB_PASSWORD="$(vault kv get -field=password secret/myapp/db)"
  set -x
}
```

### Detecting Secrets in Code

```bash
# gitleaks — scan for hardcoded secrets in git history
gitleaks detect --source . --verbose

# Scan a single file
gitleaks detect --source . --no-git --path ./scripts/deploy.sh

# Pre-commit hook with gitleaks
# .pre-commit-config.yaml:
# repos:
#   - repo: https://github.com/zricethezav/gitleaks
#     rev: v8.18.1
#     hooks:
#       - id: gitleaks

# truffleHog — another popular scanner
trufflehog git file://. --only-verified
```

[↑ Back to TOC](#table-of-contents)

---

## Shell Script Testing with bats

`bats` (Bash Automated Testing System) is the standard framework for testing shell scripts. Many teams skip this — and then spend hours debugging subtle shell bugs in production.

```bash
# Install bats-core
git clone https://github.com/bats-core/bats-core.git
cd bats-core && sudo ./install.sh /usr/local

# Or via package manager:
dnf install bats     # Fedora
brew install bats-core
```

### Writing bats Tests

```bash
#!/usr/bin/env bats
# test/deploy.bats

# Load bats helpers if installed
load 'test_helper/bats-support/load'
load 'test_helper/bats-assert/load'

# Setup runs before each test
setup() {
  # Create a temp directory for test files
  TEST_TMPDIR="$(mktemp -d)"
  export TEST_TMPDIR
  
  # Source the script under test (functions only, not main execution)
  source "${BATS_TEST_DIRNAME}/../scripts/deploy_lib.sh"
}

# Teardown runs after each test
teardown() {
  rm -rf "$TEST_TMPDIR"
}

@test "validate_version accepts semver" {
  run validate_version "1.2.3"
  assert_success
}

@test "validate_version rejects invalid format" {
  run validate_version "not-a-version"
  assert_failure
  assert_output --partial "Invalid version format"
}

@test "get_current_replicas returns integer" {
  # Mock kubectl by creating a fake one in PATH
  cat > "$TEST_TMPDIR/kubectl" << 'MOCK'
#!/bin/bash
echo '{"spec": {"replicas": 3}}'
MOCK
  chmod +x "$TEST_TMPDIR/kubectl"
  export PATH="$TEST_TMPDIR:$PATH"
  
  run get_current_replicas "myapp" "production"
  assert_success
  assert_output "3"
}

@test "rollback emits correct kubectl command" {
  run --separate-stderr rollback_deployment "myapp" "production"
  assert_output --partial "kubectl rollout undo"
}
```

```bash
# Run the tests
bats test/deploy.bats

# Run all tests in a directory
bats test/

# TAP output for CI
bats --formatter tap test/ > test-results.tap

# JUnit XML for CI (requires bats-support + bats-assert)
bats --formatter junit test/ > test-results.xml
```

[↑ Back to TOC](#table-of-contents)

---

## Makefile as a Task Runner

Makefiles are not just for C compilation. They are an excellent, dependency-aware task runner for DevOps workflows. Every project should have a Makefile at the root that documents how to build, test, and deploy the project.

```makefile
# Makefile — project task runner
# Usage: make <target>
# Run 'make help' to see all targets

SHELL := /bin/bash
.ONESHELL:            # run all recipe lines in a single shell
.DEFAULT_GOAL := help

# Variables with defaults (override with: make deploy VERSION=2.0.0)
SERVICE     ?= myapp
NAMESPACE   ?= production
VERSION     ?= $(shell git describe --tags --abbrev=0 2>/dev/null || echo "dev")
REGISTRY    ?= registry.example.com

.PHONY: help build test lint deploy rollback clean

##@ General
help: ## Show this help
	@awk 'BEGIN {FS = ":.*##"; printf "Usage: make \033[36m<target>\033[0m\n\n"} \
	  /^[a-zA-Z_-]+:.*?##/ { printf "  \033[36m%-20s\033[0m %s\n", $$1, $$2 } \
	  /^##@/ { printf "\n\033[1m%s\033[0m\n", substr($$0, 5) }' $(MAKEFILE_LIST)

##@ Development
lint: ## Run linters
	shellcheck scripts/*.sh
	python -m flake8 scripts/
	yamllint k8s/

test: ## Run test suite
	bats test/
	pytest tests/ -v

##@ Build
build: ## Build and push container image
	podman build -t $(REGISTRY)/$(SERVICE):$(VERSION) .
	podman push $(REGISTRY)/$(SERVICE):$(VERSION)

##@ Deployment
deploy: ## Deploy to Kubernetes (VERSION= required)
	@if [[ "$(NAMESPACE)" == "production" ]]; then \
	  read -p "Deploying to PRODUCTION. Are you sure? [y/N] " -n 1 -r; echo; \
	  [[ $$REPLY =~ ^[Yy]$$ ]] || exit 1; \
	fi
	kubectl set image deployment/$(SERVICE) \
	  $(SERVICE)=$(REGISTRY)/$(SERVICE):$(VERSION) \
	  -n $(NAMESPACE)
	kubectl rollout status deployment/$(SERVICE) -n $(NAMESPACE) --timeout=5m

rollback: ## Roll back last deployment
	kubectl rollout undo deployment/$(SERVICE) -n $(NAMESPACE)

##@ Cleanup
clean: ## Remove build artefacts
	rm -rf dist/ *.egg-info .pytest_cache __pycache__
	find . -name "*.pyc" -delete
```

```bash
# Using the Makefile
make help             # show all targets
make lint             # run linters
make test             # run tests
make build VERSION=2.3.1   # build specific version
make deploy VERSION=2.3.1 NAMESPACE=staging   # deploy to staging
make rollback NAMESPACE=production            # roll back production
```

[↑ Back to TOC](#table-of-contents)

---

## Automation Anti-Patterns to Avoid

Learning what not to do is as important as learning the correct patterns. These are the mistakes that cause production incidents.

### The "Galaxy Brain" Script

This is a script so clever that no one, including its author six months later, can understand what it does. Favour clarity over cleverness. A 50-line script that anyone on the team can read and modify is worth more than a 10-line masterpiece that only one person understands.

```bash
# Clever but unreadable — avoid this
IFS=$'\n' read -d '' -ra hosts < <(kubectl get nodes -o jsonpath='{range .items[*]}{.status.addresses[?(@.type=="InternalIP")].address}{"\n"}{end}' 2>/dev/null && printf '\0')

# Clear and readable — prefer this
HOSTS=()
while IFS= read -r ip; do
  [[ -n "$ip" ]] && HOSTS+=("$ip")
done < <(kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}' | tr ' ' '\n')
```

### The "Works on My Machine" Script

Scripts that assume a specific user, shell version, PATH, or tool location will fail in CI, in containers, or on a colleague's machine.

```bash
# Fragile — assumes specific PATH and shell features
python manage.py migrate  # which python? which manage.py?

# Robust
cd "$(dirname "$0")/.."   # always relative to script location
/usr/bin/env python3 ./manage.py migrate
# Or use a full virtualenv path:
./.venv/bin/python manage.py migrate
```

### The Silent Failure

```bash
# Dangerous — the script reports success even if the deployment failed
deploy_service "api"
deploy_service "worker"
deploy_service "scheduler"
echo "All services deployed"   # this always prints, even if deployments failed

# Safe — use set -e and check every exit code
set -euo pipefail

deploy_service "api"      || { echo "api deployment failed"; exit 1; }
deploy_service "worker"   || { echo "worker deployment failed"; exit 1; }
deploy_service "scheduler"|| { echo "scheduler deployment failed"; exit 1; }
echo "All services deployed successfully"
```

### The Undocumented Assumptions Script

```bash
# What is this script's purpose? What does it need? What does it change?
# Bad: no header, no usage, no documentation

# Good: always include a header block
#!/usr/bin/env bash
# -----------------------------------------------------------------------------
# rotate-logs.sh — Rotate application logs and upload to S3
#
# Usage: ./rotate-logs.sh [--dry-run] [--verbose]
#
# Required environment:
#   AWS_PROFILE      — AWS profile to use (or AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY)
#   LOG_DIR          — Directory containing logs to rotate (default: /var/log/myapp)
#   S3_BUCKET        — Target S3 bucket for archival
#
# Dependencies: aws-cli, gzip, bash >= 4.0
# Author: Jane Smith <jane@company.com>
# Last updated: 2026-01-15
# -----------------------------------------------------------------------------
set -euo pipefail
```

[↑ Back to TOC](#table-of-contents)

---

## Event-Driven Automation with inotifywait

Sometimes you need to react to filesystem changes: a config file was updated, a deployment artefact was deposited, a certificate was renewed. `inotifywait` from the `inotify-tools` package lets you watch for these events.

```bash
# Install
dnf install inotify-tools -y

# Watch a file and react to changes
while inotifywait -e close_write /etc/nginx/conf.d/; do
  echo "Config changed, reloading nginx..."
  nginx -t && systemctl reload nginx
done

# Watch a directory for new files (deployment pipeline trigger)
inotifywait -m -e create /opt/deployments/ | while read -r dir event file; do
  echo "New file: $dir$file"
  if [[ "$file" =~ \.tar\.gz$ ]]; then
    /opt/scripts/deploy-artifact.sh "$dir$file"
  fi
done

# Watch recursively with specific events
inotifywait -r -m -e modify,create,delete,move \
  --format '%T %w %f %e' \
  --timefmt '%Y-%m-%d %H:%M:%S' \
  /etc/myapp/ \
  >> /var/log/config-changes.log &

# Use --exclude to skip certain paths
inotifywait -r -m -e modify /var/www \
  --exclude '/var/www/cache' \
  | while read path action file; do
    echo "Changed: $path$file ($action)"
  done
```

A practical production use: auto-reload an application's configuration when a Kubernetes ConfigMap-mounted file changes. Since Kubernetes updates mounted ConfigMap files in-place, you can watch for those changes and send SIGHUP to the application process without a pod restart.

[↑ Back to TOC](#table-of-contents)

---

## Template Rendering with envsubst and gomplate

Hard-coding environment-specific values in configuration files is a common problem. The solution is templating: keep a single template file and render it differently per environment.

### envsubst — Simple Variable Substitution

`envsubst` is a small POSIX utility (part of GNU gettext) that replaces `${VAR}` and `$VAR` patterns in text with values from the environment.

```bash
# Template file: k8s/deployment.yaml.tpl
# apiVersion: apps/v1
# kind: Deployment
# metadata:
#   name: ${SERVICE_NAME}
# spec:
#   replicas: ${REPLICAS}
#   template:
#     spec:
#       containers:
#       - name: ${SERVICE_NAME}
#         image: ${IMAGE_REPO}/${SERVICE_NAME}:${VERSION}

# Render the template
export SERVICE_NAME=api
export REPLICAS=3
export IMAGE_REPO=registry.example.com
export VERSION=2.1.0

envsubst < k8s/deployment.yaml.tpl > k8s/deployment.yaml

# Only substitute specific variables (leave others as-is)
# Useful when the template has ${} syntax used for something else (e.g., nginx)
envsubst '${SERVICE_NAME} ${VERSION}' < nginx.conf.tpl > nginx.conf

# Common pattern: render and apply in one step
envsubst < k8s/deployment.yaml.tpl | kubectl apply -f -
```

### gomplate — Full-Featured Templating

`gomplate` is a more powerful templating tool that supports conditionals, loops, and multiple data sources (files, environment variables, Vault, AWS SSM, Consul).

```bash
# Install
curl -Lo /usr/local/bin/gomplate https://github.com/hairyhenderson/gomplate/releases/latest/download/gomplate_linux-amd64
chmod +x /usr/local/bin/gomplate

# Template file: config.yaml.tmpl
cat > config.yaml.tmpl << 'EOF'
database:
  host: {{ env.Getenv "DB_HOST" "localhost" }}
  port: {{ env.Getenv "DB_PORT" "5432" | conv.ToInt }}
  pool_size: {{ env.Getenv "DB_POOL_SIZE" "10" | conv.ToInt }}
  {{- if env.Getenv "DB_SSL_REQUIRED" "" }}
  sslmode: require
  {{- else }}
  sslmode: prefer
  {{- end }}

features:
  {{- range $k, $v := (env.Environ | strings.HasPrefix "FEATURE_") }}
  {{ $k | strings.TrimPrefix "FEATURE_" | strings.ToLower }}: {{ $v }}
  {{- end }}
EOF

# Render with environment variables
DB_HOST=db.internal \
DB_PORT=5432 \
DB_POOL_SIZE=20 \
DB_SSL_REQUIRED=true \
FEATURE_NEW_CHECKOUT=true \
FEATURE_DARK_MODE=false \
gomplate -f config.yaml.tmpl -o config.yaml

# Render using a data file as the source
gomplate -d config=config.json -f template.yaml.tmpl
```

[↑ Back to TOC](#table-of-contents)

---

## Automation War Story: The Deployment That Deleted Production

This is a composite story based on a real pattern that has caused incidents at multiple companies. Names and details are fictional.

You join a Series B fintech startup as its second DevOps engineer. On your first day, you are shown the deployment script. It is a 300-line bash file called `deploy.sh` that has grown organically over two years. Every engineer has added to it. It has no tests.

The script takes a service name and environment as arguments: `./deploy.sh payment-service production`. Under the hood, it SSHs to each production host in a loop, stops the service, deploys the new binary, and restarts the service.

Three weeks after you join, a developer deploys the payment service. They accidentally type `./deploy.sh payment-service` — omitting the environment argument. The script does not validate its arguments. It falls through to a default value of `"*"` that was set three months earlier by someone who thought it would be "useful to deploy everywhere at once."

The script then SSHs to every host in every environment — staging, QA, and production — and deploys an untested build of the payment service simultaneously. Production goes down. A dozen microservices that depend on payment-service start failing. Customers cannot complete purchases.

Recovery takes 45 minutes. The direct cost is significant. The post-mortem identifies 12 missing safeguards in the script. You end up rewriting it completely.

The rewrite includes:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Argument validation — fail loudly with usage if args are wrong
SERVICE="${1:?Usage: $0 <service> <environment>}"
ENVIRONMENT="${2:?Usage: $0 <service> <environment>}"

# Allowlist validation — reject anything not in the known list
VALID_ENVIRONMENTS=("staging" "qa" "production")
VALID_SERVICES=("api" "payment-service" "user-service" "order-service")

validate_in_list() {
  local value="$1" list_name="$2"
  local -n list="$2"
  for item in "${list[@]}"; do
    [[ "$item" == "$value" ]] && return 0
  done
  echo "Error: '$value' is not a valid ${list_name/VALID_/}" >&2
  exit 1
}

validate_in_list "$ENVIRONMENT" VALID_ENVIRONMENTS
validate_in_list "$SERVICE"     VALID_SERVICES

# Production requires explicit confirmation
if [[ "$ENVIRONMENT" == "production" ]]; then
  echo "WARNING: You are about to deploy $SERVICE to PRODUCTION"
  echo "Current version: $(get_current_version "$SERVICE" production)"
  echo "New version:     ${VERSION:?'VERSION must be set'}"
  read -rp "Type 'yes' to confirm: " confirm
  [[ "$confirm" == "yes" ]] || { echo "Aborted"; exit 0; }
fi

# Rest of the deployment logic...
```

The rewrite also adds a `--dry-run` flag, logs every action with timestamps, sends a Slack notification before and after each deployment, and checks that the service is healthy after deployment before proceeding to the next host.

The lesson: treat deployment scripts with the same care as production code. Test them. Validate inputs. Require confirmation for destructive operations. Make failures loud, not silent.

[↑ Back to TOC](#table-of-contents)

---

## Python: structlog for Structured Logging

Print statements and the standard `logging` module produce unstructured text that is hard to parse in log aggregation systems. `structlog` adds structured (key-value) logging to Python with minimal boilerplate.

```python
import structlog
import logging
import sys

# Configure structlog once at application startup
structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        # In production: JSONRenderer; in development: ConsoleRenderer
        structlog.processors.JSONRenderer()
        if not sys.stderr.isatty()
        else structlog.dev.ConsoleRenderer(),
    ],
    wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    context_class=dict,
    logger_factory=structlog.PrintLoggerFactory(),
)

log = structlog.get_logger()


def deploy_service(service: str, version: str, namespace: str) -> bool:
    """Deploy a service and return True on success."""
    log = structlog.get_logger().bind(
        service=service,
        version=version,
        namespace=namespace,
    )
    
    log.info("deployment_started")
    
    try:
        # ... deployment logic ...
        result = run_kubectl_set_image(service, version, namespace)
        log.info("deployment_completed", duration_ms=result.duration_ms)
        return True
    except KubectlError as e:
        log.error("deployment_failed", error=str(e), exit_code=e.returncode)
        return False


# Output (JSON, sent to stdout, ingested by Fluentd/Loki):
# {"service": "payment-service", "version": "2.1.0", "namespace": "production",
#  "event": "deployment_started", "level": "info", "timestamp": "2026-01-15T10:23:45Z"}
# {"service": "payment-service", "version": "2.1.0", "namespace": "production",
#  "event": "deployment_completed", "duration_ms": 12453, "level": "info", ...}
```

[↑ Back to TOC](#table-of-contents)

---

## Cron Patterns for Automation

Even though systemd timers are the modern recommendation, cron is still everywhere. You need to know the practical patterns that make cron jobs reliable.

### Preventing Overlapping Runs with flock

The single biggest cron reliability problem is two instances of the same job running simultaneously because the first one has not finished when the next scheduled time arrives.

```bash
# Use flock to ensure only one instance runs at a time
# If the lock cannot be acquired, the second instance exits immediately (-n)
*/5 * * * * /usr/bin/flock -n /tmp/my-job.lock /opt/scripts/my-job.sh

# Or with a timeout: wait up to 10 seconds for the lock before giving up
*/5 * * * * /usr/bin/flock -w 10 /tmp/my-job.lock /opt/scripts/my-job.sh
```

### Capturing Cron Output

Cron silently discards stdout and stderr unless configured otherwise. A job that fails silently is as good as a job that never ran.

```bash
# Option 1: Log to a file with timestamps
*/5 * * * * /opt/scripts/my-job.sh >> /var/log/my-job.log 2>&1

# Option 2: Log with timestamps using ts (part of moreutils)
*/5 * * * * /opt/scripts/my-job.sh 2>&1 | /usr/bin/ts '[%Y-%m-%d %H:%M:%S]' >> /var/log/my-job.log

# Option 3: systemd-cat — sends to journal with a tag
*/5 * * * * /usr/bin/systemd-cat -t my-job /opt/scripts/my-job.sh

# Option 4: Send failures to a Slack webhook
*/5 * * * * OUTPUT=$(/opt/scripts/my-job.sh 2>&1); \
  RC=$?; \
  [ $RC -ne 0 ] && curl -s -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"cron job my-job.sh failed (rc=$RC):\n$OUTPUT\"}" \
    "$SLACK_WEBHOOK_URL"
```

### Running Cron Jobs as a Specific User

```bash
# System crontab (/etc/cron.d/) allows specifying the user:
# min hour dom month dow user command
*/5 * * * * deploy /opt/scripts/sync-configs.sh

# Principle of least privilege: always run cron jobs as the most restricted
# user that can still do what the job needs
```

[↑ Back to TOC](#table-of-contents)

---

## Key Tools Reference Table

| Task | Tool | Command |
|------|------|---------|
| YAML/JSON processing | `yq` | `yq '.key' file.yaml` |
| JSON processing | `jq` | `jq '.items[].name' file.json` |
| Template rendering | `envsubst` | `envsubst < template.tpl` |
| Template rendering (advanced) | `gomplate` | `gomplate -f template.tmpl` |
| Bash testing | `bats` | `bats test/` |
| Python CLI | `click` | `@click.command()` |
| Python structured logging | `structlog` | `log = structlog.get_logger()` |
| Python testing | `pytest` | `pytest tests/ -v` |
| Shell linting | `shellcheck` | `shellcheck script.sh` |
| Python linting | `flake8` / `ruff` | `ruff check .` |
| Watch file changes | `inotifywait` | `inotifywait -m -e close_write /path/` |
| Secret scanning | `gitleaks` | `gitleaks detect --source .` |
| Prevent overlapping cron | `flock` | `flock -n /tmp/lock.lck script.sh` |
| Task runner | `make` | `make help` |
| Parallel execution | `xargs -P` | `xargs -P 8 -I{} ./process.sh {}` |
| Parallel execution | `parallel` | `parallel -j 8 ./process.sh ::: items` |

[↑ Back to TOC](#table-of-contents)

---

## Advanced sed and awk Patterns

`sed` and `awk` are two of the most powerful text processing tools in the Linux arsenal. Every DevOps engineer encounters them in legacy scripts and log processing pipelines, and they are still the right tool for many tasks.

### sed — Stream Editor

```bash
# Basic substitution
sed 's/old/new/' file.txt          # first occurrence per line
sed 's/old/new/g' file.txt         # all occurrences per line (global)
sed 's/old/new/gi' file.txt        # case-insensitive global

# In-place editing (always use -i.bak to keep a backup)
sed -i.bak 's/DEBUG/INFO/g' /etc/myapp/config.conf
# Remove the backup after confirming
rm /etc/myapp/config.conf.bak

# Address ranges — apply a command only to certain lines
sed '5,10s/old/new/g' file.txt     # lines 5-10 only
sed '/START/,/END/s/old/new/g' file.txt  # between START and END markers
sed '/^#/d' file.txt               # delete comment lines
sed '/^$/d' file.txt               # delete empty lines

# Capture groups (ERE with -E)
# Reformat "YYYY-MM-DD" to "DD/MM/YYYY"
sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/g' dates.txt

# Multi-line operations: append a line after a match
sed '/^server_name/a\\    # added by automation' nginx.conf

# Delete a line matching a pattern
sed '/unwanted_config/d' config.conf

# Print only matching lines (like grep but with sed)
sed -n '/ERROR/p' app.log

# Print line numbers of matching lines
sed -n '/ERROR/=' app.log
```

### awk — Data Processing

```bash
# awk basics: awk 'pattern { action }' file
# $0 = whole line, $1 = first field, NR = line number, NF = number of fields

# Print specific columns from output
df -h | awk 'NR>1 {print $5, $6}'    # disk usage % and mount point

# Sum a column
awk '{sum += $3} END {print "Total:", sum}' numbers.txt

# Filter rows and reformat
awk -F',' '$4 > 1000 {printf "%-20s %s\n", $1, $4}' sales.csv

# Process apache/nginx access logs — top 10 IP addresses
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# Calculate average response time from nginx logs
# Assumes last field is response time in milliseconds
awk '{sum += $NF; count++} END {print "Avg:", sum/count, "ms"}' access.log

# Multi-file processing — add filename to output
awk '{print FILENAME ":" NR ": " $0}' /etc/hosts /etc/passwd | grep "root"

# BEGIN and END blocks
awk '
  BEGIN { print "=== Report ===" }
  /ERROR/ { errors++ }
  /WARN/  { warns++ }
  END {
    print "Errors:", errors+0
    print "Warnings:", warns+0
  }
' app.log

# Process CSV — handle quoted fields with FS trick
awk -v FPAT='([^,]+)|(\"[^\"]+\")' '{print $2}' data.csv

# Format output as a table with printf
awk '{printf "%-15s %-10s %s\n", $1, $2, $3}' data.txt
```

[↑ Back to TOC](#table-of-contents)

---

## Python: dataclasses and Type Hints for Configuration

When your automation scripts grow beyond 100 lines, unstructured dictionaries become hard to maintain. Python dataclasses provide a clean way to represent configuration with type validation.

```python
from dataclasses import dataclass, field
from typing import Optional, List
import os


@dataclass
class DatabaseConfig:
    host: str
    port: int = 5432
    name: str = "app"
    user: str = "app"
    password: str = field(default="", repr=False)  # repr=False hides from logs
    pool_size: int = 10
    ssl_required: bool = False

    @classmethod
    def from_env(cls) -> "DatabaseConfig":
        """Build config from environment variables."""
        return cls(
            host=os.environ["DB_HOST"],
            port=int(os.getenv("DB_PORT", "5432")),
            name=os.getenv("DB_NAME", "app"),
            user=os.getenv("DB_USER", "app"),
            password=os.environ["DB_PASSWORD"],
            pool_size=int(os.getenv("DB_POOL_SIZE", "10")),
            ssl_required=os.getenv("DB_SSL_REQUIRED", "").lower() in ("true", "1", "yes"),
        )


@dataclass
class DeploymentConfig:
    service: str
    version: str
    namespace: str = "production"
    replicas: Optional[int] = None
    extra_labels: dict = field(default_factory=dict)
    health_check_timeout: int = 300  # seconds

    def __post_init__(self):
        """Validate configuration after init."""
        if not self.version:
            raise ValueError("version cannot be empty")
        if self.replicas is not None and self.replicas < 1:
            raise ValueError(f"replicas must be >= 1, got {self.replicas}")


# Usage
config = DeploymentConfig(
    service="payment-api",
    version="2.3.1",
    namespace="production",
    replicas=5,
)
print(config)
# DeploymentConfig(service='payment-api', version='2.3.1', namespace='production',
#                  replicas=5, extra_labels={}, health_check_timeout=300)
```

[↑ Back to TOC](#table-of-contents)

---

## Bash Script Best Practices Checklist

Before any script goes into production, review it against this checklist:

**Safety**
- `set -euo pipefail` is the first non-comment line
- All user-supplied inputs are validated before use
- Destructive operations require explicit confirmation or a `--force` flag
- The script handles SIGTERM and cleans up temporary files via a trap

**Correctness**
- Variables are quoted: `"$var"` not `$var`
- Arrays are iterated with `"${array[@]}"` not `$array`
- Paths with spaces are handled: `"$dir/$file"` not `$dir/$file`
- Integer comparisons use `-eq`/`-lt`; string comparisons use `==`/`!=`
- `[[ ]]` is used instead of `[ ]` for conditional expressions
- `$(...)` is used instead of backticks

**Maintainability**
- Script has a header block with description, usage, and dependencies
- Functions have single responsibilities and meaningful names
- Constants are declared with `readonly` or `declare -r`
- Long scripts have a `main()` function called at the end
- Complex regex or awk programs have a comment explaining the logic

**Observability**
- The script logs start, completion, and key milestones
- Errors include enough context to diagnose the failure without re-running
- Exit codes are meaningful (0 = success, 1 = general error, 2 = usage error)

**Testing**
- Pure-logic functions are extracted into a library file and tested with bats
- The script can be tested with `--dry-run` without causing side effects
- CI runs shellcheck against the script and fails on warnings

[↑ Back to TOC](#table-of-contents)

---

## Python: Writing Idiomatic Automation Scripts

### Context Managers for Resource Safety

```python
import contextlib
import tempfile
import os


@contextlib.contextmanager
def temporary_kubeconfig(cluster_name: str, token: str):
    """
    Create a temporary kubeconfig file and yield its path.
    Always cleans up the file, even if an exception is raised.
    """
    kubeconfig = {
        "apiVersion": "v1",
        "kind": "Config",
        "clusters": [{"name": cluster_name, "cluster": {"server": "https://k8s.example.com"}}],
        "users": [{"name": "ci-runner", "user": {"token": token}}],
        "contexts": [{"name": cluster_name, "context": {"cluster": cluster_name, "user": "ci-runner"}}],
        "current-context": cluster_name,
    }
    
    with tempfile.NamedTemporaryFile(
        mode='w', suffix='.yaml', prefix='kubeconfig-', delete=False
    ) as f:
        import yaml
        yaml.dump(kubeconfig, f)
        tmpfile = f.name
    
    try:
        old_kubeconfig = os.environ.get('KUBECONFIG')
        os.environ['KUBECONFIG'] = tmpfile
        yield tmpfile
    finally:
        os.environ.pop('KUBECONFIG', None)
        if old_kubeconfig:
            os.environ['KUBECONFIG'] = old_kubeconfig
        os.unlink(tmpfile)


# Usage
with temporary_kubeconfig("prod-cluster", os.environ["K8S_TOKEN"]) as kubeconfig:
    subprocess.run(["kubectl", "get", "pods", "-n", "production"], check=True)
# Kubeconfig file is deleted and env var restored here, even on exception
```

### Error Hierarchy for Automation Tools

```python
class AutomationError(Exception):
    """Base class for all automation errors."""
    def __init__(self, message: str, exit_code: int = 1):
        super().__init__(message)
        self.exit_code = exit_code


class DeploymentError(AutomationError):
    """Deployment-specific errors."""
    pass


class HealthCheckError(AutomationError):
    """Health check failed after deployment."""
    def __init__(self, service: str, attempts: int):
        super().__init__(
            f"Health check for {service} failed after {attempts} attempts",
            exit_code=2,
        )
        self.service = service
        self.attempts = attempts


class RollbackError(AutomationError):
    """Rollback itself failed — requires human intervention."""
    pass


# Usage in main
def main():
    try:
        deploy("payment-service", "2.1.0", "production")
    except HealthCheckError as e:
        log.error("health_check_failed", service=e.service, attempts=e.attempts)
        try:
            rollback("payment-service", "production")
        except RollbackError as re:
            log.critical("rollback_failed_human_intervention_required", error=str(re))
            sys.exit(3)
        sys.exit(e.exit_code)
    except DeploymentError as e:
        log.error("deployment_failed", error=str(e))
        sys.exit(e.exit_code)
```

[↑ Back to TOC](#table-of-contents)

---

## Practical Regex Patterns for Automation

Regular expressions appear constantly in automation work: parsing log lines, validating inputs, extracting fields from command output. These patterns cover the most common DevOps scenarios.

```bash
# IP address validation (IPv4)
IP_PATTERN='^(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$'
[[ "$ip" =~ $IP_PATTERN ]] && echo "valid" || echo "invalid"

# Semantic version validation
SEMVER='^[0-9]+\.[0-9]+\.[0-9]+(-[a-zA-Z0-9.]+)?(\+[a-zA-Z0-9.]+)?$'
[[ "$version" =~ $SEMVER ]] || { echo "Invalid version: $version"; exit 1; }

# Extract version number from Docker image reference
# e.g., "registry.example.com/myapp:2.3.1-alpine" → "2.3.1-alpine"
version=$(echo "$image" | grep -oP '(?<=:)[^@]+')

# Extract timestamp from log line
# e.g., "2026-01-15T10:23:45.123Z ERROR something happened"
timestamp=$(echo "$line" | grep -oP '^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}')

# Find all Kubernetes resource names in a YAML manifest
grep -oP '(?<=name: )[^\s]+' manifest.yaml

# Extract HTTP status codes from nginx access log
grep -oP '" [0-9]{3} ' access.log | grep -oP '[0-9]{3}' | sort | uniq -c | sort -rn

# Validate hostname
HOSTNAME_PATTERN='^[a-zA-Z0-9]([a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?(\.[a-zA-Z0-9]([a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*$'
[[ "$hostname" =~ $HOSTNAME_PATTERN ]] || { echo "Invalid hostname"; exit 1; }
```

```python
import re

# Extract all environment variable references from a template
ENVVAR_PATTERN = re.compile(r'\$\{([A-Z_][A-Z0-9_]*)\}|\$([A-Z_][A-Z0-9_]*)')

def get_template_vars(template: str) -> set[str]:
    """Return all ${VAR} and $VAR references in a template string."""
    matches = ENVVAR_PATTERN.findall(template)
    return {m[0] or m[1] for m in matches}


# Extract key=value pairs from a flat config file
CONFIG_LINE = re.compile(r'^\s*([A-Z_][A-Z0-9_]*)\s*=\s*(.+?)\s*$', re.IGNORECASE)

def parse_env_file(path: str) -> dict[str, str]:
    """Parse a KEY=VALUE config file, ignoring comments and blank lines."""
    result = {}
    with open(path) as f:
        for line in f:
            line = line.strip()
            if not line or line.startswith('#'):
                continue
            m = CONFIG_LINE.match(line)
            if m:
                result[m.group(1)] = m.group(2).strip('"\'')
    return result
```

[↑ Back to TOC](#table-of-contents)

---

## Scripting for Cloud APIs: AWS boto3 Patterns

Most production environments live in a cloud provider. Writing automation that interacts with cloud APIs is a core DevOps skill. Here are the boto3 patterns you will use most often.

### Session and Client Initialisation

```python
import boto3
from botocore.exceptions import ClientError
import logging

log = logging.getLogger(__name__)


def get_boto3_session(role_arn: str | None = None, region: str = "eu-west-1") -> boto3.Session:
    """
    Get a boto3 session, optionally assuming a cross-account role.
    Falls back to environment variables / instance profile automatically.
    """
    if role_arn:
        sts = boto3.client("sts", region_name=region)
        creds = sts.assume_role(
            RoleArn=role_arn,
            RoleSessionName="automation-script",
            DurationSeconds=3600,
        )["Credentials"]
        return boto3.Session(
            aws_access_key_id=creds["AccessKeyId"],
            aws_secret_access_key=creds["SecretAccessKey"],
            aws_session_token=creds["SessionToken"],
            region_name=region,
        )
    return boto3.Session(region_name=region)


# Usage
session = get_boto3_session(role_arn="arn:aws:iam::123456789:role/deploy-role")
ec2 = session.client("ec2")
```

### EC2 Instance Management

```python
def get_instances_by_tag(session: boto3.Session, tag_name: str, tag_value: str) -> list[dict]:
    """Return all EC2 instances with a specific tag."""
    ec2 = session.client("ec2")
    paginator = ec2.get_paginator("describe_instances")
    
    instances = []
    for page in paginator.paginate(
        Filters=[
            {"Name": f"tag:{tag_name}", "Values": [tag_value]},
            {"Name": "instance-state-name", "Values": ["running"]},
        ]
    ):
        for reservation in page["Reservations"]:
            instances.extend(reservation["Instances"])
    
    return instances


def rotate_asg_instances(session: boto3.Session, asg_name: str) -> None:
    """Trigger a rolling replacement of all instances in an ASG."""
    autoscaling = session.client("autoscaling")
    
    log.info("starting_instance_refresh", asg_name=asg_name)
    response = autoscaling.start_instance_refresh(
        AutoScalingGroupName=asg_name,
        Strategy="Rolling",
        Preferences={
            "MinHealthyPercentage": 90,
            "InstanceWarmup": 300,
        },
    )
    refresh_id = response["InstanceRefreshId"]
    log.info("instance_refresh_started", refresh_id=refresh_id)
    return refresh_id
```

### SSM Parameter Store for Secrets

```python
def get_parameter(session: boto3.Session, name: str, decrypt: bool = True) -> str:
    """Fetch a parameter from SSM Parameter Store."""
    ssm = session.client("ssm")
    try:
        response = ssm.get_parameter(Name=name, WithDecryption=decrypt)
        return response["Parameter"]["Value"]
    except ClientError as e:
        if e.response["Error"]["Code"] == "ParameterNotFound":
            raise KeyError(f"SSM parameter not found: {name}") from e
        raise


def put_parameter(
    session: boto3.Session,
    name: str,
    value: str,
    parameter_type: str = "SecureString",
    overwrite: bool = True,
) -> None:
    """Write a parameter to SSM Parameter Store."""
    ssm = session.client("ssm")
    ssm.put_parameter(
        Name=name,
        Value=value,
        Type=parameter_type,
        Overwrite=overwrite,
    )
    log.info("ssm_parameter_updated", name=name)
```

[↑ Back to TOC](#table-of-contents)

---

## Summary: The Scripting and Automation Mindset

The difference between a script that solves a problem once and an automation system that reliably solves a class of problems comes down to a few core principles:

**Design for failure.** Every external system you call — Kubernetes, AWS, a database, a REST API — will fail eventually. Your scripts must handle failures gracefully: retry transient errors, report permanent failures clearly, and never leave systems in an inconsistent state.

**Design for observability.** A script that runs in a cron job or CI pipeline and produces no output is as useful as a black box. Every significant action should be logged. Every failure should include enough context to diagnose the problem without re-running the script.

**Design for idempotency.** Running a script twice should produce the same result as running it once. This makes your automation safe to retry, safe to run in parallel, and safe to re-run after partial failures. The easiest way to achieve this is to check the current state before making changes: "if the deployment already has this image, skip it."

**Design for the next person.** Your script will be read, modified, and debugged by future engineers — possibly under pressure during an incident. Make their job easy: clear variable names, short functions, helpful comments for non-obvious decisions.

**Start simple.** The best automation is often a well-crafted 30-line bash script with good error handling. Do not reach for Python, Ansible, or Terraform until bash stops being the right tool. Complexity is a cost, and simple solutions are easier to debug when something goes wrong at 3am.

[↑ Back to TOC](#table-of-contents)

---

## Debugging Automation Scripts

When a script fails in CI or on a production system, you need fast, systematic debugging tools.

### Bash Debugging Techniques

```bash
# Trace execution — prints every command before running it
set -x
# Or run the script with:
bash -x ./deploy.sh

# Print a subset of lines with set -x / set +x
set -x
complex_function
set +x

# Dry run pattern — run the script but echo commands instead of executing
DRY_RUN="${DRY_RUN:-false}"
run_cmd() {
  if [[ "$DRY_RUN" == "true" ]]; then
    echo "[DRY RUN] $*"
  else
    "$@"
  fi
}
run_cmd kubectl apply -f deployment.yaml

# Debug with bashdb (bash debugger)
bashdb ./script.sh
# Inside bashdb: step, next, continue, print $var, break 42

# Check what a script sends to stdout vs stderr
./script.sh > /tmp/stdout.txt 2> /tmp/stderr.txt
echo "Exit code: $?"

# Trace a running bash process
strace -e trace=execve -p $(pgrep -f "deploy.sh") 2>&1 | head -50
```

### Python Debugging in Automation

```python
import pdb
import os

# Drop into debugger at a specific point
# (never in production — use only in development)
if os.getenv("DEBUG_DEPLOY"):
    pdb.set_trace()

# Use breakpoint() in Python 3.7+ (respects PYTHONBREAKPOINT env var)
if os.getenv("DEBUG_DEPLOY"):
    breakpoint()

# Remote debugging with debugpy (works with VS Code and PyCharm)
# pip install debugpy
import debugpy
debugpy.listen(5678)
print("Waiting for debugger attachment on port 5678...")
debugpy.wait_for_client()

# Better logging for debugging without a debugger
import structlog
structlog.configure(wrapper_class=structlog.make_filtering_bound_logger(0))  # DEBUG level
log = structlog.get_logger()
log.debug("entering_function", service=service, version=version)
```

### Common Debugging Scenarios

```bash
# Script exits unexpectedly with set -e — find out which line
# Add this at the top:
trap 'echo "Script failed at line $LINENO: $BASH_COMMAND" >&2' ERR

# Variable is empty when it should not be
set -u  # makes referencing undefined variables an error
# Or check explicitly:
echo "DB_HOST=${DB_HOST:-<unset>}"
declare -p DB_HOST 2>&1  # shows if the variable exists and its attributes

# Command not found in CI but works locally
# Check PATH differences:
echo "PATH: $PATH"
which kubectl  # is it in a non-standard location?
type -a python3  # is there a function/alias shadowing it?

# Script works interactively but fails in cron
# Cron uses a minimal environment — add explicit PATH
PATH=/usr/local/bin:/usr/bin:/bin
# Or use full paths everywhere:
/usr/local/bin/kubectl apply -f /opt/k8s/deployment.yaml

# File encoding issues (Windows CRLF line endings in a script)
file script.sh          # should say "ASCII text" not "with CRLF line terminators"
sed -i 's/\r//' script.sh    # remove CR characters
dos2unix script.sh            # or use dos2unix
```

[↑ Back to TOC](#table-of-contents)

---

## Scripting for Kubernetes Operations

Kubernetes is the operational environment for most modern applications. These are the scripting patterns you will use to automate Kubernetes operations.

### Waiting for Resources

A common automation requirement is waiting until a resource reaches a desired state before continuing.

```bash
# Wait for a deployment to complete rollout
kubectl rollout status deployment/myapp -n production --timeout=5m

# Wait for a pod to be ready (useful after creating a job)
kubectl wait pod -l job-name=migration -n production \
  --for=condition=Ready --timeout=120s

# Wait for a node to become Ready after a reboot
NODE="worker-01"
kubectl wait node/"$NODE" --for=condition=Ready --timeout=5m

# Custom wait loop — for resources that kubectl wait doesn't support well
wait_for_endpoint() {
  local url="$1" max_attempts="${2:-60}" interval="${3:-5}"
  local attempt=0
  while (( attempt++ < max_attempts )); do
    if curl -sf "$url" > /dev/null 2>&1; then
      echo "Endpoint ready: $url"
      return 0
    fi
    echo "Attempt $attempt/$max_attempts: waiting for $url"
    sleep "$interval"
  done
  echo "Timed out waiting for: $url" >&2
  return 1
}

wait_for_endpoint "http://myapp.production.svc/health" 60 5
```

### Extracting Data from kubectl

```bash
# Get a specific field with jsonpath
kubectl get deployment myapp -n production \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Get multiple fields as tab-separated values
kubectl get pods -n production \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Get all container images across all namespaces
kubectl get pods --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}: {range .spec.containers[*]}{.image}{"\n"}{end}{end}' \
  | sort

# Use go-template for more complex formatting
kubectl get nodes -o go-template='
{{- range .items}}
{{.metadata.name}} {{index .metadata.labels "kubernetes.io/role"}} {{.status.conditions | last | .status}}
{{- end}}'

# Output as custom columns (no external tools needed)
kubectl get pods -n production \
  -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,IMAGE:.spec.containers[0].image'
```

[↑ Back to TOC](#table-of-contents)

---

## Choosing Between Bash and Python

A common question is when to write a bash script versus a Python script. Here is a practical guide.

**Choose bash when:**
- The task is primarily about running other commands and handling their output
- The logic is simple enough to fit in under 50-100 lines without becoming hard to read
- You need to pipe commands together or work with stdout/stderr directly
- You are writing a quick one-off script or a simple wrapper
- The target systems are guaranteed to have bash but may not have Python

**Choose Python when:**
- The logic involves complex data structures (nested dicts, lists of objects)
- You need robust error handling with meaningful exception types
- The script will be tested with a test framework
- You need to call REST APIs, parse JSON/YAML, or do complex string processing
- The script is long-lived and will be maintained by multiple engineers
- You need async operations, threading, or concurrency
- You are building something that others will import as a library

**The "200-line rule":** if your bash script exceeds 200 lines, seriously consider rewriting it in Python. Long bash scripts become unmaintainable quickly. Python's syntax for functions, classes, error handling, and data structures scales much better.

**Never:** mix heavy logic between bash and Python in the same script. If a bash script calls a Python heredoc for the "complex parts", you have two languages to debug and test. Choose one language for the dominant logic.

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/). Non-commercial use only. Share alike with attribution. See [LICENSE.md](./LICENSE.md).*
