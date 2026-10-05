# ShellX Study Guide — Phase 0: Foundations, ELI5 & SDE Interview Prep

---

## 🗺️ Full Reading Plan

| Phase | Files | Theme |
|---|---|---|
| **0** | — | Foundations & ELI5 (this document) |
| **1** | `shellx.hpp`, `main.cpp`, `executor.cpp`, `pipeline.cpp` | The Core Engine — fork/exec/pipe |
| **2** | `signals.cpp/hpp`, `jobs.cpp/hpp` | Async Control — Signals & Background Jobs |
| **3** | `parser.cpp/hpp`, `redirection.cpp/hpp` | Input Processing — Parsing & I/O Redirection |
| **4** | `builtins.cpp/hpp`, `CMakeLists.txt`, `tests/`, `docs/` | Peripherals — Builtins, Build, Tests |

---

## 🧒 ELI5 — What Is ShellX?

Imagine you're talking to your computer through a walkie-talkie.

- You press the button and say: **"show me all files"**
- The walkie-talkie **hears** you → that's the **Parser**
- It **finds** the `ls` program on your computer → that's `execvp()`
- It creates a **new temporary worker** to run it → that's `fork()`
- It **waits** until that worker finishes → that's `waitpid()`

ShellX builds this walkie-talkie from scratch using only raw Linux system calls.

### ELI5 — The 5 Concepts That Run Everything

| Concept | ELI5 |
|---|---|
| **`fork()`** | Boss says "make an exact copy of yourself to do this job" |
| **`execvp()`** | The copy says "stop being a shell, become `ls` right now" |
| **`waitpid()`** | Boss sits at desk waiting for the worker to come back |
| **pipe `\|`** | A conveyor belt: worker 1 drops output, worker 2 picks it up |
| **signal** | A phone buzz interrupting whatever you're doing right now |

### The Flow of Every Command You Type

```
You type: ls -la | grep txt > out.txt

[Parser]   → tokens: ["ls", "-la", "|", "grep", "txt", ">", "out.txt"]
              → Pipeline { commands: [cmd1, cmd2], background: false }

[Executor] → cmd1 (ls -la): fork() → child execs ls, stdout → pipe write end
           → cmd2 (grep txt > out.txt): fork() → child execs grep
                                         stdin ← pipe read end
                                         stdout → open("out.txt")
           → parent: close all pipe fds → waitpid(all children)
```

---

## 📚 Foundation Concepts Reference

### Process & Memory

| Concept | Definition |
|---|---|
| **Process** | A running program with its own PID, memory, open fds, and signal table |
| **PCB** | Process Control Block — kernel data structure tracking process state |
| **Program vs Process** | Program = static code on disk. Process = code running in memory |
| **Virtual memory** | Each process sees its own private address space; kernel maps to real RAM |
| **Text/Data/Stack/Heap** | Segments of a process's memory: code / global vars / call stack / dynamic alloc |

### File Descriptors

```
Every process inherits these 3:
  fd 0 → stdin  (keyboard by default)
  fd 1 → stdout (screen by default)
  fd 2 → stderr (screen by default)

open("file.txt") → returns fd 3 (next available)
dup2(3, 1)       → fd 1 now points to file.txt (stdout redirected)
close(3)         → original fd 3 released (fd 1 still open to file)
```

### Key POSIX Syscalls

| Syscall | Signature | What it does |
|---|---|---|
| `fork()` | `pid_t fork()` | Creates child = copy of parent |
| `execvp()` | `int execvp(const char* file, char* const argv[])` | Replaces current process image |
| `waitpid()` | `pid_t waitpid(pid_t pid, int* status, int opts)` | Collects child exit status |
| `pipe()` | `int pipe(int pipefd[2])` | Creates pipefd[0]=read, pipefd[1]=write |
| `dup2()` | `int dup2(int oldfd, int newfd)` | Duplicates oldfd onto newfd |
| `open()` | `int open(const char* path, int flags, mode_t mode)` | Opens file, returns fd |
| `close()` | `int close(int fd)` | Releases fd |
| `sigaction()` | `int sigaction(int sig, struct sigaction* act, struct sigaction* old)` | Install signal handler |
| `kill()` | `int kill(pid_t pid, int sig)` | Send signal to process |
| `chdir()` | `int chdir(const char* path)` | Change working directory |
| `getcwd()` | `char* getcwd(char* buf, size_t size)` | Get current directory |
| `_exit()` | `void _exit(int status)` | Immediate exit, no stdio flush |

