# 1. What a System Call Is

A **system call (syscall)** is the **controlled way a user program asks the Linux kernel to do something privileged**.

User programs cannot directly:

* Access hardware
* Allocate kernel memory
* Create processes
* Read disk blocks
* Manage network interfaces

Instead, they must **request the kernel** through a system call.

Think of it like:

| Layer        | Responsibility                                         |
| ------------ | ------------------------------------------------------ |
| User Space   | Your programs (bash, python, curl, nginx)              |
| Kernel Space | Hardware access, process scheduling, memory management |

System calls are the **gateway between user space and kernel space**.

```
+-----------------------+
| User Applications     |
|  (bash, python, curl) |
+----------|------------+
           |
           | System Call
           v
+-----------------------+
| Linux Kernel          |
|  - Process mgmt       |
|  - Memory mgmt        |
|  - File systems       |
|  - Networking         |
+----------|------------+
           |
           v
        Hardware
```

---

# 2. Why System Calls Exist

The OS must **protect the system**.

Imagine if every program could:

* Write directly to disk sectors
* Modify kernel memory
* Interrupt the CPU

The system would crash constantly.

So Linux enforces:

```
User mode  (restricted)
Kernel mode (privileged)
```

System calls safely **switch the CPU from user mode to kernel mode**.

---

# 3. Example: Reading a File

When you write this program:

```c
int fd = open("file.txt", O_RDONLY);
read(fd, buffer, 100);
```

Your program is actually invoking system calls:

```
open()
read()
```

The flow looks like:

```
User Program
     |
     | open()
     v
C Library (glibc wrapper)
     |
     | syscall instruction
     v
Linux Kernel
     |
     | VFS
     | filesystem driver
     v
Disk
```

Important detail:

`open()` you call is **not the raw syscall**.

It is a **glibc wrapper** around the syscall.

---

# 4. Major Categories of Linux System Calls

Linux has **300+ system calls**.

Main groups:

| Category          | Examples                 |
| ----------------- | ------------------------ |
| Process Control   | fork, exec, exit         |
| File Operations   | open, read, write, close |
| Device Management | ioctl                    |
| Information       | getpid, getuid           |
| Communication     | pipe, socket             |
| Memory Management | mmap, brk                |

---

# 5. Most Important System Calls

Here are some you should know deeply.

### Process Management

```
fork()
execve()
wait()
exit()
getpid()
```

Example:

```
bash
  |
  | fork()
  v
child process
  |
  | execve()
  v
/bin/ls
```

---

### File Operations

```
open()
read()
write()
close()
stat()
```

Example:

```
read(fd, buffer, 100)
```

Kernel:

1. Verify fd
2. Locate file
3. Read page cache
4. Copy data to user buffer

---

### Memory Management

```
mmap()
brk()
munmap()
```

Example:

```
mmap() → map file into memory
```

Used by:

* databases
* shared memory
* large file access

---

### Networking

```
socket()
bind()
listen()
accept()
send()
recv()
```

Example:

A web server:

```
socket()
bind()
listen()
accept()
```

---

# 6. How a System Call Actually Works (Deep)

Let's say you run:

```
cat file.txt
```

Internally:

```
open()
read()
write()
close()
```

But what actually happens?

### Step 1 — Program calls glibc

```
read(fd, buf, 100)
```

glibc converts it to a syscall.

---

### Step 2 — CPU instruction

On modern Linux:

```
syscall
```

Older CPUs used:

```
int 0x80
```

---

### Step 3 — CPU switches to kernel mode

The CPU:

* switches privilege level
* jumps to kernel syscall handler

---

### Step 4 — Kernel finds syscall handler

Linux maintains a **syscall table**:

```
syscall number → kernel function
```

Example:

```
0  → read
1  → write
2  → open
```

---

### Step 5 — Kernel executes syscall

For `read()`:

```
sys_read()
   |
   → vfs_read()
        |
        → filesystem driver
```

---

### Step 6 — Return result

Kernel returns:

```
bytes read OR error
```

Then CPU switches back to **user mode**.

---

# 7. Example: System Call with `strace`

You can watch system calls with:

```
strace ls
```

Example output:

```
execve("/bin/ls", ["ls"], ...)
brk(NULL)
openat(AT_FDCWD, ".", O_RDONLY)
getdents64(...)
write(1, "file.txt\n", 9)
```

This shows exactly **what the program asks the kernel to do**.

---

# 8. Example: `fork()` System Call

Example C program:

```c
#include <unistd.h>
#include <stdio.h>

int main() {
    fork();
    printf("Hello\n");
}
```

