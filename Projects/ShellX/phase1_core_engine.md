# ShellX Study Guide — Phase 1: The Core Engine

> Files: [`shellx.hpp`](file:///home/kumaresh/Desktop/Dev/Personal/ShellX/include/shellx.hpp) · [`main.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/ShellX/src/main.cpp) · [`executor.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/ShellX/src/executor.cpp) · [`pipeline.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/ShellX/src/pipeline.cpp)

---

## 🗺️ What Phase 1 Covers

These 4 files are the **entire beating heart** of the shell. Everything else is in service of what happens here.

```
shellx.hpp    → Defines the vocabulary (what is a Command? a Pipeline?)
main.cpp      → The REPL loop — the shell's skeleton
executor.cpp  → Decides HOW to run a pipeline (builtin? single? multi?)
pipeline.cpp  → Implements multi-command piped execution
```

**Read order**: `shellx.hpp` → `main.cpp` → `executor.cpp` → `pipeline.cpp`

---

## ─── FILE 1: `shellx.hpp` ───

### Full Annotated Code

```cpp
#ifndef SHELLX_HPP          // Include guard — prevents double-inclusion
#define SHELLX_HPP          // if this file is #included by multiple .cpp files

#include <string>
#include <vector>

// POSIX headers — these are the raw OS interfaces
#include <unistd.h>         // fork, execvp, pipe, dup2, close, getpid, chdir, getcwd, _exit
#include <sys/wait.h>       // waitpid, WIFEXITED, WEXITSTATUS, WIFSIGNALED, WTERMSIG, WNOHANG
#include <sys/types.h>      // pid_t, mode_t
#include <fcntl.h>          // open, O_RDONLY, O_WRONLY, O_CREAT, O_TRUNC, O_APPEND
#include <signal.h>         // sigaction, sigemptyset, SIGCHLD, SIGINT, SIGTERM, SA_RESTART
#include <cerrno>           // errno, ENOENT, EACCES, EINTR
#include <cstring>          // memset, strerror
#include <cstdio>           // perror
#include <cstdlib>          // getenv, exit

// --- Resource limits ---
constexpr int MAX_PIPELINE_LENGTH = 16;   // max commands in a pipe chain
constexpr size_t MAX_INPUT_LENGTH = 4096; // max chars per line (also a common OS limit)
constexpr int MAX_BACKGROUND_JOBS = 64;   // max concurrent background jobs

// --- Command representation ---
struct Command {
    std::vector<std::string> args;  // args[0] = program name, rest = arguments
    std::string input_file;         // "" if no '<', otherwise the filename
    std::string output_file;        // "" if no '>' or '>>', otherwise the filename
    bool append_mode = false;       // true for '>>', false for '>'
    // NOTE: no background flag here — '&' applies to the WHOLE pipeline,
    // not a single command. Pipeline::background is the single source of truth.
};

struct Pipeline {
    std::vector<Command> commands;  // 1+ commands chained by '|'
    bool background = false;        // trailing '&' on the whole pipeline
};

#endif // SHELLX_HPP
```

### Deep Analysis

#### The Two Structs Are the Shared Language of the Entire Codebase

```
Parser creates them  →  Executor consumes them  →  Pipeline module uses Commands inside them
```

Every other module is either **building** or **reading** these two structs. They are the lingua franca.

#### `Command` Design

| Field | Type | Purpose |
|---|---|---|
| `args` | `vector<string>` | `args[0]` = program, `args[1..]` = arguments. Directly maps to `argv` for `execvp` |
| `input_file` | `string` | Empty string = no redirection (idiomatic sentinel, avoids optional<>) |
| `output_file` | `string` | Empty string = no redirection |
| `append_mode` | `bool` | Disambiguates `>` (truncate) vs `>>` (append) |

**Critical design note**: There is NO `background` field in `Command`. Background applies to the entire pipeline, not individual commands. Putting it in `Command` would create the possibility of inconsistent state (cmd1 background=true, cmd2 background=false in same pipeline — what does that mean?). This is a **Single Source of Truth** design principle.

#### `Pipeline` Design

A pipeline can have 1 command (e.g., `ls`) or N commands (e.g., `ls | grep | wc`). The executor checks `commands.size()` to decide the execution strategy.

#### Include Guards vs `#pragma once`

```cpp
#ifndef SHELLX_HPP   // Traditional include guard
#define SHELLX_HPP
// ...
#endif
```
vs
```cpp
#pragma once         // Compiler extension — not ISO C++ standard but universally supported
```

ShellX uses the traditional form for correctness and portability. Both prevent a header being processed twice during compilation.

#### Why `constexpr` over `#define`?

```cpp
// BAD (macro):
#define MAX_PIPELINE_LENGTH 16  // No type. No scope. Debugger can't see it. Substituted blindly.

// GOOD (constexpr):
constexpr int MAX_PIPELINE_LENGTH = 16;  // Typed. Scoped. In debug symbols. Evaluated at compile time.
```

---

## ─── FILE 2: `main.cpp` ───

### Full Annotated Code

```cpp
#include "shellx.hpp"
#include "parser.hpp"
#include "executor.hpp"
#include "signals.hpp"
#include "jobs.hpp"
#include <iostream>
#include <string>

int main() {
    // 1. Install signal handlers FIRST — before the loop, before any fork().
    //    If a background child finishes before handlers are installed, we miss SIGCHLD.
    //    The zombie would persist unnoticed.
    installSignalHandlers();

    std::string line;
    int last_status = 0;

    while (true) {
        // 2. Check signal flags at the TOP of each iteration.
        //    Signal handlers only set flags (async-signal-safe).
        //    Actual work (waitpid, printf) is done here, safely.

        // SIGCHLD: a background child finished → reap it
        if (g_sigchld_pending) {
            g_sigchld_pending = 0;       // Clear flag first (before reapFinishedJobs)
            reapFinishedJobs();           // Non-blocking waitpid for all background jobs
        }

        // SIGINT: user pressed Ctrl+C while no foreground child → reprint prompt
        if (g_sigint_received) {
            g_sigint_received = 0;
            std::cout << "\n";           // Move past the '^C' the terminal echoed
        }

        // 3. Print prompt (unbuffered flush with std::flush)
        std::cout << "shellx> " << std::flush;

        // 4. READ: blocks here until user presses Enter (or EOF via Ctrl+D)
        if (!std::getline(std::cin, line)) {
            // EOF (Ctrl+D) — graceful exit
            std::cout << "\n";
            killAllJobs();   // SIGTERM to all background jobs (no wait — shell is done)
            break;
        }

        // 5. Enforce max input length (getline itself has no cap)
        if (line.size() > MAX_INPUT_LENGTH) {
            std::cerr << "shellx: input too long (max " << MAX_INPUT_LENGTH << " characters)\n";
            continue;
        }

        // 6. Skip empty input
        if (line.empty()) continue;

        // 7. Skip whitespace-only input (e.g., user pressed Space then Enter)
        bool all_whitespace = true;
        for (char c : line) {
            if (!std::isspace(static_cast<unsigned char>(c))) {
                all_whitespace = false;
                break;
            }
        }
        if (all_whitespace) continue;

        // 8. PARSE: tokenize → build Pipeline struct
        Pipeline pipeline = parse(line);
        if (pipeline.commands.empty()) {
            // Parse returned empty = error (message already printed by parser)
            continue;
        }

        // 9. EXECUTE: fork/exec/wait (or builtin in-process)
        last_status = executePipeline(pipeline);

        // 10. Exit sentinel: executePipeline returns -1 when user typed "exit"
        if (last_status == -1) break;

        // 11. Reap background jobs that finished DURING the foreground command's execution
        if (g_sigchld_pending) {
            g_sigchld_pending = 0;
            reapFinishedJobs();
        }
    }

    // Return 0 for normal exit (or last_status if it was a real exit code)
    return last_status >= 0 ? last_status : 0;
}
```

### Deep Analysis

#### The REPL Loop Architecture

```
┌────────────────────────────────────────┐
│              while(true)               │
│                                        │
│  [Top of loop]                         │
│   ↓ Check g_sigchld_pending            │
│   ↓ Check g_sigint_received            │
│                                        │
│  [Prompt + Read]                       │
│   ↓ print "shellx> "                  │
│   ↓ getline() ← BLOCKS HERE           │
│                                        │
│  [Validate]                            │
│   ↓ EOF check  → killAllJobs() + break │
│   ↓ length check                       │
│   ↓ empty/whitespace check             │
│                                        │
│  [Parse]                               │
│   ↓ parse(line) → Pipeline             │
│   ↓ empty pipeline? → continue        │
│                                        │
│  [Execute]                             │
│   ↓ executePipeline(pipeline)          │
│   ↓ return -1? → break (exit cmd)     │
│                                        │
│  [Post-execute cleanup]                │
│   ↓ reap finished background jobs     │
│                                        │
└────────────────────────────────────────┘
```

#### Why Clear the Flag BEFORE the Action?

```cpp
if (g_sigchld_pending) {
    g_sigchld_pending = 0;   // ← Clear FIRST
    reapFinishedJobs();       // ← Then act
}
```

If you cleared AFTER: another SIGCHLD could arrive during `reapFinishedJobs()`, set the flag, and then you'd clear it — losing that notification. Clearing first means: "I've acknowledged the pending signal, if another comes while I'm processing, I'll catch it next iteration."

#### Why `std::flush` on the Prompt?

`cout` is line-buffered by default. `"shellx> "` doesn't end with `\n`, so without flush it might sit in the buffer and never appear. `std::flush` forces the buffer to the terminal immediately.

#### The Exit Sentinel Pattern

```cpp
// builtins.cpp: "exit" returns -1 to signal "break the loop"
if (name == "exit") {
    killAllJobs();
    return -1;  // ← sentinel
}

// main.cpp: interpreter sees -1, breaks
last_status = executePipeline(pipeline);
if (last_status == -1) break;
```

This is cleaner than using a global flag. The return value carries both the exit status AND the "should I stop?" signal using an out-of-band value (-1 is never a real exit code).

#### Why Signal Handler Installation Must Be First

```cpp
int main() {
    installSignalHandlers();  // ← MUST be first
    // ...
    while (true) { fork(); ... }
}
```

If a background child finishes before SIGCHLD is installed, the default handler runs (which is "ignore"). The child becomes a zombie with no notification to the shell. Installing before any fork() eliminates this window.

#### `static_cast<unsigned char>` on `isspace`

```cpp
if (!std::isspace(static_cast<unsigned char>(c)))
```

`isspace()` is undefined behavior if passed a negative value. On platforms where `char` is signed, characters > 127 are negative. Casting to `unsigned char` first makes all values non-negative. This is the correct, portable way to call `isspace`, `isalpha`, `isdigit`, etc.

---

## ─── FILE 3: `executor.cpp` ───

### Full Annotated Code

```cpp
// executeSingle: runs one external command via fork/exec/wait
static int executeSingle(const Command& cmd, bool background) {
    pid_t pid = fork();

    if (pid == -1) {
        perror("fork");   // print system error message (e.g., "fork: Resource temporarily unavailable")
        return 1;
    }

    if (pid == 0) {
        // ══════════════ CHILD PROCESS ══════════════

        // SIGINT: restore to default so child dies normally on Ctrl+C
        // (shell's custom handler was inherited — undo it)
        struct sigaction sa;
        sa.sa_handler = SIG_DFL;
        sigemptyset(&sa.sa_mask);
        sa.sa_flags = 0;
        sigaction(SIGINT, &sa, nullptr);

        // Apply file redirections (< > >>)
        // If redirect fails (e.g., file not found), child exits with code 1
        if (!applyRedirections(cmd)) {
            _exit(1);  // NOT exit() — avoid double-flushing inherited stdio buffers
        }

        // Build null-terminated char* array for execvp
        std::vector<const char*> argv;
        argv.reserve(cmd.args.size() + 1);
        for (const auto& arg : cmd.args) {
            argv.push_back(arg.c_str());  // raw C string pointer (valid as long as cmd lives)
        }
        argv.push_back(nullptr);   // execvp requires null terminator

        // execvp replaces this process image entirely
        // On success, this line is NEVER reached
        execvp(argv[0], const_cast<char* const*>(argv.data()));

        // If we get here, exec failed
        perror(argv[0]);                    // prints "ls: No such file or directory"
        if (errno == ENOENT) _exit(127);    // command not found
        else                 _exit(126);    // found but not executable
    }

    // ══════════════ PARENT PROCESS ══════════════

    if (background) {
        // Build the command string for the jobs list display
        std::ostringstream oss;
        for (size_t j = 0; j < cmd.args.size(); ++j) {
            if (j > 0) oss << " ";
            oss << cmd.args[j];
        }
        int job_num = addJob(pid, oss.str());
        std::cout << "[" << job_num << "] " << pid << "\n";
        return 0;   // Return immediately — don't block
    }

    // Foreground: block until child finishes
    int status;
    pid_t result;
    do {
        result = waitpid(pid, &status, 0);  // 0 = blocking wait
    } while (result == -1 && errno == EINTR);  // retry if interrupted by signal

    if (result == -1) {
        perror("waitpid");
        return 1;
    }

    if (WIFEXITED(status))   return WEXITSTATUS(status);         // normal exit
    if (WIFSIGNALED(status)) return 128 + WTERMSIG(status);      // killed by signal

    return 1;  // fallback
}

// executePipeline: the main dispatch function
int executePipeline(const Pipeline& pipeline) {
    if (pipeline.commands.empty()) return 0;

    if (pipeline.commands.size() == 1) {
        const Command& cmd = pipeline.commands[0];

        // Built-in? Run in parent (no fork ever)
        if (isBuiltin(cmd.args[0])) {
            return executeBuiltin(cmd);
        }

        // Single external command
        return executeSingle(cmd, pipeline.background);
    }

    // Multi-command pipeline — background piped pipelines not supported
    if (pipeline.background) {
        std::cerr << "shellx: backgrounded pipelines (cmd1 | cmd2 &) are not supported\n";
        return 1;
    }

    return executePipelineChain(pipeline);  // → pipeline.cpp
}
```

### Deep Analysis

#### The Dispatch Decision Tree

```
executePipeline(pipeline)
        │
        ├── pipeline.commands.empty()? → return 0
        │
        ├── commands.size() == 1?
        │       ├── isBuiltin(name)? → executeBuiltin()   [runs in parent, no fork]
        │       └── else           → executeSingle()      [fork + exec + wait/track]
        │
        └── commands.size() > 1?
                ├── background?    → error (unsupported)
                └── else          → executePipelineChain() [N forks + N-1 pipes]
```

This is a classic **strategy pattern** — the dispatch function picks the right strategy based on shape of input.

#### Child Signal Restoration — Why It Matters

```cpp
// In child, after fork():
struct sigaction sa;
sa.sa_handler = SIG_DFL;   // Reset to default behavior
sigemptyset(&sa.sa_mask);
sa.sa_flags = 0;
sigaction(SIGINT, &sa, nullptr);
```

The shell has installed a custom SIGINT handler that does NOT terminate the shell. This handler is inherited by every child. If we don't reset it:
- User presses Ctrl+C
- SIGINT sent to process group
- Shell's custom handler runs in child → child just sets a flag, doesn't die
- `cat`, `sleep`, and other commands become unkillable via Ctrl+C

Resetting to `SIG_DFL` means the child terminates normally on Ctrl+C.

#### The `argv` Construction Pattern

```cpp
std::vector<const char*> argv;
argv.reserve(cmd.args.size() + 1);           // pre-allocate (performance)
for (const auto& arg : cmd.args) {
    argv.push_back(arg.c_str());             // pointer into string's internal buffer
}
argv.push_back(nullptr);                     // NULL terminator — required by execvp

execvp(argv[0], const_cast<char* const*>(argv.data()));
```

**Why `const_cast`?** `execvp` is declared as `int execvp(const char*, char* const[])` — the inner strings are `char*` (not `const char*`) for historical C reasons, even though it doesn't actually modify them. The cast is safe here.

**Lifetime**: The `cmd` (and thus the `std::string` objects) lives on the stack until exec replaces the process. If exec succeeds, memory is irrelevant. If exec fails, `argv` and `cmd` both still exist — safe to call `perror`.

#### EINTR Retry Loop

```cpp
do {
    result = waitpid(pid, &status, 0);
} while (result == -1 && errno == EINTR);
```

Why? While the shell is blocked in `waitpid()` waiting for a foreground child, a background child might finish. That delivers SIGCHLD to the shell. Even with SA_RESTART, some syscalls are not automatically restarted on all signals. The explicit retry loop is a belt-and-suspenders approach — if waitpid returns EINTR, just call it again.

#### Exit Code Conventions

```cpp
if (WIFEXITED(status))   return WEXITSTATUS(status);
if (WIFSIGNALED(status)) return 128 + WTERMSIG(status);
```

`128 + N` is the POSIX convention for "killed by signal N":
- Ctrl+C sends SIGINT (signal 2) → exit code 130 (128+2)
- `kill -9 pid` sends SIGKILL (signal 9) → exit code 137 (128+9)

This lets scripts check `$?` and know HOW a process died.

#### Background Execution Flow

```
fork() returns child_pid to parent
    Parent immediately adds to g_jobs list
    Parent prints "[1] 12345"
    Parent returns 0 to REPL
    REPL prints "shellx> " again
    ...meanwhile child runs independently...
    Child finishes → kernel sends SIGCHLD to shell
    g_sigchld_pending = 1 (set by handler)
    At top of next REPL iteration:
        reapFinishedJobs() → waitpid(WNOHANG) → reaps child → prints "[1] Done"
```

---

## ─── FILE 4: `pipeline.cpp` ───

### Full Annotated Code

```cpp
int executePipelineChain(const Pipeline& pipeline) {
    int n = static_cast<int>(pipeline.commands.size());

    // Step 1: Create N-1 pipes
    // Flat array: pipe i → pipe_fds[2*i] = read end, pipe_fds[2*i+1] = write end
    std::vector<int> pipe_fds(2 * (n - 1));

    for (int i = 0; i < n - 1; ++i) {
        if (pipe(&pipe_fds[2 * i]) == -1) {
            perror("pipe");
            // Cleanup: close all pipes we already created
            for (int j = 0; j < 2 * i; ++j) close(pipe_fds[j]);
            return 1;
        }
    }

    // Step 2: Fork N children
    std::vector<pid_t> child_pids(n);

    for (int i = 0; i < n; ++i) {
        pid_t pid = fork();
        if (pid == -1) {
            perror("fork");
            // Cleanup: close all pipes
            for (int j = 0; j < 2 * (n - 1); ++j) close(pipe_fds[j]);
            // Reap children already forked
            for (int j = 0; j < i; ++j) {
                int status;
                pid_t result;
                do { result = waitpid(child_pids[j], &status, 0); }
                while (result == -1 && errno == EINTR);
            }
            return 1;
        }

        if (pid == 0) {
            // ══════════════ CHILD i ══════════════

            // Restore SIGINT to default
            struct sigaction sa;
            sa.sa_handler = SIG_DFL;
            sigemptyset(&sa.sa_mask);
            sa.sa_flags = 0;
            sigaction(SIGINT, &sa, nullptr);

            // Wire stdin: if not first command, read from pipe i-1's read end
            if (i > 0) {
                if (dup2(pipe_fds[2 * (i - 1)], STDIN_FILENO) == -1) {
                    perror("dup2"); _exit(1);
                }
            }

            // Wire stdout: if not last command, write to pipe i's write end
            if (i < n - 1) {
                if (dup2(pipe_fds[2 * i + 1], STDOUT_FILENO) == -1) {
                    perror("dup2"); _exit(1);
                }
            }

            // Close ALL pipe fds in this child (both ends of every pipe)
            // (dup2 already set up what we need via fds 0 and 1)
            for (int j = 0; j < 2 * (n - 1); ++j) {
                close(pipe_fds[j]);
            }

            // Apply file redirections (can override pipe wiring)
            if (!applyRedirections(pipeline.commands[i])) _exit(1);

            // Build argv and exec
            const auto& args = pipeline.commands[i].args;
            std::vector<const char*> argv;
            argv.reserve(args.size() + 1);
            for (const auto& arg : args) argv.push_back(arg.c_str());
            argv.push_back(nullptr);

            execvp(argv[0], const_cast<char* const*>(argv.data()));
            perror(argv[0]);
            _exit(errno == ENOENT ? 127 : 126);
        }

        // Parent: record this child's PID
        child_pids[i] = pid;
    }

    // Step 3: Parent closes ALL pipe fds (critical for EOF propagation)
    for (int j = 0; j < 2 * (n - 1); ++j) {
        close(pipe_fds[j]);
    }

    // Step 4: Collect all children. Pipeline exit status = last command's status.
    int pipeline_status = 0;
    for (int i = 0; i < n; ++i) {
        int status;
        pid_t result;
        do {
            result = waitpid(child_pids[i], &status, 0);
        } while (result == -1 && errno == EINTR);

        if (result == -1) { perror("waitpid"); continue; }

        // Only care about the last command's status
        if (i == n - 1) {
            if (WIFEXITED(status))   pipeline_status = WEXITSTATUS(status);
            else if (WIFSIGNALED(status)) pipeline_status = 128 + WTERMSIG(status);
        }
    }

    return pipeline_status;
}
```

### Deep Analysis

#### The Flat Array Indexing Scheme

For N=3 commands (`cmd0 | cmd1 | cmd2`), we need 2 pipes = 4 fds:

```
pipe_fds = [r0, w0, r1, w1]
              │   │   │   │
index:        0   1   2   3

pipe i → read end  = pipe_fds[2*i]
       → write end = pipe_fds[2*i + 1]

cmd0: write stdout → w0 = pipe_fds[1]  (dup2(pipe_fds[2*0+1], STDOUT))
cmd1: read stdin   ← r0 = pipe_fds[0]  (dup2(pipe_fds[2*0],   STDIN))
      write stdout → w1 = pipe_fds[3]  (dup2(pipe_fds[2*1+1], STDOUT))
cmd2: read stdin   ← r1 = pipe_fds[2]  (dup2(pipe_fds[2*1],   STDIN))
```

#### Full fd Wiring Diagram for `ls | grep txt | wc -l`

```
BEFORE fork (parent has all 4 fds):
  Parent: fd0=stdin, fd1=stdout, fd2=stderr, fd3=r0, fd4=w0, fd5=r1, fd6=w1

CHILD 0 (ls):
  dup2(fd4, 1)  → fd1 now points to pipe0-write
  close fd3,4,5,6
  exec ls       → ls output → pipe0

CHILD 1 (grep txt):
  dup2(fd3, 0)  → fd0 now points to pipe0-read
  dup2(fd6, 1)  → fd1 now points to pipe1-write
  close fd3,4,5,6
  exec grep     → reads from pipe0, writes to pipe1

CHILD 2 (wc -l):
  dup2(fd5, 0)  → fd0 now points to pipe1-read
  (last cmd, no stdout redirect to pipe)
  close fd3,4,5,6
  exec wc       → reads from pipe1, outputs to terminal

PARENT (after all forks):
  close fd3,4,5,6  ← CRITICAL: if parent keeps w0 or w1 open,
                     grep/wc never see EOF and hang forever

PARENT waitpid(child0), waitpid(child1), waitpid(child2)
```

#### Why Fork ALL Children Before Waiting

```cpp
// WRONG approach (sequential — would deadlock):
for cmd in commands:
    fork_and_exec(cmd)
    waitpid(child)      // ← DEADLOCK if pipe buffer fills up!
```

If you fork cmd0, exec it, then immediately wait for it: cmd0 is writing to a pipe. If the pipe buffer (64KB) fills up before cmd1 reads from it, cmd0 blocks on write. But we're waiting for cmd0 to finish before forking cmd1. Deadlock.

**Correct approach (ShellX)**: Fork ALL children first, THEN waitpid all of them. Children run concurrently and the pipe buffer is drained by downstream commands in real time.

#### Partial Failure Cleanup

```cpp
if (pid == -1) {
    // Close all pipes
    for (int j = 0; j < 2 * (n - 1); ++j) close(pipe_fds[j]);
    // Reap already-forked children
    for (int j = 0; j < i; ++j) {
        waitpid(child_pids[j], &status, 0);
    }
    return 1;
}
```

If the 3rd fork() fails (out of PIDs), we've already launched 2 children. They'll run and exit — we must waitpid them to prevent zombies. We also close all pipe fds to prevent leaks. Resource cleanup on partial failure is a critical systems programming pattern.

#### Why Only the Last Command's Exit Status?

```cpp
if (i == n - 1) {
    pipeline_status = WEXITSTATUS(status);
}
```

This matches bash's default behavior (without `set -o pipefail`). The idea: in `ls | grep | wc`, you care about whether `wc` succeeded. The intermediate commands are plumbing.

---

## 🎯 SDE Interview Q&A — Phase 1 (Scripted Answers)

---

### ─── Data Structures ───

**Q1. How is a command represented in ShellX? Why split into Command and Pipeline?**

> "ShellX uses two structs. A `Command` holds one executable unit: its argument list, and optional input/output file for redirection. A `Pipeline` holds a vector of Commands joined by pipes, plus a background flag.
>
> The split exists because a pipeline is a unit of execution but a command is a unit of I/O. The background flag lives only in Pipeline because `&` applies to the whole pipeline, not individual commands. Putting it in Command would create undefined state — which command controls background behavior when they disagree? Single source of truth."

---

**Q2. Why does `Command` store `args` as `vector<string>` but `execvp` needs `char* const[]`? How do you bridge that?**

> "execvp is a C POSIX API. It takes a null-terminated array of C strings. std::vector<string> is a C++ abstraction. To bridge them, I build a `std::vector<const char*>` by calling `.c_str()` on each string, appending a nullptr at the end, then passing `.data()` to execvp with a const_cast to satisfy its non-const signature. The underlying std::string objects stay alive through the stack until exec replaces the process — so the raw pointers are valid."

---

### ─── The REPL ───

**Q3. Walk me through exactly what happens between when the user presses Enter and when the prompt reappears.**

> "The user presses Enter, getline() returns the line string. We validate length and skip whitespace. We call parse(line) which tokenizes and builds a Pipeline struct. We call executePipeline(pipeline).
>
> If it's a built-in like `cd`, it runs directly in the parent — chdir() is called, returns 0, no fork happened.
>
> If it's a single external command like `ls`, we fork(). The parent waits blocking in waitpid(). The child restores SIGINT to default, applies any redirections, builds the argv array, calls execvp() — its image is replaced by `ls`. ls runs, exits. The kernel sends SIGCHLD (but we're already in waitpid, so we catch it directly). waitpid returns with the exit status. The parent decodes it with WIFEXITED/WEXITSTATUS. We check if it's the -1 exit sentinel. At the top of next iteration, we check signal flags, print 'shellx> ', and block on getline again."

