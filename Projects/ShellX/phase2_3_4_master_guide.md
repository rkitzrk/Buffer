# ShellX Master Study Guide — Phases 2, 3 & 4

> **Audience:** Competitive Programmer / Candidate aiming for Top-Tier SDE Roles (Google, Meta, Amazon, Uber, Systems/Infra teams).  
> **Coverage:** Phase 2 (Async Control & Signals), Phase 3 (Parsing & I/O Redirection), Phase 4 (Builtins, Build System, Testing & Project Pitch).

---

# 🗺️ Roadmap & Architecture Map

```
                  ┌──────────────────────────────────────────────┐
                  │                 User Terminal                │
                  └──────────────────────┬───────────────────────┘
                                         │ Line input
                                         ▼
┌────────────────────────────────────────────────────────────────────────────────┐
│ PHASE 3: PARSING & INPUT VALIDATION                                           │
│ parser.cpp: tokenize() ──► rejectUnsupported() ──► Pipeline & Command AST     │
└────────────────────────────────────────┬───────────────────────────────────────┘
                                         │
                   Is Built-in?          │ External Command / Pipeline
          ┌──────────────────────────────┴──────────────────────────────┐
          ▼                                                             ▼
┌──────────────────────────────┐               ┌─────────────────────────────────┐
│ PHASE 4: BUILT-IN EXECUTION  │               │ PHASE 1/3: EXECUTOR & FORK      │
│ builtins.cpp (cd, pwd, exit) │               │ executor.cpp & pipeline.cpp     │
│ In-process (parent execution)│               │ redirection.cpp (dup2 & open)   │
└──────────────────────────────┘               └────────────────┬────────────────┘
                                                                │
                                              Background Job?   │
                                         ┌──────────────────────┴───────┐
                                         │ Yes                          │ No (Foreground)
                                         ▼                              ▼
┌──────────────────────────────────────────────────────┐       ┌─────────────────┐
│ PHASE 2: ASYNC SIGNAL CONTROL & JOB LIFECYCLE        │       │ Blocking wait   │
│ signals.cpp (sigaction, SA_RESTART, flag-only)       │       │ with EINTR loop │
│ jobs.cpp (g_jobs table, reapFinishedJobs with WNOHANG│       └─────────────────┘
│ killAllJobs with SIGTERM on exit)                    │
└──────────────────────────────────────────────────────┘
```

---

# ══════════════════════════════════════════════
# PHASE 2: Async Control — Signals & Background Jobs
# ══════════════════════════════════════════════

