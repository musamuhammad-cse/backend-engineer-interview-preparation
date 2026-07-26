# Linux — Advanced Tier

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** Linux intermediate (signals, strace, lsof, iostat, systemd, cgroups)  
> **Estimated time:** 10–14 hours

---

## Table of Contents

1. cgroups v1/v2
2. Namespaces and Container Isolation
3. Linux Security Modules (SELinux/AppArmor)
4. Capabilities and seccomp
5. BPF and bpftrace
6. Performance Tuning (/proc/sys)
7. Network Performance (sysctl)
8. Filesystem Performance
9. Debugging Production Incidents
10. Q&A

---

## 1. cgroups v1/v2

cgroups (control groups) limit and account for resource usage (CPU, memory, I/O, PID) per process group.

### cgroups v1 (legacy)

```
# Hierarchy mounted at /sys/fs/cgroup
/sys/fs/cgroup/
├── cpu/
├── cpuacct/
├── cpuset/
├── memory/
├── blkio/
├── devices/
├── freezer/
├── net_cls/
├── net_prio/
├── pids/
└── systemd/
```

```bash
# Create a cgroup
mkdir /sys/fs/cgroup/memory/myapp
echo 500M > /sys/fs/cgroup/memory/myapp/memory.limit_in_bytes
echo 1234 > /sys/fs/cgroup/memory/myapp/cgroup.procs

# Limit CPU to 50% of one core
mkdir /sys/fs/cgroup/cpu/myapp
echo 50000 > /sys/fs/cgroup/cpu/myapp/cpu.cfs_quota_us
echo 100000 > /sys/fs/cgroup/cpu/myapp/cpu.cfs_period_us

# Limit to 100 PIDs
mkdir /sys/fs/cgroup/pids/myapp
echo 100 > /sys/fs/cgroup/pids/myapp/pids.max
```

### cgroups v2 (unified hierarchy)

Default on modern Linux (Ubuntu 22.04+, Fedora 31+, Debian 11+).

```
# Single hierarchy at /sys/fs/cgroup
/sys/fs/cgroup/
├── cgroup.controllers    # Available controllers
├── cgroup.subtree_control  # Controllers enabled for children
├── system.slice/
│   ├── nginx.service/
│   │   ├── memory.current
│   │   ├── memory.max
│   │   ├── cpu.weight
│   │   ├── cpu.max
│   │   ├── io.max
│   │   └── pids.max
│   └── ...
└── user.slice/
```

```bash
# Create cgroup
mkdir /sys/fs/cgroup/myapp
echo "+memory +cpu +pids" > /sys/fs/cgroup/cgroup.subtree_control

# Limit memory
echo 500M > /sys/fs/cgroup/myapp/memory.max
echo 1G > /sys/fs/cgroup/myapp/memory.swap.max

# Limit CPU
echo "50000 100000" > /sys/fs/cgroup/myapp/cpu.max  # 50% of one core

# CPU weight (relative share)
echo 100 > /sys/fs/cgroup/myapp/cpu.weight  # Default 100

# Limit PIDs
echo 100 > /sys/fs/cgroup/myapp/pids.max

# Attach process
echo 1234 > /sys/fs/cgroup/myapp/cgroup.procs
```

### OOM in cgroups

```bash
# Check if cgroup OOM occurred
cat /sys/fs/cgroup/myapp/memory.events
# oom 1
# oom_kill 1

# OOM kills are reported in kernel log
dmesg | grep -i "oom"
```

---

## 2. Namespaces and Container Isolation

### Namespace types

| Namespace | Isolates | Created by |
|-----------|----------|-----------|
| PID (`CLONE_NEWPID`) | Process IDs | Docker, systemd |
| Network (`CLONE_NEWNET`) | Network interfaces, routing, iptables | Docker |
| Mount (`CLONE_NEWNS`) | Mount points | Docker |
| UTS (`CLONE_NEWUTS`) | Hostname, domain name | Docker |
| IPC (`CLONE_NEWIPC`) | System V IPC, POSIX message queues | Docker |
| User (`CLONE_NEWUSER`) | UID/GID mapping | Docker (user namespace remapping) |
| Cgroup (`CLONE_NEWCGROUP`) | cgroup root | Docker |

