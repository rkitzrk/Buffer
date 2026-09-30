**ShellX** is a Unix shell in C++20 that runs commands with POSIX syscalls (`fork`, `execvp`, `waitpid`, `pipe`, `dup2`, `sigaction`). You already have algorithms; this project is **OS + processes + file descriptors**. Phase 0 is that mental model. Source files start in Phase 1.

---

## Roadmap (files, most important first)

| Phase | Goal | Files (read in this order) |
|---|---|---|
| **0** | Foundations (this message) | Concepts, not source |
| **1** | Core loop + process execution | `README.md` → `include/shellx.hpp` → `src/main.cpp` → `src/executor.cpp` → `src/pipeline.cpp` (+ their `.hpp`) |
| **2** | Grammar + parent vs child | `src/parser.cpp` → `src/redirection.cpp` → `src/builtins.cpp` |
| **3** | Async jobs + signals | `src/signals.cpp` → `src/jobs.cpp` |
| **4** | Build, tests, docs, gaps | `CMakeLists.txt` → `tests/test_shellx.sh` → `docs/*` |

Reply **“go to Phase 1”** when you want the file-by-file walkthrough plus Phase 1 interview questions.

---

# Phase 0 — Foundations

Think of the kernel as a referee: it owns **processes**, **open files**, and **signals**. A shell is a program whose job is: **read a line → parse → start other programs → wait (or not) → print a prompt again**.

## 0.1 What a shell actually is

A **program** is a file on disk (`/bin/ls`). A **process** is a running instance: PID, address space, open FDs, current working directory, signal dispositions.

The shell is itself a process. It must **not** become `ls`. If it called `exec("ls")` on itself, it would *turn into* `ls` and never return to the prompt. So it **forks** a child, and the **child** execs.

**REPL:** Read line → Evaluate → Print result (prompt) → Loop. In ShellX this is `main.cpp`.

## 0.2 Processes (the CP analog)

| CP | OS |
|---|---|
| Function call: same stack, returns | `exec`: this process is replaced; it does not “return” on success |
| Thread (shared heap) | `fork`: new process, copy of memory (copy-on-write), **own** PID |
| `return` from `main` | process **exits**; parent must **reap** exit status or it becomes a **zombie** |

**`fork()`**

- Parent: return value = child’s PID (`> 0`)
- Child: return value = `0`
- Failure: `-1` (no child created)

After fork, **both** continue from the same instruction. You branch on the return value.

**Copy-on-write:** the kernel does not copy all pages immediately. Pages are shared read-only until one process writes; then that page is copied. `fork()` is cheap until someone mutates a lot of memory.

**`execvp(file, argv)`**

- Replaces the process image (code, heap, stack) with a new program
- File descriptors **stay open** (unless `O_CLOEXEC`) — that is why redirection works *before* exec
- Searches `$PATH` if `file` has no `/`
- On **success**, it never returns
- On **failure**, it returns `-1` and sets `errno`

**`waitpid(pid, &status, flags)`**

- Collects a dead child’s exit status (reaps)
- `flags = 0`: **block** until that child exits
- `flags = WNOHANG`: return immediately; `0` means still running
- `pid = -1`: any child
- Macros: `WIFEXITED`, `WEXITSTATUS`, `WIFSIGNALED`, `WTERMSIG`

## 0.3 File descriptors (FDs)

An FD is a small integer in **this process’s** FD table:

| FD | Name | Default |
|---|---|---|
| 0 | stdin | keyboard / terminal |
| 1 | stdout | terminal |
| 2 | stderr | terminal |

`open()` → new FD. `close()` → drop it. **`dup2(old, new)`** makes `new` refer to the same open file as `old`. After `dup2(fd, 1)`, writes to stdout go to that file.

**Inheritance:** the child of `fork()` gets a **copy of the FD table**. Same open file objects in the kernel (shared offset, etc.).

