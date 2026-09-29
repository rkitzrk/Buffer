# Phase 0 — Complete SDE Interview Q&A

> Every question a recruiter or interviewer can ask about the foundations of this project.
> Organized by topic. Study answers, then try to explain them without looking.

---

## Section 1: Processes & Memory

**Q1. What is a process? How is it different from a program?**

A **program** is a static file on disk (ELF binary). A **process** is a running instance of that program — it has:
- A unique PID
- Its own virtual address space (stack, heap, code, data)
- Open file descriptors
- A current working directory
- Signal handlers
- CPU registers (program counter, stack pointer)

Multiple processes can run the same program simultaneously (e.g., two terminals both running `ls`) — they share the *code* in memory (read-only) but have separate stacks and heaps.

---

**Q2. Describe the memory layout of a process.**

```
High address
┌──────────────────┐
│  Stack           │ ← function call frames, local variables (grows ↓)
│       ↓         │
│   [gap]          │
│       ↑         │
│  Heap            │ ← malloc/new (grows ↑)
│  BSS             │ ← uninitialized global/static variables (zero-filled)
│  Data            │ ← initialized global/static variables
│  Text            │ ← machine code (read-only, shared between processes)
└──────────────────┘
Low address (0x0)
```

Stack overflows (stack pointer crosses into heap) cause `SIGSEGV`.

---

**Q3. What is virtual memory? Why does each process have its own address space?**

**Virtual memory** is an abstraction: the OS + MMU (Memory Management Unit) gives each process the illusion that it has the entire address space to itself. The OS maps virtual pages to physical pages.

Benefits:
- **Isolation**: process A can't read process B's memory (protection)
- **Simplicity**: every process starts at the same virtual address; no relocation needed
- **Overcommit**: processes can claim more memory than physically available (lazy allocation)

---

**Q4. What happens at the OS level when you run `./shellx`?**

1. Shell calls `fork()` → creates a child process
2. Child calls `execvp("./shellx", argv)` → kernel loads the ELF binary, sets up stack/heap/text segments, jumps to `main()`
3. Parent calls `waitpid()` → blocks until shellx exits
4. Child (shellx) runs; when it exits, parent is notified

---

**Q5. What is a zombie process? How is it created? How is it cleaned up?**

A **zombie** is a process that has finished executing but whose entry still exists in the process table because its parent hasn't called `waitpid()` yet.

Creation:
```
Child exits → becomes zombie (status stored in kernel)
              ↑
Parent hasn't called waitpid() yet
```

Cleanup:
- Parent calls `waitpid(child_pid, &status, 0)` → kernel removes the zombie entry
- If parent exits, zombie is **reparented to init (PID 1)**, which calls `waitpid()` automatically

**Why zombies are a problem:** Each zombie consumes a PID slot. If a long-running server creates thousands of children without reaping, it exhausts the PID table and can't create new processes.

---

**Q6. What is an orphan process?**

An **orphan** is a process whose parent has died before the child. The kernel automatically reparents orphans to init (PID 1), which periodically calls `wait()` to reap them.

In ShellX: on `exit`, we send `SIGTERM` to background jobs and immediately exit — those children become orphans, reparented to init. This is intentional and documented.

---

**Q7. What is context switching?**