```bash
# List namespaces for a process
ls -la /proc/1234/ns/
# lrwxrwxrwx 1 root root 0 Jul 26 10:00 cgroup -> 'cgroup:[4026531835]'
# lrwxrwxrwx 1 root root 0 Jul 26 10:00 ipc -> 'ipc:[4026531839]'
# lrwxrwxrwx 1 root root 0 Jul 26 10:00 mnt -> 'mnt:[4026531841]'
# lrwxrwxrwx 1 root root 0 Jul 26 10:00 net -> 'net:[4026531993]'
# lrwxrwxrwx 1 root root 0 Jul 26 10:00 pid -> 'pid:[4026531836]'
# lrwxrwxrwx 1 root root 0 Jul 26 10:00 user -> 'user:[4026531837]'
# lrwxrwxrwx 1 root root 0 Jul 26 10:00 uts -> 'uts:[4026531838]'

# Check if two processes share a namespace
# Same inode number = same namespace
```

### User namespace

```bash
# Create a user namespace with UID mapping
unshare --user --map-root-user bash
# Inside: you are root (UID 0) mapped to your original UID outside

# Check mapping
cat /proc/self/uid_map
#          0       1000          1
# Inside UID Outside UID  Length
```

### Creating a container with namespaces

```bash
# Create a new process in isolated namespaces
unshare --fork --pid --mount --uts --net --ipc bash
```

---

## 3. Linux Security Modules (SELinux/AppArmor)

### SELinux

```
# Check status
getenforce
# Enforcing | Permissive | Disabled

# Set mode
setenforce 0    # Permissive (log only)
setenforce 1    # Enforcing

# Context
ls -Z /var/www/html/index.html
# system_u:object_r:httpd_sys_content_t:s0
#   user:role:type:level

# Check log
ausearch -m avc -ts recent
```

### AppArmor

```bash
# Check status
aa-status

# Profiles in /etc/apparmor.d/
/etc/apparmor.d/
├── usr.sbin.nginx
├── usr.sbin.mysqld
└── ...

# Profile example
cat /etc/apparmor.d/usr.sbin.nginx
# include <tunables/global>
# /usr/sbin/nginx {
#   /etc/nginx/** r,
#   /var/log/nginx/** rw,
#   /var/www/** r,
#   /run/nginx.pid rw,
#   network tcp,
# }

# Set profile mode
aa-complain /usr/sbin/nginx   # Log violations
aa-enforce /usr/sbin/nginx    # Block violations
```

---

## 4. Capabilities and seccomp

### Linux capabilities

Instead of giving root (all powers), grant specific capabilities.

```bash
# Common capabilities
cap_net_bind_service    # Bind to port < 1024
cap_net_admin           # Network administration (iptables, routing)
cap_sys_admin           # System administration (mount, swapon)
cap_sys_ptrace          # Trace processes
cap_sys_nice            # Change process priority
cap_dac_override        # Bypass file permission checks
cap_chown               # Change file ownership
cap_kill                # Send signals
cap_setuid              # Change UID

# Set capabilities on binary
setcap cap_net_bind_service=+ep /usr/bin/myapp

# Check capabilities
getcap /usr/bin/myapp
getpcaps 1234          # Capabilities of running process

# Drop capabilities in systemd
# [Service]
# CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_NET_ADMIN
# AmbientCapabilities=CAP_NET_BIND_SERVICE
```

### seccomp (secure computing mode)

```bash
# Check if seccomp is enabled for a process
cat /proc/1234/status | grep Seccomp
# Seccomp: 2   (0=disabled, 1=strict, 2=filtered)

# seccomp profiles for Docker
# Default: blocks ~44 dangerous syscalls
# Custom: define allowed/denied syscalls
```