---

**Q4. What is the exit sentinel pattern? Why use -1 as a sentinel?**

> "When the user types `exit`, the builtin handler calls killAllJobs() and returns -1. executePipeline passes this back to main. main checks `if (last_status == -1) break`. This is an out-of-band signal: real exit codes are 0-255 (POSIX), so -1 is guaranteed to never be a real exit status. It's a clean way to distinguish 'exit was requested' from 'command ran and failed' without needing a global flag or exception."

---

### ─── Fork/Exec ───

**Q5. Why does the child call `_exit()` instead of `exit()` after exec fails?**

> "exit() is a C library function that does cleanup before the real syscall: it flushes all stdio buffers, runs atexit() handlers, closes stdio streams. After fork(), the child inherits the parent's (shell's) stdio buffers. If exec fails and the child calls exit(), those inherited buffers get flushed — causing whatever was buffered in the shell's stdout to be written twice to the terminal. _exit() is the raw syscall — it terminates immediately without any cleanup, avoiding double flush."

---

**Q6. What happens if `execvp` fails with ENOENT vs EACCES? Why do you return different exit codes?**

> "ENOENT means 'no such file or directory' — the command simply doesn't exist in PATH. We return 127.
>
> EACCES means 'permission denied' — the file EXISTS but lacks execute permission. We return 126.
>
> These are POSIX shell conventions. The distinction matters for scripts: 127 means 'check your spelling or PATH', 126 means 'check your file permissions'. They let the calling script diagnose the failure correctly via $?."

