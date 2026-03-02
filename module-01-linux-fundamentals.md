# Module 01: Linux Fundamentals

> Part of the [DevOps Career Course](./README.md) by UncleJS

[![CC BY-NC-SA 4.0](https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/) ![Module 01 of 15](https://img.shields.io/badge/module-01%20of%2015-grey) ![Level](https://img.shields.io/badge/level-Beginner-brightgreen) ![bash 5.2+](https://img.shields.io/badge/bash-5.2%2B-4EAA25?logo=gnubash&logoColor=white) ![systemd 256+](https://img.shields.io/badge/systemd-256%2B-blueviolet) ![Linux](https://img.shields.io/badge/Linux-RHEL%20%C2%B7%20Ubuntu%20%C2%B7%20Debian-FCC624?logo=linux&logoColor=black)

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
- [Intermediate: Environment Variables & Shell Configuration](#intermediate-environment-variables--shell-configuration)
- [Advanced: Linux Performance & Diagnostics](#advanced-linux-performance--diagnostics)
- [Advanced: Storage & Filesystems](#advanced-storage--filesystems)
- [Advanced: Kernel & Security Hardening](#advanced-kernel--security-hardening)
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
- Configure environment variables and customize your shell environment
- Diagnose performance bottlenecks using CPU, memory, and I/O tools
- Manage disk partitions, filesystems, and mounts
- Apply basic kernel tuning and Linux security hardening

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
| `/sbin` | System administration binaries |
| `/lib` | Shared libraries for `/bin` and `/sbin` |
| `/opt` | Optional/third-party software |
| `/proc` | Virtual filesystem for process/system info |
| `/sys` | Virtual filesystem for kernel/device info |
| `/dev` | Device files (disks, terminals, etc.) |
| `/run` | Runtime data (PIDs, sockets) — cleared at boot |
| `/mnt` / `/media` | Mount points for external storage |

### Navigation Commands

```bash
pwd                     # Print working directory (where am I right now?)
ls                      # List files in current directory
ls -la                  # List all files (including hidden) with details
ls -lh                  # Human-readable file sizes
ls -lt                  # Sort by modification time (newest first)
ls -lS                  # Sort by size (largest first)
cd /var/log             # Change to /var/log
cd ..                   # Go up one directory
cd ~                    # Go to your home directory
cd -                    # Go back to previous directory
tree /etc -L 2          # Display directory tree 2 levels deep
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
cp -p file.txt backup/  # Copy preserving permissions and timestamps
mv old.txt new.txt      # Rename a file
mv file.txt /tmp/       # Move a file to /tmp
ln -s /path/to/file symlink   # Create a symbolic link
ln file.txt hardlink.txt      # Create a hard link
touch newfile.txt             # Create empty file / update timestamp
```

### Finding Files

```bash
find /var -name "*.log"             # Find all .log files under /var
find . -type f -mtime -1            # Files modified in the last 1 day
find . -type f -size +10M           # Files larger than 10 MB
find /etc -name "*.conf" -type f    # Config files only
find . -perm 777                    # Find files with 777 permissions
find . -user alice                  # Files owned by alice
find . -name "*.sh" -exec chmod +x {} \;  # Find and make executable
locate nginx.conf                   # Fast file search using index (updatedb first)
which python3                       # Show full path of a command
whereis nginx                       # Locate binary, source, and man page
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
tail -F /var/log/syslog # Follow — keeps working even if file is rotated

# Searching
grep "error" file.txt           # Find lines containing "error"
grep -i "error" file.txt        # Case-insensitive search
grep -r "TODO" ./src/           # Recursive search through directory
grep -n "error" file.txt        # Show line numbers
grep -v "debug" file.txt        # Invert match — lines NOT containing "debug"
grep -c "error" file.txt        # Count matching lines
grep -l "error" /var/log/*.log  # Show only filenames that contain the pattern
grep -A 3 "ERROR" file.txt      # Show 3 lines after each match (context)
grep -B 2 "ERROR" file.txt      # Show 2 lines before each match
grep -E "error|warning" file.txt # Extended regex — match either pattern

# Comparison and counting
diff file1.txt file2.txt        # Show differences between two files
diff -u file1.txt file2.txt     # Unified diff format (used in patches)
wc -l file.txt                  # Count lines in a file
wc -w file.txt                  # Count words
wc -c file.txt                  # Count bytes

# Sorting and deduplication
sort file.txt                   # Sort alphabetically
sort -n numbers.txt             # Sort numerically
sort -rn numbers.txt            # Sort numerically, reverse
sort -k2 file.txt               # Sort by second column
uniq file.txt                   # Remove consecutive duplicate lines
sort file.txt | uniq -c         # Count occurrences of each unique line
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

First character:
  -  = regular file
  d  = directory
  l  = symbolic link
  b  = block device (disk)
  c  = character device (terminal)
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
chmod 600 ~/.ssh/id_ed25519   # Private key: owner read/write only
chmod 700 ~/.ssh/             # SSH dir: owner access only
chmod +x deploy.sh      # Add execute permission for everyone
chmod u+x deploy.sh     # Add execute for owner only
chmod g-w file.txt      # Remove write from group
chmod o= file.txt       # Remove all permissions from others
chmod -R 755 /var/www/  # Recursively set permissions

chown alice file.txt          # Change owner to alice
chown alice:devops file.txt   # Change owner and group
chown -R alice:devops /app/   # Recursively change ownership
chgrp devops file.txt         # Change group only
```

### Special Permissions

```bash
# Setuid (s) — run as file owner regardless of who executes
chmod u+s /usr/bin/passwd

# Setgid (s) — files in directory inherit the group
chmod g+s /var/shared/

# Sticky bit (t) — only file owner can delete in a directory
chmod +t /tmp

# Numeric equivalents (4 digits)
chmod 4755 file    # setuid + rwxr-xr-x
chmod 2755 dir     # setgid + rwxr-xr-x
chmod 1777 /tmp    # sticky + rwxrwxrwx

# View with ls
ls -la /tmp   # drwxrwxrwt — 't' shows sticky bit
```

### Access Control Lists (ACLs)

When standard `rwx` isn't enough — ACLs allow per-user, per-group permissions.

```bash
# View ACL on a file
getfacl file.txt

# Grant bob read access without changing standard perms
setfacl -m u:bob:r file.txt

# Grant the ops group write access
setfacl -m g:ops:rw file.txt

# Set a default ACL on a directory (inherited by new files)
setfacl -d -m u:bob:rw /var/shared/

# Remove an ACL entry
setfacl -x u:bob file.txt

# Remove all ACLs
setfacl -b file.txt
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
# Visual mode   — press 'v' to select text
# Command mode  — press ':' in normal mode

# Essential commands (in normal mode):
i           # Enter insert mode (before cursor)
a           # Enter insert mode (after cursor)
o           # Open new line below and enter insert mode
Esc         # Return to normal mode
:w          # Save (write)
:q          # Quit
:wq         # Save and quit
:q!         # Quit without saving (force)
:wqa        # Save and quit all open files
dd          # Delete current line
yy          # Copy (yank) current line
p           # Paste below current line
P           # Paste above current line
x           # Delete character under cursor
u           # Undo
Ctrl+r      # Redo
gg          # Go to first line
G           # Go to last line
:42         # Go to line 42
/word       # Search forward for "word"
?word       # Search backward for "word"
n           # Jump to next search result
N           # Jump to previous search result
:%s/old/new/g   # Replace all occurrences in file
:set number     # Show line numbers
:set paste      # Paste mode (no auto-indent)
Ctrl+v then I   # Visual block insert (multi-line edit)
```

### nano

`nano` is beginner-friendly — all shortcuts are shown at the bottom of the screen.

```bash
nano file.txt       # Open file in nano

Ctrl+O              # Save (Write Out)
Ctrl+X              # Exit
Ctrl+W              # Search
Ctrl+\              # Search and replace
Ctrl+K              # Cut current line
Ctrl+U              # Paste (Uncut)
Ctrl+G              # Help
Alt+U               # Undo
Ctrl+_              # Go to line number
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
command 2>&1            # Redirect stderr to same place as stdout
command &> all.log      # Write both stdout and stderr to all.log
command 2>/dev/null     # Discard error output
command > /dev/null 2>&1  # Discard all output

# Here-document — write multi-line input to a command
cat << EOF > config.yaml
host: localhost
port: 5432
EOF

# Process substitution — treat command output as a file
diff <(sort file1.txt) <(sort file2.txt)
```

### Pipes

The `|` operator sends the **output** of one command as the **input** to the next.

```bash
ls -la | grep ".log"             # List files, filter for .log
ps aux | grep nginx              # Find nginx processes
cat access.log | sort | uniq    # Sort and deduplicate
cat access.log | wc -l          # Count log entries
ls -la | awk '{print $5, $9}'   # Show only size and filename
df -h | grep -v tmpfs           # Disk usage without tmpfs entries

# tee — write to file AND stdout simultaneously
command | tee output.log         # Write to both screen and file
command | tee -a output.log      # Append to file
make build 2>&1 | tee build.log  # Capture build output with errors

# xargs — turn stdin lines into command arguments
find . -name "*.log" | xargs rm        # Delete all found log files
cat hosts.txt | xargs -I{} ping -c1 {} # Ping each host in file
ls *.txt | xargs -P4 gzip              # Compress 4 files in parallel
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: System & Process Management

```bash
# Process inspection
ps aux              # Show all running processes
ps aux | grep nginx # Find specific process
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu  # Sort by CPU usage
top                 # Interactive process viewer (q to quit)
htop                # Enhanced process viewer (install separately)
pgrep nginx         # Find PIDs for processes named nginx
pstree              # Show process tree

# Sending signals
kill 1234           # Send SIGTERM to process with PID 1234
kill -9 1234        # Send SIGKILL (force terminate) to PID 1234
kill -HUP 1234      # Send SIGHUP (reload config — useful for nginx)
killall nginx       # Kill all processes named nginx
pkill -f "python app.py"  # Kill by full command pattern

# Background & foreground jobs
sleep 300 &         # Run in background
jobs                # List background jobs
fg %1               # Bring job 1 to foreground
bg %1               # Send stopped job to background
Ctrl+Z              # Pause (stop) current foreground process
nohup command &     # Run immune to hangup (survives logout)
disown %1           # Detach job from shell

# Disk and memory
df -h               # Show disk usage (human-readable)
df -hT              # Show disk usage with filesystem type
du -sh /var/log/    # Show size of /var/log directory
du -ah /var | sort -rh | head -10  # Top 10 largest items
free -h             # Show RAM and swap usage
uptime              # How long the system has been running + load average
vmstat 1 5          # Virtual memory stats, 5 samples every 1 second
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
ssh -v user@host            # Verbose mode (debug connection issues)

# Generate an SSH key pair
ssh-keygen -t ed25519 -C "your_email@example.com"
# Creates: ~/.ssh/id_ed25519 (private) and ~/.ssh/id_ed25519.pub (public)

# Key types and when to use them
# ed25519  — modern, fast, secure (recommended)
# rsa      — widely compatible, use -b 4096 for strength
# ecdsa    — elliptic curve, good for older systems

# Copy your public key to a remote server
ssh-copy-id user@hostname
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@hostname  # Specify key

# Securely copy files
scp file.txt user@host:/remote/path/     # Local → Remote
scp user@host:/remote/file.txt ./        # Remote → Local
scp -r folder/ user@host:/remote/path/  # Copy directory

# rsync — efficient sync (only transfers changes)
rsync -avz ./local/ user@host:/remote/  # Sync directory to remote
rsync -avz --delete ./local/ user@host:/remote/  # Sync + delete extras
rsync -avz -e "ssh -p 2222" ./local/ user@host:/remote/  # Custom port
```

### SSH Config File

The `~/.ssh/config` file lets you define shortcuts and per-host settings:

```
# ~/.ssh/config

Host prod
    HostName 192.168.1.10
    User alice
    IdentityFile ~/.ssh/prod_key
    Port 22

Host bastion
    HostName bastion.example.com
    User ec2-user
    IdentityFile ~/.ssh/aws_key

Host internal
    HostName 10.0.0.50
    User ubuntu
    ProxyJump bastion           # Tunnel through bastion host
    IdentityFile ~/.ssh/aws_key

Host *
    ServerAliveInterval 60      # Send keepalive every 60 seconds
    AddKeysToAgent yes          # Add key to ssh-agent automatically
```

```bash
# With the config above:
ssh prod            # Connects to 192.168.1.10 as alice
ssh internal        # Connects via bastion (SSH jump host)

# SSH agent — cache passphrase in memory
eval $(ssh-agent)
ssh-add ~/.ssh/id_ed25519

# SSH tunneling
ssh -L 5432:db-host:5432 user@bastion  # Forward local port 5432 to remote DB
ssh -L 8080:localhost:80 user@server   # Browse remote web server locally
ssh -R 8080:localhost:3000 user@server # Expose local app on remote port
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Command History & Shortcuts

```bash
history             # Show command history
history | grep ssh  # Search history for ssh commands
history -c          # Clear history
!!                  # Re-run the last command
!git                # Re-run the last command starting with "git"
!123                # Re-run command number 123 from history
!$                  # Last argument of the previous command
!^                  # First argument of the previous command
^old^new            # Replace "old" with "new" in last command and run
sudo !!             # Re-run last command with sudo

# Keyboard shortcuts
Ctrl+R              # Reverse search through history (type to search)
Ctrl+C              # Cancel the current command
Ctrl+D              # Exit current shell (EOF)
Ctrl+L              # Clear the terminal screen
Ctrl+A              # Jump to beginning of line
Ctrl+E              # Jump to end of line
Ctrl+U              # Delete from cursor to beginning of line
Ctrl+K              # Delete from cursor to end of line
Ctrl+W              # Delete the word before the cursor
Alt+F               # Move forward one word
Alt+B               # Move backward one word
Tab                 # Auto-complete commands and file paths
Tab Tab             # Show all completions when ambiguous

# HISTCONTROL — suppress duplicates and leading-space commands
export HISTCONTROL=ignoreboth
export HISTSIZE=10000
export HISTFILESIZE=20000
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Root Access & User Management

```bash
sudo command            # Run command as root (superuser)
sudo -i                 # Open a root shell session
sudo -u bob command     # Run command as another user
su username             # Switch to another user
su -                    # Switch to root user (with root's environment)
passwd                  # Change your own password
sudo passwd username    # Change another user's password

# User management
sudo useradd -m alice           # Create user alice with home directory
sudo useradd -m -s /bin/bash -G sudo alice  # Create with shell and sudo group
sudo usermod -aG sudo alice     # Add alice to sudo group
sudo usermod -aG docker alice   # Add alice to docker group
sudo usermod -s /bin/bash alice # Change login shell
sudo userdel alice              # Delete user alice
sudo userdel -r alice           # Delete user AND home directory
id                              # Show current user's UID, GID, groups
id alice                        # Show alice's UID, GID, groups
whoami                          # Print current username
groups                          # List all groups current user belongs to

# Group management
sudo groupadd devops            # Create a group
sudo groupdel devops            # Delete a group
cat /etc/passwd                 # List all users
cat /etc/group                  # List all groups
getent passwd alice             # Look up user info

# sudoers — fine-grained sudo control
sudo visudo                     # Edit /etc/sudoers safely
# Allow alice to run only specific commands:
# alice ALL=(ALL) /usr/bin/systemctl, /usr/bin/apt
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
curl -L https://example.com  # Follow redirects
nc -zv hostname 80          # Test if port 80 is open (netcat)
nc -l 8080                  # Listen on port 8080
netstat -tulnp              # Show listening ports and services
ss -tulnp                   # Modern replacement for netstat
ss -s                       # Socket statistics summary
wget https://example.com/file.tar.gz   # Download a file
ip addr show                # Show IP addresses and interfaces
ip route show               # Show routing table
ip link show                # Show interface status
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: System Information

```bash
uname -a            # Full kernel and system information
uname -r            # Kernel version only
hostname            # Show system hostname
hostname -I         # Show all IP addresses
hostnamectl         # Show and set hostname (systemd systems)
who                 # Show who is logged in
w                   # Show who is logged in and what they're doing
last                # Show login history
lastlog             # Show last login for all users
lsb_release -a      # Show Linux distribution info
cat /etc/os-release # Distribution info (works everywhere)
lscpu               # CPU information (cores, sockets, architecture)
lsmem               # Memory information
lsblk               # List block devices (disks and partitions)
lsblk -f            # Show with filesystem types
lspci               # List PCI devices
lsusb               # List USB devices
dmidecode -t system # Hardware info from BIOS/UEFI
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

# Run a command on multiple servers
for server in web01 web02 web03; do
  echo "=== $server ==="; ssh $server uptime
done

# Check disk usage across servers and alert if over 80%
for server in web01 web02 web03; do
  usage=$(ssh $server "df / | tail -1 | awk '{print \$5}'" | tr -d '%')
  [ "$usage" -gt 80 ] && echo "ALERT: $server disk at ${usage}%"
done
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Package Management

```bash
# Debian/Ubuntu (apt)
sudo apt update                 # Update package index
sudo apt upgrade                # Upgrade all packages
sudo apt full-upgrade           # Upgrade + remove obsolete packages
sudo apt install nginx          # Install a package
sudo apt install -y nginx       # Install without prompting
sudo apt remove nginx           # Remove a package (keep config)
sudo apt purge nginx            # Remove package and config files
sudo apt autoremove             # Remove unused dependencies
apt search nginx                # Search for a package
apt show nginx                  # Show package details
dpkg -l | grep nginx            # List installed packages matching nginx
dpkg -L nginx                   # List files installed by a package
apt-cache depends nginx         # Show package dependencies

# Hold a package at its current version
sudo apt-mark hold nginx
sudo apt-mark unhold nginx

# RHEL/Rocky Linux/CentOS (dnf/yum)
sudo dnf update                 # Update all packages
sudo dnf install nginx          # Install a package
sudo dnf remove nginx           # Remove a package
sudo dnf search nginx           # Search for a package
sudo dnf info nginx             # Show package details
sudo dnf list installed         # List all installed packages
sudo dnf history                # Show transaction history
sudo dnf history undo last      # Undo the last transaction
rpm -qa | grep nginx            # List installed RPMs matching nginx
rpm -ql nginx                   # List files installed by nginx RPM
rpm -qf /usr/bin/nginx          # Which package owns this file?

# Snap (cross-distro)
sudo snap install code --classic   # Install VS Code
snap list                          # List installed snaps
sudo snap refresh                  # Update all snaps
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
sudo systemctl is-active nginx  # Check if running (returns "active" or "inactive")
sudo systemctl is-enabled nginx # Check if enabled for boot

# View service list
systemctl list-units --type=service         # All active services
systemctl list-units --type=service --all   # Including inactive
systemctl list-unit-files --type=service    # All installed service files

# View logs for a service
journalctl -u nginx             # All logs for nginx
journalctl -u nginx -f          # Follow logs in real time
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx --since "2026-01-01" --until "2026-01-02"
journalctl -p err               # Show only error-level entries
journalctl -p err -xe           # Error + context + extra info (use when debugging)
journalctl --disk-usage         # How much space logs are using
journalctl --vacuum-size=500M   # Trim logs to 500 MB
journalctl --vacuum-time=7d     # Delete logs older than 7 days

# System targets (like runlevels)
systemctl get-default               # Show current target (e.g. multi-user.target)
sudo systemctl set-default graphical.target
sudo systemctl isolate rescue.target  # Switch to rescue mode

# Writing a custom service unit
sudo tee /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Application
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server
Restart=on-failure
RestartSec=5s
Environment=PORT=3000
EnvironmentFile=/etc/myapp/env

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now myapp
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Environment Variables & Shell Configuration

Environment variables are key-value pairs available to all processes in a shell session. They control behavior across tools, languages, runtimes, and scripts.

### Working with Environment Variables

```bash
# View variables
env                         # All environment variables
printenv                    # Same — all env vars
printenv HOME               # One specific variable
echo $PATH                  # Print PATH
echo $USER                  # Current username
echo $HOME                  # Home directory
echo $SHELL                 # Current shell
echo $PWD                   # Current directory (same as pwd)

# Set a variable
MY_VAR="hello"              # Shell variable (not exported to child processes)
export MY_VAR="hello"       # Environment variable (available to child processes)
export PORT=3000

# Unset a variable
unset MY_VAR

# Temporary override for one command
PORT=8080 node server.js    # Run with PORT=8080 without permanently exporting

# Check if a variable is set
[ -z "$MY_VAR" ] && echo "not set" || echo "set to: $MY_VAR"
```

### Important System Variables

| Variable | Purpose |
|---|---|
| `PATH` | Directories searched for executables |
| `HOME` | Current user's home directory |
| `USER` / `LOGNAME` | Current username |
| `SHELL` | Path to current shell |
| `TERM` | Terminal type |
| `EDITOR` | Default text editor |
| `LANG` / `LC_ALL` | Locale and character encoding |
| `TZ` | Timezone (e.g. `America/New_York`) |
| `TMPDIR` | Temporary files directory |
| `PS1` | Shell prompt string |
| `LD_LIBRARY_PATH` | Additional shared library search paths |

```bash
# Modify PATH to include a custom bin directory
export PATH="$HOME/.local/bin:$PATH"

# Set default editor
export EDITOR=vim

# Set timezone
export TZ="UTC"
date   # Now shows UTC time

# View the complete PATH entries one per line
echo $PATH | tr ':' '\n'
```

### Shell Configuration Files

Bash reads configuration files in a specific order depending on how the shell is invoked:

```
Login shell:
  /etc/profile        ← system-wide, runs first
  /etc/profile.d/*.sh ← modular system-wide scripts
  ~/.bash_profile     ← user personal (runs if exists)
  ~/.bash_login       ← fallback if .bash_profile missing
  ~/.profile          ← fallback if both above missing

Interactive non-login shell (e.g. opening a new terminal tab):
  /etc/bash.bashrc    ← system-wide
  ~/.bashrc           ← user personal (most commonly edited)

Logout:
  ~/.bash_logout
```

```bash
# ~/.bashrc — the file you'll edit most often

# Custom aliases
alias ll='ls -la --color=auto'
alias grep='grep --color=auto'
alias k='kubectl'
alias tf='terraform'
alias dc='docker compose'

# Useful functions
mkcd() { mkdir -p "$1" && cd "$1"; }
extract() {
  case "$1" in
    *.tar.gz)  tar xzf "$1" ;;
    *.tar.bz2) tar xjf "$1" ;;
    *.zip)     unzip "$1" ;;
    *.gz)      gunzip "$1" ;;
    *)         echo "Don't know how to extract $1" ;;
  esac
}

# Custom prompt (PS1) showing user, host, directory, git branch
parse_git_branch() {
  git branch 2>/dev/null | grep '^*' | colrm 1 2
}
export PS1='\[\e[32m\]\u@\h\[\e[0m\]:\[\e[34m\]\w\[\e[33m\]$(git_branch)\[\e[0m\]\$ '

# PATH additions
export PATH="$HOME/.local/bin:$HOME/bin:$PATH"

# History settings
export HISTCONTROL=ignoreboth
export HISTSIZE=10000
export HISTFILESIZE=20000
shopt -s histappend    # Append to history instead of overwriting

# Apply changes without logging out
source ~/.bashrc
# or shorthand:
. ~/.bashrc
```

### .env Files

`.env` files store environment variables for an application, keeping secrets out of code:

```bash
# .env
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=myapp
DATABASE_PASSWORD=secret123
API_KEY=abc123def456
NODE_ENV=development
```

```bash
# Load a .env file in a script
export $(grep -v '^#' .env | xargs)

# Or using a dedicated tool (dotenv)
# Python: from dotenv import load_dotenv
# Node.js: require('dotenv').config()

# Never commit .env to Git — add to .gitignore:
echo ".env" >> .gitignore
echo ".env.local" >> .gitignore
```

[↑ Back to TOC](#table-of-contents)

---

## Advanced: Linux Performance & Diagnostics

### CPU Performance

```bash
# Load average
uptime          # 1, 5, 15 minute load averages
# Load > number of CPU cores = system is overloaded

nproc           # Number of CPU cores available
lscpu           # Detailed CPU info

# Real-time CPU monitoring
top             # press 1 to show per-core stats
htop            # Interactive, color-coded
mpstat -P ALL 1 # Per-CPU stats every 1 second (sysstat package)
sar -u 1 5      # CPU utilization, 5 samples every 1 second

# CPU-intensive process profiling
pidstat -u 1    # Per-process CPU usage
perf top        # Real-time perf analysis (requires kernel headers)
strace -p 1234  # Trace system calls for PID 1234
```

### Memory Performance

```bash
free -h                         # RAM and swap overview
cat /proc/meminfo               # Detailed kernel memory stats
vmstat 1 5                      # Virtual memory stats
# Key columns: si/so (swap in/out), bi/bo (block I/O)

# Identify memory hogs
ps aux --sort=-%mem | head -10  # Top 10 by memory
pmap -x 1234                    # Memory map for PID 1234

# Cache and buffers
sync && echo 3 > /proc/sys/vm/drop_caches  # Flush page cache (root only, careful!)
cat /proc/sys/vm/swappiness     # Swap aggressiveness (0–100, default 60)
sudo sysctl vm.swappiness=10    # Reduce swap tendency
```

### Disk I/O Performance

```bash
# Disk I/O stats
iostat -xz 1        # Extended I/O stats every 1 second
# Key columns: %util (busy %), await (avg wait ms), r/s, w/s
iotop               # Interactive I/O monitor by process
iotop -o            # Only show processes doing I/O

# Disk speed testing
dd if=/dev/zero of=/tmp/test bs=1M count=1024 oflag=dsync  # Write test
dd if=/tmp/test of=/dev/null bs=1M                         # Read test

# Find I/O-heavy processes
pidstat -d 1        # Per-process disk I/O

# Check for disk errors
sudo dmesg | grep -i "error\|fail"
sudo journalctl -k | grep -i "error\|fail"   # Kernel messages only
sudo smartctl -a /dev/sda                    # S.M.A.R.T. disk health
```

### Network Performance

```bash
# Bandwidth monitoring
iftop                           # Real-time bandwidth by connection
nethogs                         # Per-process bandwidth usage
nload                           # Interface bandwidth graph

# Connection stats
ss -s                           # Summary of socket stats
ss -tulnp                       # All listening sockets
ss -tp                          # All TCP connections with process info
ss -o state established '( dport = :443 )'   # Established HTTPS connections

# Packet stats
ip -s link show eth0            # Interface packet counters
netstat -s                      # Protocol-level stats (TCP retransmits, etc.)

# Latency testing
ping -c 100 8.8.8.8 | tail -5  # 100-ping latency stats
hping3 -S -p 443 -c 100 example.com  # TCP latency (SYN packets)
```

### System-Wide Performance Snapshot

```bash
# The USE method — Utilization, Saturation, Errors per resource
# CPU: utilization (mpstat), saturation (load avg), errors (dmesg)
# Memory: utilization (free), saturation (vmstat si/so), errors (dmesg)
# Disk: utilization (iostat %util), saturation (await), errors (dmesg)
# Network: utilization (iftop), saturation (ss), errors (ip -s link)

# Quick snapshot tools
sar -A 1 5          # All stats via sysstat
dstat               # Combined CPU/disk/net/mem stats

# Performance investigation workflow
uptime              # 1. Check load average
dmesg | tail -10    # 2. Check recent kernel messages
vmstat 1 5          # 3. Check CPU/memory/I/O overall
pidstat 1           # 4. Find high-CPU/memory processes
iostat -xz 1        # 5. Check disk I/O
iftop               # 6. Check network usage
```

[↑ Back to TOC](#table-of-contents)

---

## Advanced: Storage & Filesystems

### Disk Partitioning

```bash
# View block devices and partitions
lsblk                       # Tree view of disks and partitions
lsblk -f                    # Include filesystem types and UUIDs
fdisk -l                    # List all disks and partition tables
parted -l                   # Modern alternative to fdisk

# Create and manage partitions
sudo fdisk /dev/sdb         # Interactive partition editor for /dev/sdb
sudo gdisk /dev/sdb         # GPT partition tool (use for disks > 2TB or UEFI)

# Partition with parted (scriptable)
sudo parted /dev/sdb mklabel gpt
sudo parted /dev/sdb mkpart primary ext4 0% 100%
```

### Filesystems

```bash
# Create filesystems
sudo mkfs.ext4 /dev/sdb1            # Create ext4 filesystem
sudo mkfs.xfs /dev/sdb1             # Create XFS filesystem (common on RHEL)
sudo mkfs.btrfs /dev/sdb1           # Create Btrfs filesystem

# Mount and unmount
sudo mount /dev/sdb1 /mnt/data              # Mount a partition
sudo umount /mnt/data                       # Unmount

# Check filesystem type
file -s /dev/sdb1
blkid /dev/sdb1

# Persistent mounts via /etc/fstab
# Format: <device> <mountpoint> <type> <options> <dump> <pass>
# UUID=abc-123 /mnt/data ext4 defaults 0 2
sudo blkid /dev/sdb1   # Get UUID
sudo vim /etc/fstab
sudo mount -a           # Apply all fstab entries without rebooting

# Filesystem health
sudo e2fsck -n /dev/sdb1    # Check ext4 (dry-run, read-only)
sudo xfs_check /dev/sdb1    # Check XFS
df -iH                       # Show inode usage (can run out before disk space!)
```

### Logical Volume Manager (LVM)

LVM adds a flexible abstraction layer between physical disks and filesystems — resizable volumes without repartitioning.

```bash
# LVM hierarchy: Physical Volumes (PV) → Volume Group (VG) → Logical Volume (LV)

# Create a Physical Volume
sudo pvcreate /dev/sdb1
pvs                         # List Physical Volumes

# Create a Volume Group
sudo vgcreate datavg /dev/sdb1
vgs                         # List Volume Groups

# Create a Logical Volume
sudo lvcreate -L 50G -n datalv datavg   # Fixed size
sudo lvcreate -l 100%FREE -n datalv datavg  # Use all space
lvs                                     # List Logical Volumes

# Use the LV like a regular partition
sudo mkfs.ext4 /dev/datavg/datalv
sudo mount /dev/datavg/datalv /mnt/data

# Extend a volume online (no downtime!)
sudo lvextend -L +20G /dev/datavg/datalv         # Add 20 GB
sudo resize2fs /dev/datavg/datalv                # Resize ext4 to match
sudo xfs_growfs /mnt/data                        # Resize XFS (mounted)
```

[↑ Back to TOC](#table-of-contents)

---

## Advanced: Kernel & Security Hardening

### sysctl — Kernel Parameters

`sysctl` lets you read and write kernel parameters at runtime, or persistently via `/etc/sysctl.conf`.

```bash
# View all parameters
sysctl -a

# Read a specific parameter
sysctl net.ipv4.ip_forward
sysctl vm.swappiness

# Set a parameter at runtime (not persistent across reboots)
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w vm.swappiness=10

# Persist parameters in /etc/sysctl.d/
sudo tee /etc/sysctl.d/99-custom.conf << 'EOF'
# Enable IP forwarding (required for routing/NAT, containers)
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# Reduce swappiness for better application performance
vm.swappiness = 10

# Increase max open files
fs.file-max = 2097152

# Harden the network stack
net.ipv4.tcp_syncookies = 1         # SYN flood protection
net.ipv4.conf.all.rp_filter = 1     # Reverse path filtering
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1
EOF

sudo sysctl --system    # Apply all /etc/sysctl.d/ files
```

### ulimit — Resource Limits

```bash
ulimit -a               # Show all limits for current shell
ulimit -n               # Max open file descriptors
ulimit -n 65536         # Set max open files (current session)

# Persistent limits via /etc/security/limits.conf
sudo tee -a /etc/security/limits.conf << 'EOF'
# Format: <domain> <type> <item> <value>
*       soft    nofile    65536
*       hard    nofile    131072
nginx   soft    nofile    65536
nginx   hard    nofile    65536
EOF
```

### SSH Hardening

```bash
# /etc/ssh/sshd_config — key settings to harden
sudo vim /etc/ssh/sshd_config

# Recommended settings:
Port 2222                       # Non-standard port (obscurity, not security)
PermitRootLogin no              # Never allow direct root login
PasswordAuthentication no       # Keys only, no passwords
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
MaxAuthTries 3                  # Limit brute force attempts
ClientAliveInterval 300         # Disconnect idle sessions after 5 min
ClientAliveCountMax 2
AllowUsers alice bob            # Whitelist specific users
AllowGroups sshusers            # Or whitelist a group
X11Forwarding no                # Disable unless needed
Banner /etc/ssh/banner.txt      # Show legal notice before login

sudo systemctl restart sshd     # Apply changes
sudo sshd -t                    # Test config before restarting
```

### Fail2ban — Intrusion Prevention

```bash
# Install
sudo apt install fail2ban   # Ubuntu
sudo dnf install fail2ban   # RHEL/Fedora

# Configure
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo vim /etc/fail2ban/jail.local

# Example jail config:
# [sshd]
# enabled = true
# port = 2222
# maxretry = 3
# bantime = 3600
# findtime = 600

sudo systemctl enable --now fail2ban

# Check status
sudo fail2ban-client status sshd
sudo fail2ban-client status          # All jails
sudo fail2ban-client set sshd unbanip 1.2.3.4   # Unban an IP
```

### Audit & Compliance

```bash
# auditd — kernel-level audit logging
sudo apt install auditd
sudo systemctl enable --now auditd

# Define rules — what to audit
sudo auditctl -w /etc/passwd -p wa -k passwd_changes   # Watch passwd writes
sudo auditctl -w /etc/sudoers -p wa -k sudoers_changes
sudo auditctl -a always,exit -F arch=b64 -S execve -k commands  # Log all commands

# Persist rules
sudo vim /etc/audit/rules.d/custom.rules

# View audit log
sudo ausearch -k passwd_changes
sudo aureport --summary          # Summary report
sudo aureport -a                 # All events
```

[↑ Back to TOC](#table-of-contents)

---

## Tools & Commands Reference

| Command | Purpose |
|---|---|
| `pwd` | Print working directory |
| `ls -la` | List all files with details |
| `cd`, `mkdir`, `rm`, `cp`, `mv` | Filesystem navigation & management |
| `find`, `locate` | Find files by name, type, age |
| `cat`, `less`, `head`, `tail -f` | View file contents |
| `grep`, `grep -r` | Search text with patterns |
| `diff`, `wc`, `sort`, `uniq` | Compare files, count, sort |
| `chmod`, `chown`, `setfacl` | Change permissions, ownership, ACLs |
| `vim`, `nano` | Edit files in the terminal |
| `ps aux`, `top`, `htop`, `pgrep` | Process inspection |
| `kill`, `killall`, `pkill` | Send signals to processes |
| `nohup`, `jobs`, `fg`, `bg` | Job control |
| `df`, `du`, `free`, `vmstat` | Disk and memory usage |
| `iostat`, `iotop` | Disk I/O performance |
| `mpstat`, `sar` | CPU performance statistics |
| `ssh`, `scp`, `rsync` | Secure remote access and file sync |
| `ssh-keygen`, `ssh-copy-id` | SSH key management |
| `sudo`, `su`, `passwd`, `useradd` | Privilege and user management |
| `ping`, `curl`, `nc`, `ss` | Network diagnostics |
| `ip addr`, `ip route` | Interface and routing management |
| `uname`, `hostname`, `lscpu`, `lsblk` | System information |
| `history`, `!!`, `Ctrl+R` | Command history |
| `systemctl`, `journalctl` | Service and log management |
| `apt` / `dnf` | Package management |
| `export`, `env`, `printenv` | Environment variables |
| `source` / `.` | Apply shell config changes |
| `sysctl` | Read/write kernel parameters |
| `ulimit` | Shell resource limits |
| `lsblk`, `fdisk`, `parted` | Disk partitioning |
| `mkfs`, `mount`, `umount` | Filesystem creation and mounting |
| `lvcreate`, `vgcreate`, `pvcreate` | LVM volume management |
| `fail2ban-client` | Intrusion prevention status |
| `auditctl`, `ausearch` | Kernel audit management |

[↑ Back to TOC](#table-of-contents)

---

## Hands-On Labs

### Lab 1.1 — Filesystem Exploration

1. Open a terminal and run `pwd` — note your location
2. Navigate to `/etc` and list all files with `ls -la`
3. Find all files in `/etc` that end in `.conf`: `find /etc -name "*.conf"`
4. View the first 20 lines of `/etc/hosts`: `head -20 /etc/hosts`
5. Search for your own username in `/etc/passwd`: `grep "$USER" /etc/passwd`
6. Count how many `.conf` files are in `/etc`: `find /etc -name "*.conf" | wc -l`

### Lab 1.2 — File Permissions

1. Create a test script: `echo '#!/bin/bash\necho "Hello DevOps!"' > test.sh`
2. Try running it: `./test.sh` — it will fail (no execute permission)
3. Add execute permission: `chmod +x test.sh`
4. Run it again: `./test.sh` — it works
5. Check permissions: `ls -la test.sh`
6. Remove execute permission: `chmod -x test.sh`
7. Grant a specific user (bob) read access using ACL: `setfacl -m u:bob:r test.sh`

### Lab 1.3 — Process & System Management

1. Run `ps aux` and find the process with PID 1
2. Check disk usage: `df -h`
3. Check memory: `free -h`
4. Start a background process: `sleep 300 &`
5. Find its PID: `ps aux | grep sleep`
6. Kill it: `kill <PID>`
7. Check CPU and memory stats: `vmstat 1 5`

### Lab 1.4 — SSH Key Setup

1. Generate an SSH key pair: `ssh-keygen -t ed25519 -C "devops-lab"`
2. View your public key: `cat ~/.ssh/id_ed25519.pub`
3. If you have a second machine or VM, copy the key: `ssh-copy-id user@remote-host`
4. Test passwordless login: `ssh user@remote-host`
5. Create an SSH config entry for the host in `~/.ssh/config`

### Lab 1.5 — Command Chaining Challenge

1. Find all `.log` files in `/var/log`: `find /var/log -name "*.log" 2>/dev/null`
2. Count how many exist: `find /var/log -name "*.log" 2>/dev/null | wc -l`
3. Show the 5 largest log files: `find /var/log -name "*.log" 2>/dev/null -exec du -sh {} \; | sort -rh | head -5`
4. Search all logs for the word "failed" and count occurrences: `grep -ri "failed" /var/log/ 2>/dev/null | wc -l`

### Lab 1.6 — Shell Configuration

1. Add a custom alias to `~/.bashrc`: `alias ports='ss -tulnp'`
2. Add a `PATH` entry: `export PATH="$HOME/.local/bin:$PATH"`
3. Write a shell function `mkcd` that creates and changes into a directory
4. Apply the changes: `source ~/.bashrc`
5. Test: `mkcd /tmp/testdir && pwd`

### Lab 1.7 — Performance Investigation

1. Run `uptime` and check the load average against `nproc` (number of CPUs)
2. Run `vmstat 1 10` and identify which column shows swapping activity
3. Find the top 5 memory-consuming processes: `ps aux --sort=-%mem | head -6`
4. Run `iostat -xz 1 5` and identify the `%util` column for each disk
5. Use `journalctl -p err -n 50` to find the 50 most recent error messages

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

- [The Linux Command Line (free book)](https://linuxcommand.org/tlcl.php) — William Shotts
- [Linux Journey](https://linuxjourney.com/) — Interactive Linux learning
- [Explainshell](https://explainshell.com/) — Paste any command to see what each part does
- [man pages](https://man7.org/linux/man-pages/) — Official Linux manual pages
- [Brendan Gregg's Linux Performance](https://www.brendangregg.com/linuxperf.html) — Deep performance analysis
- [Linux Hardening Guide](https://www.cisecurity.org/cis-benchmarks/) — CIS Benchmarks
- [LVM Administration Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_and_managing_logical_volumes/)
- [Glossary: Bash](./glossary.md#b), [Daemon](./glossary.md#d), [SSH](./glossary.md#s)
- **Certification**: Linux Foundation Certified Sysadmin (LFCS)

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)*
