# Linux — Intermediate Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** Linux basics (files, permissions, processes, SSH)  
> **Estimated time:** 8–10 hours

---

## Table of Contents

1. Process Signals (Deep Dive)
2. strace and lsof
3. Disk Management
4. Memory Management
5. I/O Performance
6. systemd Deep Dive
7. Cron and Scheduling
8. ulimit
9. Firewall (iptables/ufw)
10. Q&A

---

## 1. Process Signals (Deep Dive)

### Common signals

| Signal | Number | Default action | Can catch? | Use case |
|--------|--------|---------------|------------|----------|
| SIGTERM | 15 | Terminate | Yes | Graceful shutdown |
| SIGKILL | 9 | Terminate | No | Force kill |
| SIGSTOP | 19 | Stop | No | Pause process |
| SIGCONT | 18 | Continue | Yes | Resume process |
| SIGHUP | 1 | Terminate | Yes | Reload config / restart |
| SIGINT | 2 | Terminate | Yes | Ctrl+C |
| SIGQUIT | 3 | Core dump | Yes | Terminate with core dump |
| SIGUSR1 | 10 | Terminate | Yes | User-defined (log rotation) |
| SIGUSR2 | 12 | Terminate | Yes | User-defined |
| SIGPIPE | 13 | Terminate | Yes | Broken pipe |
| SIGCHLD | 17 | Ignore | Yes | Child process terminated |

### Signal handling in scripts

```bash
# Trap signals for cleanup
#!/bin/bash
cleanup() {
    echo "Cleaning up..."
    rm -f /tmp/lock
    exit 0
}
trap cleanup SIGTERM SIGINT SIGHUP

# Run main process
echo "Running..."
while true; do sleep 1; done
```

### Core dumps

```bash
# Enable core dumps
ulimit -c unlimited
echo "/var/coredumps/core.%e.%p" > /proc/sys/kernel/core_pattern

# Triggered by SIGQUIT (Ctrl+\)
# Analyze with gdb: gdb /path/to/binary /path/to/core
```

---

## 2. strace and lsof

### strace (system call tracer)

Trace system calls made by a process:

```bash
# Trace a command
strace -f php artisan cache:clear

# Attach to running process
strace -p 1234

# Common options
strace -e openat,read,write -p 1234    # Filter specific syscalls
strace -c -p 1234                       # Count syscalls (summary)
strace -T -p 1234                       # Show time per syscall
strace -tt -p 1234                      # Show timestamps
strace -o /tmp/strace.log -p 1234      # Write to file
```

**Debugging scenarios:**

```bash
# Why is a process slow?
strace -c -p 1234
# Shows which syscalls take the most time

# Why can't PHP connect to the database?
strace -e connect -p $(pidof php-fpm)

# What files is a process reading/writing?
strace -e openat,read,write -p 1234
```

### lsof (list open files)

```bash
# List all open files for a process
lsof -p 1234

# List processes using a specific file
lsof /var/log/nginx/access.log

# List network connections
lsof -i :8080                     # Connections on port 8080
lsof -i TCP:5432                  # TCP connections to port 5432

# List Unix domain sockets
lsof -U

# List files by type
lsof -d 0-10                      # File descriptors 0-10
lsof -a -p 1234 -i4               # IPv4 for PID 1234

# Show process name (not just PID)
lsof -c nginx                     # All files opened by nginx processes
```

**Common uses:**
```bash
# Port already in use?
lsof -i :8080
# Kill the process using the port
kill -9 $(lsof -t -i :8080)

# File still in use after delete?
lsof | grep "(deleted)"
# These files are held open by processes but unlinked (disk space not freed)

# What connections does my database have?
lsof -i @10.0.1.5:5432
```

### fuser

```bash
# Find processes using a file
fuser /var/log/nginx/access.log

# Find process using a port
fuser 8080/tcp

# Kill processes using a file
fuser -k /var/log/nginx/access.log
```

---

## 3. Disk Management

### Viewing disk info

```bash
# Disk usage
df -h                # Human-readable filesystem usage
df -h /var/log       # Specific mount point
du -sh /var/log      # Directory size
du -h --max-depth=1  # Top-level directories

# Block devices
lsblk                # List block devices (tree view)
lsblk -f             # With filesystem info
fdisk -l             # Partition table
```

### Mounting filesystems

```bash
# Mount
mount /dev/sdb1 /mnt/data
mount -t ext4 /dev/sdb1 /mnt/data
umount /mnt/data

# Persistent mount (/etc/fstab)
/dev/sdb1  /mnt/data  ext4  defaults,noatime  0  2

# Check fstab
mount -a  # Mount all entries in fstab
```