Output:

```
Hello
Hello
```

Why?

`fork()` creates **an exact copy of the process**.

Result:

```
Parent Process
     |
     | fork()
     v
+-----------+
| Parent    |
+-----------+
| Child     |
+-----------+
```

Both continue executing the same code.

---

# 9. System Call vs Library Function

Important interview question.

| Type             | Example  | Kernel involved  |
| ---------------- | -------- | ---------------- |
| System call      | read()   | YES              |
| Library function | printf() | NO (until write) |

Example:

```
printf("Hello")
```

internally becomes:

```
write(1, "Hello")
```

Only `write()` is the syscall.

---

# 10. Direct Syscall Example

You can bypass libc.

Example:

```c
#include <sys/syscall.h>
#include <unistd.h>

int main() {
    syscall(SYS_write, 1, "Hello\n", 6);
}
```

This calls the syscall **directly**.

---

# 11. Where Syscalls Are Defined in Linux

In the kernel source:

```
/usr/include/asm/unistd_64.h
```

Example:

```
#define __NR_read 0
#define __NR_write 1
#define __NR_open 2
```

And kernel implementations:

```
kernel/sys.c
fs/read_write.c
```

---

# 12. Real Example: `curl https://google.com`

You asked earlier about curl networking. System calls used:

```
socket()
connect()
send()
recv()
close()
```

Flow:

```
curl
 |
 | socket()
 v
Kernel creates TCP socket
 |
 | connect()
 v
TCP handshake
 |
 | send()
 v
HTTP request
 |
 | recv()
 v
HTTP response
```

---

# 13. How to List All Syscalls

```
ausyscall --dump
```

or

```
man syscalls
```

---

# 14. Tools to Observe System Calls

### strace

```
strace ls
```

### trace specific syscall

```
strace -e open ls
```

### follow forks

```
strace -f curl google.com
```

---

# 15. Important Interview Insight

The interviewer often wants you to connect:

```
Application
→ libc
→ system call
→ kernel
→ device driver
→ hardware
```

Example:

```
printf()
 → write()
 → sys_write()
 → VFS
 → filesystem driver
 → block driver
 → disk
```

---

# 16. One Key Thing Most People Miss

System calls are **expensive**.

Why?

Because they involve:

* privilege switch
* context change
* kernel scheduling checks

That is why high-performance software:

* batches operations
* uses async IO
* uses mmap

---

# Important Linux System Calls

| Category            | System Call            | What It Does                                                   | Example Usage                     |
| ------------------- | ---------------------- | -------------------------------------------------------------- | --------------------------------- |
| **Process Control** | `fork()`               | Creates a new child process by duplicating the calling process | Used by shells to launch programs |
|                     | `execve()`             | Replaces current process image with a new program              | Running `/bin/ls` from bash       |
|                     | `exit()`               | Terminates the calling process                                 | Program finishing execution       |
|                     | `wait()` / `waitpid()` | Waits for a child process to finish                            | Parent waits for child            |
|                     | `getpid()`             | Returns process ID                                             | Used for logging/debugging        |
|                     | `getppid()`            | Returns parent process ID                                      | Process hierarchy tracking        |
|                     | `kill()`               | Sends signal to a process                                      | `kill -9 <pid>`                   |

---

| Category            | System Call           | What It Does                             | Example Usage                |
| ------------------- | --------------------- | ---------------------------------------- | ---------------------------- |
| **File Operations** | `open()` / `openat()` | Opens a file and returns file descriptor | `open("file.txt", O_RDONLY)` |
|                     | `read()`              | Reads data from a file descriptor        | Reading file contents        |
|                     | `write()`             | Writes data to a file descriptor         | Writing logs                 |
|                     | `close()`             | Closes a file descriptor                 | Release file resources       |
|                     | `lseek()`             | Moves file offset pointer                | Random file access           |
|                     | `stat()`              | Retrieves file metadata                  | File size, permissions       |
|                     | `fstat()`             | Gets metadata for an open file           | Used with file descriptor    |

---

| Category              | System Call  | What It Does                      | Example Usage              |
| --------------------- | ------------ | --------------------------------- | -------------------------- |
| **Memory Management** | `mmap()`     | Maps files or devices into memory | Used by databases          |
|                       | `munmap()`   | Unmaps memory region              | Release mapped memory      |
|                       | `brk()`      | Changes process heap size         | Used by `malloc()`         |
|                       | `mprotect()` | Changes memory protection         | Control read/write/execute |

---