---

## 5. BPF and bpftrace

### bpftrace examples

```bash
# Install: apt install bpftrace

# Syscall tracing
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'

# Count syscalls per process
bpftrace -e 'tracepoint:syscalls:sys_enter_read { @[comm] = count(); }'

# Files opened by process
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[pid, comm] = count(); }'

# Disk I/O latency histogram
bpftrace -e 'kprobe:blk_account_io_start { @start[tid] = nsecs; }
             kretprobe:blk_account_io_done /@start[tid]/ {
               @us = hist((nsecs - @start[tid]) / 1000);
               delete(@start[tid]);
             }'
```

### BCC tools

```bash
# Install: apt install bpfcc-tools

# File opens per process
filetop

# TCP connections
tcptop

# Cache misses
cachestat

# New processes
execsnoop

# I/O latency
biolatency
```

---

## 6. Performance Tuning (/proc/sys)

### Memory tuning

```bash
# Reduce swappiness (default 60)
echo 10 > /proc/sys/vm/swappiness

# Increase max map count (for Elasticsearch)
echo 262144 > /proc/sys/vm/max_map_count

# Dirty page ratios
echo 5 > /proc/sys/vm/dirty_ratio                    # Max dirty pages (% of RAM) before write
echo 10 > /proc/sys/vm/dirty_background_ratio         # Start background write at this %

# OOM settings
echo 1 > /proc/sys/vm/overcommit_memory               # Always allow overcommit
echo 100 > /proc/sys/vm/overcommit_ratio              # % of RAM for overcommit
```

### Filesystem tuning

```bash
# Increase max file watchers (for inotify)
echo 524288 > /proc/sys/fs/inotify/max_user_watches

# Increase max open files
echo 100000 > /proc/sys/fs/file-max

# Increase AIO (for databases)
echo 1048576 > /proc/sys/fs/aio-max-nr
```

---

## 7. Network Performance (sysctl)

### TCP tuning

```bash
# These go in /etc/sysctl.conf

# Increase TCP buffer sizes
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Enable TCP BBR congestion control
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# Enable TCP Fast Open
net.ipv4.tcp_fastopen = 3

# Increase backlog
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# Reduce TIME_WAIT
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1

# Keepalive
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 60
net.ipv4.tcp_keepalive_probes = 5

# Connection tracking
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30

# Disable slow start after idle
net.ipv4.tcp_slow_start_after_idle = 0

# Apply
sysctl -p
```

---

## 8. Filesystem Performance

### Mount options

```bash
# noatime — disable access time updates (biggest gain)
mount -o noatime,data=writeback /dev/sda1 /mnt/data

# ext4 options
# data=ordered (default): metadata written after data
# data=writeback: metadata written anytime (faster, risk on crash)
# nobarrier: disable write barriers (faster, risk on power loss)
# noatime: skip access time updates

# XFS options
# nobarrier, largeio, swalloc
```

### I/O schedulers

```bash
# Check current scheduler
cat /sys/block/sda/queue/scheduler

# Set scheduler (NVMe → none, SSD → none/mq-deadline, HDD → bfq)
echo none > /sys/block/nvme0n1/queue/scheduler

# Tuning
echo 512 > /sys/block/sda/queue/nr_requests    # Max I/O requests
echo 2048 > /sys/block/sda/queue/read_ahead_kb  # Read-ahead
```

### SSD-specific

```bash
# Enable TRIM/discard
fstrim -v /

# Check alignment
parted /dev/sda align-check opt 1

# I/O stats (nvme-cli)
nvme list
nvme smart-log /dev/nvme0
```

---

## 9. Debugging Production Incidents

### High CPU