**Why close extra FDs in pipelines:** a pipe’s **write end** staying open in *any* process means the reader never sees EOF → `cat` / `wc` **hangs forever**. This is the #1 shell bug.

## 0.4 Pipes

`pipe(fds)` → `fds[0]` read, `fds[1]` write. Kernel buffer between them. **Unrelated to files on disk** until you `dup2` them onto 0/1.

For `A | B | C` you need **2 pipes** (N commands → N−1 pipes). Command `i` reads from pipe `i-1` and writes to pipe `i`.

## 0.5 Signals (async events)

A signal is an interrupt: `SIGINT` (Ctrl+C), `SIGCHLD` (child died), `SIGTERM`, `SIGKILL` (unkillable).

Handlers run **asynchronously**. Almost nothing is safe inside them: no `malloc`, no `std::cout`, no `vector` push. Safe: write a `volatile sig_atomic_t` flag.

ShellX: handlers **only set flags**. Reaping happens in the **REPL**, not in the handler. That avoids a race with foreground `waitpid()`.

**`sigaction` vs `signal()`:** `sigaction` has defined flags (`SA_RESTART`, `SA_NOCLDSTOP`). Prefer it.

**`SA_RESTART`:** interrupted syscalls (like `read`) restart instead of failing with `EINTR`. ShellX **still** retries `waitpid` on `EINTR` (belt and suspenders).

## 0.6 Built-ins vs external commands

Each process has its **own** cwd, env, etc.

- `ls` → must fork+exec (`/bin/ls`)
- `cd` → **must** run in the **shell** (`chdir`). A child `chdir` dies with the child; parent cwd unchanged
- `pwd` could theoretically be `/bin/pwd`, but ShellX implements it as a builtin (`getcwd`)
- `exit` / `jobs` must be builtins (they manage **this** process / job table)

## 0.7 `_exit` vs `exit`

After `fork()`, parent and child share **stdio buffers** until COW. `exit()` flushes buffers → possible **double print**. Failed-exec children use **`_exit()`** (no flush). Standard shell practice.

## 0.8 Exit codes (interview gold)

| Code | Meaning |
|---|---|
| 0 | success |
| 1–125 | program’s own errors |
| **126** | found, not executable (`EACCES`) |
| **127** | not found (`ENOENT`) |
| **128 + N** | killed by signal N (`SIGINT` = 2 → 130) |

Pipeline status (bash default, ShellX too): **last command only** (no `pipefail`).

## 0.9 C++ you need beyond CP

- Headers: `unistd.h`, `sys/wait.h`, `fcntl.h`, `signal.h`
- `errno`, `perror`, `strerror`
- `const char*` argv for `execvp` — extra `nullptr` terminator
- `volatile sig_atomic_t` for flags
- CMake: compile `src/*.cpp`, `-I include`, C++20
- RAII is **not** used for FDs here; you `close()` by hand (systems style)

You do **not** need graphs/DP. You **do** need to draw FDs on paper.

## 0.10 What ShellX does *not* do (say this honestly)

- No `;` `&&` `||` command substitution
- No `cmd1 | cmd2 &` (job tracks **one** PID)
- **No process groups** (`setpgid`). Background jobs can still get **Ctrl+C** from the terminal. Documented limitation, not an accident.

---

# Phase 0 — Interview questions and answers

These are the questions interviewers actually ask for a “I built a shell” project, plus fundamentals they use to check you are not reciting a tutorial. Docs in the repo already cover some; extra ones are marked **(not in the old guide)**.

### Processes and fork/exec

**Q1. What does `fork()` actually do?**  
Duplicates the calling process: same code, memory (COW), open FDs, cwd. Returns `0` in child, child PID in parent, `-1` on failure.

**Q2. Why both `fork()` and `exec()`? Why not just `exec()`?**  
`exec` replaces *this* process. The shell would become `ls` and never prompt again. Fork first; exec only in the child.

