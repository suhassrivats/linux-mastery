# 1. First Principle: The Troubleshooting Methodology

A good answer in interviews is this framework:

### Step 1 — Understand the Symptom

Ask questions like:

* What exactly is failing?
* When did it start?
* Is it intermittent or constant?
* What changed recently?
* Is it one machine or many?

Example questions:

```
Is the issue CPU, memory, disk, or network related?
Is it application level or system level?
Did a deploy happen recently?
```

---

### Step 2 — Check System Health Quickly

Run quick **high-level health commands**.

```
uptime
top
htop
vmstat
iostat
free -h
df -h
```

This tells you immediately:

| Resource | What to check   |
| -------- | --------------- |
| CPU      | high load       |
| Memory   | swap usage      |
| Disk     | full filesystem |
| IO       | disk bottleneck |

---

### Step 3 — Identify the Bottleneck

Classic bottlenecks:

1️⃣ CPU
2️⃣ Memory
3️⃣ Disk I/O
4️⃣ Network
5️⃣ Locks / contention

---

# 2. CPU Troubleshooting

### Check CPU Load

```
uptime
```

Example:

```
load average: 12.4 11.2 9.5
```

If system has **4 cores**, load **>4** means overload.

---

### Find CPU consuming processes

```
top
```

or

```
htop
```

Sort by CPU.

---

### Per-process CPU

```
ps aux --sort=-%cpu | head
```

---

### Per core usage

```
mpstat -P ALL 1
```

Important fields:

```
%user
%system
%iowait
```

High **iowait** = disk bottleneck.

---

### Deep CPU profiling

```
perf top
perf record
perf report
```

Used for **kernel / application profiling**.

---

# 3. Memory Troubleshooting

### Memory overview

```
free -h
```

Look for:

```
available
swap usage
```

---

### Detailed memory stats

```
vmstat 1
```

Important columns:

| Field | Meaning  |
| ----- | -------- |
| si    | swap in  |
| so    | swap out |

If swap is active → memory pressure.

---

### Which process uses memory

```
ps aux --sort=-%mem | head
```

---

### Memory leaks

```
pmap <pid>
smem
```

---

### Kernel memory

```
cat /proc/meminfo
```

---

# 4. Disk Troubleshooting

### Check disk usage

```
df -h
```

If disk is full → many services break.

---

### Which directory is big

```
du -sh *
```

Or

```
du -h --max-depth=1 /
```

---

### Disk IO bottleneck

```
iostat -x 1
```

Key metrics:

| Metric | Meaning      |
| ------ | ------------ |
| %util  | disk busy    |
| await  | latency      |
| svctm  | service time |

If **%util ≈ 100%** → disk saturated.

---

### Which process is causing IO

```
iotop
```

---

### Open files

```
lsof
```

---

# 5. Network Troubleshooting

### Check connectivity

```
ping
```

---

### DNS resolution

```
dig
nslookup
```

---

### Active connections

```
ss -tulpn
```

or

```
netstat -tulpn
```

---

### Network traffic

```
iftop
nload
```

---

### Packet capture

```
tcpdump
```

Example:

```
tcpdump -i eth0 port 443
```

---

### Trace route

```
traceroute
```

---

# 6. Process Troubleshooting

List processes:

```
ps aux
```

Tree view:

```
pstree
```

---

Kill process:

```
kill -9 <pid>
```

---

Check process limits:

```
ulimit -a
```

---

# 7. Log Analysis

Logs are **the most important debugging tool**.

### System logs

```
/var/log/syslog
/var/log/messages
```

---

### Kernel logs

```
dmesg
```

---

### Service logs (systemd)

```
journalctl -u nginx
```

or

```
journalctl -xe
```

---

### Live log monitoring

```
tail -f /var/log/app.log
```

---

# 8. File System Problems

Check inode exhaustion:

```
df -i
```

If inodes full → cannot create files.

---

Check mounted filesystem:

```
mount
```

---

Check disk errors:

```
dmesg | grep -i error
```

---

# 9. Service Troubleshooting

Check if service is running:

```
systemctl status nginx
```

Start service:

```
systemctl start nginx
```

Restart:

```
systemctl restart nginx
```

---

Check logs:

```
journalctl -u nginx
```

---

# 10. Advanced Debugging Tools

These impress interviewers.

### strace

Trace system calls.

```
strace -p <pid>
```

Example use case:

* program stuck
* file access issues
* network calls

---

### ltrace

Trace library calls.

---

### gdb

Debug running process.

---

### perf

CPU profiling.

```
perf top
```

---

### eBPF tools (modern SRE)

Examples:

```
bcc
bpftrace
```

Example:

```
execsnoop
opensnoop
biolatency
```

These trace **kernel events in production safely**.

---

# 11. Real SRE Troubleshooting Framework

A strong interview answer:

### 1️⃣ Reproduce the problem

Confirm issue.

---

### 2️⃣ Identify the scope

Is it:

* single host
* rack
* cluster
* region

---

### 3️⃣ Identify resource bottleneck

Check:

```
CPU
Memory
Disk
Network
```

---

### 4️⃣ Correlate with recent changes

Check:

* deployments
* config changes
* infra changes

---

### 5️⃣ Use logs + metrics

Look at:

* system logs
* application logs
* monitoring dashboards

---

### 6️⃣ Form hypothesis

Example:

```
High disk IO causing latency
```

---

### 7️⃣ Validate hypothesis

Use tools:

```
iostat
iotop
```

---

### 8️⃣ Fix and verify

Example:

* kill runaway process
* increase resources
* rollback deployment

---

# 12. Example Interview Answer

If asked:

**"How do you troubleshoot a slow Linux server?"**

Answer like this:

> First I try to identify if the issue is CPU, memory, disk, or network related. I start with high level tools like `uptime`, `top`, `vmstat`, and `iostat` to understand overall system health.
>
> If CPU is high, I use `top` or `ps` to find the offending process.
> If memory is low, I check `free -h` and `vmstat` for swap usage.
> If disk is slow, I use `iostat` and `iotop` to identify disk bottlenecks.
> For networking issues, I check connections using `ss` and packet flows with `tcpdump`.
>
> I also inspect logs using `journalctl` or files in `/var/log` to correlate system behavior with errors.
>
> If needed, I go deeper using tools like `strace`, `perf`, or eBPF tools for kernel-level debugging.

---

# 13. Tools That Impress in Senior / Principal Interviews

Mentioning these shows **deep Linux expertise**:

| Tool     | Purpose              |
| -------- | -------------------- |
| strace   | syscall tracing      |
| perf     | CPU profiling        |
| bpftrace | kernel observability |
| tcpdump  | packet capture       |
| iostat   | disk performance     |
| vmstat   | memory stats         |
| mpstat   | CPU per core         |
| lsof     | open files           |
| sar      | historical metrics   |

---

# 14. Quick Troubleshooting Command Set (Memorize)

```
top
htop
vmstat
iostat
mpstat
free -h
df -h
du -sh
ss -tulpn
lsof
journalctl
dmesg
tcpdump
strace
```
