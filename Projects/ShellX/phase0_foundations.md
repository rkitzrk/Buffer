# Phase 0 — Foundations Before Reading ShellX

> You're a Codeforces Specialist — you understand recursion, graphs, DP, and complexity deeply.
> This guide bridges the gap from **algorithmic thinking → systems programming thinking**.
> The mental model shift is the hardest part. The code is actually simpler than DP.

---

## 0.1 — The Process Mental Model

### What is a Process?

A **process** is a running instance of a program. The OS gives each process:
- Its own **virtual address space** (stack, heap, code, data segments)
- A set of **file descriptors** (open files, pipes, sockets)
- A **PID** (process ID)
- Signal handlers, working directory, environment variables

```
Process Address Space (Virtual Memory):
┌──────────────────┐  High address
│      Stack       │  ← grows downward (local vars, return addrs)
│        ↓        │
│                  │
│        ↑        │
│      Heap        │  ← grows upward (malloc/new)
│──────────────────│
│   BSS Segment    │  ← uninitialized globals
│   Data Segment   │  ← initialized globals
│   Text Segment   │  ← executable code (read-only)
└──────────────────┘  Low address (0x0)
```

**Key insight for competitive programmers:** Think of a process like a node in a tree. `fork()` creates a child node. The parent and child are separate — changes in one don't affect the other (except via explicit IPC).

---

## 0.2 — File Descriptors (FDs) — The Most Critical Concept

### What is a File Descriptor?

An FD is just a small non-negative integer (0, 1, 2, 3, ...) that is an index into a per-process **file descriptor table**. Each entry points to a kernel **open file description** which tracks the actual file/pipe/socket.

```
Process FD Table            Kernel Open File Table       Inode Table
┌────────────────┐          ┌─────────────────────┐     ┌──────────┐
│ fd=0 (stdin)  │────────→ │ offset, flags, mode  │───→ │  file A  │
│ fd=1 (stdout) │────────→ │ offset, flags, mode  │───→ │  file B  │
│ fd=2 (stderr) │────────→ │ offset, flags, mode  │───→ │  file C  │
│ fd=3          │────────→ │ ...                  │
└────────────────┘
```

### The Three Standard FDs (always open at program start)

| FD | Name   | Default Target |
|----|--------|---------------|
| 0  | stdin  | keyboard      |
| 1  | stdout | terminal      |
| 2  | stderr | terminal      |

### Critical Operations

```c
// Open a file → get a new fd
int fd = open("file.txt", O_RDONLY);     // returns 3 (first free slot)

// Duplicate: make fd2 point to same open file description as fd1
dup2(fd, STDOUT_FILENO);  // fd 1 now points to file.txt
                           // Now printf() writes to file.txt!

// Always close what you don't need
close(fd);   // after dup2, the original fd is redundant
```