### Filesystem check

```bash
# Check filesystem (unmount first)
umount /dev/sdb1
fsck /dev/sdb1
fsck -f /dev/sdb1     # Force check even if clean
```

---

## 4. Memory Management

### Viewing memory

```bash
# Free memory
free -h
              total        used        free      shared  buff/cache   available
Mem:           15Gi       4.2Gi       1.5Gi       245Mi       9.6Gi       9.8Gi
Swap:         2.0Gi       200Mi       1.8Gi

# Detailed memory info
cat /proc/meminfo
# MemTotal, MemFree, MemAvailable, Buffers, Cached, SwapTotal, SwapFree

# Per-process memory
ps aux --sort=-%mem | head -10
top -o %MEM
```

### Memory concepts

| Term | Meaning |
|------|---------|
| **RSS** | Resident Set Size — physical memory used (includes shared pages) |
| **VSZ** | Virtual Memory Size — total virtual address space |
| **PSS** | Proportional Set Size — RSS / number of processes sharing (accurate) |
| **USS** | Unique Set Size — memory unique to this process |
| **Buffers** | Memory used for filesystem metadata |
| **Cached** | Memory used for page cache (file content cached) |
| **Available** | Estimated memory available for new allocations |
| **Swap** | Memory on disk (when RAM is full) |

> **Trap:** RSS overcounts memory used by shared libraries. If 10 processes load the same glibc, RSS counts it 10 times. PSS divides shared pages — use PSS for accurate per-process memory.

### Swap

```bash
# Create swap file
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# Make permanent (/etc/fstab)
/swapfile  none  swap  sw  0  0

# Check swap usage
swapon --show
cat /proc/swaps

# Swap priority
swapon -p 10 /swapfile   # Higher priority = used first
```

### OOM Killer

```bash
# Adjust OOM score for a process
echo -1000 > /proc/1234/oom_score_adj  # Less likely to be killed
echo 1000 > /proc/1234/oom_score_adj   # More likely to be killed

# Check OOM score
cat /proc/1234/oom_score

# View OOM killer history
dmesg | grep -i "killed process"
```

> **Trap:** Setting `oom_score_adj` to -1000 doesn't guarantee a process won't be killed. It only makes it less likely. The kernel always picks the highest `oom_score` process in an OOM situation.

---

## 5. I/O Performance

### iostat

```bash
# Install: apt install sysstat

# Basic usage
iostat -x 1           # Extended stats, every 1 second
iostat -x 1 5         # 5 samples

# Key columns:
# %util — percentage of time device was busy (100% = saturated)
# rkB/s — KB read per second
# wkB/s — KB written per second
# await — average time (ms) for I/O request
# svctm — average service time (ms) per I/O
# avgqu-sz — average queue length
```

**Interpreting %util:**

| %util | Meaning |
|-------|---------|
| < 60% | Normal |
| 60–80% | Moderate load |
| 80–90% | Heavy load, may cause latency |
| > 90% | Near saturation, significant latency |

> **Trap:** %util at 100% on SSD doesn't mean the device is saturated the same way it does on HDD. Modern SSDs can handle multiple I/Os in parallel. Use `avgqu-sz` and `await` for SSDs instead.

### iotop

```bash
# Interactive I/O monitoring per process
iotop
iotop -oP           # Only processes doing I/O, show PIDs only
```

---

## 6. systemd Deep Dive

### Unit file structure

```
/etc/systemd/system/myapp.service
```

```ini
[Unit]
Description=My Application
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=notify
User=deploy
WorkingDirectory=/var/www/app
ExecStart=/usr/local/bin/myapp server
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
RestartSec=5
Environment=APP_ENV=production
EnvironmentFile=/etc/myapp/env
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### Service types

| Type | Behavior |
|------|----------|
| `simple` | Process runs in foreground. systemd assumes running immediately. |
| `forking` | Process forks (daemonizes). systemd waits for parent to exit. |
| `oneshot` | Run once, exits. systemd considers it active while running. |
| `notify` | Process sends `sd_notify()` when ready. |
| `dbus` | Process registers on D-Bus bus. |

### journalctl filtering

```bash
# Last hour
journalctl --since "1 hour ago"

# Time range
journalctl --since "2026-07-26 10:00:00" --until "2026-07-26 11:00:00"

# Service specific
journalctl -u nginx -u php8.2-fpm

# Priority filtering
journalctl -p err           # ERROR and above
journalctl -p info -p err   # INFO and ERROR (inclusive range)