### Linux Signals

| Signal | Number | Default | When |
|---|---|---|---|
| `SIGCHLD` | 17 | Ignored | Child process changed state |
| `SIGINT` | 2 | Terminate | User pressed Ctrl+C |
| `SIGTERM` | 15 | Terminate | Polite kill request |
| `SIGKILL` | 9 | Kill | Force kill (uncatchable) |
| `SIGSTOP` | 19 | Stop | Pause process (uncatchable) |
| `SIGHUP` | 1 | Terminate | Terminal disconnected |

### Exit Codes

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | General error |
| 126 | Found but not executable (EACCES) |
| 127 | Command not found (ENOENT) |
| 128 + N | Killed by signal N |

---

## 🎯 SDE Interview Q&A — Complete Scripted Answers

> These are exact scripts. Say them out loud. Practice them. Every question below is a real interview question.

---

### ─── SECTION 1: Processes ───

---

**Q1. What is a process? How is it different from a program?**

> "A program is a static file on disk — just bytes of compiled code. A process is what happens when the OS loads that program into memory and starts running it. A process has its own address space, a unique PID, open file descriptors, a stack, a heap, and a signal disposition table. Multiple processes can run the same program simultaneously — for example, you can open two terminals each running `bash`; same program, two separate processes with completely separate memory."

---

**Q2. What happens when you call `fork()`?**

> "fork() creates an exact copy of the calling process. The kernel duplicates the process's memory space, file descriptor table, and signal handlers. After fork() returns, there are now two processes running the same code from the same point. In the parent, fork() returns the child's PID — a positive number. In the child, fork() returns 0. On failure, it returns -1 in the parent only. The standard idiom is:
> ```c
> pid_t pid = fork();
> if (pid == 0) {
>     // child code
> } else {
>     // parent code
> }
> ```
> The child gets a *copy-on-write* copy of the parent's memory — pages are shared until either side writes to them."

---

**Q3. What is Copy-on-Write (CoW) in the context of fork()?**

> "After fork(), the OS doesn't immediately duplicate all memory pages. Instead, both parent and child share the same physical pages marked as read-only. The moment either process tries to write to a page, the OS makes a private copy of just that page for the writer. This is called copy-on-write. It makes fork() fast — especially when followed immediately by exec(), which throws the whole memory away anyway, so no copy is ever needed."

---

**Q4. What is a zombie process? How do you prevent it?**

> "A zombie is a process that has finished executing but hasn't been reaped by its parent yet. When a process exits, the kernel keeps a small record of its PID and exit status in the process table — it can't fully clean up until the parent calls waitpid() to collect that exit status. Until then, the process is in the 'zombie' state — it's dead but still occupies an entry in the process table.
>
> Prevention: the parent must call waitpid() for every child it forks. In ShellX, foreground children are reaped immediately after they exit. Background children are reaped via SIGCHLD — when a child exits, the kernel sends SIGCHLD to the parent, and the REPL calls waitpid(WNOHANG) to clean up."

---

**Q5. What is an orphan process?**

> "An orphan is a process whose parent has died. When a parent exits without waiting for its children, those children become orphans. The kernel automatically re-parents orphans to PID 1 (init or systemd). Init periodically calls waitpid() so orphans don't become zombies. In ShellX, when the user types `exit`, we send SIGTERM to background jobs and then exit without waiting — those children become orphans and init adopts them."

---

**Q6. What is the difference between `exit()` and `_exit()`?**