The OS scheduler pauses one process (saves its CPU registers, stack pointer, program counter to the kernel's PCB — Process Control Block) and resumes another (restores its saved state). This is **context switching**.

Cost: ~1–10 microseconds. Syscalls cause a context switch from user-mode to kernel-mode (and back).

---

**Q8. What is the difference between kernel mode and user mode?**

| | User Mode | Kernel Mode |
|--|-----------|-------------|
| Access | Restricted — can't access hardware directly | Full access to hardware, memory |
| Entry | Via syscall instruction | Normal kernel execution |
| Crash impact | Segfault kills only your process | Kernel panic kills the system |

Every `fork()`, `open()`, `write()` etc. is a **syscall** — a controlled jump into kernel mode, execution of kernel code, then return to user mode.

---

## Section 2: File Descriptors

**Q9. What is a file descriptor? What is it an index into?**

An FD is a small non-negative integer that is an index into the process's **file descriptor table** (a per-process array in the kernel). Each entry points to a **kernel open file description** (shared structure) which tracks:
- File offset (position)
- Open flags (`O_RDONLY`, `O_WRONLY`, etc.)
- Reference count
- Pointer to the inode

```
Process FD Table     Open File Table (kernel)    Inode
fd=0 ──────────────→ { offset=0, flags, ... } ──→ /dev/tty
fd=1 ──────────────→ { offset=0, flags, ... } ──→ /dev/tty
fd=3 ──────────────→ { offset=42, flags, .. } ──→ /home/out.txt
```

---

**Q10. What are the three standard file descriptors? What happens if you close fd 0?**

| FD | Macro | Default |
|----|-------|---------|
| 0 | `STDIN_FILENO` | keyboard (terminal) |
| 1 | `STDOUT_FILENO` | terminal |
| 2 | `STDERR_FILENO` | terminal |

If you `close(0)` and then call `open("file.txt")`, the OS assigns the lowest available FD — which is now 0. So `scanf()` would read from `file.txt`. This is the mechanic behind I/O redirection.

---

**Q11. What does `dup2(oldfd, newfd)` do exactly? What if newfd is already open?**

`dup2(oldfd, newfd)`:
1. If `newfd` is already open, **atomically closes it first**
2. Makes `newfd` point to the same open file description as `oldfd`
3. Both `oldfd` and `newfd` now refer to the same file

```c
int fd = open("out.txt", O_WRONLY|O_CREAT|O_TRUNC, 0644);
dup2(fd, STDOUT_FILENO);  // fd 1 now → out.txt
close(fd);                // close original; fd 1 still open
// Now printf() → out.txt
```

**Atomicity matters:** `close(newfd); dup(oldfd)` is NOT equivalent — there's a race window where another thread could grab `newfd` between the close and dup.

---

**Q12. What is the difference between `dup()` and `dup2()`?**

- `dup(fd)`: duplicates `fd` to the lowest available fd number; you don't control which fd you get
- `dup2(oldfd, newfd)`: duplicates to a *specific* fd number (`newfd`); closes it first if open

For shell redirection, you always use `dup2()` because you need to target exactly fd 0, 1, or 2.

---

**Q13. What happens to file descriptors across `fork()`?**

The child inherits a **copy** of the parent's FD table. Both parent and child have their own fd entries pointing to the **same** underlying open file descriptions. This means:
- The file offset is shared (seek in one, affects the other)
- The reference count on the open file description increases

This is why after `fork()`, both parent and child must close pipe fds they don't need — otherwise the write-end of a pipe stays open in the reader's process and EOF is never delivered.

---

**Q14. What happens to file descriptors across `exec()`?**

By default, **all FDs survive exec** (they remain open in the new program). This is exactly how redirection works: the shell sets up `dup2()` calls in the child, then exec-s the command. The new program finds fd 1 pointing to a file, not the terminal.

Exception: FDs opened with `O_CLOEXEC` flag are automatically closed on `exec()`. This is the safe default for library code that opens FDs internally.

---

**Q15. What is the maximum number of file descriptors a process can have?**

Controlled by two limits:
- **Soft limit**: `ulimit -n` (commonly 1024 or 65536)
- **Hard limit**: maximum the soft limit can be raised to

Check with:
```bash
ulimit -n       # soft limit
ulimit -Hn      # hard limit
cat /proc/sys/fs/file-max   # system-wide limit
```

FD exhaustion causes `open()` to return -1 with `errno == EMFILE`.

---

## Section 3: fork() / exec() / waitpid()

**Q16. What exactly does `fork()` return, and why does it return twice?**

`fork()` creates a new process by duplicating the calling process. It returns:
- In the **parent**: the PID of the new child (> 0)
- In the **child**: 0
- On error: -1 (child not created)

It "returns twice" because after `fork()`, there are two processes — each with their own copy of the stack. Each resumes execution right after the `fork()` call and gets its own return value.

```c
pid_t pid = fork();
if (pid == -1) { /* error */ }
else if (pid == 0) {
    // I am the child
} else {
    // I am the parent; pid = child's PID
}
```

---

**Q17. What is copied during `fork()`? What is shared?**

**Copied (COW):**
- Stack, heap, BSS, data segments (via Copy-on-Write)
- File descriptor table (entries are copied, but underlying open file descriptions are shared)
- Signal handlers, signal mask
- Environment variables, working directory

**Shared (not copied):**
- Underlying open file descriptions (file offset is shared)
- Memory-mapped files with `MAP_SHARED`
- Physical memory pages (until one process writes — Copy-on-Write)

**Copy-on-Write:** After `fork()`, the OS doesn't actually copy physical memory. Pages are marked read-only and shared. When either process writes to a page, a fault occurs and the kernel makes a private copy for that process. This makes `fork()` cheap.

---

**Q18. What is `exec()`? What happens to the process after exec?**

`exec()` replaces the current process's image with a new program. After `exec()`:
- Code, stack, heap, BSS, data are replaced with the new program's
- **FDs survive** (unless `O_CLOEXEC`)
- PID stays the same
- Signal handlers are reset to `SIG_DFL`
- `exec()` **never returns** on success — the old code is gone

The `execvp` variant:
- `v` = arguments as an array (vector)
- `p` = searches `PATH` automatically (you don't need the full path)

```c
char* argv[] = {"ls", "-la", NULL};  // must be NULL-terminated
execvp("ls", argv);
// If we reach here, exec failed
perror("ls");
_exit(127);
```

---

**Q19. Why do we use `_exit()` instead of `exit()` after a failed `exec()` in the child?**

`exit()` does:
1. Calls functions registered with `atexit()`
2. **Flushes and closes all stdio buffers** (stdout, stderr)
3. Then calls `_exit()`

The problem: after `fork()`, the child inherits the parent's buffered stdio state. If the parent had buffered output that wasn't flushed yet, `exit()` in the child would flush it — causing that output to appear **twice** (once when the child exits, once when the parent eventually flushes).

`_exit()` skips all cleanup and terminates immediately. It's the correct choice in the child after `fork()` when exec fails.

---

**Q20. What is `waitpid()`? What are the status macros used to inspect the exit status?**

```c
pid_t result = waitpid(pid, &status, options);
```

- Suspends the calling process until the specified child changes state
- Returns the child's PID on success, -1 on error, 0 if `WNOHANG` and no child ready

**Status inspection macros:**

| Macro | Meaning |
|-------|---------|
| `WIFEXITED(status)` | True if child exited normally (via `exit()` or `return`) |
| `WEXITSTATUS(status)` | The actual exit code (only valid if `WIFEXITED`) |
| `WIFSIGNALED(status)` | True if child was killed by a signal |
| `WTERMSIG(status)` | The signal number that killed it (only if `WIFSIGNALED`) |
| `WIFSTOPPED(status)` | True if child was stopped (not terminated) |

---

**Q21. What is `WNOHANG`? When do you use it?**

`WNOHANG` makes `waitpid()` **non-blocking**: if no child has finished, it returns immediately with 0 instead of blocking.

Used in ShellX's `reapFinishedJobs()`:
```c
pid_t result = waitpid(job.pid, &status, WNOHANG);
// result == 0 → still running
// result > 0 → exited, we can reap
// result == -1 → error
```

Without `WNOHANG`, calling `waitpid()` for a background job in the REPL would block the shell until that job finishes — defeating the purpose of background execution.

---

**Q22. What is `EINTR`? Why do we retry `waitpid()` on `EINTR`?**

`EINTR` (Error: INTerrupted) is returned by blocking syscalls when a signal is delivered while they're waiting. E.g., if `waitpid()` is blocking and a `SIGCHLD` arrives for a *different* child, the `waitpid()` may return -1 with `errno == EINTR`.

**The retry pattern:**
```c
pid_t result;
do {
    result = waitpid(pid, &status, 0);
} while (result == -1 && errno == EINTR);
```

In ShellX, while waiting for a foreground child, a background child might finish and deliver `SIGCHLD`. Without the retry, the foreground `waitpid()` would spuriously fail.

Note: `SA_RESTART` in `sigaction()` flags auto-restarts many syscalls, but making the retry explicit in code documents intent clearly.

---

**Q23. What is `waitpid(-1, ...)` vs `waitpid(pid, ...)`?**

- `waitpid(pid, ...)` — waits for a **specific** child
- `waitpid(-1, ...)` — waits for **any** child (equivalent to `wait()`)
- `waitpid(-pgid, ...)` — waits for any child in process group `pgid`

ShellX avoids `waitpid(-1, ...)` because reaping "any" child in a signal handler could accidentally reap the foreground child, causing the foreground `waitpid()` to fail with `ECHILD`.

---

## Section 4: Pipes

**Q24. What is a pipe? How does it work at the kernel level?**

A pipe is a **kernel-managed, in-memory, unidirectional byte stream** with two ends:
- `fds[0]` — read end
- `fds[1]` — write end

The kernel maintains a fixed-size buffer (typically 64KB on Linux). Write to `fds[1]`, read from `fds[0]`. The data is FIFO.

Blocking behavior:
- Read on empty pipe: blocks until data is available (or write end is closed → returns 0/EOF)
- Write on full pipe: blocks until space is available (or read end is closed → `SIGPIPE`)

---

**Q25. What is the critical fd discipline rule for pipelines? Why?**

**Rule: Every process must close every pipe fd it doesn't own.**

Why:
- A pipe's read end sees **EOF only when ALL write-end fds are closed**
- If the parent (or any other process) holds the write-end open, the reader blocks forever — even if the actual writing process has exited

Example: In `ls | grep cpp`:
```
After fork of ls:   parent has fds[0] and fds[1]; ls-child has fds[0] and fds[1]
After fork of grep: parent has fds[0] and fds[1]; grep-child has fds[0] and fds[1]

ls-child:   dup2(fds[1], stdout); close(fds[0]); close(fds[1]); exec ls
grep-child: dup2(fds[0], stdin);  close(fds[0]); close(fds[1]); exec grep
parent:     close(fds[0]); close(fds[1]);  waitpid for both
```

If parent doesn't close `fds[1]`, grep reads forever because the write-end is still open.

---

**Q26. Implement a 2-command pipeline: `ls | wc -l` using raw syscalls.**

```c
int fds[2];
pipe(fds);

pid_t pid1 = fork();
if (pid1 == 0) {                    // child 1: ls
    dup2(fds[1], STDOUT_FILENO);    // stdout → write end
    close(fds[0]);                  // close read end (not needed)
    close(fds[1]);                  // close write end original
    execlp("ls", "ls", NULL);
    _exit(127);
}

pid_t pid2 = fork();
if (pid2 == 0) {                    // child 2: wc
    dup2(fds[0], STDIN_FILENO);     // stdin ← read end
    close(fds[0]);
    close(fds[1]);                  // critical: close write end in reader
    execlp("wc", "wc", "-l", NULL);
    _exit(127);
}

close(fds[0]);   // parent closes both
close(fds[1]);
waitpid(pid1, NULL, 0);
waitpid(pid2, NULL, 0);
```

---

**Q27. What is `SIGPIPE`? When is it generated?**

`SIGPIPE` is sent to a process when it **writes to a pipe whose read end is closed** (no reader).

Default action: terminate the process.

Common scenario: `cat /dev/urandom | head -5` — after `head` reads 5 lines and exits, `cat` gets `SIGPIPE` on its next write. Shells typically ignore `SIGPIPE` or handle it gracefully.

---

**Q28. What is the difference between a pipe and a named pipe (FIFO)?**

| | Pipe | Named Pipe (FIFO) |
|--|------|--------------------|
| Lifetime | Exists only while process holds fd | Exists as a filesystem entry |
| Access | Only related processes (parent/child) | Any process that knows the path |
| Creation | `pipe(fds)` | `mkfifo("path", mode)` |
| Usage | `ls \| grep` | Inter-process communication between unrelated processes |

---

## Section 5: Signals

**Q29. What is a signal? How is it delivered?**

A signal is an asynchronous notification sent to a process. Delivery path:
1. Something triggers the signal (Ctrl+C, another process calling `kill()`, hardware fault)
2. Kernel sets a pending bit in the process's signal mask
3. Before the process returns to user mode (e.g., after a syscall or context switch), kernel checks pending signals
4. If unblocked, kernel redirects the process to the signal handler (or takes default action)

Signals are **not queued** (except real-time signals) — if two `SIGCHLD` arrive before the handler runs, only one invocation of the handler is guaranteed.

---

**Q30. What is `sigaction()` vs `signal()`? Why prefer `sigaction()`?**

`signal()` is the old API. Behavior varies across Unix systems:
- On some: handler is reset to `SIG_DFL` after one invocation (must reinstall)
- `SA_RESTART` behavior is undefined

`sigaction()` provides:
- **Reliable semantics**: handler stays installed
- **`SA_RESTART`**: auto-restart interrupted syscalls
- **`SA_NOCLDSTOP`**: don't signal when child is stopped (only on exit)
- **Signal mask control**: block other signals during handler execution

Always use `sigaction()` in production code.

---

**Q31. What is async-signal safety? Why can't signal handlers call `printf()`?**

A signal handler can interrupt **any** point in the main program — including inside `malloc()` or `printf()`, which use internal locks (mutexes) on their data structures.

If the handler also calls `malloc()`:
- Main code is inside malloc, holding the malloc lock
- Handler is called, also tries to lock → **deadlock**

Or if main is in the middle of updating `malloc()`'s heap structures when handler runs → **heap corruption**.

`printf()` is not async-signal-safe because it:
- Uses buffered I/O (locks, internal state)
- May call `malloc()` internally

**Async-signal-safe functions** (safe to call from handlers): `write()`, `_exit()`, `kill()`, `signal()`, setting `sig_atomic_t` variables.

**The pattern used in ShellX:**
```c
volatile sig_atomic_t g_sigchld_pending = 0;

void sigchld_handler(int sig) {
    g_sigchld_pending = 1;  // only safe operation
}
```

---

**Q32. What is `volatile sig_atomic_t`? Why both keywords?**

```c
volatile sig_atomic_t g_sigchld_pending;
```

- **`sig_atomic_t`**: an integer type that is guaranteed to be read/written **atomically** on this platform (no partial reads). Required because the signal handler and main loop both access this variable.
- **`volatile`**: tells the compiler **not to cache** this variable in a register. Without `volatile`, the compiler might optimize the main loop to read it once and assume it doesn't change, missing updates from the signal handler.

Both are required together for correct signal flag variables.

---

**Q33. What is `SA_RESTART`? What syscalls does it NOT restart?**

`SA_RESTART` tells the kernel to automatically restart certain interrupted syscalls when a signal handler returns, instead of returning -1 with `errno == EINTR`.

**Not restarted by `SA_RESTART`:**
- `select()`, `pselect()`, `poll()`, `epoll_wait()` — return early with whatever is ready
- `nanosleep()`, `usleep()` — restart but with remaining time
- `wait()`, `waitpid()` — implementation-dependent

This is why ShellX still has the explicit EINTR retry loop despite using `SA_RESTART` — for documentation clarity and because `waitpid` restart semantics vary.

---

**Q34. What is `SA_NOCLDSTOP`? Why does ShellX use it?**

`SA_NOCLDSTOP` tells the kernel: **only send SIGCHLD when a child terminates, not when it is stopped** (e.g., by `SIGSTOP` or `SIGTSTP`).

Without it, pressing Ctrl+Z (which sends SIGTSTP) would trigger the shell's SIGCHLD handler even though no job actually finished — causing spurious calls to `reapFinishedJobs()` and a confusing `waitpid()` on a still-running process.

---

**Q35. What happens to signal handlers after `fork()` and after `exec()`?**

- After `fork()`: child **inherits** all of the parent's signal handlers (same function pointers)
- After `exec()`: all signal handlers are **reset to `SIG_DFL`** (because the handler function no longer exists in the new program's address space). Signals that were set to `SIG_IGN` remain `SIG_IGN`.

In ShellX: after `fork()`, before `exec()`, we restore `SIGINT` to `SIG_DFL` in the child — so the child responds normally to Ctrl+C:
```c
struct sigaction sa;
sa.sa_handler = SIG_DFL;
sigaction(SIGINT, &sa, nullptr);
execvp(...);
```

---

**Q36. What is a signal mask? What is `sigemptyset()` and `sigaddset()`?**

The **signal mask** is a per-process set of signals that are currently **blocked** (pending but not delivered). Blocked signals are held by the kernel until unblocked.

```c
sigset_t mask;
sigemptyset(&mask);          // empty set (no signals blocked)
sigaddset(&mask, SIGCHLD);   // add SIGCHLD to the set
sigprocmask(SIG_BLOCK, &mask, NULL);  // block SIGCHLD
// ... critical section ...
sigprocmask(SIG_UNBLOCK, &mask, NULL); // unblock
```

In `sigaction()`, `sa.sa_mask` specifies additional signals to block **during handler execution** (the signal being handled is always blocked during its handler).

---

## Section 6: I/O Redirection

**Q37. Trace exactly what happens for `echo hello > out.txt` at the syscall level.**

1. Shell parses: command=`echo`, args=`["echo", "hello"]`, output_file=`out.txt`
2. Shell calls `fork()` → creates child
3. **In child:**
   ```c
   int fd = open("out.txt", O_WRONLY|O_CREAT|O_TRUNC, 0644);
   // fd = 3 (first free slot)
   dup2(3, 1);    // fd 1 (stdout) now points to out.txt
   close(3);      // close original; fd 1 still open → out.txt
   execvp("echo", ["echo", "hello", NULL]);
   // echo writes "hello\n" to fd 1 → goes to out.txt
   ```
4. **In parent:** `waitpid(child_pid, &status, 0)`

---

**Q38. What are the `open()` flags used for `>` vs `>>` vs `<`?**

| Operator | Flags |
|----------|-------|
| `>` (output truncate) | `O_WRONLY \| O_CREAT \| O_TRUNC` |
| `>>` (output append) | `O_WRONLY \| O_CREAT \| O_APPEND` |
| `<` (input) | `O_RDONLY` |

File mode `0644` = owner read+write, group read, others read.

---

**Q39. What does `O_TRUNC` do? What if the file doesn't exist?**

`O_TRUNC`: if the file already exists and is successfully opened, truncate it to zero length.

Combined with `O_CREAT`: if file doesn't exist, create it; if it does exist, truncate it.

---

**Q40. How does input redirection `< file` work? Where is it applied?**

Always **in the child, between `fork()` and `exec()`**:
```c
int fd = open("file.txt", O_RDONLY);
dup2(fd, STDIN_FILENO);   // fd 0 (stdin) now reads from file.txt
close(fd);
execvp("cat", argv);       // cat reads from fd 0 → from file.txt
```

The command doesn't know its stdin was redirected — it just reads from fd 0.

---

## Section 7: Built-ins and Shell Mechanics

**Q41. Why must built-in commands like `cd` run in the parent process (no fork)?**

The working directory is a **per-process** attribute. If `cd /tmp` ran in a child:
```
Parent (shell):   cwd = /home/user
  fork() → child: cwd = /home/user (inherited copy)
  child: chdir("/tmp") → child's cwd = /tmp
  child: exits
Parent (shell):   cwd = /home/user ← UNCHANGED
```

The parent's working directory would never change. So `cd` must call `chdir()` directly in the parent process — no fork.

Same reasoning applies to `exit` (must actually exit the shell process) and variable assignment.

---

**Q42. What is the difference between `exit(127)` and `_exit(127)` and `return 127` from main?**

| | Effect |
|--|--------|
| `return 127` from main | Calls `exit(127)` implicitly |
| `exit(127)` | Runs `atexit` handlers, flushes stdio, calls `_exit(127)` |
| `_exit(127)` | Immediately terminates; no cleanup; safe after `fork()` |

In a child process after failed `exec()`, use `_exit()` to avoid:
- Running parent's `atexit` handlers in the child
- Double-flushing parent's buffered stdio output

---

**Q43. Why does exit code 127 mean "command not found" and 126 mean "not executable"?**

These are **POSIX conventions** adopted by shells:
- **127**: command not found (`errno == ENOENT` after `execvp`)
- **126**: command found but not executable (permission denied, `errno == EACCES`; or file exists but is a directory)
- **128+N**: process killed by signal N (e.g., SIGINT=2 → exit code 130)

These are conventions, not kernel-enforced rules. Programs can return any value 0-255.

---

**Q44. What exit code does a shell report for a pipeline `cmd1 | cmd2 | cmd3`?**

By POSIX default (and bash's default non-`pipefail` behavior): the exit status of the **last command** in the pipeline.

ShellX implements this. `false | true` → reports 0 (success).

With bash's `set -o pipefail`: reports the exit status of the **rightmost command that failed**. Not implemented in ShellX (documented in known limitations).

---

## Section 8: System-Level Concepts

**Q45. What is the difference between `kill(pid, SIGTERM)` and `kill(pid, SIGKILL)`?**

| | SIGTERM | SIGKILL |
|--|---------|---------|
| Can be caught? | Yes — process can install a handler | No — uncatchable |
| Can be ignored? | Yes | No |
| Allows cleanup? | Yes (if handler does cleanup) | No — immediate termination |
| Appropriate for? | Graceful shutdown | Last resort, force kill |

ShellX uses `SIGTERM` on exit for background jobs — polite, gives them a chance to clean up.

---

**Q46. What is `strace`? What would you use it for when debugging ShellX?**

`strace` intercepts and prints all **syscalls** made by a process.

```bash
strace -f ./shellx   # -f traces child processes too (fork-following)
```

Useful for:
- Verifying that `fork()`, `execvp()`, `waitpid()` are called correctly
- Detecting fd leaks (seeing `open()` calls without matching `close()`)
- Finding pipeline hangs (seeing which `read()` is blocking)
- Checking signal delivery (`rt_sigaction`, `rt_sigreturn`)

---

**Q47. What is `lsof`? How do you use it to find fd leaks?**

`lsof -p <pid>` lists all open files for that process. You'd run it against the ShellX process to verify no unexpected fds are open after command execution.

---

**Q48. What is `pstree`? How do you use it to check for zombies?**

```bash
pstree -p $$   # $$ = current shell's PID
```

Shows the process tree. Zombie processes appear as `(defunct)` in `ps aux`. If you see `shellx`'s children listed as `(defunct)` that don't disappear, you have a zombie bug.

---

## Section 9: C++ / Language-Level Questions

**Q49. What does `const char* const*` mean? Why does `execvp` need it?**

```c
int execvp(const char* file, char* const argv[]);
```

- `char* const argv[]` = array of pointers to char, where the **pointers** are const (can't change what they point to), but the pointed-to chars can be modified
- We often have `const char*` strings in C++ (`std::string::c_str()`) and need to cast: `const_cast<char* const*>(argv.data())`

The `execvp` signature is technically a POSIX historical wart — it should be `const char* const argv[]`.

---

**Q50. What is `perror()`? How does it differ from `strerror()`?**

```c
perror("open");        // prints: "open: No such file or directory\n"
strerror(errno)        // returns: "No such file or directory" (string)
```

`perror(prefix)` prints `prefix: <errno description>` to `stderr`. It uses the global `errno` variable set by the last failing syscall.

`strerror(errno)` returns the string without printing. Useful when you want to format it yourself with `fprintf`.

---

## Section 10: Classic Interview Coding Questions

**Q51. Write a function that runs a command with fork/exec/wait and returns its exit code.**

```cpp
#include <unistd.h>
#include <sys/wait.h>
#include <cerrno>

int run_command(const char* cmd, char* const argv[]) {
    pid_t pid = fork();
    if (pid == -1) {
        perror("fork");
        return -1;
    }
    if (pid == 0) {                 // child
        execvp(cmd, argv);
        perror(cmd);
        _exit(errno == ENOENT ? 127 : 126);
    }
    // parent
    int status;
    pid_t result;
    do {
        result = waitpid(pid, &status, 0);
    } while (result == -1 && errno == EINTR);

    if (WIFEXITED(status))   return WEXITSTATUS(status);
    if (WIFSIGNALED(status)) return 128 + WTERMSIG(status);
    return 1;
}
```

---

**Q52. Write code to implement `echo hello > out.txt` using syscalls.**

```cpp
pid_t pid = fork();
if (pid == 0) {
    int fd = open("out.txt", O_WRONLY|O_CREAT|O_TRUNC, 0644);
    if (fd == -1) { perror("out.txt"); _exit(1); }
    if (dup2(fd, STDOUT_FILENO) == -1) { perror("dup2"); _exit(1); }
    close(fd);
    execlp("echo", "echo", "hello", nullptr);
    _exit(127);
}
waitpid(pid, nullptr, 0);
```

---

**Q53. What are three ways a pipeline can hang? How do you fix each?**

1. **Parent doesn't close pipe fds** → reader never gets EOF → fix: `close(fds[0]); close(fds[1])` in parent after all forks
2. **Child doesn't close unused pipe end** → e.g., `grep` holds write-end open → fix: in every child, close ALL pipe fds after `dup2()`
3. **Pipe buffer is full, writer blocks** → occurs if reader is slow or dead → fix: ensure reader is running before writer; don't buffer unnecessarily

---

**Q54. What is the race condition if a signal handler calls `waitpid(-1, ...)`?**

Scenario:
- Shell is blocked in `waitpid(fg_pid, &status, 0)` for foreground job
- Background job finishes → kernel sends `SIGCHLD`
- Signal handler calls `waitpid(-1, &status, WNOHANG)`
- Handler reaps the foreground job (not the background one!)
- Original foreground `waitpid(fg_pid, ...)` returns -1 with `ECHILD` — child already reaped

**Fix (ShellX's approach):** Handler ONLY sets a flag. Reaping happens in the main loop where it's known whether a foreground job is running.

---

**Q55. Explain Copy-on-Write in the context of `fork()`. What's the performance implication?**

After `fork()`, the kernel doesn't copy all pages. Instead:
1. Parent and child share the same physical pages (mapped read-only in both)
2. When either process **writes** to a page, a page fault occurs
3. Kernel allocates a new physical page, copies the content, maps it to the writing process
4. Now each process has its own private copy of that page

**Performance:** `fork()` is O(1) for large processes (only the page table is copied, not the data). Cost is paid lazily, only when pages are actually written.

**Implication for ShellX:** Even large shells fork quickly. The child immediately calls `exec()`, which throws away all inherited pages anyway.

---

*This covers every possible question from Phase 0 foundations. When ready → say "Phase 1" and I'll cover the 4 most critical ShellX source files with deep annotations.*