# JSON output
journalctl -o json

# Disk usage
journalctl --disk-usage

# Rotate logs
journalctl --vacuum-size=500M    # Keep max 500MB
journalctl --vacuum-time=7d      # Keep 7 days
```

---

## 7. Cron and Scheduling

### Crontab format

```
# ┌── minute (0-59)
# │ ┌── hour (0-23)
# │ │ ┌── day of month (1-31)
# │ │ │ ┌── month (1-12)
# │ │ │ │ ┌── day of week (0-7, Sun=0)
# │ │ │ │ │
  * * * * * command
```

```bash
# Edit crontab
crontab -e

# Examples
*/5 * * * * /usr/bin/php /var/www/app/artisan schedule:run >> /dev/null 2>&1
0 3 * * * /usr/local/bin/backup.sh
0 9-17 * * 1-5 /usr/local/bin/workday.sh  # Every hour 9-5, weekdays
@daily /usr/local/bin/daily.sh
@reboot /usr/local/bin/startup.sh

# List crontabs
crontab -l

# System crontab (/etc/crontab) — different format (has user field)
0 3 * * * root /usr/local/bin/backup.sh
```

### at (one-time tasks)

```bash
echo "php artisan report:generate" | at now + 1 hour
atq     # List queued jobs
atrm 5  # Remove job 5
```

---

## 8. ulimit

### Resource limits

```bash
# View all limits for current shell
ulimit -a

# Key limits:
ulimit -n   # Open file descriptors (most commonly hit)
ulimit -u   # Max user processes
ulimit -c   # Core file size
ulimit -s   # Stack size

# Set limits
ulimit -n 65536

# Persistent (in /etc/security/limits.conf)
deploy   soft   nofile   65536
deploy   hard   nofile   65536
```

### Common limits issues

```
# "Too many open files" error
→ ulimit -n is too low for the application
→ Check current: cat /proc/{pid}/limits | grep "open files"

# "Cannot fork" error
→ ulimit -u is too low (max user processes)
→ Check current: cat /proc/{pid}/limits | grep "processes"
```

---

## 9. Firewall (iptables/ufw)

### iptables

```bash
# List rules
iptables -L -n -v           # List rules with counters
iptables -t nat -L -n       # NAT table (port forwarding)

# Allow SSH (port 22)
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP/HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow established connections (return traffic)
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Drop all other incoming traffic
iptables -P INPUT DROP

# Save rules
iptables-save > /etc/iptables/rules.v4
```

### ufw (Uncomplicated Firewall)

```bash
ufw enable
ufw allow ssh
ufw allow 80/tcp
ufw allow 443/tcp
ufw deny 3306    # Block MySQL from outside
ufw status verbose
```

---

## 10. Q&A

**Q: How do you find which process is using port 8080?**
A: `lsof -i :8080` or `fuser 8080/tcp` or `ss -tulnp | grep 8080`.

**Q: How do you trace system calls of a process?**
A: `strace -p $PID`. Filter with `-e` (e.g., `strace -e openat,read -p $PID`).

**Q: What's the difference between `df` and `du`?**
A: `df` shows filesystem-level disk usage (how much space is free on the partition). `du` shows directory-level usage (how much space specific directories use).

**Q: How do you check memory usage per process?**
A: `ps aux --sort=-%mem` or `top -o %MEM` or `cat /proc/$PID/status | grep VmRSS`.

**Q: What's the difference between RSS, VSZ, and PSS?**
A: RSS = physical memory (includes shared pages counted multiple times). VSZ = virtual address space. PSS = RSS with shared pages divided evenly.

**Q: How do you increase the max number of open files?**
A: `ulimit -n 65536` (session), or `/etc/security/limits.conf` (persistent), or `LimitNOFILE=65536` in systemd unit.

**Q: What's a zombie process and how do you clean it?**
A: Process that exited but its parent didn't call `wait()`. Kill the parent (or wait for it to call wait). Zombie processes take no resources (just a PID table entry).

**Q: How do you create a systemd service?**
A: Create `/etc/systemd/myapp.service` with [Unit], [Service], [Install] sections. Run `systemctl daemon-reload` then `systemctl start myapp`.

**Q: What's `strace -c` useful for?**
A: Shows a summary/count of all system calls called by a process and the time spent in each. Useful for identifying performance bottlenecks.

**Q: How do you find a deleted file still consuming disk space?**
A: `lsof | grep "(deleted)"`. The file is unlinked but held open by a process. Kill or restart the process to free the space.