| Category                   | System Call | What It Does             | Example Usage     |
| -------------------------- | ----------- | ------------------------ | ----------------- |
| **File System Management** | `mkdir()`   | Creates directory        | `mkdir test`      |
|                            | `rmdir()`   | Removes directory        | `rmdir test`      |
|                            | `unlink()`  | Deletes file             | `rm file.txt`     |
|                            | `rename()`  | Renames file             | `mv file1 file2`  |
|                            | `chmod()`   | Changes file permissions | `chmod 755 file`  |
|                            | `chown()`   | Changes file owner       | `chown user file` |

---

| Category              | System Call | What It Does                       | Example Usage                |
| --------------------- | ----------- | ---------------------------------- | ---------------------------- |
| **Device Management** | `ioctl()`   | Device-specific control operations | Configure network interfaces |
|                       | `read()`    | Read from device                   | Reading keyboard input       |
|                       | `write()`   | Write to device                    | Writing to terminal          |

---

| Category       | System Call | What It Does                | Example Usage               |
| -------------- | ----------- | --------------------------- | --------------------------- |
| **Networking** | `socket()`  | Creates network socket      | Start TCP/UDP communication |
|                | `bind()`    | Assigns address to socket   | Bind server to port         |
|                | `listen()`  | Marks socket as passive     | Wait for connections        |
|                | `accept()`  | Accepts incoming connection | Web server connection       |
|                | `connect()` | Connects to remote socket   | Client connecting to server |
|                | `send()`    | Sends data through socket   | HTTP request                |
|                | `recv()`    | Receives data from socket   | HTTP response               |

---

| Category                              | System Call        | What It Does                                    | Example Usage     |
| ------------------------------------- | ------------------ | ----------------------------------------------- | ----------------- |
| **Inter-Process Communication (IPC)** | `pipe()`           | Creates communication channel between processes | Shell pipelines   |
|                                       | `dup()` / `dup2()` | Duplicates file descriptor                      | Redirect stdout   |
|                                       | `shmget()`         | Creates shared memory segment                   | Fast IPC          |
|                                       | `shmat()`          | Attaches shared memory                          | Map shared memory |
|                                       | `msgget()`         | Creates message queue                           | Process messaging |

---

| Category    | System Call   | What It Does             | Example Usage           |
| ----------- | ------------- | ------------------------ | ----------------------- |
| **Signals** | `signal()`    | Sets signal handler      | Handle SIGINT           |
|             | `sigaction()` | Advanced signal handling | Reliable signal control |
|             | `pause()`     | Waits for signal         | Process sleep           |
|             | `alarm()`     | Sends signal after time  | Timers                  |

---

| Category            | System Call       | What It Does             | Example Usage           |
| ------------------- | ----------------- | ------------------------ | ----------------------- |
| **Time Management** | `gettimeofday()`  | Gets current time        | Logging timestamps      |
|                     | `nanosleep()`     | Sleep for time interval  | Precise sleep           |
|                     | `clock_gettime()` | Get high resolution time | Performance measurement |

---

# The **10 Most Important Syscalls to Know (Interview Focus)**

If an interviewer asks **“Name important Linux system calls”**, these are the ones to mention first:

| System Call | Why Important                |
| ----------- | ---------------------------- |
| `fork()`    | Creates new processes        |
| `execve()`  | Executes programs            |
| `wait()`    | Process synchronization      |
| `open()`    | File access                  |
| `read()`    | File/device reading          |
| `write()`   | File/device writing          |
| `mmap()`    | Memory mapping               |
| `socket()`  | Networking                   |
| `connect()` | Network client connections   |
| `accept()`  | Server accepting connections |

---

# Example: Real System Calls Behind a Command

Running:

```bash
ls
```

Actual system calls:

```
execve("/bin/ls")
openat()
getdents64()
write()
close()
```

Observed with:

```bash
strace ls
```

---

# Quick Mental Model

You can remember syscalls in **five groups**:

```
Process → fork, exec, wait
File → open, read, write, close
Memory → mmap, brk
Network → socket, bind, connect
IPC → pipe, shared memory
```

---

Let’s walk through a **real Linux process creation flow from the terminal** and see exactly how **`fork()`**, **`exec()`**, and **PIDs** behave. 

We will follow what happens when you run:

```
ls -l
```

from the terminal.

---

# 1. Initial State: Terminal and Shell

When you open a terminal, the OS starts a **terminal emulator** (for example iTerm2, GNOME Terminal, or Terminal (macOS)).

Inside the terminal, a **shell** runs such as:

* Bash
* Zsh

Example process tree:

```
systemd (PID 1)
   │
   └── terminal (PID 2000)
           │
           └── bash (PID 2050)
```