---

**Q7. What is the race condition risk with EINTR in waitpid, and how does ShellX handle it?**

> "While the parent is blocked in waitpid() waiting for a foreground child, a background child might finish and deliver SIGCHLD. Even with SA_RESTART on the SIGCHLD handler, certain conditions can cause waitpid to return -1 with errno=EINTR instead of auto-restarting — particularly when a signal arrives just before the syscall starts.
>
> ShellX wraps every waitpid call in a do-while retry loop: `do { result = waitpid(...); } while (result == -1 && errno == EINTR)`. This distinguishes 'real error' (errno is not EINTR) from 'interrupted by signal, retry'. It's belt-and-suspenders with SA_RESTART."

---

### ─── Pipelines ───

**Q8. For `cmd0 | cmd1 | cmd2`, walk through every dup2() call made in each child.**

> "We have 2 pipes: pipe0 (fds 3,4) and pipe1 (fds 5,6).
>
> Child 0 (cmd0): dup2(4, 1) — stdout now writes to pipe0's write end. No stdin change (reads from terminal).
>
> Child 1 (cmd1): dup2(3, 0) — stdin now reads from pipe0's read end. dup2(6, 1) — stdout now writes to pipe1's write end.
>
> Child 2 (cmd2): dup2(5, 0) — stdin now reads from pipe1's read end. No stdout change (writes to terminal).
>
> After each dup2, all children close the original pipe fds (3,4,5,6). The parent also closes all of them after forking. This ensures EOF propagates correctly."