**Q3. If `exec` succeeds, what is the next line of C++ after `execvp`?**  
It never runs. Only the failure path after `execvp` runs.

**Q4. Difference between `execl`, `execv`, `execvp`? (not in old guide)**  
`l` = list of args in the call; `v` = argv array; `p` = search `PATH`. ShellX uses **`execvp`**.

**Q5. Why `const_cast` on argv for `execvp`? (not in old guide)**  
POSIX `execvp` takes `char * const argv[]` (non-const `char`). C++ `c_str()` is `const char*`. The program is not supposed to modify argv; the cast is an API mismatch.

**Q6. What is copy-on-write? Does fork copy the whole heap?**  
Not immediately. Pages shared until a write. Large heaps still make *later* writes expensive, but fork itself is not a full memcpy.

**Q7. `fork` fails with `EAGAIN`. What happened? (not in old guide)**  
Resource limit: too many processes (`RLIMIT_NPROC`) or temporary kernel memory pressure. Shell should `perror` and not assume a child exists.

**Q8. Parent vs child: who runs first after fork? (not in old guide)**  
Undefined. Scheduler decides. Never assume order. That is why you cannot “print then wait” without synchronization if both touch the same resource incorrectly — for a shell, you just wait in the parent for FG jobs.

### File descriptors and redirection

**Q9. What is a file descriptor?**  
Index into the process FD table, pointing at a kernel file description (file, pipe, socket, tty).

**Q10. Walk through `ls > out.txt` in the child.**  
`open("out.txt", O_WRONLY|O_CREAT|O_TRUNC, 0644)` → `dup2(fd, STDOUT_FILENO)` → `close(fd)` → `execvp("ls", ...)`. `ls` thinks it writes to stdout.

**Q11. Why close the original fd after `dup2`?**  
Otherwise you leak an FD. `ls` would inherit an extra open file. In pipes, leaking the *write* end causes hangs.

**Q12. `>` vs `>>` flags?**  
`>`: `O_WRONLY | O_CREAT | O_TRUNC`. `>>`: `O_WRONLY | O_CREAT | O_APPEND`. Mode `0644` for create.

**Q13. Why apply redirection in the child, not the parent? (not in old guide)**  
Parent is the shell. Remapping parent’s stdout would send the **prompt** into the file. Child remaps, then execs; parent FDs unchanged.

**Q14. Does `exec` close FDs?**  
No, unless `FD_CLOEXEC`. Redirected 0/1/2 stay.

**Q15. stdin/stdout/stderr numbers?**  
0, 1, 2.

### Pipes and IPC

**Q16. Walk through `cmd1 | cmd2`.**  
`pipe()` → two FDs. Fork A: `dup2(write, 1)`, close all pipe FDs, exec cmd1. Fork B: `dup2(read, 0)`, close all, exec cmd2. Parent closes **both** ends. Parent `waitpid` both. If anyone leaves a write end open, cmd2 never gets EOF.

**Q17. Why N−1 pipes for N commands?**  
Each pipe connects one pair of neighbors.

**Q18. Why close pipe FDs in the parent?**  
Parent does not read/write the pipe. Holding the write end → last reader never sees EOF.

**Q19. Pipe vs socket vs shared memory? (not in old guide)**  
Pipe: unidirectional, related processes (typically), kernel buffer. Socket: bidirectional, can be networked. Shared memory: fastest, need extra sync. Shells use pipes because they match `stdout → stdin`.

**Q20. What if the writer fills the pipe and the reader is slow?**  
Writer **blocks** on `write` until the reader consumes. Kernel pipe capacity is finite (often 64 KiB).

**Q21. Pipeline exit status?**  
ShellX / bash default: **last command**. `true | false` → status 1.

### Built-ins

**Q22. Why must `cd` be a builtin?**  
`chdir` in a child does not change the shell’s cwd.