> "Both terminate the process. But `exit()` is a C library function that does cleanup before calling the underlying _exit() syscall — it flushes all stdio buffers, calls atexit() handlers, and closes stdio streams. `_exit()` is the raw syscall — it terminates immediately with no cleanup.
>
> In ShellX, child processes always use `_exit()` after exec fails. The reason: the child inherits copies of the parent's stdio buffers. If exec fails and we call `exit()`, those inherited buffers get flushed — causing whatever was in the parent's stdout buffer to be written twice. `_exit()` skips this entirely."

---

**Q7. What does `execvp()` do? Why does it replace the process?**

> "execvp() loads a new program into the current process's address space and starts executing it. The p suffix means it searches the PATH environment variable automatically. exec() replaces the process image — the text segment, data, stack, and heap are all replaced with the new program's content. The PID stays the same, but everything else changes.
>
> The reason we fork() before exec() is so we have a disposable copy. The original shell process is not replaced — only the child is. If exec fails, the child calls _exit() and the parent continues."

---

**Q8. What is `execv` vs `execvp` vs `execve`?**

> "These are all variants of the exec family:
> - `execv(path, argv)` — takes full path, no PATH search
> - `execvp(file, argv)` — takes name, searches PATH automatically
> - `execve(path, argv, envp)` — takes full path + explicit environment array
>
> ShellX uses `execvp` because users type command names like `ls`, not full paths like `/bin/ls`. The OS resolves the name through PATH."

---

### ─── SECTION 2: File Descriptors & Pipes ───

---

**Q9. What is a file descriptor?**

> "A file descriptor is a small non-negative integer that's an index into the process's open file table. When a process opens a file, socket, or pipe, the kernel returns an fd. The process uses this fd in all subsequent read(), write(), close() calls. fds 0, 1, 2 are always stdin, stdout, stderr by convention. Each process has its own fd table, though two fds in different processes can refer to the same underlying kernel file description — this is how pipe inheritance works."

---

**Q10. Explain `dup2(oldfd, newfd)`.**

> "dup2() makes newfd point to the same open file description as oldfd. If newfd is already open, it's closed first atomically. After the call, both oldfd and newfd refer to the same file — writes to either go to the same place.
>
> The classic use: to redirect stdout to a file:
> ```c
> int fd = open("out.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
> dup2(fd, STDOUT_FILENO);  // fd 1 now points to out.txt
> close(fd);                // fd still open via fd 1, close original
> ```
> After this, printf() writes go to out.txt instead of the terminal."

---

**Q11. How does `pipe()` work? Walk through a full example.**

> "pipe() creates a unidirectional data channel. It fills an array of two file descriptors: pipefd[0] is the read end, pipefd[1] is the write end. Data written to pipefd[1] is available to read from pipefd[0]. It's a kernel buffer — typically 64KB on Linux.
>
> For `ls | grep txt`:
> 1. Parent calls `pipe(fds)` → fds[0]=read, fds[1]=write
> 2. Fork child 1 (ls):
>    - `dup2(fds[1], STDOUT_FILENO)` → ls output goes into pipe
>    - `close(fds[0]); close(fds[1])` → close originals
>    - `execvp("ls", ...)`
> 3. Fork child 2 (grep):
>    - `dup2(fds[0], STDIN_FILENO)` → grep reads from pipe
>    - `close(fds[0]); close(fds[1])` → close originals
>    - `execvp("grep", ...)`
> 4. Parent: `close(fds[0]); close(fds[1])` → critical!
> 5. Parent: `waitpid(all children)`"

---

**Q12. Why must the parent close pipe fds after forking? What happens if it doesn't?**

> "A pipe's read end returns EOF only when ALL write ends are closed. If the parent holds onto fds[1] (the write end) without closing it, the grep process reading from fds[0] will block forever — it keeps waiting for more data because the write end is still open from the parent's perspective. The command hangs.
>
> The rule in ShellX is: every process closes every fd it doesn't own. This single rule prevents all pipeline hangs and fd leaks."