---

**Q9. Why must you fork ALL children before calling waitpid on any of them?**

> "Because of pipe buffer limits. A pipe's kernel buffer is typically 64KB on Linux. If cmd0 produces more than 64KB of output and we've only forked cmd0 while waiting for it to finish, cmd0 will block on write (pipe full, nobody reading). But we're in waitpid waiting for cmd0. Deadlock.
>
> The solution: fork all N children first. They all run concurrently. cmd1 reads from the pipe as fast as cmd0 writes. The buffer never fills. Only after all forks do we enter a waitpid loop to collect results."

---

**Q10. Why does ShellX only return the last command's exit status for a pipeline?**

> "This matches bash's default behavior without `pipefail`. The reasoning: in `cmd0 | cmd1 | cmd2`, the user's intent is usually to know whether the final output was produced correctly. cmd0 and cmd1 are plumbing — intermediate processors. If cmd2 fails, the pipeline failed. If cmd2 succeeds, the data made it through.
>
> bash's `set -o pipefail` changes this to return the last nonzero exit status of any command in the pipeline — more strict, useful in scripts where intermediate failures matter."

---

**Q11. How would you detect and handle a broken pipe in a pipeline?**

> "A broken pipe occurs when a writer writes to a pipe whose read end is closed — e.g., the downstream command has already exited. The kernel sends SIGPIPE to the writing process.
>
> By default, SIGPIPE terminates the writer immediately — which is usually fine (cmd0 stops producing if cmd1 is done). If you want to handle it gracefully, you install a SIGPIPE handler or set `O_NOSIGNAL` and check for EPIPE on write. In ShellX, SIGPIPE uses default disposition — writers are killed by SIGPIPE when downstream exits early, which is the correct shell behavior."