So currently:

| Process  | PID  |
| -------- | ---- |
| terminal | 2000 |
| bash     | 2050 |

---

# 2. User Runs a Command

You type:

```
ls -l
```

Now the shell must **start a new program**.

The shell **does NOT replace itself immediately**.

Instead it performs:

```
fork()  → create child process
exec()  → child loads new program
```

---

# 3. Step 1 — `fork()` Happens

The shell calls the system call:

```
fork()
```

`fork()` **creates a new child process that is almost identical to the parent**.

Important points:

* child gets **new PID**
* parent PID stays same
* child inherits memory, file descriptors, environment

Example after fork:

```
systemd (1)
   │
terminal (2000)
   │
bash (2050)
   │
   ├── bash (2050)  ← parent
   │
   └── bash (2100)  ← child
```

Now we have:

| Process     | PID  | PPID |
| ----------- | ---- | ---- |
| bash parent | 2050 | 2000 |
| bash child  | 2100 | 2050 |

Notice:

```
child.PPID = parent.PID
```

Both processes are **still bash** at this point.

---

# 4. Return Value of `fork()`

`fork()` returns **different values** to parent and child.

| Process | fork() return value |
| ------- | ------------------- |
| Parent  | child PID           |
| Child   | 0                   |

Example:

```
fork() → returns 2100 to parent
fork() → returns 0 to child
```

This lets programs run different code paths.

Example pseudo-code:

```
pid = fork()

if pid == 0:
    # child process
else:
    # parent process
```

---

# 5. Step 2 — Child Executes `exec()`

The child process now **replaces itself with the program you asked for**.

Shell calls:

```
execve("/bin/ls", ["ls","-l"], env)
```

Before exec:

```
PID 2100 = bash
```

After exec:

```
PID 2100 = ls
```

Important:

**PID does NOT change.**

Only the **program image** changes.

Process tree now:

```
systemd (1)
   │
terminal (2000)
   │
bash (2050)
   │
   └── ls (2100)
```

So:

| Process | PID  | Program |
| ------- | ---- | ------- |
| shell   | 2050 | bash    |
| child   | 2100 | ls      |

---

# 6. Step 3 — `ls` Runs

Now the `ls` program runs and performs system calls:

```
openat()
getdents64()
write()
close()
```

Example:

```
open directory
read entries
print output
```

---

# 7. Step 4 — Process Exit

After printing output:

```
exit()
```

Process `2100` terminates.

Now the kernel sends a signal:

```
SIGCHLD
```

to the parent shell.

---

# 8. Step 5 — Parent Calls `wait()`

The shell calls:

```
waitpid(2100)
```

This allows the shell to:

* collect exit status
* remove zombie process

Now process tree returns to:

```
systemd
   │
terminal
   │
bash
```

---

# 9. Real Example Using Commands

Run:

```
ps -f
```

Example:

```
UID   PID  PPID  CMD
me   2050  2000  bash
```

Now run:

```
sleep 30
```

During execution:

```
ps -f
```

Output:

```
UID   PID  PPID  CMD
me   2050  2000  bash
me   2100  2050  sleep 30
```

Observe:

| Field | Meaning    |
| ----- | ---------- |
| PID   | process ID |
| PPID  | parent PID |

---

# 10. Visual Timeline

Full timeline of `ls` execution:

```
User types: ls
      │
      ▼
bash (PID 2050)
      │
      │ fork()
      ▼
child bash (PID 2100)
      │
      │ execve("/bin/ls")
      ▼
ls (PID 2100)
      │
      │ run program
      ▼
exit()
      │
      │
bash waitpid()
```

---

# 11. Important Insight: Copy-On-Write

When `fork()` happens, Linux **does NOT copy all memory immediately**.

Instead it uses:

```
Copy-On-Write (COW)
```

Meaning:

* parent and child share memory pages
* copy only happens if modified

This makes `fork()` very fast.

---

# 12. Special Case: Shell Pipelines

Example:

```
ls | grep txt
```

Shell does:

```
fork() → ls
fork() → grep
pipe()
dup2()
exec()
```

Process tree:

```
bash
 ├── ls
 └── grep
```

Both processes run simultaneously.

---

# 13. Real Syscall Trace

Run:

```
strace ls
```

You’ll see:

```
execve("/bin/ls")
brk()
openat()
getdents64()
write()
exit_group()
```

---

# 14. The Key Interview Summary

When running a command from a shell:

```
1. shell calls fork()
2. child process created
3. child calls execve()
4. program runs
5. program exits
6. parent calls wait()
```

---



