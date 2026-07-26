# Linux — Basic Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** None — ground-level Linux proficiency  
> **Estimated time:** 4–6 hours

---

## Table of Contents

1. Filesystem Hierarchy
2. File Permissions
3. Users and Groups
4. Process Management
5. Package Management
6. Text Processing
7. SSH
8. systemd Basics
9. Basic Networking
10. Q&A

---

## 1. Filesystem Hierarchy

| Path | Purpose |
|------|---------|
| `/bin` | Essential user binaries (ls, cp, mv) |
| `/sbin` | System binaries (fdisk, mkfs) |
| `/usr` | User system resources (bin, lib, share, local) |
| `/etc` | Configuration files (passwd, nginx.conf, hosts) |
| `/var` | Variable data (logs, databases, spool) |
| `/tmp` | Temporary files (cleared on reboot) |
| `/home` | User home directories |
| `/root` | Root user home |
| `/proc` | Virtual filesystem — process and kernel info |
| `/sys` | Virtual filesystem — device and kernel parameters |
| `/dev` | Device files (sda, tty, null, random) |
| `/opt` | Optional third-party software |
| `/mnt` | Temporary mount points |

### File types

```bash
# Check file type
$ ls -l /bin/ls
-rwxr-xr-x 1 root root 142560 Mar 15 2026 /bin/ls
# ^ first char: - = regular file, d = directory, l = symlink
#               s = socket, p = pipe, b = block device, c = character device

$ ls -l /dev/sda
brw-rw---- 1 root disk 8, 0 Mar 15 2026 /dev/sda
# b = block device
```

### Virtual filesystems

```bash
# /proc — kernel and process info
/proc/cpuinfo          # CPU details
/proc/meminfo          # Memory details
/proc/loadavg          # System load
/proc/{pid}/fd/        # Open file descriptors for process
/proc/{pid}/status     # Process state, memory, capabilities
/proc/{pid}/environ    # Environment variables

# /sys — kernel parameters and device info
/sys/block/sda/        # Block device info
/sys/class/net/eth0/   # Network interface info
```

---

## 2. File Permissions

### Permission notation

```bash
$ ls -l script.sh
-rwxr--r-- 1 deploy developers 1234 Mar 15 10:00 script.sh
││││││││││
│││││││││└── Other: execute
││││││││└─── Other: read
│││││││└──── Other: write (sticky bit position)
││││││└───── Group: execute
│││││└────── Group: read
││││└─────── Group: write (SGID position)
│││└──────── Owner: execute (SUID position)
││└───────── Owner: read
│└────────── Owner: write
└─────────── File type (- regular, d directory)
```

| Permission | Numeric | Files | Directories |
|------------|---------|-------|-------------|
| `r` | 4 | Read content | List files |
| `w` | 2 | Modify content | Create/delete files |
| `x` | 1 | Execute | Enter directory |

### Special permissions

```bash
# SUID (Set User ID) — runs with owner's permissions
chmod u+s /usr/bin/passwd
-rwsr-xr-x  # s in owner's execute position

# SGID (Set Group ID) — runs with group's permissions
chmod g+s /shared/dir
drwxrws---  # s in group's execute position

# Sticky bit — only owner can delete own files
chmod +t /tmp
drwxrwxrwt  # t in other's execute position
```

### Changing permissions

```bash
chmod 755 script.sh     # rwxr-xr-x
chmod +x script.sh      # Add execute
chmod -R g+w /app       # Recursively add group write
chown deploy:developers file.txt  # Change owner:group
```

---

## 3. Users and Groups

### User management

```bash
# List users
cat /etc/passwd
# Format: username:x:UID:GID:display name:home:shell

# Add user
useradd -m -s /bin/bash deploy

# Set password
passwd deploy

# Switch user
su - deploy           # Login shell
sudo -u deploy command  # Run as user

# User info
id deploy             # UID, GID, groups
whoami                # Current user
```

### Sudo

```bash
# /etc/sudoers (edit with visudo)
deploy ALL=(ALL) ALL                    # Full sudo access
deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl  # No password for specific command
%developers ALL=(ALL) ALL               # Group sudo
```

### Group management