---

**Q12. What would you change to support backgrounded pipelines (`cmd1 | cmd2 &`)?**

> "Currently the Job struct holds a single pid_t. For a backgrounded pipeline, we'd need to track all N child PIDs in the job. So Job would need a `vector<pid_t> pids`. addJob() would take a vector. reapFinishedJobs() would call waitpid(WNOHANG) for each PID and only mark the job Done when all pids have been reaped. The pipeline would need to not call waitpid synchronously — just fork all, record all pids, return immediately. This also requires process group support (setpgid) for proper Ctrl+C isolation."

---

**Q13. What is a pipe buffer? What is its size on Linux? What happens when it fills up?**

> "A pipe buffer is a kernel-managed ring buffer that temporarily holds data written to the write end until it's read from the read end. On Linux, the default size is 65536 bytes (64KB), configurable per-pipe via fcntl(F_SETPIPE_SZ) up to a system limit.
>
> When the buffer is full: write() to the write end blocks until there's space. When the buffer is empty: read() from the read end blocks until data arrives. When all write ends are closed: read() returns 0 (EOF). This last property is why closing unused pipe ends in all processes is non-negotiable."

---

**Q14. What is the difference between `pipe()` and `socketpair()`?**

> "`pipe()` creates a unidirectional channel — data flows one way only. If you need bidirectional communication, you need two pipes.
>
> `socketpair()` creates a pair of connected Unix domain sockets that are bidirectional — each end can both read and write. It's used for bidirectional parent-child IPC.
>
> For shell pipelines, pipe() is sufficient because data always flows forward — cmd0 → cmd1 → cmd2."