**Q23. Could `pwd` be `/bin/pwd`?**  
Yes; it only *prints* cwd. Builtin is simpler and matches `cd`.

**Q24. Why is `exit` a builtin?**  
It must terminate **this** process and optionally signal jobs. An external `exit` would only exit the child.

**Q25. `cd` with no arguments?**  
ShellX: `chdir($HOME)` via `getenv("HOME")`.

### Wait, zombies, jobs

**Q26. What is a zombie?**  
Exited process whose status was not `wait`ed. Occupies a slot in the process table. Parent *must* reap.

**Q27. Blocking vs `WNOHANG` wait?**  
Block: FG command, shell waits. `WNOHANG`: poll BG jobs without stalling the prompt.

**Q28. How does ShellX reap BG jobs? (differs from naive tutorials)**  
`SIGCHLD` handler **only sets** `g_sigchld_pending`. REPL calls `reapFinishedJobs()` → `waitpid(pid, WNOHANG)` per job. **No `waitpid` in the handler.** Avoids racing the FG `waitpid` for the same PID.

**Q29. Why not reap in the `SIGCHLD` handler?**  
`waitpid` in a handler can steal the FG child’s status; handler + `waitpid` in executor both try to reap. Also, printing/STL in a handler is unsafe.

**Q30. One `SIGCHLD` for many children? (not in old guide)**  
Signals can **coalesce**. Always loop until `waitpid` says nothing left. ShellX loops the **job list**, not `waitpid(-1)` in the handler.

**Q31. What is an orphan? (not in old guide)**  
Parent died first. Child is reparented to **init (PID 1)** / systemd, which reaps it. ShellX `exit` sends `SIGTERM` to BG jobs and does **not** wait; init cleans up.

**Q32. `WIFEXITED` vs `WIFSIGNALED`?**  
Normal `_exit(code)` vs killed by signal. ShellX maps signal death to `128 + sig`.

**Q33. Why retry `waitpid` on `EINTR`?**  
A signal (e.g. `SIGCHLD` from a *different* child) can interrupt a blocking wait for the FG child. Retry until you reap the intended PID.

### Signals and Ctrl+C

**Q34. What happens on Ctrl+C by default?**  
Kernel sends `SIGINT` to the **foreground process group** of the terminal. Default action: **terminate**.

**Q35. Why install a `SIGINT` handler in the shell?**  
Otherwise Ctrl+C kills the **shell**. Handler sets a flag; shell stays alive.

**Q36. Why restore `SIG_DFL` for `SIGINT` in the child before exec?**  
Child **inherits** the shell’s handler. `ls` would then ignore Ctrl+C or only set a flag. Default restores “kill this process on Ctrl+C”.

**Q37. Known gap: BG jobs vs Ctrl+C.**  
No `setpgid` / job control. BG children stay in the shell’s FG group, so **terminal SIGINT can hit them too**. Say this unprompted — it shows you understand process groups.

**Q38. `sigaction` vs `signal()`?**  
`signal()` is poorly specified across Unixes. `sigaction` is portable and supports `SA_RESTART`, `SA_NOCLDSTOP`.

**Q39. What is `SA_NOCLDSTOP`?**  
Do not generate `SIGCHLD` when a child **stops** (Ctrl+Z / `SIGSTOP`), only when it **exits**. ShellX has no job-control stop/continue.

**Q40. What is async-signal-safe? (not in old guide)**  
Functions POSIX allows in handlers (`write`, `_exit`, set `sig_atomic_t`). Not: `printf`, `malloc`, `new`, `std::vector`.

**Q41. Why `volatile sig_atomic_t`?**  
`sig_atomic_t`: atomic wrt signals. `volatile`: compiler must not cache the flag in a register across the loop.

### Shell design / ShellX-specific

**Q42. What is a REPL?**  
Loop: prompt, read, parse, execute, repeat. EOF (`Ctrl+D`) → `getline` fails → ShellX `killAllJobs` and exit.

