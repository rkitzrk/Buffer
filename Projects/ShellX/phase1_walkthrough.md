# Phase 1 — The 4 Most Critical Files

> These 4 files contain the entire soul of the shell. Every other file supports them.
> Read in this order: signals → executor → pipeline → main

**File priority ranking:**
1. `signals.cpp` — sets the safety contract everything else depends on
2. `executor.cpp` — the fork/exec/wait core; most interview-questioned
3. `pipeline.cpp` — hardest to write correctly; fd discipline is the crux
4. `main.cpp` — the REPL glue; shows how all pieces integrate

---

## File 1: `src/signals.cpp` (40 lines — most impact per line)

### Full Annotated Source

```cpp
#include "signals.hpp"
#include <cstring>

// --- Global signal flags ---
volatile sig_atomic_t g_sigchld_pending = 0;
volatile sig_atomic_t g_sigint_received = 0;
```

**Why `volatile sig_atomic_t` and not just `int`?**
- `sig_atomic_t` — platform-guaranteed to be read/written atomically. On a 64-bit machine,
  a plain `int` write might not be atomic (torn write). The signal handler sets this and
  the main loop reads it — they are in different "execution contexts" with no lock.
- `volatile` — tells the compiler: "don't cache this in a register; re-read from memory
  every time." Without `volatile`, the optimizer might hoist the check out of the REPL
  loop entirely because it looks like a variable that "never changes" from the main thread's
  perspective. This would cause the main loop to never see the flag update.
- Both are REQUIRED together. One without the other is insufficient.

---

```cpp
static void sigchld_handler(int /*signo*/) {
    g_sigchld_pending = 1;
}
```

**Why `static`?** Internal linkage — this function is not exported. Only `installSignalHandlers()`
needs it. Prevents name collisions and makes the intent explicit.

**Why `/*signo*/` (commented param)?** The handler signature must match `void handler(int)` —
the `int` is the signal number. We don't need it (we already know it's SIGCHLD), but we must
accept it. Commenting the name suppresses the "unused parameter" warning.

**The key design decision — flag-only handler:**

The alternative would be:
```cpp
// WRONG — do NOT do this
static void sigchld_handler(int) {
    waitpid(-1, &status, WNOHANG);  // DON'T reap here
    // update job list               // DON'T touch STL here
    printf("job done\n");           // DON'T call printf here
}
```

Why is this wrong?
1. `waitpid()` in the handler could reap the **foreground child** that the main code is
   also waiting on → foreground `waitpid()` returns `ECHILD` (child already reaped) → bug
2. `printf` is not async-signal-safe (uses internal locks; handler could deadlock with main)
3. Touching `std::vector<Job>` (STL) in the handler races with the main loop reading it

The flag-only pattern eliminates all three problems. The handler is 1 line and trivially correct.

---

```cpp
void installSignalHandlers() {
    struct sigaction sa;

    std::memset(&sa, 0, sizeof(sa));
```

**Why `memset` to zero?** The `sigaction` struct has several fields. Uninitialized padding bits
could hold garbage values that affect behavior on some kernels. Zero-initialize to be safe —
this is standard practice.

---

```cpp
    sa.sa_handler = sigchld_handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART | SA_NOCLDSTOP;
    sigaction(SIGCHLD, &sa, nullptr);
```

**`sigemptyset(&sa.sa_mask)`** — While the SIGCHLD handler is executing, don't additionally
block any other signals. If we added SIGINT to the mask, Ctrl+C would be silently ignored
during child reaping.

**`SA_RESTART`** — If a blocking syscall (like `read()` or `waitpid()`) is interrupted by
SIGCHLD, automatically restart it instead of returning -1/EINTR. "Belt" — ShellX also has
explicit EINTR retry loops ("suspenders") for documentation clarity.

**`SA_NOCLDSTOP`** — Without this, if you Ctrl+Z a child (SIGTSTP), the parent gets SIGCHLD.
The handler sets `g_sigchld_pending = 1`. The REPL calls `reapFinishedJobs()`. It calls
`waitpid(pid, &status, WNOHANG)` — but the child isn't dead, just stopped. `waitpid` returns 0.
No harm done but it's a spurious wakeup and bad practice. `SA_NOCLDSTOP` prevents this entirely.