```bash
groupadd developers
usermod -aG developers deploy   # Add user to group
gpasswd -d deploy developers    # Remove user from group
groups deploy                   # List groups for user
```

---

## 4. Process Management

### Viewing processes

```bash
# ps — process snapshot
ps aux                    # All processes
ps aux --sort=-%mem       # Sort by memory
ps aux | grep nginx        # Filter
ps -ef --forest           # Tree view

# top — real-time view
top                       # Interactive
  htop                    # Better UI (install separately)

# pgrep/pkill
pgrep -u deploy php       # Find PIDs
pkill -f "php artisan"    # Kill by pattern
```

### Process states

| State | Meaning |
|-------|---------|
| `R` | Running or runnable |
| `S` | Sleeping (interruptible) |
| `D` | Uninterruptible sleep (I/O) |
| `Z` | Zombie (terminated, waiting for parent) |
| `T` | Stopped |
| `t` | Tracing stop |

### Sending signals

```bash
kill -15 PID    # SIGTERM — graceful shutdown (default)
kill -9 PID     # SIGKILL — force kill
kill -1 PID     # SIGHUP — reload config
kill -USR1 PID  # User-defined signal (often log rotation)

# By name
pkill -f "php-fpm: pool www"
killall nginx
```

### Process priority

```bash
nice -n -20 ./high-priority  # Run with high priority (-20 = highest, +19 = lowest)
renice -n 5 -p 1234          # Change priority of running process
```

### Background and foreground

```bash
command &           # Run in background
nohup command &     # Run in background, immune to hup
disown              # Remove job from shell's job table
Ctrl+Z              # Suspend (stop)
bg                  # Resume in background
fg                  # Resume in foreground
jobs                # List background jobs
```

---

## 5. Package Management

### Debian/Ubuntu (apt)

```bash
# Update package lists
apt update

# Install
apt install nginx

# Remove
apt remove nginx        # Keeps config
apt purge nginx         # Removes config
apt autoremove          # Remove unused dependencies

# Search
apt search nginx

# List installed
apt list --installed

# Upgrade all
apt upgrade
apt dist-upgrade
```

### Red Hat/CentOS (yum/dnf)

```bash
yum install nginx
yum remove nginx
yum search nginx
yum update
```

---

## 6. Text Processing

### grep

```bash
# Search in files
grep "ERROR" /var/log/nginx/error.log

# Recursive
grep -r "DB_HOST" /etc/

# Common options
grep -i "error" file       # Case-insensitive
grep -v "debug" file       # Exclude matching lines
grep -c "ERROR" file       # Count matches
grep -n "ERROR" file       # Show line numbers
grep -A5 "ERROR" file      # 5 lines after match
grep -B5 "ERROR" file      # 5 lines before match
grep -C5 "ERROR" file      # 5 lines around match

# Extended regex (ERE)
grep -E "^[0-9]{3}" file
egrep "^[0-9]{3}" file    # Same as grep -E
```

### sed

```bash
# Replace (in-place)
sed -i 's/old/new/g' file

# Delete lines matching pattern
sed -i '/^#/d' /etc/nginx/nginx.conf  # Remove comments

# Print specific lines
sed -n '10,20p' file      # Print lines 10-20

# Replace only on lines matching pattern
sed '/ERROR/s/OLD/NEW/g' file
```

### awk

```bash
# Print columns
awk '{print $1, $3}' /var/log/nginx/access.log

# Filter by column value
awk '$9 == 500 {print $1, $7}' /var/log/nginx/access.log  # 500 errors

# Sum values
awk '{sum += $10} END {print sum}' access.log  # Total bytes

# With field separator
awk -F: '{print $1}' /etc/passwd  # List usernames

# Line number condition
awk 'NR > 1 {print}' file  # Skip header line
```

### cut, sort, uniq

```bash
# Extract fields
cut -d' ' -f1,7 /var/log/nginx/access.log

# Sort
sort -k4 /var/log/nginx/access.log       # Sort by column 4
sort -n -k4                               # Numeric sort

# Unique (requires sort first)
sort access.log | uniq -c | sort -rn     # Count occurrences
```

---

## 7. SSH

### Basics

