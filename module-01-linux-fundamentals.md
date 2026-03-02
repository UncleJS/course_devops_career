# Module 01: Linux Fundamentals

> Part of the [DevOps Career Course](./README.md) by UncleJS

---

## Table of Contents

- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [Beginner: Navigation & File Management](#beginner-navigation--file-management)
- [Beginner: File Operations](#beginner-file-operations)
- [Beginner: File Permissions](#beginner-file-permissions)
- [Beginner: Text Editors](#beginner-text-editors)
- [Beginner: Piping & Redirection](#beginner-piping--redirection)
- [Beginner: System & Process Management](#beginner-system--process-management)
- [Beginner: SSH & Remote Access](#beginner-ssh--remote-access)
- [Beginner: Command History & Shortcuts](#beginner-command-history--shortcuts)
- [Beginner: Root Access & User Management](#beginner-root-access--user-management)
- [Beginner: Network Commands](#beginner-network-commands)
- [Beginner: System Information](#beginner-system-information)
- [Intermediate: Chaining Commands](#intermediate-chaining-commands)
- [Intermediate: Package Management](#intermediate-package-management)
- [Intermediate: Systemd & Services](#intermediate-systemd--services)
- [Tools & Commands Reference](#tools--commands-reference)
- [Hands-On Labs](#hands-on-labs)
- [Further Reading](#further-reading)

---

## Overview

Linux is the operating system that runs the internet. Over 90% of cloud servers, containers, and production systems run on Linux. Before you can work with Docker, Kubernetes, Terraform, or any DevOps tool, you need to be comfortable on the command line.

This module covers everything you need to navigate, manage, and administer a Linux system from the terminal — the foundational skill for every topic in this course.

[↑ Back to TOC](#table-of-contents)

---

## Learning Objectives

By the end of this module you will be able to:

- Navigate a Linux filesystem and manage files and directories confidently
- Read, search, and manipulate file contents from the command line
- Understand and configure file permissions and ownership
- Edit files using `vi`/`vim` and `nano`
- Pipe commands together and redirect input/output
- Monitor and manage running processes
- Connect to remote servers securely using SSH
- Use `sudo`, `su`, and manage user accounts
- Chain multiple commands to solve real-world tasks

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Navigation & File Management

The Linux filesystem is a tree structure that starts at `/` (the root). Everything — files, devices, processes — is a file in Linux.

### Key Directories

| Path | Description |
|---|---|
| `/` | Root of the filesystem |
| `/home` | User home directories |
| `/etc` | System configuration files |
| `/var` | Variable data (logs, databases, mail) |
| `/tmp` | Temporary files (cleared on reboot) |
| `/usr` | User programs and libraries |
| `/bin` | Essential binaries |
| `/proc` | Virtual filesystem for process/system info |

### Navigation Commands

```bash
pwd                     # Print working directory (where am I right now?)
ls                      # List files in current directory
ls -la                  # List all files (including hidden) with details
ls -lh                  # Human-readable file sizes
cd /var/log             # Change to /var/log
cd ..                   # Go up one directory
cd ~                    # Go to your home directory
cd -                    # Go back to previous directory
```

### File & Directory Management

```bash
mkdir projects          # Create a directory
mkdir -p a/b/c          # Create nested directories in one command
rm file.txt             # Delete a file
rm -r folder/           # Delete a directory and all its contents
rm -rf folder/          # Force delete without prompts (use with caution!)
cp source.txt dest.txt  # Copy a file
cp -r src/ dst/         # Copy a directory recursively
mv old.txt new.txt      # Rename a file
mv file.txt /tmp/       # Move a file to /tmp
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: File Operations

```bash
cat file.txt            # Print entire file to screen
less file.txt           # Page through a file (q to quit, / to search)
head -n 20 file.txt     # Show first 20 lines
tail -n 20 file.txt     # Show last 20 lines
tail -f /var/log/syslog # Follow a file in real time (great for logs)
grep "error" file.txt   # Find lines containing "error"
grep -i "error" file.txt  # Case-insensitive search
grep -r "TODO" ./src/   # Recursive search through directory
diff file1.txt file2.txt  # Show differences between two files
wc -l file.txt          # Count lines in a file
wc -w file.txt          # Count words
find /var -name "*.log" # Find all .log files under /var
find . -type f -mtime -1  # Files modified in the last 1 day
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: File Permissions

Every file in Linux has three permission sets: **owner**, **group**, and **others**. Each set has three bits: **read (r)**, **write (w)**, **execute (x)**.

### Reading Permissions

```
-rwxr-xr--  1  alice  devops  1234  Jan 1 12:00  script.sh
 |||||||||||
 |rwx        = owner (alice) can read, write, execute
    r-x      = group (devops) can read, execute
       r--   = others can only read
```

### Octal Notation

| Octal | Permissions | Meaning |
|---|---|---|
| `7` | `rwx` | Read + Write + Execute |
| `6` | `rw-` | Read + Write |
| `5` | `r-x` | Read + Execute |
| `4` | `r--` | Read only |
| `0` | `---` | No permissions |

### chmod & chown

```bash
chmod 755 script.sh     # rwxr-xr-x — owner all, group/others read+execute
chmod 644 config.txt    # rw-r--r-- — owner read/write, others read
chmod +x deploy.sh      # Add execute permission for everyone
chmod u+x deploy.sh     # Add execute for owner only
chmod -R 755 /var/www/  # Recursively set permissions

chown alice file.txt          # Change owner to alice
chown alice:devops file.txt   # Change owner and group
chown -R alice:devops /app/   # Recursively change ownership
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Text Editors

### vim / vi

`vim` is the most powerful terminal editor — essential for editing configs on remote servers.

```bash
vim file.txt        # Open file in vim

# MODES:
# Normal mode   — default, navigate and run commands
# Insert mode   — press 'i' to enter, type text, Esc to exit
# Command mode  — press ':' in normal mode

# Essential commands (in normal mode):
i           # Enter insert mode
Esc         # Return to normal mode
:w          # Save (write)
:q          # Quit
:wq         # Save and quit
:q!         # Quit without saving (force)
dd          # Delete current line
yy          # Copy (yank) current line
p           # Paste below current line
/word       # Search for "word"
n           # Jump to next search result
u           # Undo
Ctrl+r      # Redo
gg          # Go to first line
G           # Go to last line
:set number # Show line numbers
```

### nano

`nano` is beginner-friendly — all shortcuts are shown at the bottom of the screen.

```bash
nano file.txt       # Open file in nano

Ctrl+O              # Save (Write Out)
Ctrl+X              # Exit
Ctrl+W              # Search
Ctrl+K              # Cut current line
Ctrl+U              # Paste (Uncut)
Ctrl+G              # Help
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Piping & Redirection

Piping and redirection are how you chain Linux commands together — a fundamental DevOps skill.

### Redirection

```bash
command > file.txt      # Write stdout to file (overwrites)
command >> file.txt     # Append stdout to file
command < file.txt      # Read stdin from file
command 2> error.log    # Write stderr to error.log
command &> all.log      # Write both stdout and stderr to all.log
command 2>/dev/null     # Discard error output
```

### Pipes

The `|` operator sends the **output** of one command as the **input** to the next.

```bash
ls -la | grep ".log"             # List files, filter for .log
ps aux | grep nginx              # Find nginx processes
cat access.log | sort | uniq    # Sort and deduplicate
cat access.log | wc -l          # Count log entries
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: System & Process Management

```bash
ps aux              # Show all running processes
ps aux | grep nginx # Find specific process
top                 # Interactive process viewer (q to quit)
htop                # Enhanced process viewer (install separately)
kill 1234           # Send SIGTERM to process with PID 1234
kill -9 1234        # Send SIGKILL (force terminate) to PID 1234
killall nginx       # Kill all processes named nginx

df -h               # Show disk usage (human-readable)
df -hT              # Show disk usage with filesystem type
du -sh /var/log/    # Show size of /var/log directory
free -h             # Show RAM and swap usage
uptime              # How long the system has been running + load average
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: SSH & Remote Access

SSH (Secure Shell) is how you connect to remote servers — you will use this constantly as a DevOps engineer.

```bash
ssh user@hostname           # Connect to remote host
ssh user@192.168.1.10       # Connect using IP address
ssh -p 2222 user@host       # Connect on a non-standard port
ssh -i ~/.ssh/my_key user@host  # Connect using a specific key

# Generate an SSH key pair
ssh-keygen -t ed25519 -C "your_email@example.com"
# Creates: ~/.ssh/id_ed25519 (private) and ~/.ssh/id_ed25519.pub (public)

# Copy your public key to a remote server
ssh-copy-id user@hostname

# Securely copy files
scp file.txt user@host:/remote/path/     # Local → Remote
scp user@host:/remote/file.txt ./        # Remote → Local
scp -r folder/ user@host:/remote/path/  # Copy directory

# SSH config file (~/.ssh/config) for shortcuts:
# Host myserver
#     HostName 192.168.1.10
#     User alice
#     IdentityFile ~/.ssh/mykey
# Then just: ssh myserver
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Command History & Shortcuts

```bash
history             # Show command history
history | grep ssh  # Search history for ssh commands
!!                  # Re-run the last command
!git                # Re-run the last command starting with "git"
!123                # Re-run command number 123 from history
Ctrl+R              # Reverse search through history (type to search)
Ctrl+C              # Cancel the current command
Ctrl+L              # Clear the terminal screen
Ctrl+A              # Jump to beginning of line
Ctrl+E              # Jump to end of line
Ctrl+U              # Delete from cursor to beginning of line
Tab                 # Auto-complete commands and file paths
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Root Access & User Management

```bash
sudo command            # Run command as root (superuser)
sudo -i                 # Open a root shell session
su username             # Switch to another user
su -                    # Switch to root user
passwd                  # Change your own password
sudo passwd username    # Change another user's password

# User management
sudo useradd -m alice           # Create user alice with home directory
sudo usermod -aG sudo alice     # Add alice to sudo group
sudo userdel alice              # Delete user alice
id                              # Show current user's UID, GID, groups
whoami                          # Print current username
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Network Commands

```bash
ping google.com             # Test network connectivity
ping -c 4 192.168.1.1       # Send exactly 4 pings
curl https://example.com    # Fetch a URL
curl -I https://example.com # Fetch HTTP headers only
curl -o file.txt https://example.com/file  # Download and save
nc -zv hostname 80          # Test if port 80 is open (netcat)
nc -l 8080                  # Listen on port 8080
netstat -tulnp              # Show listening ports and services
ss -tulnp                   # Modern replacement for netstat
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: System Information

```bash
uname -a            # Full kernel and system information
uname -r            # Kernel version only
hostname            # Show system hostname
hostname -I         # Show all IP addresses
who                 # Show who is logged in
w                   # Show who is logged in and what they're doing
last                # Show login history
lsb_release -a      # Show Linux distribution info
cat /etc/os-release # Distribution info (works everywhere)
lscpu               # CPU information
lsblk               # List block devices (disks and partitions)
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Chaining Commands

Combine commands to solve real-world DevOps tasks in one line.

```bash
# Find all text files
find . -name "*.txt"

# Find and read all text files
find . -name "*.txt" -exec cat {} \;

# Find lines containing "ERROR" in all log files
find /var/log -name "*.log" -exec grep -l "ERROR" {} \;

# Extract, sort, and deduplicate error messages
grep -r "ERROR" /var/log/ | awk '{print $NF}' | sort | uniq -c | sort -rn

# Save error results to a file
grep -r "ERROR" /var/log/ > /tmp/errors.txt

# Copy all log files to a backup folder preserving directory structure
find /var/log -name "*.log" | while read f; do
  cp --parents "$f" /backup/
done

# Archive logs older than 7 days
find /var/log -name "*.log" -mtime +7 -exec gzip {} \;

# Monitor a log file and alert on errors
tail -f /var/log/app.log | grep --line-buffered "ERROR"

# Count HTTP status codes in access log
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Show top 10 largest files in a directory
du -ah /var | sort -rh | head -10
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Package Management

```bash
# Debian/Ubuntu (apt)
sudo apt update                 # Update package index
sudo apt upgrade                # Upgrade all packages
sudo apt install nginx          # Install a package
sudo apt remove nginx           # Remove a package
sudo apt autoremove             # Remove unused dependencies
apt search nginx                # Search for a package
apt show nginx                  # Show package details
dpkg -l | grep nginx            # List installed packages matching nginx

# RHEL/Rocky Linux/CentOS (dnf/yum)
sudo dnf update                 # Update all packages
sudo dnf install nginx          # Install a package
sudo dnf remove nginx           # Remove a package
sudo dnf search nginx           # Search for a package
rpm -qa | grep nginx            # List installed RPMs matching nginx
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Systemd & Services

Most Linux servers use `systemd` to manage services. You will need this for running web servers, databases, and container runtimes.

```bash
sudo systemctl start nginx      # Start a service
sudo systemctl stop nginx       # Stop a service
sudo systemctl restart nginx    # Restart a service
sudo systemctl reload nginx     # Reload config without restart
sudo systemctl enable nginx     # Start on boot
sudo systemctl disable nginx    # Do not start on boot
sudo systemctl status nginx     # Show status and recent logs

# View logs for a service
journalctl -u nginx             # All logs for nginx
journalctl -u nginx -f          # Follow logs in real time
journalctl -u nginx --since "1 hour ago"
journalctl -p err               # Show only error-level entries
```

[↑ Back to TOC](#table-of-contents)

---

## Tools & Commands Reference

| Command | Purpose |
|---|---|
| `pwd` | Print working directory |
| `ls -la` | List all files with details |
| `cd`, `mkdir`, `rm`, `cp`, `mv` | Filesystem navigation & management |
| `cat`, `less`, `head`, `tail` | View file contents |
| `grep` | Search text with patterns |
| `find` | Find files by name, type, age |
| `diff`, `wc` | Compare files, count lines/words |
| `chmod`, `chown` | Change permissions and ownership |
| `vim`, `nano` | Edit files in the terminal |
| `ps`, `top`, `kill` | Process management |
| `df`, `du`, `free` | Disk and memory usage |
| `ssh`, `scp`, `ssh-keygen` | Secure remote access |
| `sudo`, `su`, `passwd` | Privilege escalation and user management |
| `ping`, `curl`, `nc`, `netstat` | Network diagnostics |
| `uname`, `hostname`, `who` | System information |
| `history`, `!!`, `Ctrl+R` | Command history |
| `systemctl`, `journalctl` | Service and log management |
| `apt` / `dnf` | Package management |

[↑ Back to TOC](#table-of-contents)

---

## Hands-On Labs

### Lab 1.1 — Filesystem Exploration

1. Open a terminal and run `pwd` — note your location
2. Navigate to `/etc` and list all files with `ls -la`
3. Find all files in `/etc` that end in `.conf`: `find /etc -name "*.conf"`
4. View the first 20 lines of `/etc/hosts`: `head -20 /etc/hosts`
5. Search for your own username in `/etc/passwd`: `grep "$USER" /etc/passwd`

### Lab 1.2 — File Permissions

1. Create a test script: `echo '#!/bin/bash\necho "Hello DevOps!"' > test.sh`
2. Try running it: `./test.sh` — it will fail (no execute permission)
3. Add execute permission: `chmod +x test.sh`
4. Run it again: `./test.sh` — it works
5. Check permissions: `ls -la test.sh`
6. Remove execute permission: `chmod -x test.sh`

### Lab 1.3 — Process & System Management

1. Run `ps aux` and find the process with PID 1
2. Check disk usage: `df -h`
3. Check memory: `free -h`
4. Start a background process: `sleep 300 &`
5. Find its PID: `ps aux | grep sleep`
6. Kill it: `kill <PID>`

### Lab 1.4 — SSH Key Setup

1. Generate an SSH key pair: `ssh-keygen -t ed25519 -C "devops-lab"`
2. View your public key: `cat ~/.ssh/id_ed25519.pub`
3. If you have a second machine or VM, copy the key: `ssh-copy-id user@remote-host`
4. Test passwordless login: `ssh user@remote-host`

### Lab 1.5 — Command Chaining Challenge

1. Find all `.log` files in `/var/log`: `find /var/log -name "*.log" 2>/dev/null`
2. Count how many exist: `find /var/log -name "*.log" 2>/dev/null | wc -l`
3. Show the 5 largest log files: `find /var/log -name "*.log" 2>/dev/null -exec du -sh {} \; | sort -rh | head -5`
4. Search all logs for the word "failed" and count occurrences: `grep -ri "failed" /var/log/ 2>/dev/null | wc -l`

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

- [The Linux Command Line (free book)](https://linuxcommand.org/tlcl.php) — William Shotts
- [Linux Journey](https://linuxjourney.com/) — Interactive Linux learning
- [Explainshell](https://explainshell.com/) — Paste any command to see what each part does
- [man pages](https://man7.org/linux/man-pages/) — Official Linux manual pages
- [Glossary: Bash](./glossary.md#b), [Daemon](./glossary.md#d), [SSH](./glossary.md#s)
- **Certification**: Linux Foundation Certified Sysadmin (LFCS)

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)*