**`nullptr` as third arg** — The third arg is `struct sigaction* oldact` — if non-null, the
old handler is saved there. We don't need it.

---

```cpp
    std::memset(&sa, 0, sizeof(sa));
    sa.sa_handler = sigint_handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;
    sigaction(SIGINT, &sa, nullptr);
```

**Why re-memset?** We reuse the `sa` struct. The previous SIGCHLD setup had `SA_NOCLDSTOP`
in `sa_flags`. Forgetting to clear it before the SIGINT setup would have an unintended
`SA_NOCLDSTOP` on SIGINT (meaningless but sloppy). Always re-zero when reusing struct.

**Why is SIGINT's handler different from SIGKILL?** SIGKILL cannot be caught or ignored —
the kernel delivers it unconditionally. SIGINT CAN be caught. We install a custom handler
so the shell itself doesn't die on Ctrl+C, while still letting children die (they get
SIG_DFL restored before exec).

---

### The Timing Problem (Critical Interview Insight)

```
installSignalHandlers()       ← MUST happen before any fork()
        |
      fork() creates background child
        |
    child runs...finishes...kernel sends SIGCHLD
        |
    sigchld_handler() fires → g_sigchld_pending = 1
        |
    REPL loop checks flag → reapFinishedJobs()
```

If handler was installed AFTER a fork, and the child finished between the fork and the
handler installation, the SIGCHLD would use the OLD default handler (which is `SIG_DFL` =
ignore for SIGCHLD on many systems). The child becomes a **zombie** that is never reaped.

---

## File 2: `src/executor.cpp` (122 lines — the fork/exec/wait core)

### The Two-Function Design

```
executePipeline()     ← public entry point; dispatch logic
      |
      ├─ isBuiltin()  → executeBuiltin()   (no fork, parent runs)
      ├─ single cmd   → executeSingle()    (fork+exec+wait)
      └─ multi cmd    → executePipelineChain()  (pipeline.cpp)
```

### executeSingle() — Full Walkthrough

```cpp
static int executeSingle(const Command& cmd, bool background) {
    pid_t pid = fork();

    if (pid == -1) {
        perror("fork");
        return 1;
    }
```

**Why handle `pid == -1` first?** `fork()` fails when:
- System is out of PIDs (`EAGAIN`)
- Process has hit its max process limit (`RLIMIT_NPROC`)
- Not enough memory for page table (`ENOMEM`)

If we don't check, the code falls through. `pid == 0` (child branch) won't match `-1 == 0`,
so we'd fall into parent code. But then `waitpid(-1, ...)` would wait for ANY child — catastrophic.
Always check fork failure first.

---

```cpp
    if (pid == 0) {
        // --- Child process ---

        struct sigaction sa;
        sa.sa_handler = SIG_DFL;
        sigemptyset(&sa.sa_mask);
        sa.sa_flags = 0;
        sigaction(SIGINT, &sa, nullptr);
```

**Why restore SIGINT to SIG_DFL in the child?**

The shell installed a custom SIGINT handler (sigint_handler). After `fork()`, the child inherits
this handler. So if you run `sleep 10` and press Ctrl+C:
- Terminal sends SIGINT to the foreground process group
- Child (sleep) has the shell's custom SIGINT handler → sets a flag and returns → sleep keeps running!

That's wrong. We want `sleep` to die on Ctrl+C.

`SIG_DFL` for SIGINT = "terminate the process" — which is what you expect.

**Note:** After `exec()`, signal handlers are automatically reset to `SIG_DFL` anyway.
But we call `sigaction()` before exec — this is a safety measure for the window between
`fork()` and `exec()` where redirection setup or other code might trigger an unwanted signal.
Also documents intent clearly.

---

```cpp
        if (!applyRedirections(cmd)) {
            _exit(1);
        }
```

**Why redirections before exec?** `exec()` replaces the process image. The new program
(e.g., `ls`) has no knowledge of your redirection intent. You must rewire the FDs *before*
exec so the new program finds them already set up.

**Why `_exit(1)` not `exit(1)`?** The child inherited the parent's stdio buffers.
`exit()` flushes them. If the parent had `printf("shellx> ")` in a buffer (not yet flushed),
the child's `exit()` would flush it — printing the prompt twice. `_exit()` skips all cleanup.