```bash
# Connect
ssh deploy@example.com
ssh -p 2222 deploy@example.com

# Key-based auth
ssh-keygen -t ed25519 -C "deploy@example.com"
ssh-copy-id deploy@example.com

# Config file (~/.ssh/config)
Host inventory-prod
    HostName ec2-54-123-45-67.compute-1.amazonaws.com
    User deploy
    IdentityFile ~/.ssh/inventory-prod
    Port 22
# Then: ssh inventory-prod

# SSH tunnels
ssh -L 8080:localhost:80 deploy@example.com  # Local port forwarding
ssh -R 8080:localhost:80 deploy@example.com  # Remote port forwarding
```

### SCP and rsync

```bash
# Copy files
scp file.txt deploy@example.com:/var/www/
scp -r /local/dir deploy@example.com:/remote/dir

# Rsync (differential, efficient)
rsync -avz /local/dir/ deploy@example.com:/remote/dir/
```

---

## 8. systemd Basics

```bash
# Service management
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx         # SIGHUP (config reload)
systemctl enable nginx         # Start on boot
systemctl disable nginx
systemctl status nginx         # Active/inactive, logs, PID
systemctl is-active nginx

# System state
systemctl list-units --type=service  # All services
systemctl list-units --failed         # Failed services
systemctl daemon-reload               # Reload unit files after edit
```

### journalctl

```bash
# View logs
journalctl -u nginx              # Service logs
journalctl -u nginx -f           # Follow
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx -p err       # Priority: emerg, alert, crit, err, warning, info, debug
journalctl -k                    # Kernel logs
```

---

## 9. Basic Networking

### curl

```bash
curl http://example.com
curl -I http://example.com       # Headers only
curl -v http://example.com       # Verbose (show request/response)
curl -X POST -d '{"key":"value"}' -H "Content-Type: application/json" http://api.example.com
curl -o output.html http://example.com  # Save to file
curl -k https://example.com      # Allow insecure SSL
curl --connect-timeout 5 http://example.com
```

### ping, dig, nslookup

```bash
ping -c 4 google.com       # Ping with count
dig example.com            # DNS lookup
dig -x 8.8.8.8             # Reverse DNS
nslookup example.com       # DNS lookup (simpler)
```

### netstat/ss

```bash
# Socket statistics (modern replacement for netstat)
ss -tuln                   # Listening TCP and UDP ports
ss -tulnp                  # With process names
ss -tan | grep ESTAB       # Established connections
ss -tan | grep TIME_WAIT   # Connections in TIME_WAIT

# Traditional netstat
netstat -tuln
netstat -tulnp
```

### ip command

```bash
ip addr                    # Interface IP addresses
ip route                   # Routing table
ip link set eth0 up/down   # Enable/disable interface
```

---

## 10. Q&A

**Q: What's the difference between a soft link and a hard link?**
A: Symlink (soft): reference to another file path. Can cross filesystems. Points to path (broken if target deleted). Hard link: same inode as target. Cannot cross filesystems. Target can be deleted — file persists through hard links.

**Q: What's `ps aux` show?**
A: All processes (a), user-format (u), processes not attached to terminal (x). Shows PID, CPU%, MEM%, VSZ/RSS, STAT, START, TIME, COMMAND.

**Q: What's the difference between SIGTERM and SIGKILL?**
A: SIGTERM (15): graceful shutdown — process can catch and clean up. SIGKILL (9): force kill — process cannot catch, immediate termination. Always try SIGTERM first.

**Q: What's an inode?**
A: Data structure storing file metadata (permissions, ownership, timestamps, size, disk block locations) except name. Each file has one inode identified by inode number.

**Q: How do you find files by size?**
A: `find / -type f -size +100M -exec ls -lh {} \;`

**Q: How do you check disk usage?**
A: `df -h` (filesystem-level), `du -sh /var/log` (directory-level), `ncdu` (interactive).

**Q: What's a zombie process?**
A: Process that has terminated but still has an entry in the process table because its parent hasn't called `wait()`. The parent must reap it. If the parent never does, the zombie stays forever.

**Q: How do you see which ports are listening?**
A: `ss -tuln` or `netstat -tuln`. Shows TCP/UDP listening ports, addresses, and state.