---

**Q15. In `pipeline.cpp`, what happens if `fork()` fails partway through creating N children?**

> "We've successfully forked children 0 through i-1 but fork() returns -1 for child i. These running children will execute and eventually exit. If we don't wait for them, they become zombies.
>
> ShellX handles this explicitly: on fork failure, it closes all pipe fds, then enters a waitpid loop for children 0 through i-1, blocking until each exits. Only then does it return 1 (error). This is careful resource management — no fd leaks, no zombies, even on partial failure."

---

### ─── Design Questions ───

**Q16. How would you add process group support to fix the Ctrl+C background job issue?**

> "Right now background jobs stay in the shell's process group, so Ctrl+C sends SIGINT to them too. Fix:
>
> 1. After fork(), before exec(), the child calls `setpgid(0, 0)` — puts itself in a new process group (its own PID becomes the PGID).
> 2. The parent also calls `setpgid(child_pid, child_pid)` — race condition protection: either call can succeed first.
> 3. For foreground jobs, call `tcsetpgrp(STDIN_FILENO, child_pgid)` to give the terminal to the foreground group.
> 4. On foreground job completion, call `tcsetpgrp(STDIN_FILENO, shell_pgid)` to take back the terminal.
>
> This separates the signal delivery namespace — Ctrl+C targets only the foreground group."