---

```cpp
        std::vector<const char*> argv;
        argv.reserve(cmd.args.size() + 1);
        for (const auto& arg : cmd.args) {
            argv.push_back(arg.c_str());
        }
        argv.push_back(nullptr);   // execvp requires NULL terminator

        execvp(argv[0], const_cast<char* const*>(argv.data()));
```

**Why `NULL` terminator?** `execvp` uses C-style arrays — it doesn't know the array size.
It reads until it hits a `NULL` pointer. Forgetting `nullptr` → undefined behavior (reads garbage).

**Why `execvp` and not `execv`?**
- `execv` needs the full path: `/bin/ls`
- `execvp` searches `PATH` automatically: just `"ls"` works
The 'p' means "path search."

**`const_cast<char* const*>` — why?**
`argv.data()` is `const char**` (vector of `const char*`).
`execvp` signature is `char* const*` (array of pointers-to-char, where pointers are const).
These are technically different types. The cast is necessary and safe — `execvp` doesn't
actually modify the strings.

---

```cpp
        perror(argv[0]);
        if (errno == ENOENT) {
            _exit(127); // command not found
        } else {
            _exit(126); // found but not executable
        }
```

**Why is `errno` still valid after `execvp` fails?**
`execvp` sets `errno` when it fails. Since `execvp` didn't succeed, we're still running
the same process (child) with the same memory, so `errno` is intact and correct.

**Why check errno AFTER perror?** `perror()` uses `errno` internally too. Since `perror`
is not async-signal-safe, it could theoretically change errno... but we're not in a signal
handler here so it's fine. Still, checking after `perror` is correct (perror doesn't change errno).

---

```cpp
    // --- Parent process ---

    if (background) {
        std::ostringstream oss;
        for (size_t j = 0; j < cmd.args.size(); ++j) {
            if (j > 0) oss << " ";
            oss << cmd.args[j];
        }
        int job_num = addJob(pid, oss.str());
        std::cout << "[" << job_num << "] " << pid << "\n";
        return 0;
    }
```

**Background path:** Parent does NOT call `waitpid()`. It records the child's PID in the job
list and returns immediately. The child runs independently. When it finishes, `SIGCHLD` fires,
the flag gets set, and `reapFinishedJobs()` handles it next REPL iteration.

---

```cpp
    int status;
    pid_t result;
    do {
        result = waitpid(pid, &status, 0);
    } while (result == -1 && errno == EINTR);
```

**The EINTR retry loop — the most commonly asked question about this function.**

Scenario:
1. Shell forks foreground child (e.g., `sleep 10`)
2. Shell calls `waitpid(sleep_pid, &status, 0)` — blocks
3. A background child from a previous command finishes → kernel sends SIGCHLD
4. Signal handler runs (sets flag, returns immediately)
5. `waitpid()` is interrupted → returns -1 with `errno == EINTR`
6. Without retry: parent sees error, returns 1 — **sleep is still running but shell thinks it's done!**
7. With retry: loop continues, `waitpid()` blocks again, correctly waits for sleep to finish

`SA_RESTART` would restart most syscalls automatically, but the explicit loop is the belt-AND-suspenders approach.

---

```cpp
    if (WIFEXITED(status)) {
        return WEXITSTATUS(status);
    } else if (WIFSIGNALED(status)) {
        return 128 + WTERMSIG(status);
    }
```

**Why `128 + signal_number`?** This is the universal shell convention:
- Exit codes 0–127: normal exit status
- Exit codes 128+N: process was killed by signal N

`kill -l` shows signal numbers: SIGINT=2, SIGKILL=9, SIGSEGV=11...
So `Ctrl+C` killing a process → exit code 130 (128+2).
Bash uses this, ZSH uses this, ShellX uses this.

---

### executePipeline() — The Dispatch Logic

```cpp
int executePipeline(const Pipeline& pipeline) {
    if (pipeline.commands.size() == 1) {
        const Command& cmd = pipeline.commands[0];

        if (isBuiltin(cmd.args[0])) {
            return executeBuiltin(cmd);    // no fork — runs in parent
        }
        return executeSingle(cmd, pipeline.background);
    }

    if (pipeline.background) {
        std::cerr << "shellx: backgrounded pipelines not supported\n";
        return 1;
    }

    return executePipelineChain(pipeline);
}
```