---

**Q13. What is an fd leak? Why is it dangerous?**

> "An fd leak is when a process opens file descriptors and never closes them. Each process has a limit on how many fds it can have open (typically 1024 by default, configurable to millions). Leaked fds waste kernel resources. In pipeline code specifically, leaked write ends of pipes prevent downstream commands from ever seeing EOF, causing deadlocks. In long-running servers, fd leaks eventually exhaust the limit and all future open()/accept() calls fail."

---

**Q14. For a pipeline of N commands, how many pipes and processes do you need?**

> "For N commands you need exactly N-1 pipes and N child processes. The pattern is:
> - Pipe `i` (0-indexed) connects command `i` to command `i+1`
> - Command `i` reads from pipe `i-1` (if not first) and writes to pipe `i` (if not last)
> - The first command reads from its original stdin; the last writes to its original stdout
>
> In ShellX's pipeline.cpp, we create `2*(n-1)` fds in a flat array. `pipe_fds[2*i]` = read end of pipe i, `pipe_fds[2*i+1]` = write end of pipe i."

---

### ─── SECTION 3: Signals ───

---

**Q15. What is a signal? Name 5 common signals.**

> "A signal is an asynchronous notification sent to a process to notify it of an event. It's like an interrupt at the software level. Signals can be sent by the kernel, by other processes (with appropriate permissions), or by the process itself.
>
> Five common signals:
> 1. SIGINT (2) — Ctrl+C from terminal, default: terminate
> 2. SIGTERM (15) — polite termination request, can be caught
> 3. SIGKILL (9) — force kill, cannot be caught or ignored
> 4. SIGCHLD (17) — sent to parent when child changes state
> 5. SIGSEGV (11) — segmentation fault, memory access violation"

---

**Q16. What is `sigaction()` and how is it better than `signal()`?**