**The Golden Rule (you'll see this everywhere in ShellX):**
> **Every process must close every file descriptor it doesn't use.**
> If you leave a pipe's write-end open in the reader process, the reader will *never* see EOF. It'll hang forever.

### fork() and FDs

When you `fork()`, the child **inherits a copy of the parent's FD table**. Both parent and child have their own copy of fd=3 pointing to the same open file. Both must close it when done.

---

## 0.3 — `fork()`, `exec()`, `waitpid()` — The Core Triad

### fork()

```c
pid_t pid = fork();
```

- Returns **twice**: once in parent (with child's PID), once in child (with 0)
- Think of it as: the process is "cloned" — everything is duplicated
- After fork: parent and child run independently (different stacks, different PIDs)

```
Before fork():     After fork():
[Parent]           [Parent]    [Child]
   |                   |           |
   fork() ─────────────┤           │
                        └─── 0 ←──┘
                        pid > 0    pid == 0
```

### exec()

```c
execvp("ls", argv);   // replace THIS process's image with "ls"
```

- Does NOT create a new process — it **replaces** the current process's code, stack, heap with a new program
- After exec: the current code is gone; the new program runs
- FDs survive exec (unless `O_CLOEXEC` is set)
- Signal handlers are reset to default

**The fork+exec pattern** (how every shell runs commands):
```
parent: fork() → child: exec("ls") → child is now "ls"
parent: waitpid(child_pid) → blocks until "ls" finishes
```

### waitpid()

```c
pid_t result = waitpid(pid, &status, 0);
```

- Parent calls this to **reap** a finished child
- Without this, finished children become **zombies** (the process is dead but its entry stays in the process table until parent reaps it)
- `WNOHANG` flag: non-blocking — return immediately if no child has finished

---

## 0.4 — Pipes

A pipe is a kernel buffer with two ends: a **read end** (fd[0]) and a **write end** (fd[1]).

```c
int fds[2];
pipe(fds);   // fds[0] = read end, fds[1] = write end
```

Data written to `fds[1]` can be read from `fds[0]`. The pipe is in-kernel; no file on disk.

### How `ls | grep cpp` works

```
Step 1: Create pipe
  fds[0] ←────────────── fds[1]

Step 2: Fork child 1 (ls)
  Child 1: dup2(fds[1], STDOUT_FILENO)  → ls writes to pipe
           close all pipe fds
           exec("ls")

Step 3: Fork child 2 (grep)
  Child 2: dup2(fds[0], STDIN_FILENO)   → grep reads from pipe
           close all pipe fds
           exec("grep", "cpp")

Step 4: Parent closes all pipe fds, waits for both children
```

**Why parent must close pipe fds:** If parent keeps `fds[1]` open, `grep` will never see EOF on its stdin — the write-end is still open (parent has it). `grep` blocks forever.

---

## 0.5 — Signals

Signals are asynchronous notifications sent to a process by the kernel or another process.

| Signal  | Default Action | Typical Cause |
|---------|---------------|---------------|
| SIGINT  | Terminate     | Ctrl+C        |
| SIGCHLD | Ignore        | Child exited  |
| SIGTERM | Terminate     | `kill pid`    |
| SIGKILL | Terminate (uncatchable) | `kill -9` |
| SIGSEGV | Core dump     | Invalid memory access |

### Signal Handlers

```c
struct sigaction sa;
sa.sa_handler = my_handler;    // function to call
sigemptyset(&sa.sa_mask);      // don't block other signals while handling
sa.sa_flags = SA_RESTART;      // auto-restart interrupted syscalls
sigaction(SIGINT, &sa, NULL);  // install handler
```

### Async-Signal Safety (Critical!)

Signal handlers can interrupt *any* point in your code — even in the middle of `malloc()`.
If your handler calls `malloc()` while the main code was also in the middle of `malloc()`, you get **undefined behavior** (double-locking, heap corruption).

**Rule:** Signal handlers must only call **async-signal-safe** functions. The safe list includes:
- `write()` (not `printf()` — printf uses buffered I/O with locks)
- Setting `volatile sig_atomic_t` variables
- `_exit()` (not `exit()`)

**The correct pattern (used in ShellX):**
```c
// Handler: ONLY set a flag
volatile sig_atomic_t g_sigchld_pending = 0;
void sigchld_handler(int sig) {
    g_sigchld_pending = 1;  // that's it — nothing else
}

// Main loop: do the real work after returning from handler
if (g_sigchld_pending) {
    g_sigchld_pending = 0;
    reapFinishedJobs();  // safe to call waitpid, printf here
}
```

---

## 0.6 — I/O Redirection Mechanics

`echo hello > out.txt` works like this:

```c
// In the child process (after fork, before exec):
int fd = open("out.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
dup2(fd, STDOUT_FILENO);   // fd 1 now points to out.txt
close(fd);                  // close original fd (dup2 made a copy)
execvp("echo", argv);       // echo writes to fd 1 → goes to out.txt
```

The `echo` program doesn't know or care about redirection — it always writes to `stdout` (fd 1). The shell secretly rewired fd 1 before exec.

---

## 0.7 — Key Syscall Reference Card

| Syscall | Signature | What it does |
|---------|-----------|-------------|
| `fork()` | `pid_t fork()` | Clone process; returns twice |
| `execvp()` | `int execvp(const char* file, char* const argv[])` | Replace process image |
| `waitpid()` | `pid_t waitpid(pid_t pid, int* status, int options)` | Wait/reap child |
| `pipe()` | `int pipe(int fds[2])` | Create kernel pipe |
| `dup2()` | `int dup2(int oldfd, int newfd)` | Clone fd; newfd now points to same file |
| `open()` | `int open(const char* path, int flags, mode_t mode)` | Open file, get fd |
| `close()` | `int close(int fd)` | Release fd |
| `chdir()` | `int chdir(const char* path)` | Change working directory |
| `getcwd()` | `char* getcwd(char* buf, size_t size)` | Get working directory |
| `kill()` | `int kill(pid_t pid, int sig)` | Send signal to process |
| `sigaction()` | `int sigaction(int sig, struct sigaction* act, ...)` | Install signal handler |
| `_exit()` | `void _exit(int status)` | Immediate exit (no cleanup) |

---

## 0.8 — Mental Model Differences: Competitive vs Systems

| Competitive Programming | Systems Programming |
|------------------------|---------------------|
| Single process | Multiple processes communicating |
| No side effects (pure functions) | Everything has side effects (FDs, memory, signals) |
| Time/space complexity | Correctness under concurrency & signal delivery |
| Clean inputs | Syscalls can fail at any time; check every return value |
| One execution path | fork() creates two simultaneous execution paths |
| Crash = WA | Crash = zombie, fd leak, hang |

**The hardest adjustment:** After `fork()`, you have **two execution contexts** running the code simultaneously. The `pid == 0` branch is the child; `pid > 0` is the parent. Many bugs come from forgetting which context you're in.

---

## 0.9 — Pre-Reading Checklist

Before moving to Phase 1, make sure you can answer these mentally:

- [ ] What is a file descriptor? What's fd 0, 1, 2?
- [ ] What does `fork()` return in the parent vs. child?
- [ ] What happens to FDs after `fork()`?
- [ ] Why do we use `_exit()` instead of `exit()` in a child after failed `exec()`?
- [ ] What is a zombie process?
- [ ] Why can't signal handlers call `printf()`?
- [ ] What does `dup2(fd, STDOUT_FILENO)` do?
- [ ] Why must the parent close pipe fds after forking?

---

## Resources to Skim (30–60 min each)

| Resource | What to read |
|----------|-------------|
| `man 2 fork` | Return values, FD inheritance |
| `man 2 execvp` | How path search works, what survives exec |
| `man 2 waitpid` | `WIFEXITED`, `WEXITSTATUS`, `WNOHANG`, zombie explanation |
| `man 2 pipe` | Pipe semantics, EOF behavior |
| `man 2 dup2` | What happens if newfd is already open |
| `man 7 signal` | Signal safety, async-signal-safe function list |
| APUE Chapter 8 | "Advanced Programming in the Unix Environment" — process model (gold standard) |
| CS:APP Chapter 8 | "Computer Systems: A Programmer's Perspective" — exceptional control flow |

---

*Confirm when ready → I'll send Phase 1 (most critical files in the ShellX codebase).*