---

**Q17. What is the time complexity of building and executing a pipeline of N commands?**

> "Building: O(N) pipe() calls + O(N) fork() calls. Each pipe() and fork() is O(1) from the user's perspective (kernel operations under the hood are O(address space size) for fork due to page table copying, but with CoW it's amortized cheap). Closing pipe fds: O(N) per child = O(N²) total across N children.
>
> Waiting: O(N) waitpid calls.
>
> Total: O(N²) in the worst case due to the inner close loop in each child. For MAX_PIPELINE_LENGTH=16, this is 256 operations — trivial in practice."

---

## 🎯 Phase 1 Summary — Key Mental Models

| Concept | How ShellX implements it |
|---|---|
| **REPL** | `while(true)` in main.cpp with flag checks at top |
| **Fork/exec/wait** | executeSingle() — 3 steps for every external command |
| **Built-in dispatch** | String comparison before any fork |
| **Single/multi branch** | commands.size() == 1 vs > 1 in executePipeline |
| **Pipe wiring** | Flat fd array + dup2 indexing scheme |
| **EOF propagation** | Parent closes all pipe fds after all forks |
| **Deadlock prevention** | Fork all before waiting any |
| **Exit status** | WIFEXITED/WIFSIGNALED macros + 128+N convention |
| **Sentinel pattern** | Return -1 for "exit was requested" |
| **Signal restoration** | Child resets SIGINT to SIG_DFL right after fork |

---

> ✅ **Say "proceed to Phase 2"** when ready. Phase 2 covers `signals.cpp/hpp` and `jobs.cpp/hpp` — the async control layer. This is where race conditions live and where the most subtle systems programming decisions are made.