> "Both install signal handlers, but sigaction() is more reliable and portable. The old signal() function has implementation-defined behavior — on some systems the handler is reset to SIG_DFL after each signal (making it one-shot). sigaction() has well-defined semantics:
> - The `sa_flags` field controls behavior precisely (SA_RESTART, SA_NOCLDSTOP, etc.)
> - `sa_mask` lets you block other signals while the handler runs
> - The handler stays installed until explicitly changed
>
> In ShellX, we use sigaction() with SA_RESTART (auto-restart interrupted syscalls) and SA_NOCLDSTOP (don't fire SIGCHLD when children are merely stopped, only when they exit)."

---

**Q17. What is async-signal safety? Why does it matter?**

> "When a signal arrives, the handler can interrupt the process at any point — even in the middle of a malloc() or printf() call. If the signal handler also calls malloc() or printf(), we have re-entrancy — the function is called again while it's already running. Most libc functions are NOT async-signal-safe because they use internal locks or global state.
>
> If a signal handler calls a non-async-signal-safe function, the result is undefined behavior — potential deadlock (if the handler tries to acquire a lock the main code already holds) or data corruption.
>
> The only safe pattern: signal handlers should do minimal work. In ShellX, both handlers just set a `volatile sig_atomic_t` flag and return. The actual work (reaping jobs, reprinting prompt) happens in the main REPL loop where it's safe."

---

**Q18. What is `volatile sig_atomic_t`? Why both keywords?**

> "`sig_atomic_t` is an integer type guaranteed to be read/written atomically by the hardware — no partial reads even on architectures where int isn't natively atomic.
>
> `volatile` tells the compiler to never cache this variable in a register or optimize away reads/writes — always go to memory. Without volatile, the compiler might see that `g_sigchld_pending` isn't modified in the main loop's visible code and optimize `if (g_sigchld_pending)` to be always-false.
>
> Together: `volatile sig_atomic_t` is the only correct type for a variable shared between a signal handler and the main program."

---

**Q19. What is SA_RESTART? When would you NOT want it?**

> "SA_RESTART tells the kernel to automatically restart certain slow system calls if they're interrupted by a signal. Without it, syscalls like read(), waitpid(), accept() return -1 with errno=EINTR when a signal arrives, requiring manual retry logic.
>
> With SA_RESTART: kernel automatically retries, cleaner code.
>
> When you'd NOT want it: if you deliberately want signals to break out of a blocking call — for example, a server that uses SIGINT to gracefully shut down by breaking out of an accept() loop. In that case, you'd omit SA_RESTART and check for EINTR explicitly to distinguish 'real error' from 'interrupted by signal'."

---

**Q20. Why does ShellX set SIGINT to SIG_DFL in child processes after fork?**

> "After fork(), the child inherits the parent's signal handlers. The shell has installed a custom SIGINT handler that just sets a flag and returns — it intentionally doesn't die from Ctrl+C. But child processes (like `cat`, `sleep`) should behave normally and terminate when the user presses Ctrl+C.
>
> So every child, immediately after fork(), resets SIGINT to SIG_DFL (the default action = terminate). This way: Ctrl+C kills the child normally, the child's death delivers SIGCHLD to the parent, and the parent's custom handler catches it."

---

**Q21. What is the race condition in signal handling that ShellX avoids?**

> "The dangerous scenario: a background child finishes. The kernel sends SIGCHLD to the shell. If the SIGCHLD handler called waitpid() directly, AND the foreground child's executor also calls waitpid() at the same time, you could have two concurrent waitpid() calls racing to reap the same child. The loser gets ECHILD (no such child).
>
> ShellX's design eliminates this: the SIGCHLD handler only sets `g_sigchld_pending = 1`. All actual waitpid() calls happen in one of two places: (1) the executor's blocking waitpid() for foreground children, (2) reapFinishedJobs() in the REPL for background children. There is exactly one waitpid() call site per PID. No race."

---

### ─── SECTION 4: Shell Internals ───

---

**Q22. Why are `cd`, `pwd`, `exit` built-in commands? Why can't they be external programs?**

> "`cd` changes the working directory. If it ran in a child process, only that child's directory would change — the shell's directory would be unaffected. Built-ins run in the parent process itself, directly calling chdir(). Same reasoning applies to `exit` — calling exit() in a child would only exit the child, not the shell. pwd reads the shell's own working directory, so it also needs to run in the parent. These commands need to affect the shell process's own state, which is impossible from a child."

---

**Q23. What is a REPL? How does ShellX implement one?**

> "REPL stands for Read-Evaluate-Print Loop. It's the core pattern of any interactive interpreter — shells, Python, database consoles, all use it.
>
> ShellX's REPL in main.cpp:
> 1. **Read** — `getline(cin, line)` blocks waiting for user input
> 2. **Evaluate** — `parse(line)` → Pipeline struct, then `executePipeline(pipeline)`
> 3. **Print** — the executed command prints its own output
> 4. **Loop** — go back to step 1
>
> The loop also checks signal flags at each iteration: if SIGCHLD fired, reap finished jobs; if SIGINT fired, reprint the prompt."

---

**Q24. What is the background execution model (`&`) in ShellX?**

> "When a command ends with `&`, it's a background job. After fork(), instead of calling blocking waitpid(), the parent immediately adds the child's PID to the job list and returns. The child runs concurrently with the shell's REPL.
>
> When the child eventually finishes, the kernel sends SIGCHLD to the shell. The shell's SIGCHLD handler sets a flag. At the next REPL iteration, the shell calls reapFinishedJobs() which calls waitpid(WNOHANG) — non-blocking — for each tracked job PID. If the job is done, it's reaped and removed from the list."

---

**Q25. What is `waitpid(pid, &status, WNOHANG)` vs `waitpid(pid, &status, 0)`?**

> "The third argument is options flags.
>
> `0` (blocking): waits indefinitely until the specified child exits. Used for foreground commands — we want to block until the user's command completes.
>
> `WNOHANG` (non-blocking): returns immediately even if the child hasn't exited. Returns 0 if child is still running, child's PID if it has exited. Used for background job reaping — we don't want to block the REPL just to check if background jobs finished."

---

**Q26. How do you extract the exit status from `waitpid`'s status integer?**

> "The status integer returned via waitpid is not directly the exit code — it's a packed bit field. You must use macros to decode it:
>
> - `WIFEXITED(status)` — true if child exited normally (via exit() or return from main)
> - `WEXITSTATUS(status)` — extracts the actual exit code (0-255), only valid if WIFEXITED
> - `WIFSIGNALED(status)` — true if child was killed by a signal
> - `WTERMSIG(status)` — extracts the signal number that killed it, only valid if WIFSIGNALED
>
> ShellX returns `128 + WTERMSIG(status)` for signal-killed children, following the standard shell convention."

---

### ─── SECTION 5: C++ & Systems Programming ───

---

**Q27. What is `constexpr` and why use it instead of `#define`?**

> "constexpr declares a value computed at compile time. Unlike #define, it has a type, is scoped, participates in the type system, and is visible to the debugger.
>
> ```cpp
> #define MAX_JOBS 64           // no type, no scope, preprocessor substitution
> constexpr int MAX_JOBS = 64; // typed, scoped, inspectable by debugger
> ```
>
> In ShellX, constants like MAX_PIPELINE_LENGTH and MAX_BACKGROUND_JOBS are constexpr — they're safer and more idiomatic modern C++."

---

**Q28. Why does `execvp` take `char* const argv[]` (array of C strings) instead of `std::vector<std::string>`?**

> "execvp is a C POSIX API — it predates C++ and works at the system call level with C-style null-terminated strings. std::string and std::vector are C++ abstractions that don't survive across exec boundaries anyway (exec replaces the entire process image).
>
> In ShellX, we build a `std::vector<const char*>` by calling `.c_str()` on each std::string argument, push_back a nullptr at the end (required sentinel for argv), then pass `.data()` to execvp with a const_cast. The vector owns the lifetime — it lives long enough because exec either succeeds (process replaced) or fails (vector goes out of scope normally)."

---

**Q29. What is `std::move` and why is it used when building the pipeline?**

> "std::move converts an lvalue to an rvalue reference, enabling the move constructor or move assignment. Instead of copying an object's data, the move 'steals' its resources (the internal buffer of a string or vector) — leaving the source in a valid-but-empty state.
>
> In parser.cpp: `pipeline.commands.push_back(std::move(cmd))` — rather than copying the Command struct (which contains a vector of strings), we move it. The cmd variable is emptied (fine since we're done with it) and the pipeline now owns those strings without any allocation."

---

**Q30. What is the difference between `struct` and `class` in C++?**

> "The only difference is default access specifier: struct members are public by default, class members are private by default. Both can have constructors, methods, inheritance.
>
> In ShellX, Command, Pipeline, and Job are structs — they're plain data holders (POD-like) with no encapsulation logic needed. Using struct signals to the reader 'this is data, not a stateful object'."

---

### ─── SECTION 6: OS Deep-Dive ───

---

**Q31. What is virtual memory? How does it relate to processes?**

> "Virtual memory is an abstraction layer between a process's view of memory and physical RAM. Each process gets its own private address space (e.g., 0 to 2^48 on x86-64). The CPU's MMU (Memory Management Unit) translates virtual addresses to physical addresses via page tables maintained by the kernel.
>
> Benefits: isolation (process A can't accidentally access process B's memory), overcommit (more virtual memory can be allocated than physical RAM exists — pages are backed lazily), and enables fork's copy-on-write optimization."

---

**Q32. What is context switching?**

> "A context switch is when the OS scheduler preempts a running process and switches the CPU to run a different process. The kernel saves the current process's register state, program counter, and other CPU state into its PCB, then loads another process's saved state and jumps to its program counter.
>
> Context switches have overhead — saving/restoring registers, TLB flushes for different address spaces. This is why threads (same address space, no TLB flush needed) context switch faster than processes."

---

**Q33. What is the difference between blocking and non-blocking I/O?**

> "Blocking I/O: a read() or write() call blocks (suspends) the calling process until the operation completes — data is available, buffer has space, etc. The process gives up the CPU while waiting.
>
> Non-blocking I/O: the call returns immediately with EAGAIN/EWOULDBLOCK if the operation can't complete right now. The process must retry later.
>
> ShellX uses blocking waitpid() for foreground commands (intentionally block until child exits) and non-blocking waitpid(WNOHANG) for background job reaping (don't block the REPL)."

---

**Q34. What is the kernel's process table? What happens when it fills up?**

> "The process table is a kernel data structure that tracks all running processes. Each entry (PCB) stores PID, state, memory maps, open fds, signal handlers, etc. Linux's default PID limit is 32768 (configurable via /proc/sys/kernel/pid_max).
>
> If the limit is hit, fork() returns -1 with errno=EAGAIN — no new processes can be created. Zombie processes consume entries even though they're dead — this is why unreaped zombies are a real resource problem."

---

**Q35. What happens to open file descriptors after `fork()`?**

> "The child inherits copies of all the parent's file descriptors. Both parent and child now have file descriptors referring to the same underlying open file descriptions — the same seek position, same flags. The kernel reference counts the underlying file description, and it's not fully released until all fd references are closed.
>
> This inheritance is essential for pipe-based IPC: the parent creates a pipe, forks, and the child inherits the pipe fds. But it also means: after forking, both parent and child must close pipe ends they don't need — otherwise the reference count stays > 0 and EOF never arrives."

---

**Q36. What is the difference between a pipe and a socket?**

> "A pipe is a unidirectional, in-memory kernel buffer for communication between related processes (parent-child or siblings). It has no network involvement. It's half-duplex — data flows one way; you need two pipes for bidirectional communication.
>
> A socket is bidirectional and can communicate across a network, or locally via Unix domain sockets. Sockets have addresses, can be connection-oriented (TCP) or connectionless (UDP).
>
> ShellX uses anonymous pipes (pipe()) — the simplest IPC for chaining command output to input."

---

**Q37. How does the shell know if a command is a built-in or external?**

> "The shell checks the command name against a hardcoded list of built-in names before forking. In ShellX's executor.cpp:
> ```cpp
> if (isBuiltin(cmd.args[0])) {
>     return executeBuiltin(cmd);  // run in parent, no fork
> }
> return executeSingle(cmd, background);  // fork + exec
> ```
> The check is just string comparison. Real shells like bash also handle keywords (if, while, for), aliases, and functions as special cases before reaching external programs."

---

**Q38. What is a race condition? Give a real example from ShellX's design.**

> "A race condition is when a program's correctness depends on the relative timing of operations in concurrent execution paths, and that timing is non-deterministic.
>
> Real example ShellX explicitly avoids: if the SIGCHLD handler called waitpid() directly, and a background child exits while the foreground executor is also in waitpid(), both code paths could try to reap the same PID. One would succeed; the other would get ECHILD (no such child) and report a spurious error.
>
> Solution: SIGCHLD handler only sets a flag. Only one code path ever calls waitpid() for any given PID. Zero race condition."

---

**Q39. What is the `PATH` environment variable? How does `execvp` use it?**

> "PATH is an environment variable containing a colon-separated list of directories to search for executables — e.g., `/usr/local/bin:/usr/bin:/bin`. When you type `ls`, the shell's execvp() searches each directory in PATH in order for a file named `ls`. The first match found is executed.
>
> execvp() does this search automatically (the 'p' suffix stands for PATH). execv() requires the full absolute path — you'd have to pass `/bin/ls` explicitly."

---

**Q40. Explain the full lifecycle of running `ls | wc -l` in ShellX.**

> "Let me trace it step by step:
>
> 1. User types `ls | wc -l`, presses Enter
> 2. **Parser**: tokenizes → `['ls', '|', 'wc', '-l']`, builds Pipeline with 2 Commands
> 3. **Executor**: sees 2 commands, calls executePipelineChain()
> 4. **pipeline.cpp**: creates 1 pipe → fds[0]=read, fds[1]=write
> 5. **Fork child 1** (ls):
>    - Resets SIGINT to SIG_DFL
>    - dup2(fds[1], STDOUT_FILENO) — ls output goes into pipe
>    - Closes all pipe fds (fds[0] and fds[1])
>    - execvp('ls', ['ls', nullptr])
> 6. **Fork child 2** (wc):
>    - Resets SIGINT to SIG_DFL
>    - dup2(fds[0], STDIN_FILENO) — wc reads from pipe
>    - Closes all pipe fds
>    - execvp('wc', ['wc', '-l', nullptr])
> 7. **Parent**: closes fds[0] and fds[1] — critical for EOF signaling
> 8. **Parent**: waitpid(child1), waitpid(child2)
> 9. ls writes its output to pipe, exits
> 10. wc reads until EOF (which arrives when ls exits AND parent closed write end)
> 11. wc prints line count, exits
> 12. Both children reaped, parent returns last command's exit status
> 13. REPL prints prompt again"

---

### ─── SECTION 7: Behavioral / Design Questions ───

---

**Q41. How would you extend ShellX to support `&&` and `||`?**

> "Currently the parser explicitly rejects `&&` and `||`. To add them, I'd need to:
> 1. **Parser**: recognize `&&` and `||` as tokens and add a new `ConditionalList` structure — a sequence of pipelines with connecting operators
> 2. **Data structures**: something like `struct ConditionalList { vector<pair<Pipeline, Operator>> stages; }` where Operator is AND/OR/NONE
> 3. **Executor**: evaluate left pipeline, check exit status — if 0 and AND, execute right pipeline; if nonzero and OR, execute right pipeline; otherwise skip.
> Exit code chaining follows standard shell semantics."

---

**Q42. What would you do differently if you were scaling this shell for production?**

> "Several things:
> 1. **Process groups / job control**: use setpgid() to put each pipeline in its own process group. This isolates Ctrl+C — it reaches only the foreground group, not background jobs.
> 2. **Terminal control**: use tcsetpgrp() to transfer terminal ownership between foreground/background.
> 3. **Multi-PID background jobs**: the Job struct currently holds one PID. For `cmd1 | cmd2 &`, we'd need a vector of PIDs per job.
> 4. **History and readline**: use libreadline or linenoise for command history, arrow key navigation.
> 5. **Memory safety**: use address sanitizers in CI, consider switching to Rust for a greenfield version.
> 6. **Concurrent SIGCHLD**: add sigprocmask() to block SIGCHLD in critical sections."

---

**Q43. What is the difference between a shell built-in and a shell function?**

> "A built-in is hardcoded into the shell binary itself — like `cd`, `exit`, `jobs`. They run directly in the shell process.
>
> A shell function is user-defined code written in the shell's scripting language — like:
> ```bash
> greet() { echo "Hello $1"; }
> ```
> Functions also run in the current shell process (no fork) by default. The difference is that built-ins are compiled into the shell binary while functions are defined at runtime in the shell's own language."

---

## 🎯 Pre-Phase-1 Checklist

- [ ] I can explain fork() return values without hesitation
- [ ] I can draw the fd table before and after dup2()
- [ ] I can explain why the parent must close pipe fds
- [ ] I can explain zombie vs orphan without confusing them
- [ ] I know why async-signal handlers must be minimal
- [ ] I can explain why `cd` is a builtin
- [ ] I can trace a full pipe command execution end-to-end
- [ ] I know what WIFEXITED / WEXITSTATUS / WIFSIGNALED do
- [ ] I know the difference between exit() and _exit()

---

> ✅ **Say "proceed to Phase 1"** when ready. Phase 1 covers `shellx.hpp`, `main.cpp`, `executor.cpp`, and `pipeline.cpp` with deep code-level analysis and SDE interview questions tied to actual lines of code.