**Q43. Why parse into `Command` / `Pipeline` instead of exec’ing the raw string? (not in old guide)**  
Need structured argv, redirections, pipe stages, `&`. The kernel `exec` API is argv, not a bash line.

**Q44. Why is `&` on `Pipeline`, not `Command`?**  
`&` applies to the **whole line**. A per-command flag could disagree with the pipeline flag.

**Q45. Why reject `cmd1 | cmd2 &`?**  
`Job` stores **one PID**. A pipeline is N PIDs. Deferred until real job control.

**Q46. 126 vs 127?**  
127 not found; 126 not executable. Matches bash.

**Q47. Why `_exit` after failed exec?**  
Avoid flushing inherited stdio buffers (`exit` would).

**Q48. Why install signals *before* the first fork?**  
A BG child could die before the handler exists → missed `SIGCHLD` → zombie until something waits.

**Q49. Does ShellX implement `fg` / `bg` / process groups?**  
No. `jobs` only lists tracked BG PIDs.

**Q50. How would you add process groups? (not in old guide)**  
`setpgid` in child after fork; `tcsetpgrp` to give the tty to the FG group; shell in its own group; Ctrl+C goes only to FG group. Then BG jobs survive Ctrl+C. This is “real job control.”

### C++, Linux, build

**Q51. Why C++20 + POSIX, not a library like Boost.Process?**  
The point is to **use syscalls**. Libraries hide the interview-relevant parts.

**Q52. Why Linux not Windows? (not in old guide)**  
`fork`/`exec`/`pipe`/`signals` are POSIX. Windows is `CreateProcess`. WSL is Linux.

**Q53. What does CMake do here?**  
Compiles listed `.cpp` files, `-I include`, C++20, `-Wall -Wextra -Wpedantic`.

**Q54. AddressSanitizer — why mention it? (not in old guide)**  
Catches use-after-free / leaks in C++. Does **not** catch “forgot to close pipe FD” hangs. Use `lsof`, `strace` for FD bugs.

**Q55. How do you debug a hung pipeline?**  
`strace -f` (follow forks), `lsof -p PID` (who holds the pipe), draw FD tables. Almost always an unclosed write end.

### Comparison / CS theory they mix in

**Q56. Thread vs process for running `ls`? (not in old guide)**  
`exec` in a thread would replace the **whole process** (all threads). Shells use **processes**. Threads share cwd — `chdir` in a thread would change the shell, which is why `cd` is a function call in the same process, not a thread+exec.

**Q57. Virtual memory: why can two processes have the “same” pointer after fork? (not in old guide)**  
Virtual addresses copied; physical pages COW. Each has its own page tables.

**Q58. What is `PATH`?**  
Colon-separated list of directories. `execvp` walks it for `ls` without `/`.

**Q59. Absolute vs relative exec? (not in old guide)**  
If argv[0] contains `/`, `execvp` does **not** search PATH (`./a.out`, `/usr/bin/ls`).

**Q60. Kernel vs user space for `write(1, ...)`? (not in old guide)**  
`write` is a syscall: trap into kernel, copy to file/pipe/tty, return. The shell’s C++ is user space.

---

## How to study Phase 0 (Specialist → interviews)

1. Draw **one** picture: process tree for `ls | wc` with every FD in parent and both children **before and after** close.
2. Recite Q2, Q16, Q22, Q26, Q28, Q36, Q37 without notes.
3. On Linux: `man 2 fork`, `man 2 execve`, `man 2 pipe`, `man 2 dup2`, `man 2 waitpid`, `man 7 signal`, `man 7 pipe`.
4. Optional lab: tiny C program that only `fork`+`wait` and print PIDs — feel two processes.

When this is solid, we go **Phase 1**: `shellx.hpp`, REPL, `executeSingle`, multi-stage pipes — line-level, plus SDE questions on that code.

Say **go to Phase 1** when you want that.