> **Files:** [`include/signals.hpp`](file:///home/kumaresh/Desktop/Dev/Personal/ShellX/include/signals.hpp) · [`src/signals.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/ShellX/src/signals.cpp) · [`include/jobs.hpp`](file:///home/kumaresh/Desktop/Dev/Personal/ShellX/include/jobs.hpp) · [`src/jobs.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/ShellX/src/jobs.cpp)

---

## 1. Deep Code Walkthrough

### A. Signal Handler Architecture (`src/signals.cpp`)

```cpp
#include "signals.hpp"
#include <cstring>

// Global flags shared between signal handlers and main REPL loop
volatile sig_atomic_t g_sigchld_pending = 0;
volatile sig_atomic_t g_sigint_received = 0;

static void sigchld_handler(int /*signo*/) {
    // ASYNC-SIGNAL-SAFE: only atomic store, no syscalls, no malloc, no IO
    g_sigchld_pending = 1;
}

static void sigint_handler(int /*signo*/) {
    // Shell survives Ctrl+C; flags the REPL to print a newline and prompt
    g_sigint_received = 1;
}

void installSignalHandlers() {
    struct sigaction sa;

    // --- SIGCHLD Setup ---
    std::memset(&sa, 0, sizeof(sa));
    sa.sa_handler = sigchld_handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART | SA_NOCLDSTOP;
    sigaction(SIGCHLD, &sa, nullptr);

    // --- SIGINT Setup ---
    std::memset(&sa, 0, sizeof(sa));
    sa.sa_handler = sigint_handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART;
    sigaction(SIGINT, &sa, nullptr);
}
```

#### Line-by-Line Systems Insights:
1. **`volatile sig_atomic_t`:**
   - In competitive programming, when multiple threads or interrupts touch a flag, compiler register caching causes stale reads. `sig_atomic_t` guarantees that reads and writes are single-instruction atomic operations on the CPU architecture. `volatile` instructs the compiler never to cache the value in a register, forcing it to fetch directly from memory on every read.
2. **Flag-Only Handler Pattern:**
   - Why not call `waitpid()` inside `sigchld_handler`? Because calling `waitpid()` in a signal handler creates an **inherent race condition** with any foreground `waitpid()` running in `executor.cpp`. Moreover, functions like `printf`, `malloc`, or STL vector mutations inside a signal handler are **undefined behavior** (non-reentrant).
3. **`SA_NOCLDSTOP`:**
   - Prevents `SIGCHLD` from firing if a child is merely paused (`Ctrl+Z` / `SIGSTOP` / `SIGTSTP`) or resumed (`SIGCONT`). The shell only cares when children actually **exit or terminate**.
4. **`SA_RESTART`:**
   - Automatically restarts slow system calls (like `read()` on stdin) if interrupted by a signal, avoiding premature `EINTR` failures.

---

### B. Background Job Table & Reaping (`src/jobs.cpp`)

```cpp
#include "jobs.hpp"
#include "shellx.hpp"
#include <iostream>
#include <algorithm>
#include <sys/wait.h>
#include <signal.h>

std::vector<Job> g_jobs;

int addJob(pid_t pid, const std::string& cmd) {
    if (static_cast<int>(g_jobs.size()) >= MAX_BACKGROUND_JOBS) {
        std::cerr << "shellx: too many background jobs (max " << MAX_BACKGROUND_JOBS << ")\n";
    }
    Job job;
    job.pid = pid;
    job.command = cmd;
    job.running = true;
    g_jobs.push_back(job);
    return static_cast<int>(g_jobs.size());
}

void reapFinishedJobs() {
    for (auto it = g_jobs.begin(); it != g_jobs.end(); ) {
        if (!it->running) {
            it = g_jobs.erase(it);
            continue;
        }

        int status;
        // Non-blocking poll using WNOHANG
        pid_t result = waitpid(it->pid, &status, WNOHANG);

        if (result > 0) {
            // Process terminated
            int job_num = static_cast<int>(std::distance(g_jobs.begin(), it)) + 1;
            if (WIFEXITED(status)) {
                std::cout << "[" << job_num << "] Done (exit " << WEXITSTATUS(status)
                          << "): " << it->command << "\n";
            } else if (WIFSIGNALED(status)) {
                std::cout << "[" << job_num << "] Killed (signal " << WTERMSIG(status)
                          << "): " << it->command << "\n";
            }
            it = g_jobs.erase(it); // Erase moves iterator to next valid element
        } else {
            ++it;
        }
    }
}

void killAllJobs() {
    for (auto& job : g_jobs) {
        if (job.running) {
            kill(job.pid, SIGTERM);
            // We intentionally do NOT wait/reap here.
            // When ShellX exits, orphaned children are automatically reparented to init (PID 1).
        }
    }
}
```

#### Line-by-Line Systems Insights:
1. **`waitpid(it->pid, &status, WNOHANG)`:**
   - Competitive programming equivalent: checking if a query is ready in $O(1)$ without blocking the main event loop. `WNOHANG` returns `0` immediately if the child is still executing, `pid` if it has changed state, or `-1` on error.
2. **Safe Iterator Invalidation with `it = g_jobs.erase(it)`:**
   - In C++, erasing from a vector invalidates iterators from the point of erasure onwards. Catching the return value of `erase()` keeps traversal strictly $O(N)$ with no iterator invalidation crashes.
3. **Shutdown via `kill(pid, SIGTERM)` & Orphan Adoption:**
   - On `exit` or `EOF (Ctrl+D)`, `killAllJobs()` fires polite termination (`SIGTERM`) to every child. We do not block to reap them because the shell is terminating immediately. Any surviving processes become orphans and are adopted by PID 1 (`init` or `systemd`), which handles their final reaping.

---

## 2. Competitive Programmer to Systems Mental Model

| CP Concept | Systems ShellX Parallel |
|---|---|
| **Event Loop / Queue** | REPL checking atomic signal flags before reading stdin |
| **Lazy Evaluation** | Postponing `waitpid()` reaping until the top of the REPL loop instead of inside the signal handler |
| **State Machine Transitions** | Process lifecycle: `Running` $\rightarrow$ `Exited` (Zombie in kernel) $\rightarrow$ `Reaped` (Removed from process table) |
| **Sentinel Value** | Returning `0` from `waitpid(..., WNOHANG)` indicating "still active" vs `>0` indicating "terminated" |

---

## 3. Intense SDE Interview Q&A (Phase 2 Scripted Answers)

### Q1. "Why can't you call `printf`, `malloc`, or mutate a `std::vector` inside a signal handler?"
> **Scripted Answer:**  
> "Signal handlers are invoked asynchronously on the same thread of execution, interrupting whatever instruction the program was currently executing. 
> 
> Functions like `malloc`, `free`, and `printf` (and STL containers like `std::vector`) rely on internal locks and global data structures. If the main program is interrupted in the middle of executing `malloc()` while holding an internal heap lock, and the signal handler also calls `malloc()`, the signal handler attempts to acquire the exact same lock. Because the thread already holds it but cannot proceed, it **deadlocks with itself**.
> 
> POSIX defines a strict whitelist of **async-signal-safe** functions. In ShellX, we follow the gold-standard design: signal handlers do nothing except set a `volatile sig_atomic_t` flag and return. All heap allocation, vector updates, and I/O are deferred to the main loop where they can execute safely."

---

### Q2. "Walk me through the race condition that occurs if `SIGCHLD` calls `waitpid()` directly."
> **Scripted Answer:**  
> "Suppose the shell is running a foreground process (e.g., `make`) and has called blocking `waitpid(fg_pid, &status, 0)` in `executor.cpp`. At that exact millisecond, a background child finishes and the kernel delivers `SIGCHLD`.
> 
> If the `SIGCHLD` handler were to call `waitpid(-1, &status, WNOHANG)` to reap children, the handler could reap the foreground child before the executor's `waitpid()` returns. When the signal handler finishes and control returns to the executor, the executor's `waitpid(fg_pid)` fails with `ECHILD` (No child processes) because the child was already reaped. The shell would falsely report an execution error.
> 
> ShellX avoids this by enforcing **single ownership of waitpid calls**: the foreground executor reaps foreground PIDs directly, and background PIDs are reaped exclusively by `reapFinishedJobs()` in the REPL loop when the executor is inactive."

---

### Q3. "What happens to background jobs when the user presses `Ctrl+C` in ShellX? Is this a limitation?"
> **Scripted Answer:**  
> "In the current implementation of ShellX, background jobs are not yet placed in isolated process groups via `setpgid()`. Because all children share the shell's foreground process group, pressing `Ctrl+C` on the terminal sends `SIGINT` to the entire terminal process group — meaning background jobs also receive `SIGINT`.
> 
> In a production shell, this is solved using POSIX job control:
> 1. Each background pipeline is placed into its own process group using `setpgid(pid, pid)`.
> 2. The foreground job is given control of the terminal using `tcsetpgrp(STDIN_FILENO, pgid)`.
> 3. Terminal signals like `SIGINT` (`Ctrl+C`) and `SIGTSTP` (`Ctrl+Z`) are delivered by the tty driver only to the foreground process group, completely isolating background jobs."

---

# ══════════════════════════════════════════════
# PHASE 3: Input Processing — Lexer, Parser & Redirection
# ══════════════════════════════════════════════

> **Files:** [`include/parser.hpp`](file:///home/kumaresh/Desktop/Dev/Personal/ShellX/include/parser.hpp) · [`src/parser.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/ShellX/src/parser.cpp) · [`include/redirection.hpp`](file:///home/kumaresh/Desktop/Dev/Personal/ShellX/include/redirection.hpp) · [`src/redirection.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/ShellX/src/redirection.cpp)

---

## 1. Deep Code Walkthrough

### A. The Tokenizer DFA (`src/parser.cpp`)

The tokenizer scans the input string character-by-character using a deterministic finite-state approach:

```cpp
static std::vector<std::string> tokenize(const std::string& line) {
    std::vector<std::string> tokens;
    size_t i = 0, len = line.size();

    while (i < len) {
        if (std::isspace(line[i])) { ++i; continue; }

        // Double-quote state: preserves internal spaces and special characters
        if (line[i] == '"') {
            ++i; // consume opening quote
            std::string token;
            while (i < len && line[i] != '"') {
                token += line[i++];
            }
            if (i < len) ++i; // consume closing quote
            tokens.push_back(token);
            continue;
        }

        // Two-character operator: >> (must check before >)
        if (line[i] == '>' && i + 1 < len && line[i + 1] == '>') {
            tokens.push_back(">>");
            i += 2;
            continue;
        }

        // Single-character delimiters: | < > &
        if (line[i] == '|' || line[i] == '<' || line[i] == '>' || line[i] == '&') {
            tokens.push_back(std::string(1, line[i++]));
            continue;
        }

        // Regular word / argument
        std::string token;
        while (i < len && !std::isspace(line[i]) &&
               line[i] != '|' && line[i] != '<' && line[i] != '>' &&
               line[i] != '&' && line[i] != '"') {
            token += line[i++];
        }
        tokens.push_back(token);
    }
    return tokens;
}
```

#### Grammar Enforcement (`rejectUnsupported`):
ShellX rejects syntax it does not natively support rather than producing undefined behavior:
- `;` (Command sequencing)
- `&&` and `||` (Short-circuit logical operators)
- `$()` and backticks (Command substitution)

---

### B. Pipeline AST Construction (`parse()`)

```cpp
Pipeline parse(const std::string& line) {
    Pipeline pipeline;
    std::vector<std::string> tokens = tokenize(line);
    if (tokens.empty() || rejectUnsupported(tokens)) return pipeline;

    // Detect trailing '&'
    if (!tokens.empty() && tokens.back() == "&") {
        pipeline.background = true;
        tokens.pop_back();
        if (tokens.empty()) {
            std::cerr << "shellx: syntax error near '&'\n";
            return Pipeline{};
        }
    }

    // Split on '|'
    std::vector<std::vector<std::string>> command_token_lists;
    command_token_lists.push_back({});
    for (const auto& token : tokens) {
        if (token == "|") {
            if (command_token_lists.back().empty()) {
                std::cerr << "shellx: syntax error near '|'\n";
                return Pipeline{};
            }
            command_token_lists.push_back({});
        } else {
            command_token_lists.back().push_back(token);
        }
    }

    if (command_token_lists.back().empty()) {
        std::cerr << "shellx: syntax error near '|'\n";
        return Pipeline{};
    }

    // Parse each command's redirections and arguments
    for (auto& cmd_tokens : command_token_lists) {
        Command cmd;
        for (size_t i = 0; i < cmd_tokens.size(); ++i) {
            if (cmd_tokens[i] == "<") {
                if (i + 1 >= cmd_tokens.size()) { /* syntax error */ return Pipeline{}; }
                cmd.input_file = cmd_tokens[++i];
            } else if (cmd_tokens[i] == ">" || cmd_tokens[i] == ">>") {
                bool append = (cmd_tokens[i] == ">>");
                if (i + 1 >= cmd_tokens.size()) { /* syntax error */ return Pipeline{}; }
                cmd.output_file = cmd_tokens[++i];
                cmd.append_mode = append;
            } else {
                cmd.args.push_back(cmd_tokens[i]);
            }
        }
        if (cmd.args.empty()) return Pipeline{};
        pipeline.commands.push_back(std::move(cmd));
    }
    return pipeline;
}
```

---

### C. File Descriptor Remapping (`src/redirection.cpp`)

```cpp
#include "redirection.hpp"
#include <cstdio>

bool applyRedirections(const Command& cmd) {
    // Input Redirection: <
    if (!cmd.input_file.empty()) {
        int fd = open(cmd.input_file.c_str(), O_RDONLY);
        if (fd == -1) {
            perror(cmd.input_file.c_str());
            return false;
        }
        if (dup2(fd, STDIN_FILENO) == -1) {
            perror("dup2");
            close(fd);
            return false;
        }
        close(fd); // Original fd safely closed; STDIN_FILENO now refers to file
    }

    // Output Redirection: > or >>
    if (!cmd.output_file.empty()) {
        int flags = O_WRONLY | O_CREAT;
        flags |= (cmd.append_mode ? O_APPEND : O_TRUNC);

        // 0644 mode = rw-r--r--
        int fd = open(cmd.output_file.c_str(), flags, 0644);
        if (fd == -1) {
            perror(cmd.output_file.c_str());
            return false;
        }
        if (dup2(fd, STDOUT_FILENO) == -1) {
            perror("dup2");
            close(fd);
            return false;
        }
        close(fd);
    }
    return true;
}
```

---

## 2. Competitive Programmer to Systems Mental Model

| CP Concept | Systems ShellX Parallel |
|---|---|
| **String Parsing / Two Pointers** | Tokenizer character pointer scanning through `line` with greedy matching for `>>` before `>` |
| **AST (Abstract Syntax Tree)** | Splitting on `|` creates an array of subtrees (`Command`), where each command has attributes (`args`, `input_file`, `output_file`) |
| **Graph Edge Rewiring** | `dup2(fd, STDIN/STDOUT)` rewires the node's standard incoming/outgoing edges in the OS file table |
| **Move Semantics (`std::move`)** | Transferring ownership of dynamic vectors into the AST without allocating duplicate memory |

---

## 3. Intense SDE Interview Q&A (Phase 3 Scripted Answers)

### Q1. "Explain step-by-step how `dup2()` accomplishes redirection."
> **Scripted Answer:**  
> "In Unix, every process has a File Descriptor Table. Indices `0`, `1`, and `2` point to standard input, standard output, and standard error.
> 
> When executing `cat < input.txt > output.txt`:
> 1. In the child process (after `fork()`, before `exec()`), we call `open("input.txt", O_RDONLY)`. The kernel assigns the lowest available descriptor, say `fd = 3`.
> 2. We call `dup2(3, STDIN_FILENO)` (i.e. `dup2(3, 0)`). `dup2()` atomically closes descriptor 0 (if open) and makes descriptor 0 point to the exact same file table entry as descriptor 3.
> 3. We immediately call `close(3)`. Now, descriptor 0 is the sole handle pointing to `input.txt`.
> 4. We repeat for stdout: `open("output.txt", ...)` returns `fd = 3`. We call `dup2(3, STDOUT_FILENO)` and `close(3)`.
> 5. When `execvp()` runs, the new program inherits descriptors 0, 1, and 2 unchanged, reading from `input.txt` and writing to `output.txt` without needing any internal knowledge of redirection."

---

### Q2. "What are the security and permissions implications of `open(..., O_CREAT, 0644)`?"
> **Scripted Answer:**  
> "When `O_CREAT` is specified, the third argument `mode_t mode` is mandatory. Omitting it leads to undefined behavior because `open()` will read garbage data from the CPU register stack as the file mode.
> 
> `0644` in octal represents permissions:
> - `6` (User): Read (`4`) + Write (`2`)
> - `4` (Group): Read (`4`)
> - `4` (Others): Read (`4`)
> 
> Crucially, the actual file permissions created on disk are modified by the process's **`umask`** via the formula `mode & ~umask`. For instance, if `umask` is `0022`, the resulting permissions are `0644 & ~0022 = 0644`. If `0666` were supplied, the resulting permissions would be `0644`."

---

### Q3. "Why does ShellX enforce `MAX_INPUT_LENGTH` if `std::getline()` can read arbitrarily long strings?"
> **Scripted Answer:**  
> "While `std::getline` grows its string dynamically on the heap, an unconstrained input buffer exposes the shell to memory exhaustion (Denial of Service). In POSIX systems, `ARG_MAX` and kernel limits constrain how much data can be passed through `execve()`. Setting `MAX_INPUT_LENGTH = 4096` ensures predictability, bounds the tokenization phase to deterministic time complexity, and prevents malicious or accidental inputs (e.g. pasting an infinite binary stream) from crashing the shell process."

---

# ══════════════════════════════════════════════
# PHASE 4: Peripherals — Builtins, Build System & Testing
# ══════════════════════════════════════════════

> **Files:** [`include/builtins.hpp`](file:///home/kumaresh/Desktop/Dev/Personal/ShellX/include/builtins.hpp) · [`src/builtins.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/ShellX/src/builtins.cpp) · [`CMakeLists.txt`](file:///home/kumaresh/Desktop/Dev/Personal/ShellX/CMakeLists.txt) · [`tests/test_shellx.sh`](file:///home/kumaresh/Desktop/Dev/Personal/ShellX/tests/test_shellx.sh)

---

## 1. Deep Code Walkthrough

### A. Built-in Commands Execution (`src/builtins.cpp`)

```cpp
#include "builtins.hpp"
#include "jobs.hpp"
#include <iostream>
#include <cstdlib>

bool isBuiltin(const std::string& cmd) {
    return (cmd == "cd" || cmd == "pwd" || cmd == "exit" || cmd == "jobs");
}

int executeBuiltin(const Command& cmd) {
    const std::string& name = cmd.args[0];

    // cd builtin
    if (name == "cd") {
        const char* target = nullptr;
        if (cmd.args.size() < 2) {
            target = std::getenv("HOME");
            if (!target) {
                std::cerr << "cd: HOME not set\n";
                return 1;
            }
        } else {
            target = cmd.args[1].c_str();
        }

        if (chdir(target) != 0) {
            perror("cd");
            return 1;
        }
        return 0;
    }

    // pwd builtin
    if (name == "pwd") {
        char buf[4096];
        if (getcwd(buf, sizeof(buf)) != nullptr) {
            std::cout << buf << "\n";
        } else {
            perror("pwd");
            return 1;
        }
        return 0;
    }

    // jobs builtin
    if (name == "jobs") {
        printJobs();
        return 0;
    }

    // exit builtin
    if (name == "exit") {
        killAllJobs();
        return -1; // Sentinel: causes the REPL while-loop to terminate
    }

    std::cerr << "shellx: unknown builtin: " << name << "\n";
    return 1;
}
```

#### Line-by-Line Systems Insights:
1. **Why `cd` MUST be built-in:**
   - Every process possesses its own Working Directory pointer in kernel space. If `cd` were executed in a child process via `fork() + execvp()`, the child's working directory would change, but upon child exit, the parent shell's working directory would remain unchanged. Therefore, `chdir()` must be executed directly in the parent process.
2. **The Exit Sentinel (`-1`):**
   - In Unix exit statuses, standard return codes are unsigned 8-bit integers (`0` to `255`). Using `-1` provides an out-of-band sentinel signal that tells the outer REPL loop to break cleanly.

---

### B. Build System & Sanitizers (`CMakeLists.txt`)

```cmake
cmake_minimum_required(VERSION 3.20)
project(ShellX LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

add_executable(shellx
    src/main.cpp
    src/parser.cpp
    src/executor.cpp
    src/builtins.cpp
    src/redirection.cpp
    src/pipeline.cpp
    src/signals.cpp
    src/jobs.cpp
)
target_include_directories(shellx PRIVATE include)
target_compile_options(shellx PRIVATE -Wall -Wextra -Wpedantic)
```

#### Debug & Sanitizer Recipe:
```bash
cmake -S . -B build-debug -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -g"
cmake --build build-debug
```
- **AddressSanitizer (ASan):** Detects buffer overflows, use-after-free, double-free, and memory leaks.
- **UndefinedBehaviorSanitizer (UBSan):** Catches integer overflows, invalid bitwise shifts, and misaligned pointer dereferences.

---

### C. Automated Test Harness (`tests/test_shellx.sh`)

The test script validates:
1. **Basic Commands & Builtins:** `echo`, `pwd`, `cd`, `cd $HOME`, invalid directory handling.
2. **Redirection Mechanics:** `>` (truncation), `>>` (appending), `<` (input from file), missing file errors.
3. **Pipes:** 2-stage (`echo | cat`), 3-stage (`cat | sort | uniq`), SIGPIPE handling.
4. **Background Execution:** `sleep 0.1 &`, verification of `[1] <pid>` syntax.
5. **Syntax Rejection:** Proper rejection of `&&`, `||`, empty pipe segments `| ls`, trailing pipes `ls |`, and backgrounded pipelines `cat | cat &`.

---

## 2. Competitive Programmer to Systems Mental Model

| CP Concept | Systems ShellX Parallel |
|---|---|
| **Root of State Tree** | Process Working Directory (`chdir` mutates root of path queries for future syscalls) |
| **Sanitizers & Flags** | Using `-fsanitize=address,undefined` in CP matches how production systems teams catch memory faults in CI |
| **Unit & Integration Testing** | Test script feeding mock inputs to standard streams and asserting exit codes / substrings |

---

## 3. Intense SDE Interview Q&A (Phase 4 Scripted Answers)

### Q1. "Why can't `exit` just call `exit(0)` inside `executeBuiltin()`?"
> **Scripted Answer:**  
> "If `executeBuiltin()` called `exit(0)` directly, it would bypass RAII destructors and any final cleanup logic situated in `main()`. 
> 
> By returning an out-of-band sentinel value (`-1`), the responsibility of termination flows cleanly up the call stack. The REPL loop in `main()` breaks gracefully, allowing stack variables to unwind, open streams to flush, and `main()` to return its final exit code. This maintains structured control flow and prevents abrupt termination."

---

### Q2. "How would you design a test suite for a custom shell?"
> **Scripted Answer:**  
> "A robust shell test harness requires testing both positive and negative execution paths across 4 layers:
> 1. **Unit & Grammar Validation:** Feed malformed syntax (`| cmd`, `cmd |`, `&&`, unbalanced quotes) and verify the parser fails cleanly with specific error messages rather than crashing.
> 2. **File Descriptor Isolation:** Check that redirected commands write only to the target file and do not leak file descriptors to subsequent commands (verifiable via `/proc/$$/fd` or `lsof`).
> 3. **Concurrency & Signals:** Test rapid succession of background jobs and verify that zombie processes do not accumulate (`ps -ef | grep defunct`). Test that pressing `Ctrl+C` kills long-running children without killing the shell itself.
> 4. **Exit Status Propagation:** Assert that `$?` reflects the exact POSIX conventions (`0` on success, `127` on not found, `126` on permission denied, `128 + N` on signal death)."

---

# ══════════════════════════════════════════════
# 🏆 SDE Placement Master Pitch: How to Sell ShellX in Interviews
# ══════════════════════════════════════════════

### 1. The 60-Second Elevator Pitch
> *"For my systems programming project, I built **ShellX**, a POSIX-compliant Unix command interpreter written in C++20 on Linux using raw POSIX system calls. 
> 
> Rather than wrapping high-level libraries, I implemented process creation, multi-stage pipelines, I/O redirection, and asynchronous signal handling directly using primitives like `fork()`, `execvp()`, `pipe()`, `dup2()`, and `sigaction()`. 
> 
> The most technically demanding engineering challenge was ensuring **async-signal safety** and **deadlock-free pipeline execution**. I designed a flag-only `SIGCHLD` handler using `volatile sig_atomic_t` to eliminate race conditions with foreground reaping, enforced strict file descriptor ownership rules across parent and child processes to guarantee proper EOF propagation, and tested the codebase with AddressSanitizer and UndefinedBehaviorSanitizer."*

---

### 2. The Top 5 "Curveball" Questions Interviewers Love to Ask

#### Curveball 1: "What happens if a child writes to a pipe after the reader process has crashed or terminated?"
> **Scripted Answer:**  
> "The kernel sends a `SIGPIPE` signal to the writing process. The default action of `SIGPIPE` is to terminate the writer process immediately. If the writer has chosen to ignore `SIGPIPE` or handles it, the `write()` system call returns `-1` with `errno` set to `EPIPE` (Broken pipe). In ShellX, child processes restore default signal behavior (`SIG_DFL`), so upstream writers terminate naturally when downstream readers close their pipes."

#### Curveball 2: "Can a parent process receive a signal while executing inside `waitpid()`?"
> **Scripted Answer:**  
> "Yes. If a signal arrives while `waitpid()` is blocked, the call can be interrupted and return `-1` with `errno = EINTR`. In ShellX, we install signal handlers with `SA_RESTART`, but to guarantee zero race conditions on systems where certain calls are not automatically restarted, we wrap all foreground `waitpid()` invocations in an explicit retry loop:
> ```cpp
> do {
>     result = waitpid(pid, &status, 0);
> } while (result == -1 && errno == EINTR);
> ```
> This guarantees that background notifications do not abort foreground execution."

#### Curveball 3: "Why did you use `std::vector<const char*>` for `execvp` instead of raw C dynamic arrays (`malloc`)?"
> **Scripted Answer:**  
> "Using `std::vector<const char*>` gives us RAII memory safety in C++ without sacrificing compatibility with POSIX C APIs. We call `.c_str()` on each `std::string` argument, push a trailing `nullptr`, and pass `.data()` directly to `execvp`. If `execvp()` fails, the vector is cleanly freed when the child calls `_exit()`, avoiding any manual heap memory tracking or leaks."

#### Curveball 4: "Why did you choose `_exit()` over `exit()` when `execvp()` fails in a child?"
> **Scripted Answer:**  
> "`exit()` is a C library function that runs user `atexit` handlers and flushes standard I/O streams. Because the child inherited duplicated stdio buffer copies from the parent shell upon `fork()`, calling `exit()` would flush those buffers a second time, corrupting terminal output. `_exit()` is the direct kernel system call that terminates the child immediately without flushing inherited buffers."

#### Curveball 5: "What is the single biggest architectural enhancement you would make if given another week?"
> **Scripted Answer:**  
> "Implementing full **POSIX job control with process groups**. Currently, background jobs share the terminal's process group, meaning pressing `Ctrl+C` sends `SIGINT` to background jobs as well as foreground jobs. I would implement `setpgid()` to place each command or pipeline into a distinct process group, and use `tcsetpgrp()` to dynamically grant terminal control exclusively to the foreground group."

---

### 3. Summary Checklist for SDE Placements

- [x] Can explain `fork()`, `execvp()`, `waitpid()` with return values and error codes.
- [x] Can draw the file descriptor table before and after `dup2()` and pipe closure.
- [x] Can explain why `SIGCHLD` handler must only set an atomic flag (async-signal safety).
- [x] Can explain the difference between a zombie, an orphan, and how PID 1 reaps orphans.
- [x] Can explain why `cd` and `exit` must execute in the parent process.
- [x] Can explain why child processes must call `_exit()` instead of `exit()` on exec failure.
- [x] Can explain why all pipe file descriptors must be closed in the parent after forking.
- [x] Can explain how AddressSanitizer and UndefinedBehaviorSanitizer protect systems code.