```bash
# Find process
top -o %CPU
ps aux --sort=-%cpu | head -5

# Inside process — find thread
top -H -p 1234
# Convert thread ID to hex
printf "%x\n" 4567

# Get stack trace
pstack 1234            # Thread stacks
strace -p 1234          # System calls
perf top -p 1234        # Sampling profiler
perf record -p 1234 -g  # Record with call graph
perf report             # Analyze
```

### Memory leak

```bash
# Monitor growth
watch -n 1 'ps aux | grep myapp | grep -v grep'
# RSS column keeps growing

# Collect memory info
cat /proc/1234/status | grep VmRSS
cat /proc/1234/smaps | grep -A5 "\[heap\]"

# Use valgrind
valgrind --leak-check=full ./myapp

# Use heaptrack
heaptrack ./myapp

# Kernel memory
slabtop            # Kernel slab allocator
cat /proc/meminfo  # Check Slab field
```

### Disk I/O

```bash
# Find which processes are doing I/O
iotop -oP

# Check device-level
iostat -x 1
# If %util > 90% and await > r_await + w_await → saturated

# Find files being written
lsof +D /var/log 2>/dev/null | grep -i deleted

# Check latency
ioping -c 10 /var/log    # Measure I/O latency
ioping -c 10 -W /var/log # Include cache effects
```

### Network issues

```bash
# Dropped packets
netstat -s | grep -i drop
cat /proc/net/softnet_stat

# Connection backlog drops
cat /proc/net/netstat | grep ListenDrops

# Socket stats
ss -tan | awk '{print $1}' | sort | uniq -c

# TCP retransmits
netstat -s | grep retransmit

# Packet capture
tcpdump -i eth0 port 80 -w /tmp/capture.pcap
tcpdump -i eth0 'tcp[13] & 2 != 0'  # SYN packets
tshark -r /tmp/capture.pcap          # Analyze
```

---

## 10. Q&A

**Q: How do containers isolate processes?**
A: Namespaces (PID, Network, Mount, UTS, IPC, User, Cgroup) and cgroups (resource limits). Combined with a shared kernel.

**Q: What's the difference between cgroups v1 and v2?**
A: v1: separate hierarchies per controller → complexity, inconsistency. v2: unified hierarchy, safer defaults, pressure stall information (PSI), I/O limits.

**Q: How do you limit memory for a process without Docker?**
A: Use cgroups: `mkdir /sys/fs/cgroup/myapp && echo 500M > /sys/fs/cgroup/myapp/memory.max && echo $PID > /sys/fs/cgroup/myapp/cgroup.procs`

**Q: What's the OOM killer and how do you influence it?**
A: Kernel mechanism that kills processes when memory is exhausted. Tune with `oom_score_adj`. Check kernel logs for kills.

**Q: What's the difference between capabilities and seccomp?**
A: Capabilities: grant granular root-level privileges (bind to low port, change ownership). seccomp: filter system calls (allow/deny specific syscalls). Both reduce kernel attack surface.

**Q: How do you diagnose high CPU without tools?**
A: `top -H -p $PID` to find thread, then `/proc/$PID/stack` for kernel stack, or `gdb -p $PID` with `bt` for user-space stack.

**Q: How do you find which files a process is I/O-heavy on?**
A: `strace -e openat,read,write -p $PID` or `lsof -p $PID` or `fatrace` (fanotify-based file access monitor).

**Q: How do you tune Linux for high-throughput TCP servers?**
A: Increase buffer sizes, enable BBR, tune backlog, reduce TIME_WAIT with `tcp_tw_reuse`, increase connection tracking limits.

**Q: What's BPF used for in production?**
A: Performance tracing (bpftrace), network filtering (XDP), security (seccomp filters), observability (BCC tools like `execsnoop`, `tcptop`, `biolatency`).

**Q: How do you debug a "Too many open files" error?**
A: Check `cat /proc/$PID/limits | grep "open files"`, then `lsof -p $PID | wc -l` to count open FDs. Increase `ulimit -n` or find the file descriptor leak.
