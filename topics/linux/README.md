# Linux — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** Linux servers running Laravel + PostgreSQL + Redis, Docker containers on EC2/ECS, SSH access, troubleshooting production incidents  
> **Note:** Linux proficiency is expected for senior backend engineers. The senior signal is **troubleshooting production issues using system tools (strace, lsof, tcpdump, perf)** and **understanding OS internals (process scheduling, memory management, file descriptors, signals)** — not just basic commands.

---

## How to use this material

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud | 20 min/section |
| 2 | Run the commands on a Linux VPS or Docker container | 20 min/section |
| 3 | Answer the section's Q&A without looking, then diff | 20 min/section |
| 4 | Debug a production issue scenario using Linux tools | 20 min |

**The senior signal is performance troubleshooting with system tools.** Knowing `ls` and `grep` is table stakes; using `strace`, `perf`, `lsof`, and `tcpdump` to debug production issues is the differentiator.

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | File system hierarchy, file permissions, users/groups, process management (ps, top, kill), package management (apt, yum), text processing (grep, sed, awk, cut), SSH, systemd basics, basic networking (curl, ping, netstat) | 4–6 hours |
| [`02-intermediate.md`](./02-intermediate.md) | Process signals, strace/lsof/fuser, disk management (df, du, fdisk, lsblk), memory management (free, vmstat, /proc/meminfo), I/O scheduling (iostat, iotop), systemd service management, journalctl, cron/at, ulimit, firewall (iptables, ufw) | 8–10 hours |
| [`03-senior.md`](./03-senior.md) | Kernel tuning (sysctl), cgroups/namespaces (container fundamentals), perf profiling, tcpdump network analysis, OOM killer tuning, swap management, RLIMIT tuning, kernel panic/crash analysis, /proc filesystem deep dive, Linux security (SELinux, AppArmor, capabilities, seccomp) | 10–12 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 130+ interview questions, troubleshooting scenarios, performance analysis exercises | Ongoing drill |

---

## Coverage map

### File system
- FHS (Filesystem Hierarchy Standard): /bin, /sbin, /usr, /var, /etc, /proc, /sys, /tmp
- File types: regular, directory, symlink, socket, pipe, device (block/char)
- Permissions: rwx, sticky bit, SUID, SGID, ACLs
- Links: hard links vs symbolic links
- inodes: what they are, how to check, how to find files using inode
- Virtual filesystems: /proc, /sys, /dev, tmpfs

### Process management
- Process states: running, sleeping (interruptible/uninterruptible), zombie, stopped
- Process tree: PID, PPID, PGID, SID
- Signals: SIGTERM (15), SIGKILL (9), SIGSTOP (19), SIGCONT (18), SIGHUP (1), SIGUSR1/2
- nice/renice: process priority
- nohup, disown, setsid: detached processes
- daemonization: double-fork pattern

### Memory
- Virtual memory, paging, swapping
- RSS vs VSZ vs PSS vs USS
- /proc/meminfo: MemTotal, MemFree, Buffers, Cached, SwapTotal, SwapFree
- Buffer cache, page cache, dentry cache
- slab allocator
- OOM killer: oom_score, oom_score_adj, oom_adj

### Disk and I/O
- Block devices: lsblk, fdisk, parted
- Filesystem types: ext4, XFS, btrfs, ZFS
- Mount options: noatime, nodiratime, barrier, data=ordered
- I/O schedulers: cfq, deadline, noop, mq-deadline, kyber, BFQ
- iostat: tps, rkB/s, wkB/s, await, svctm, %util
- strace: system call tracing
- lsof: open files per process
- fsck: filesystem check

### Networking
- TCP/IP stack, socket states (LISTEN, ESTABLISHED, TIME_WAIT, CLOSE_WAIT)
- tcpdump: packet capture and analysis
- netstat/ss: socket statistics
- ip route, ip addr: routing and interface configuration
- iptables/nftables: packet filtering, NAT, port forwarding
- DNS resolution: /etc/hosts, /etc/resolv.conf, nsswitch.conf
- /etc/sysctl.conf: network tuning (net.core.somaxconn, tcp_tw_reuse, tcp_fin_timeout)

### Systemd
- Units: service, socket, timer, mount, path
- systemctl: start, stop, restart, enable, disable, status, daemon-reload
- journalctl: viewing logs, filtering by unit, time range, priority
- Unit files: [Unit], [Service], [Install] sections
- Service types: simple, forking, oneshot, notify, dbus

### Security
- Linux capabilities: CAP_NET_BIND_SERVICE, CAP_SYS_ADMIN, etc.
- seccomp: system call filtering
- SELinux: contexts, booleans, enforcing/permissive
- AppArmor: profiles, complain/enforce mode
- Namespaces: PID, net, mount, user, UTS, IPC, cgroup (container primitives)
- cgroups v1/v2: resource limits (CPU, memory, I/O, pids)

### Performance tools
- top/htop/atop: process/system monitoring
- strace: system call tracing
- ltrace: library call tracing
- perf: CPU profiling, sampling, hardware counters
- flamegraphs: CPU and off-CPU profiling
- sar: system activity reporting
- dmesg: kernel ring buffer

---

## Study order recommendation

Focus on troubleshooting tools (strace, lsof, tcpdump) and OS fundamentals (processes, memory, filesystem).

```
Week 1:  01-basic.md          + Run commands on a Linux VM/container
Week 2:  02-intermediate.md   + strace/lsof/tcpdump exercises
Week 3:  03-senior.md         + perf profiling, cgroups, sysctl tuning
Week 4+: 04-question-bank.md daily drill
```