**Dispatch order matters:**
1. Check builtin FIRST — if we forked for `cd`, the parent's cwd wouldn't change
2. Single external command → executeSingle
3. Multi-command → executePipelineChain

---

## File 3: `src/pipeline.cpp` (137 lines — the hardest to get right)

### The Central Data Structure

```cpp
std::vector<int> pipe_fds(2 * (n - 1));
```

For N=3 commands (`ls | sort | head`), N-1=2 pipes, 4 fds total:
```
pipe_fds = [p0_read, p0_write, p1_read, p1_write]
            [  0   ] [  1   ]  [  2   ] [  3   ]
index:         0        1         2        3
```

Pipe i: `pipe_fds[2*i]` = read end, `pipe_fds[2*i+1]` = write end.

```
cmd[0]──write─→[p0_write]──pipe──[p0_read]──read─→cmd[1]──write─→[p1_write]──pipe──[p1_read]──read─→cmd[2]
```

---

### The Pipe Creation Loop

```cpp
for (int i = 0; i < n - 1; ++i) {
    if (pipe(&pipe_fds[2 * i]) == -1) {
        perror("pipe");
        for (int j = 0; j < 2 * i; ++j) {
            close(pipe_fds[j]);
        }
        return 1;
    }
}
```

**Error handling:** If `pipe()` fails (e.g., fd table full) at iteration i=1, we already
created pipe[0]. We must close those fds before returning — otherwise fd leak.
Clean up "what we successfully created so far" pattern.

---

### The Fork Loop — Read This Carefully

```cpp
for (int i = 0; i < n; ++i) {
    pid_t pid = fork();
    if (pid == 0) {
        // child for command i
```

**Key insight:** All N forks happen in the PARENT process's loop. After `fork()`:
- Child goes into the `if (pid == 0)` block and eventually calls `exec()` or `_exit()`
- Parent continues the loop and forks the next child

So the parent loop runs N times total. Child processes do NOT continue the for loop —
they exit (via exec or _exit) before reaching the next iteration.

---

```cpp
        // Wire stdin from previous pipe (if not first command)
        if (i > 0) {
            if (dup2(pipe_fds[2 * (i - 1)], STDIN_FILENO) == -1) {
                perror("dup2"); _exit(1);
            }
        }

        // Wire stdout to next pipe (if not last command)
        if (i < n - 1) {
            if (dup2(pipe_fds[2 * i + 1], STDOUT_FILENO) == -1) {
                perror("dup2"); _exit(1);
            }
        }
```

**Tracing for 3 commands (i=0,1,2), 2 pipes:**

| Command i | stdin comes from | stdout goes to |
|-----------|-----------------|----------------|
| 0 (ls)    | unchanged (terminal) | `pipe_fds[1]` = p0_write |
| 1 (sort)  | `pipe_fds[0]` = p0_read | `pipe_fds[3]` = p1_write |
| 2 (head)  | `pipe_fds[2]` = p1_read | unchanged (terminal) |

---

```cpp
        // Close ALL pipe fds in this child (both ends of every pipe)
        for (int j = 0; j < 2 * (n - 1); ++j) {
            close(pipe_fds[j]);
        }
```

**THIS IS THE MOST CRITICAL STEP AND THE #1 SOURCE OF BUGS.**

Why close ALL fds (not just the ones you don't use)?

After `dup2()`, the child has:
- fd 0 (stdin) → pointing to p0_read (via dup2)
- fd 1 (stdout) → pointing to p0_write (via dup2)
- fd `pipe_fds[0]` → also pointing to p0_read (the original fd)
- fd `pipe_fds[1]` → also pointing to p0_write (the original fd)
- fd `pipe_fds[2]` → pointing to p1_read (completely unused)
- fd `pipe_fds[3]` → pointing to p1_write (completely unused)

If we only close "unused" ones (3 and 4), we still have the original pipe fds (0 and 1 in
the pipe_fds array) open. The KEY issue: `sort` (cmd 1) holds a reference to p0_write
(the write end of pipe 0). When `ls` finishes and closes its p0_write, the pipe still has
a writer (`sort`'s inherited copy). `sort` is reading from p0_read but also holds p0_write
open — so it NEVER sees EOF on its own stdin because it itself is a writer!

**Closing ALL pipe_fds makes this clean.** The only remaining references are fd 0 and fd 1
which were created by dup2 and are the intended ones.

---

```cpp
// Parent: close ALL pipe fds immediately after all forks
for (int j = 0; j < 2 * (n - 1); ++j) {
    close(pipe_fds[j]);
}
```

**"Immediately after all forks"** — The timing matters. If the parent closes after each fork:
- Fork child 0, close pipes, then fork child 1 — child 1 might not see all pipes open
The correct sequence is: fork ALL children first (so they all inherit the full pipe set),
THEN close all pipes in the parent.

---

```cpp
// Wait for all children. Retry on EINTR.
for (int i = 0; i < n; ++i) {
    int status;
    pid_t result;
    do {
        result = waitpid(child_pids[i], &status, 0);
    } while (result == -1 && errno == EINTR);

    if (i == n - 1) {    // only last command's status matters
        if (WIFEXITED(status))   pipeline_status = WEXITSTATUS(status);
        else if (WIFSIGNALED(status)) pipeline_status = 128 + WTERMSIG(status);
    }
}
```

**Why wait for ALL children even though we only care about the last?**
We need to reap all children to avoid zombies. We wait for them in order
(0, 1, 2...) which is fine — even if child 2 finishes before child 1, `waitpid(child_pids[1])`
will still block until child 1 finishes. The pipeline naturally serializes through the pipe.

**Using `waitpid(specific_pid)` not `waitpid(-1)`** — we know exactly which children we
created. Using -1 could accidentally reap a background job's child.

---

## File 4: `src/main.cpp` (89 lines — the integration glue)

### The REPL Loop Structure

```
[start of loop]
    ↓
[check SIGCHLD flag → reapFinishedJobs]   ← at TOP, before prompt
    ↓
[check SIGINT flag → print newline]
    ↓
[print prompt]
    ↓
[getline]                                  ← blocks here waiting for input
    ↓
[validate input length, emptiness]
    ↓
[parse → Pipeline]
    ↓
[executePipeline]                          ← may block (foreground) or return immediately (bg)
    ↓
[check SIGCHLD flag → reapFinishedJobs]   ← at BOTTOM, after execution
    ↓
[back to top]
```

**Why check SIGCHLD at BOTH top and bottom?**
- TOP: catches jobs that finished while the shell was blocked in `getline()` (user was typing)
- BOTTOM: catches jobs that finished during foreground command execution

---

```cpp
installSignalHandlers();
```

**First line of main — why?** If we did ANY I/O or processing before this and a
background job somehow ran (theoretically impossible here, but principle matters),
we'd miss SIGCHLD. Good practice: install handlers before anything else.

---

```cpp
if (g_sigchld_pending) {
    g_sigchld_pending = 0;    // reset FIRST
    reapFinishedJobs();       // then reap
}
```

**Why reset the flag BEFORE reaping (not after)?**

Race condition scenario if you reset AFTER:
1. `g_sigchld_pending == 1`
2. `reapFinishedJobs()` starts reaping
3. During reaping, ANOTHER child finishes → SIGCHLD → handler sets flag to 1 again
4. `reapFinishedJobs()` finishes
5. We reset `g_sigchld_pending = 0` — BUT the new SIGCHLD was missed!
6. Next REPL iteration, the flag is 0, we don't reap the second child → zombie

By resetting FIRST, any new SIGCHLD that arrives during `reapFinishedJobs()` will leave
the flag as 1, and we'll process it next iteration. No signal is ever missed.

---

```cpp
if (!std::getline(std::cin, line)) {
    // EOF (Ctrl+D) — exit gracefully
    std::cout << "\n";
    killAllJobs();
    break;
}
```

**`getline` returns false on EOF (Ctrl+D) or stream error.** Without this check, the loop
would spin infinitely on EOF — printing the prompt forever with an empty string.

**`killAllJobs()` before breaking** — sends SIGTERM to all background jobs. The shell process
then exits, and any surviving children are reparented to init. No explicit reap — by design.

---

```cpp
if (line.size() > MAX_INPUT_LENGTH) {
    std::cerr << "shellx: input too long\n";
    continue;
}
```

**`std::getline` has no length limit** — it will read however many bytes the user provides.
A malicious or accidental 1GB input would allocate 1GB of memory. The check after
`getline` is the enforcement point — the constant `MAX_INPUT_LENGTH = 4096` does nothing
by itself.

---

```cpp
last_status = executePipeline(pipeline);

if (last_status == -1) {    // exit sentinel
    break;
}
```

**Sentinel value -1:** `executeBuiltin("exit")` returns -1. This propagates up through
`executePipeline` → main loop → breaks → shell exits. It's a simple protocol for the
built-in to signal "please terminate the REPL."

Normal exit codes are 0–255. -1 as a sentinel is unambiguous.

---

```cpp
return last_status >= 0 ? last_status : 0;
```

**Why `>= 0` check?** After `break`, `last_status` could be:
- A normal exit code (0–255) from the last command → return it
- -1 (exit sentinel) → return 0 (clean exit)

The shell's own exit code becomes the return value of main, which becomes the
shell process's exit code — visible to whoever ran the shell.

---

## Phase 1 — Interview Questions & Answers

---

**Q1. Why must signal handlers be async-signal-safe? Name 3 functions that are NOT safe.**

Signal handlers can interrupt any point in program execution — including inside library
functions that use internal locks or shared state. Calling the same non-reentrant function
from both the handler and the interrupted code causes deadlock or data corruption.

Not safe:
1. `printf()` — uses stdio locks and buffered I/O
2. `malloc()` — uses heap locks; interrupted malloc + handler malloc = heap corruption
3. Any STL container operation (push_back, etc.) — may call malloc internally

---

**Q2. What is the SIGCHLD race condition that this design prevents?**

If the SIGCHLD handler called `waitpid(-1, &status, WNOHANG)`:
- Shell is blocked in `waitpid(fg_pid, &status, 0)` for foreground job
- Background job finishes → SIGCHLD fires
- Handler calls `waitpid(-1, ...)` → accidentally reaps the **foreground child**
- Foreground `waitpid(fg_pid, ...)` returns -1 with ECHILD (child already gone) → bug

ShellX's flag-only handler prevents this. Reaping only happens in the main loop
where the code knows whether a foreground job is active.

---

**Q3. Trace exactly what happens when you type `ls > out.txt` in ShellX.**

1. `getline()` reads `"ls > out.txt"`
2. `parse()` tokenizes: `["ls", ">", "out.txt"]`
   - Recognizes `>` → sets `cmd.output_file = "out.txt"`, `cmd.args = ["ls"]`
3. `executePipeline()` → single command, not builtin → `executeSingle(cmd, false)`
4. `fork()` → creates child
5. **Child:**
   - Restores SIGINT to SIG_DFL
   - `applyRedirections(cmd)`:
     - `open("out.txt", O_WRONLY|O_CREAT|O_TRUNC, 0644)` → fd=3
     - `dup2(3, 1)` → fd 1 now points to out.txt
     - `close(3)` → close original
   - `execvp("ls", ["ls", nullptr])`
   - `ls` writes its output to fd 1 → goes to out.txt
6. **Parent:** `waitpid(child_pid, &status, 0)` — blocks until ls finishes

---

**Q4. Why does `pipe()` need to be called BEFORE `fork()`?**

`pipe()` creates two connected fds in the kernel. After `fork()`, the child inherits
copies of both fds. This is the mechanism that connects parent and child through the pipe.

If you called `pipe()` AFTER fork, each process would get its own independent pipe with
no connection — they couldn't communicate.

---

**Q5. For `ls | sort | head`, explain the fd state in each child after all dup2s and close-alls.**

```
pipe_fds = [p0r, p0w, p1r, p1w]   (r=read, w=write)

Child 0 (ls):
  After dup2: fd 1 → p0w (stdout to pipe 0 write)
  After close-all pipe_fds: fd 0 (stdin) = terminal, fd 1 = p0w, no other pipe fds
  exec("ls"): ls writes to stdout (fd 1) → goes into pipe 0

Child 1 (sort):
  After dup2: fd 0 → p0r, fd 1 → p1w
  After close-all pipe_fds: fd 0 = p0r, fd 1 = p1w, no other pipe fds
  exec("sort"): sort reads from stdin (fd 0) = pipe 0, writes to stdout (fd 1) = pipe 1

Child 2 (head):
  After dup2: fd 0 → p1r (stdin from pipe 1 read)
  After close-all pipe_fds: fd 0 = p1r, fd 1 (stdout) = terminal, no other pipe fds
  exec("head"): head reads from stdin (fd 0) = pipe 1, writes to stdout = terminal

Parent: closes all pipe_fds, then waitpid for all 3 children
```

---

**Q6. What happens if the parent forgets to close the write end of a pipe?**

The reader (`grep`, `sort`, etc.) reads until it sees EOF. EOF is signaled when
ALL write-end fds are closed. If the parent holds the write end open, the reference
count on the write end is ≥ 1. The kernel never delivers EOF to the reader.
The reader blocks on `read()` forever → pipeline hangs.

---

**Q7. Why do we use `waitpid(specific_pid)` instead of `waitpid(-1)` for pipeline children?**

`waitpid(-1)` waits for ANY child — including background jobs.
If a background job finishes while we're reaping pipeline children with `waitpid(-1)`,
we might accidentally reap the background job before `reapFinishedJobs()` gets to it.
The job list still shows it as "running" — it's now a phantom entry.

Using specific PIDs ensures each `waitpid()` call reaps exactly the child we forked.

---

**Q8. What is the exit code convention for signal-killed processes? What is 130?**

Convention: `128 + signal_number`.
SIGINT = 2 → exit code 130.
SIGKILL = 9 → exit code 137.
SIGSEGV = 11 → exit code 139.

This lets calling scripts detect whether a process was killed by a signal:
```bash
./program
echo $?    # 139 → process segfaulted (killed by SIGSEGV=11)
```

---

**Q9. Why is `g_sigchld_pending` reset to 0 BEFORE `reapFinishedJobs()`, not after?**

If reset after: a SIGCHLD arriving during `reapFinishedJobs()` would set the flag to 1.
After reaping, we reset to 0 — the new SIGCHLD is silently lost. That child's death is
never noticed → zombie.

If reset before: a SIGCHLD during reaping re-sets the flag to 1. Next loop iteration,
we reap again and catch it. No signal is ever missed.

---

**Q10. Implement a minimal shell REPL (fork/exec/wait only, no signals) in ~30 lines of C.**

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    char line[1024];
    while (1) {
        printf("sh> ");
        fflush(stdout);
        if (!fgets(line, sizeof(line), stdin)) break;
        line[strcspn(line, "\n")] = 0;    // strip newline

        // tokenize by space
        char *argv[64];
        int argc = 0;
        char *tok = strtok(line, " ");
        while (tok && argc < 63) {
            argv[argc++] = tok;
            tok = strtok(NULL, " ");
        }
        argv[argc] = NULL;
        if (argc == 0) continue;

        pid_t pid = fork();
        if (pid == 0) {
            execvp(argv[0], argv);
            perror(argv[0]);
            _exit(127);
        }
        int status;
        waitpid(pid, &status, 0);
    }
    return 0;
}
```

This is the "interviewer asks you to code a shell on whiteboard" answer.

---

**Q11. What would happen if you called `exit()` instead of `_exit()` in the child after exec fails?**

`exit()` flushes all stdio buffers. The child inherited the parent shell's stdout buffer.
If the shell had a buffered prompt `"shellx> "` not yet written to the terminal,
`exit()` in the child flushes it — the prompt appears twice (child flushes it, then
parent flushes it again when it prints the next prompt). Subtle visual bug.
More critically, in C programs, `atexit()` handlers would also run — potentially
dangerous if they free memory, close files, etc. that the child shouldn't touch.

---

**Q12. What is the difference between `WIFEXITED` and `WIFSIGNALED`? Can both be true?**

- `WIFEXITED(status)` — true if child called `exit()`, `_exit()`, or returned from `main()`
- `WIFSIGNALED(status)` — true if child was terminated by a signal (Ctrl+C, SIGSEGV, etc.)

They are **mutually exclusive** — a process exits either normally or due to a signal.
Only one of these macros will be true for any given status value.

If neither is true, the child was stopped (WIFSTOPPED) — not terminated.
Always handle both cases to avoid returning an uninitialized status.

---

*Confirm when ready → I'll send Phase 2 (parser, jobs, redirection — the supporting layer).*
