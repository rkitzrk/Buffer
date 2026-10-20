# ShellX — Ultra-Intense Technical SDE Interview Playbook
## Code-Level Questions + Verbatim Scripted Answers

> **What This Is:** 60+ questions an interviewer will ask when they open your GitHub and read your code line by line. Every answer is tied to a specific file, function, or line number. Practice saying these out loud.
>
> **The Rule:** When an interviewer asks about your project, they test two things:  
> (1) Do you actually understand what you wrote?  
> (2) Can you reason about changes, mutations, and edge cases in real time?

---

# ══════════════════════════════════════
# FILE: shellx.hpp — Data Structure Questions
# ══════════════════════════════════════

---

**Q1. "Why did you define `Command` and `Pipeline` as structs instead of classes?"**

> "In C++, the only difference between struct and class is the default access specifier — public for struct, private for class. `Command` and `Pipeline` are pure data holders. They have no invariants to protect, no methods that need to enforce encapsulation, and no internal state that should be hidden. Using struct signals to any reader: 'This is data, not an object with behavior.' It's idiomatic modern C++ — using the right tool. Putting them as classes with public members would be confusing — it implies there's encapsulation somewhere that doesn't exist."

---

**Q2. "Why is the `background` flag on `Pipeline` and not on `Command`? Wouldn't it be simpler on Command?"**

> "Absolutely not — putting it on `Command` would introduce the possibility of **inconsistent state**. A pipeline like `cmd1 | cmd2 | cmd3` has three `Command` objects inside one `Pipeline`. If `background` lived on each `Command`, what does it mean when `cmd1.background = true` but `cmd2.background = false`? Which one wins? You'd need extra validation everywhere.
>
> The `&` operator in shell syntax applies to the entire pipeline as a unit — it's a property of the pipeline's execution mode, not of any individual command. Keeping it in `Pipeline` makes this semantically correct and provides a **single source of truth** — one flag, one meaning, no ambiguity."

---

**Q3. "You set `bool append_mode = false` as a default member initializer. What's the C++ standard version that enables this, and what does it guarantee?"**

> "Default member initializers in struct/class were introduced in **C++11** under the name non-static data member initializers (NSDMIs). They guarantee that even a default-constructed `Command{}` has `append_mode` set to `false`. Without this, if I wrote `Command cmd;` and immediately checked `cmd.append_mode`, I'd have undefined behavior from reading an uninitialized bool. The initializer ensures every `Command` object starts in a well-defined, safe state without requiring an explicit constructor."

---

**Q4. "Why are `MAX_PIPELINE_LENGTH`, `MAX_INPUT_LENGTH`, and `MAX_BACKGROUND_JOBS` `constexpr` instead of `#define`?"**

> "`#define` is a preprocessor text substitution — it has no type, no scope, and no visibility to the debugger or static analyzer. If I wrote `#define MAX_PIPELINE_LENGTH 16`, the preprocessor replaces every occurrence with the literal `16` before the compiler even sees the code. You can't step into it in `gdb`, you can't check its type, and it can silently conflict with other macros.
>
> `constexpr int MAX_PIPELINE_LENGTH = 16` is a typed, scoped, compile-time constant. It participates in the type system — if I accidentally wrote `MAX_PIPELINE_LENGTH = 17` the compiler catches it. It shows up in debug symbols. And it's evaluated at compile time with zero runtime overhead. Strictly superior in every dimension for constants in C++."

---

**Q5. "What does `#ifndef SHELLX_HPP` / `#define` / `#endif` do? Could you use `#pragma once` instead?"**

> "This is a **traditional include guard**. The C preprocessor processes `#include` directives by textually inserting the file contents. If `shellx.hpp` is included by both `executor.cpp` and `pipeline.cpp`, and both are compiled together, without a guard the compiler sees the struct definitions twice and throws a redefinition error.
>
> The guard works by: first time through, `SHELLX_HPP` is not defined, so we enter the block and define it. Second time through (same translation unit), `SHELLX_HPP` is already defined, so the entire file is skipped.
>
> `#pragma once` achieves the same result by telling the compiler to only process the file once, but it's a compiler extension — not part of the ISO C++ standard, though universally supported by GCC, Clang, and MSVC. I used the traditional form for maximum portability."

---

# ══════════════════════════════════════
# FILE: main.cpp — REPL Deep Dive
# ══════════════════════════════════════

---

**Q6. "Line 13: Why must `installSignalHandlers()` be called before the loop and before any `fork()`? What specific bug does this prevent?"**

> "If `installSignalHandlers()` is called after the loop starts, there's a window — even if just microseconds — where `SIGCHLD` is using its default disposition. The default action of `SIGCHLD` is to ignore the signal on most Linux systems. If a background child exits during that window, the kernel delivers `SIGCHLD`, the default handler ignores it, and our `g_sigchld_pending` flag is never set. The child becomes a permanent zombie — its exit status is never collected and its process table entry is never freed.
>
> By installing the handler before any code that could `fork()`, we eliminate this race window entirely."

---

**Q7. "Lines 20-23: Why do you clear `g_sigchld_pending = 0` BEFORE calling `reapFinishedJobs()`, not after?"**

> "This is a critical ordering decision. Consider the sequence if we cleared after:
> 1. We check `g_sigchld_pending` — it's `1`.
> 2. We call `reapFinishedJobs()`.
> 3. During `reapFinishedJobs()`, another background child finishes and delivers `SIGCHLD`.
> 4. The signal handler runs and sets `g_sigchld_pending = 1`.
> 5. `reapFinishedJobs()` returns.
> 6. We now execute `g_sigchld_pending = 0` — **clearing the new notification**.
>
> The newly-finished child is never reaped on this iteration and won't be noticed until the NEXT SIGCHLD arrives. This child becomes a temporary zombie unnecessarily.
>
> By clearing FIRST, step 4 re-sets the flag to 1, and our cleared-first approach means the next iteration will see the new `1` and reap the second child. No notifications lost."

---

**Q8. "Line 32: Why `std::flush` on the prompt and not just `"\n"` or `endl`?"**

> "The prompt `'shellx> '` has no newline at the end. `std::cout` uses buffered I/O by default — output is collected in a user-space buffer and flushed to the terminal only when the buffer is full, a newline is written, or an explicit flush occurs. Without `std::flush`, the prompt text might sit in the buffer indefinitely and the terminal would appear blank while waiting for user input.
>
> I could use `std::endl` instead, but `std::endl` writes `'\n'` AND flushes, which would break the prompt format (it would appear on its own line). `std::flush` flushes without adding characters. Also, `std::endl` has a reputation for unnecessary flushes in loops — here we actually want the flush so it's correct, but the habit of using `flush` explicitly makes the intent clear."

---

**Q9. "Lines 54-63: Why not just call `line.find_first_not_of(" \t\n\r")` instead of the manual loop?"**

> "Both work. The manual loop has two practical advantages:
> 1. **Early termination**: The moment a non-whitespace character is found, we `break` immediately. We don't scan the rest of the string. `find_first_not_of` must check every character against all characters in the supplied set for each position.
> 2. **Correctness across character sets**: I use `static_cast<unsigned char>(c)` before `std::isspace()`. On systems where `char` is signed (which is implementation-defined in C++), characters above ASCII 127 have negative values. Passing a negative value to `std::isspace()` is **undefined behavior**. The cast to `unsigned char` makes all values non-negative before the classification call. `find_first_not_of` with a string literal doesn't have this explicit safety."

---

**Q10. "Lines 75-78: `executePipeline` returns `-1` as an exit sentinel. Why is `-1` safe to use? Could it ever be a real exit code?"**

> "POSIX defines exit codes as unsigned 8-bit values in the range 0 to 255. `waitpid` extracts them via `WEXITSTATUS(status)`, which is defined to return values in `[0, 255]`. The macro `WEXITSTATUS` extracts bits 8-15 of the status integer, which are always in range.
>
> `-1` is outside the 0-255 range. It cannot be a real process exit code. I chose `-1` specifically because it's an unmistakable sentinel — it's what POSIX system calls return on failure. Any developer reading `if (last_status == -1)` immediately recognizes this as a sentinel check, not a real exit code comparison.
>
> The only function that returns `-1` in this codebase is `executeBuiltin()` when the command is `exit` — a single, explicit location."

---

**Q11. "Line 87: `return last_status >= 0 ? last_status : 0;` — when would `last_status` be negative here?"**

> "Theoretically, if the loop exits via `break` from the exit sentinel path (`last_status == -1`), `last_status` is `-1` and we return `0`. The ternary prevents the shell process itself from returning `-1` to the calling terminal — which would be nonsensical since exit codes are non-negative.
>
> If the loop exits via `getline()` returning false (Ctrl+D / EOF), `last_status` retains whatever the last executed command returned — could be `0`, could be `1`, or any value 0-255. That value is returned directly as the shell's exit code. This means the shell's exit code accurately reflects whether the last executed command succeeded."

---

# ══════════════════════════════════════
# FILE: executor.cpp — Fork/Exec Internals
# ══════════════════════════════════════

---

**Q12. "Lines 25-29 in executor.cpp: Why do you restore SIGINT to `SIG_DFL` in the child AFTER fork but BEFORE exec?"**

> "After `fork()`, the child inherits the parent shell's custom `SIGINT` handler — the one that just sets `g_sigint_received = 1` and returns without terminating the process. This is intentional for the shell itself: pressing Ctrl+C shouldn't kill the shell.
>
> But the child is about to become a completely different program via `execvp()`. When the user presses Ctrl+C while running `sleep 10` foreground, they expect `sleep` to terminate. If we don't restore `SIG_DFL`, `sleep` inherits the shell's handler, Ctrl+C just sets a flag that `sleep` never reads, and `sleep` keeps running despite the user's explicit termination intent.
>
> We do this BEFORE exec because `exec` replaces the program image but PRESERVES signal dispositions that are set to `SIG_DFL` or `SIG_IGN`. Any handler that's a function pointer gets reset to `SIG_DFL` by exec automatically — but we want `SIG_DFL` specifically for SIGINT, so setting it before exec is the explicit, correct approach. We don't want to rely on exec's implicit reset behavior."

---

**Q13. "Lines 37-42: Why `argv.reserve(cmd.args.size() + 1)` before the loop? Is it necessary?"**

> "It's a performance optimization — not strictly necessary for correctness. `std::vector::push_back` can trigger reallocations if the vector grows past its capacity. Each reallocation allocates a new buffer (typically 2x the current capacity), copies all existing elements, and frees the old buffer. For a command with 20 arguments, without `reserve`, you might get 4-5 reallocations during the push_back loop.
>
> `reserve(size + 1)` pre-allocates exactly the right capacity in a single allocation: `size` for the argument pointers plus `1` for the null terminator. Zero reallocations. In a hot path like a shell executing commands in a tight loop, this matters.
>
> The `+ 1` is for the mandatory null sentinel that `execvp` requires as the terminator of the argv array."

---

**Q14. "Line 45: Why `const_cast<char* const*>`? Isn't this unsafe?"**

> "The `const_cast` here is a POSIX C legacy issue. `execvp` is declared as:
> ```c
> int execvp(const char *file, char *const argv[]);
> ```
> The inner strings are `char*` (mutable pointers to char), not `const char*`. However, in C++, `arg.c_str()` returns `const char*` — a pointer to immutable data inside the `std::string` object. So our `vector<const char*>` contains `const char*` elements, but `execvp` expects `char*`.
>
> The `const_cast` discards the const on the pointer elements. This is safe because `execvp` does not actually modify the strings — the function signature predates C's const correctness conventions. The POSIX spec explicitly states that argv strings are not modified.
>
> Is it unsafe? The `const_cast` itself is safe when you know the callee doesn't write through the pointer, which POSIX guarantees for `execvp`. Using `const_cast` is never type-safe in the general case, but here it's documented and intentional."

---

**Q15. "Lines 49-53: After `execvp()` fails, why check `errno` for `ENOENT` vs `EACCES` specifically? Why not just `_exit(1)` for all failures?"**

> "This follows the POSIX shell specification (IEEE Std 1003.1). The distinction matters for automation and scripting:
>
> - **Exit code 127 (`ENOENT`)**: 'Command not found'. The command doesn't exist in any directory in PATH. Automated deployment tools, CI/CD pipelines, and scripts check for 127 to detect misconfigured environments or typos.
> - **Exit code 126 (`EACCES`)**: 'Command found but not executable'. The binary exists but lacks the execute permission bit. This is a permissions problem, not a PATH problem.
>
> If we returned `1` for both, a script checking `$?` after a failed command would know it failed but not why. Distinguishing 127 vs 126 gives the caller actionable diagnostic information. Real shells (bash, zsh, dash) all follow this convention."

---

**Q16. "Lines 75-77: Why is the `do-while` loop with `errno == EINTR` needed? Doesn't `SA_RESTART` handle this?"**

> "Yes, `SA_RESTART` handles it for most cases — it tells the kernel to automatically restart interrupted system calls after the signal handler returns. But `SA_RESTART` has documented exceptions on Linux. Specifically, `wait()`-family calls are NOT automatically restarted by `SA_RESTART` on some older Linux kernel versions and in certain signal delivery edge cases.
>
> More importantly: defensive programming. The do-while costs zero CPU time on the normal path (SIGCHLD doesn't fire every iteration). But if we ever add a new signal handler without `SA_RESTART`, or if the code runs on a POSIX system where `SA_RESTART` semantics differ, the retry loop is our safety net.
>
> This is belt-and-suspenders: `SA_RESTART` is the belt, the EINTR retry loop is the suspenders. In production systems code, you use both."

---

**Q17. "Lines 84-89: `WIFEXITED` vs `WIFSIGNALED` — what's in the `status` integer that makes these macros work?"**

> "The `status` argument of `waitpid` is a packed bitfield, not a direct exit code. The low 8 bits encode HOW the child terminated:
> - If bits [0:6] are all 0: the child called `exit()` or returned from main. Bits [8:15] contain the actual exit code. `WIFEXITED` checks bits [0:6] == 0 and `WEXITSTATUS` extracts bits [8:15].
> - If bit 7 is 0 and bits [0:6] are non-zero: the child was killed by a signal. Bits [0:6] contain the signal number. `WIFSIGNALED` checks this case and `WTERMSIG` extracts bits [0:6].
> - If bit 7 is set: the child stopped (not relevant here since `SA_NOCLDSTOP` prevents this).
>
> In ShellX: `128 + WTERMSIG(status)` follows the POSIX convention that signals produce exit codes above 128. Ctrl+C is SIGINT (signal 2) → exit code 130. `kill -9` is SIGKILL (signal 9) → exit code 137."

---

**Q18. "Lines 100-110: Why check `isBuiltin` before `executeSingle`? What breaks if you swap that order?"**

> "If we called `executeSingle` first for `cd /tmp`, we would `fork()` a child, the child would call `execvp('cd', ['cd', '/tmp', NULL])`. But `cd` is not a binary on the filesystem — it's a shell built-in. `execvp` would fail with `ENOENT` and the child would print `cd: No such file or directory` and exit with 127.
>
> Even if `cd` were somehow a real binary: changing directory in a child process only affects THAT child's working directory. When the child exits and we `waitpid`, the parent shell's working directory is completely unchanged. The `chdir()` system call modifies the per-process working directory in the kernel PCB. It cannot propagate up to the parent.
>
> This is why built-ins MUST run in the parent process — they need to modify the parent's state: current directory (`cd`), exit the parent itself (`exit`), or read the parent's job list (`jobs`)."

---

# ══════════════════════════════════════
# FILE: pipeline.cpp — The Hardest File
# ══════════════════════════════════════

---

**Q19. "Line 13: `std::vector<int> pipe_fds(2 * (n - 1))`. Walk me through why N commands need exactly N-1 pipes."**

> "Think of it as a chain. For `cmd0 | cmd1 | cmd2 | cmd3` (N=4):
>
> ```
> cmd0 → pipe0 → cmd1 → pipe1 → cmd2 → pipe2 → cmd3
> ```
>
> There are 3 connections for 4 nodes — that's `N - 1` edges. Each pipe creates 2 file descriptors (read end and write end), so the total fd count is `2 * (N - 1)`.
>
> I use a flat vector instead of a 2D array: `pipe_fds[2*i]` is the read end of pipe `i`, and `pipe_fds[2*i + 1]` is the write end of pipe `i`. For N=4:
> - Pipe 0: `pipe_fds[0]` (read) and `pipe_fds[1]` (write) — connects cmd0 → cmd1
> - Pipe 1: `pipe_fds[2]` (read) and `pipe_fds[3]` (write) — connects cmd1 → cmd2
> - Pipe 2: `pipe_fds[4]` (read) and `pipe_fds[5]` (write) — connects cmd2 → cmd3"

---

**Q20. "Lines 19-21: If `pipe()` fails for pipe `i`, you close pipes `0` through `i-1`. Why not just skip cleanup and return 1?"**

> "File descriptors are a finite system resource. Each process on Linux has a default fd limit of 1024 (soft limit, configurable up to `/proc/sys/fs/file-max`). If we open pipes for 5 commands and the 5th `pipe()` fails but we don't close the previous 8 fds, the shell process now has 8 leaked fd slots it can never recover. Over time, or in a long-running shell with many pipeline attempts that fail mid-creation, you'd exhaust the fd limit and `open()`, `pipe()`, `socket()` calls start failing across the board.
>
> The cleanup loop `for (int j = 0; j < 2 * i; ++j) close(pipe_fds[j])` releases exactly the fds we successfully opened before the failure. This is standard POSIX resource cleanup on partial failure — the same pattern used in kernel drivers and system daemons."

---

**Q21. "Lines 38-44: If `fork()` fails partway through, you not only close pipes but also `waitpid()` already-forked children. Why? What happens if you don't?"**

> "If fork fails on child 3 but children 0, 1, 2 were already successfully forked and are executing:
>
> - Children 0, 1, 2 are running, writing to pipes, reading from pipes.
> - We return immediately without `waitpid`.
> - The shell REPL continues to the next iteration.
> - Children 0, 1, 2 finish executing and exit.
> - The kernel marks them as zombies — holding process table entries.
> - We never call `waitpid` for these PIDs.
> - They become **permanent zombies** consuming process table entries forever.
>
> By calling `waitpid(child_pids[j], &status, 0)` for each successfully-forked child in the cleanup block, we collect their exit statuses, the kernel removes their process table entries, and no zombies are created. Resource cleanup on partial failure is non-negotiable in production systems code."

---

**Q22. "Lines 59-72: Explain the exact `dup2` calls for child `i`. What are the formulas and why?"**

> "For child `i` out of `n` total commands:
>
> **Stdin wiring (if not first command, i.e., `i > 0`):**
> - Child `i` should read from the READ end of pipe `i-1`.
> - Read end of pipe `i-1` is at index `2*(i-1)` in `pipe_fds`.
> - Call: `dup2(pipe_fds[2*(i-1)], STDIN_FILENO)`.
> - Effect: fd 0 (stdin) now points to the read end of the previous pipe.
>
> **Stdout wiring (if not last command, i.e., `i < n-1`):**
> - Child `i` should write to the WRITE end of pipe `i`.
> - Write end of pipe `i` is at index `2*i + 1` in `pipe_fds`.
> - Call: `dup2(pipe_fds[2*i + 1], STDOUT_FILENO)`.
> - Effect: fd 1 (stdout) now points to the write end of the current pipe.
>
> After both `dup2` calls, the child's stdin and stdout are wired to the right pipe ends. But the ORIGINAL pipe fd numbers (3, 4, 5, 6...) still exist as separate open descriptors pointing to the same pipes. Those must be closed next."

---

**Q23. "Lines 74-77: After `dup2`, you close ALL pipe fds — including the ones you just dup2'd into stdin/stdout. Why doesn't that close the pipe connection?"**

> "This is the most subtle fd concept in the code. When you call `dup2(pipe_fds[2*(i-1)], STDIN_FILENO)`, two descriptors now point to the same underlying Open File Description in the kernel's Open File Table:
> - `pipe_fds[2*(i-1)]` (e.g., fd 3) → read end of pipe i-1
> - `STDIN_FILENO` (fd 0) → read end of pipe i-1 (same entry)
>
> When you call `close(pipe_fds[2*(i-1)])` (i.e., `close(3)`), you close fd 3. But fd 0 still exists and still points to the read end. The kernel's reference count for that Open File Description drops from 2 to 1. It's not deleted. The pipe channel remains alive via fd 0.
>
> `close()` destroys a file descriptor slot — it decrements the reference count on the underlying Open File Description. Only when the count reaches 0 is the underlying resource freed. This is why closing the original pipe fds after dup2 is safe and necessary — safe because fd 0 still has a reference, necessary because leaving extra descriptors open prevents EOF propagation."

---

**Q24. "Lines 104-107: The parent closes ALL pipe fds after the fork loop. What exact bug occurs if this is omitted?"**

> "The POSIX EOF invariant: a reader on a pipe sees EOF only when ALL write-end descriptors for that pipe — across ALL processes in the system — are closed.
>
> After forking, the parent holds write-end descriptors for EVERY pipe in the pipeline. The children have set up their own wiring via dup2 and closed their pipe fd copies. But the parent's copies are still open.
>
> If the parent skips cleanup:
> - `cmd0` finishes writing and exits. Its write end is closed.
> - `cmd1` reads everything cmd0 wrote, then calls `read()` waiting for more.
> - From the kernel's perspective: the parent still has the write end of pipe 0 open. Reference count = 1 (parent's copy). Not zero. So `cmd1`'s `read()` blocks indefinitely.
> - The entire pipeline hangs. The shell prompt never returns.
>
> This is exactly the bug I encountered and diagnosed with `lsof`. The fix: 4 lines in the parent closing all pipe fds immediately after the fork loop."

---

**Q25. "Lines 113-133: You call `waitpid` in ORDER (child 0, then child 1, ..., then child N-1). Could a child finish out of order? Could this cause a deadlock?"**

> "Yes, children can absolutely finish out of order. `cmd0` might finish before `cmd1`, or `cmd2` might finish before `cmd0` if its work is lighter. The sequential `waitpid` loop handles this correctly because of how `waitpid(specific_pid, ...)` works: it blocks until THAT specific PID changes state, regardless of what other children do.
>
> So if we wait for `child_pids[0]` but child 1 finished first, we keep blocking in `waitpid(child_pids[0])`. Meanwhile child 1 becomes a temporary zombie (its exit status is held by the kernel). Once child 0 finishes, we collect it, then move to collecting child 1 (which is immediately available since it's already a zombie). No deadlock.
>
> Could we deadlock? Only if a child itself blocks waiting for another child that hasn't been waited for — but children don't call `waitpid` on each other here. The only blocking is pipe-related, and since all pipes are closed in the parent and properly wired, data flows correctly."

---

# ══════════════════════════════════════
# FILE: signals.cpp — Async Safety Deep Dive
# ══════════════════════════════════════

---

**Q26. "What exactly is `sigemptyset(&sa.sa_mask)` doing, and what happens if you omit it?"**

> "`sa_mask` is a signal set that specifies which additional signals should be blocked (temporarily suppressed) while the signal handler is executing. `sigemptyset` initializes the set to empty — no additional signals are blocked during handler execution.
>
> If you omit `sigemptyset`: `sa.sa_mask` contains whatever garbage happens to be on the stack from previous function calls. This could have any bit pattern, potentially blocking arbitrary signals during handler execution. For example, if bit 2 (SIGINT) happens to be set by accident, pressing Ctrl+C while the SIGCHLD handler runs would not deliver SIGINT to the shell at all — the signal would be silently deferred. This is a latent, non-deterministic bug that could manifest differently across runs.
>
> Always initialize signal sets explicitly. This is why I call `sigemptyset` before every `sigaction` call."

---

**Q27. "Why `SA_NOCLDSTOP` on the SIGCHLD handler? What specific scenario does it prevent?"**

> "Without `SA_NOCLDSTOP`, SIGCHLD is delivered to the parent not just when a child exits (our case of interest), but also when a child is stopped (receives SIGSTOP, SIGTSTP) or resumed (receives SIGCONT). This is for job control — shells that support Ctrl+Z stopping processes use these notifications.
>
> ShellX doesn't implement job stopping/resuming. If a child is stopped (e.g., via an external `kill -STOP <child_pid>`), without `SA_NOCLDSTOP` we'd get a spurious SIGCHLD, set `g_sigchld_pending = 1`, call `reapFinishedJobs()`, call `waitpid(child_pid, &status, WNOHANG)`, and WNOHANG would return 0 (child isn't dead, just stopped). We'd loop through all jobs finding nothing to reap — wasted work at best, confusing behavior at worst. `SA_NOCLDSTOP` filters these out, delivering SIGCHLD only on actual child exit or signal-kill."

---

**Q28. "Could a second SIGCHLD arrive while the first SIGCHLD handler is still running? What happens?"**

> "By default, while a signal handler is executing, that same signal is blocked (masked). This is specified in POSIX: the signal being handled is automatically added to the process's signal mask for the duration of the handler. So a second SIGCHLD arriving while the handler runs is queued — but importantly, standard Unix signals are NOT queued: only ONE pending signal of a given type can exist at a time. If three children exit simultaneously, the kernel may deliver only ONE SIGCHLD, with the other two being 'merged' into it.
>
> This is why our `reapFinishedJobs()` loops through ALL tracked jobs calling `waitpid(WNOHANG)` for each, rather than just calling it once. A single SIGCHLD notification might represent multiple children finishing. The loop drains all of them."

---

# ══════════════════════════════════════
# FILE: jobs.cpp — Job Table Internals
# ══════════════════════════════════════

---

**Q29. "In `reapFinishedJobs()`, you use `it = g_jobs.erase(it)` in a loop. Why not `g_jobs.erase(g_jobs.begin() + index)` with an index?"**

> "Because `std::vector::erase()` invalidates all iterators FROM the erased element onwards. If I had a raw index and called `erase(begin() + i)`, I'd need to remember not to increment `i` after erasing (since the next element slides into position `i`). Forgetting this produces subtle bugs — skipping elements or accessing out-of-bounds.
>
> The `it = g_jobs.erase(it)` pattern is the canonical C++ idiom for erasing during iteration. `erase()` returns an iterator to the element immediately following the erased one, which is the correct next position. The loop correctly handles two cases: erased element (use returned iterator, don't increment) and non-erased element (increment with `++it`)."

---

**Q30. "Why does `killAllJobs()` NOT call `waitpid()` after sending `SIGTERM`?"**

> "Because the shell is exiting. The comment in the code captures this: *'shell is exiting immediately afterward, so orphaned/terminated children are cleaned up by init/PID 1.'*
>
> When a process dies, its children that are still alive become orphans. The kernel immediately re-parents them to PID 1 (`init` or `systemd`). PID 1 calls `wait()` in a loop to reap all orphaned children. So even if we don't wait, no zombies persist — init adopts and cleans them.
>
> If we called `waitpid()` for each background job: if a background job ignores SIGTERM (which processes are allowed to do), we'd block indefinitely in `waitpid()` waiting for a process that refuses to die. The shell would hang on exit. Sending SIGTERM and immediately exiting is the correct, non-blocking approach. For processes that must be killed, you'd follow up with SIGKILL after a timeout — but that adds significant complexity for a shell exit path."

---

**Q31. "What's the bug if you use `g_jobs.size()` cast to `int` for checking `MAX_BACKGROUND_JOBS` but you added a warning comment saying you 'still track it'?"**

> "This is actually an intentional design decision documented in the code: if the limit is exceeded, we warn the user but still add the job. The alternative — refusing to add the job — would mean the background child runs as a zombie with no tracking. We'd have a running process with no way to manage or reap it.
>
> By tracking beyond the limit (with a warning), the job is at minimum reapable when SIGCHLD fires. The 'limit' is a soft advisory, not a hard enforcement. A better production design would have a hard limit that rejects the fork entirely before it happens, but that requires passing the limit check upward through the call chain."

---

# ══════════════════════════════════════
# FILE: parser.cpp — Lexer & Grammar
# ══════════════════════════════════════

---

**Q32. "In `tokenize()`, why is the `>>` check (two-character lookahead) placed BEFORE the single-character `>` check?"**

> "Greedy matching / maximal munch — the fundamental rule of lexer design. When you see `>>`, if you match `>` first as a single token, you've committed to treating this as a standalone `>` token, and the next loop iteration sees the second `>` as another `>` token. Now the parser sees two consecutive `>` tokens instead of one `>>` token.
>
> The parser would have to handle two consecutive `>` tokens as a special case — contaminating the parser with lexer logic. By checking `>>` first (two-character lookahead: check if `line[i] == '>'` AND `i + 1 < len` AND `line[i+1] == '>'`), we match the longest possible operator, emit a single `">>"` token, and advance `i` by 2. The parser receives a clean, unambiguous stream of tokens."

---

**Q33. "The parser's `rejectUnsupported` function checks `&&` by detecting two consecutive `&` tokens. Explain why `&&` appears as two `&` tokens."**

> "The tokenizer handles `&` as a single-character special token — line 45 in parser.cpp: `if (line[i] == '|' || line[i] == '<' || line[i] == '>' || line[i] == '&')`. So `&&` is scanned as: first character is `&` → emit token `"&"`, advance. Second character is `&` → emit token `"&"`, advance. Result: two consecutive `"&"` tokens.
>
> `rejectUnsupported` then checks: `if (t == "&" && i + 1 < tokens.size() && tokens[i + 1] == "&")` — two adjacent `&` tokens = `&&` operator, which we reject.
>
> An alternative design would tokenize `&&` as a single token by adding a two-character lookahead in the tokenizer (similar to how `>>` is handled). I chose the current approach because `&&` is explicitly unsupported — there's no need to have a clean `&&` token type if we're going to reject it immediately anyway."

---

**Q34. "What happens if a user types `echo hello > out.txt > out2.txt`? Does ShellX handle this? Should it?"**

> "ShellX does NOT detect this. The parser loops through command tokens, and when it sees `> out.txt`, it sets `cmd.output_file = 'out.txt'`. Then when it sees `> out2.txt`, it overwrites: `cmd.output_file = 'out2.txt'`. The Command struct ends up with only the last output file recorded — `out2.txt` — and the `> out.txt` redirection is silently ignored.
>
> Real shells like bash also honor only the last redirection for a given stream (multiple output redirections on the same command — last one wins). So this behavior is actually standard-compliant. However, bash at least opens and immediately closes the earlier redirection files (creating them if they don't exist). ShellX doesn't do this — `out.txt` wouldn't even be created. This is a known gap. A production shell would either warn about redundant redirections or faithfully process all of them in order."

---

**Q35. "Walk through what `parse("echo hello | cat")` returns step by step."**

> "Step 1 — `tokenize`: scans left to right, produces: `['echo', 'hello', '|', 'cat']`.
>
> Step 2 — `rejectUnsupported`: no `;`, no `&&`, no `||`, no `$()`, no backtick. Returns `false` (no rejection).
>
> Step 3 — Trailing `&` check: `tokens.back()` is `'cat'`, not `'&'`. `pipeline.background` stays `false`.
>
> Step 4 — Split on `|`: iterate tokens. `'echo'` → push to current group. `'hello'` → push to current group. `'|'` → current group `['echo', 'hello']` is non-empty, push new group. `'cat'` → push to new group. `command_token_lists = [['echo','hello'], ['cat']]`.
>
> Step 5 — Check trailing pipe: `command_token_lists.back()` = `['cat']`, non-empty. OK.
>
> Step 6 — Check length: 2 commands ≤ 16. OK.
>
> Step 7 — Build Commands:
> - Command 0: no `<`, `>`, `>>`. `args = ['echo', 'hello']`.
> - Command 1: no redirections. `args = ['cat']`.
>
> Returns: `Pipeline { commands: [Command{args:['echo','hello']}, Command{args:['cat']}], background: false }`."

---

# ══════════════════════════════════════
# FILE: redirection.cpp — fd Wiring
# ══════════════════════════════════════

---

**Q36. "In `applyRedirections`, you open the file first, call `dup2`, then `close` the original fd. What happens if you skip the final `close(fd)`?"**

> "You'd have a file descriptor leak. After `dup2(fd, STDIN_FILENO)`, two descriptors point to the same file:
> - fd `STDIN_FILENO` (0) → `input.txt`
> - fd `3` (or wherever the `open()` returned) → same `input.txt`
>
> The child now holds two open references to the same file. This wastes an fd slot. More critically, when the child eventually calls `exec()`, unless the fd is marked with `FD_CLOEXEC`, the leaked fd is inherited by the execvp'd program too. Programs like `ls` or `grep` don't expect to inherit random extra open file descriptors. This can cause resource exhaustion in long pipelines or loops.
>
> Closing the original fd after `dup2` is mandatory. The file stays open via the `STDIN_FILENO` reference — reference count drops from 2 to 1, not to 0."

---

**Q37. "What's the difference between `O_TRUNC` and `O_APPEND`? What happens at the kernel level for each?"**

> "Both are flags for `open()` that control write behavior for existing files.
>
> **`O_TRUNC`** (for `>` redirection): When the file is opened, the kernel immediately sets the file size to zero. The inode's `i_size` field is set to 0, and all data blocks are freed (or marked reusable). Subsequent writes start at byte 0, overwriting any previous content.
>
> **`O_APPEND`** (for `>>` redirection): Before every `write()` syscall, the kernel atomically sets the file offset to the current end of file (`i_size`). The key word is **atomic** — even with multiple processes concurrently appending to the same file, each `write()` is guaranteed to append after all previous writes without interleaving. The offset update and write are a single atomic operation in the kernel, preventing race conditions between concurrent appenders.
>
> `O_APPEND` is how log files work safely in production even with multiple services writing simultaneously."

---

# ══════════════════════════════════════
# FILE: builtins.cpp — Parent-Process Logic
# ══════════════════════════════════════

---

**Q38. "In the `pwd` builtin, you use a fixed `char buf[4096]`. What's the risk of this, and what's the proper way to handle arbitrarily long paths?"**

> "The risk is a potential buffer overflow if the current working directory path exceeds 4096 bytes. On Linux, `PATH_MAX` is typically 4096, so this works in practice. But `PATH_MAX` is advisory, not guaranteed to be the actual maximum path length.
>
> The correct approach for variable-length paths: call `getcwd(NULL, 0)`. When passed `NULL` with size 0, `getcwd` dynamically allocates a buffer of exactly the right size and returns a pointer. You're responsible for calling `free()` on it.
>
> Alternatively, use POSIX `pathconf(_PC_PATH_MAX, fd)` to query the actual limit dynamically. For ShellX's scope, the fixed 4096-byte buffer is adequate and matches how many real-world shell implementations handle it."

---

**Q39. "What does `getenv("HOME")` return if `$HOME` is not set? Where is this handled in the `cd` builtin?"**

> "`getenv` returns `nullptr` (a null pointer) if the environment variable is not set. In `builtins.cpp` lines 17-21:
> ```cpp
> target = std::getenv('HOME');
> if (!target) {
>     std::cerr << 'cd: HOME not set\n';
>     return 1;
> }
> ```
> The null check is mandatory. If we passed `nullptr` to `chdir()`, it would crash with SIGSEGV (segfault) since chdir dereferences the pointer to read the path. Printing a helpful error and returning 1 is the correct user-facing behavior."

---

# ══════════════════════════════════════
# WHAT-IF / MUTATION QUESTIONS
# (Interviewers ask these to test depth)
# ══════════════════════════════════════

---

**Q40. "What happens if you change `pipeline.commands.size() == 1` to `pipeline.commands.size() <= 1` in executor.cpp?"**

> "Functionally identical behavior — the condition already handles size 0 in the empty check on line 95 (`if (pipeline.commands.empty()) return 0`). A pipeline with 0 commands is caught before reaching line 100. So the check at line 100 only ever sees size 1 or greater. Changing to `<= 1` would be equivalent. But `== 1` is more precise and documents intent clearly: 'exactly one command triggers this branch'."

---

**Q41. "What happens if you remove the `static` keyword from `executeSingle` in executor.cpp?"**

> "`static` on a free function at file scope gives it **internal linkage** — it's invisible to other translation units. Without `static`, `executeSingle` would have external linkage, meaning the linker could see it from other `.cpp` files. If any other file happened to define a function also named `executeSingle`, you'd get a linker error: multiple definition.
>
> More importantly, `static` communicates design intent: 'This is an implementation detail of executor.cpp, not part of the public interface.' The public interface is `executePipeline` (declared in `executor.hpp`). Internal helpers should be `static`. This is standard C++ module encapsulation practice."

---

**Q42. "What would happen if you called `executePipelineChain` for a single-command pipeline (N=1)?"**

> "`2 * (n - 1)` = `2 * 0` = `0`. The `pipe_fds` vector would have size 0. The outer pipe creation loop `for (int i = 0; i < n - 1; ++i)` runs 0 times — no pipes created. Then one child is forked. In the child: `i = 0`, `i > 0` is false (no stdin dup2), `i < n - 1` is `0 < 0` which is false (no stdout dup2). The child closes zero pipe fds, applies redirections, and calls `execvp`. Parent closes zero pipe fds and `waitpid` for one child.
>
> It would actually work correctly! But it's still correct to not call it for single commands — `executeSingle` is cleaner and more direct, with proper background job support."

---

**Q43. "What if `MAX_PIPELINE_LENGTH` was set to 1? What commands would break?"**

> "Any pipeline with 2+ commands (`cmd1 | cmd2`) would be rejected by the parser check at `parser.cpp:157`:
> ```cpp
> if (static_cast<int>(command_token_lists.size()) > MAX_PIPELINE_LENGTH)
> ```
> With `MAX_PIPELINE_LENGTH = 1`, `command_token_lists.size() = 2 > 1` would trigger rejection with a 'pipeline too long' error. Single commands (`echo hello`) would still work. The constant effectively limits pipeline depth, acting as a resource safety valve."

---

**Q44. "Why does `reapFinishedJobs()` use `waitpid(it->pid, ...)` with a specific PID instead of `waitpid(-1, ...)`?"**

> "Using `waitpid(-1, ...)` would match ANY child process — including the current foreground command if it happened to finish at the same time. This is the exact race condition I eliminated by design: the signal handler sets a flag, reaping is deferred, and background reaping only happens in `reapFinishedJobs()` when no foreground command is executing (i.e., `executePipeline` has already returned).
>
> But using a specific PID per job (`it->pid`) makes ownership explicit: this `waitpid` call reaps exactly THIS background job, nothing else. It's defensive — even if called at the wrong time, it can't accidentally reap a foreground child with a different PID."

---

# ══════════════════════════════════════
# EXTENSION / DESIGN QUESTIONS
# ══════════════════════════════════════

---

**Q45. "How would you add environment variable expansion (`$HOME`, `$PATH`) to ShellX?"**

> "I'd add a post-tokenization expansion pass. After `tokenize()` returns the token list and before passing to the command builder, I'd scan each token for `$` followed by alphanumeric characters. For each match, extract the variable name, call `getenv(name)`, and substitute the token's `$VAR` substring with the returned value. Handle `nullptr` returns (unset variables) as empty string.
>
> This fits naturally in the parse pipeline: `tokenize()` → `expandVariables()` → `rejectUnsupported()` → command building. The expansion is purely a string transformation on the token list — no structural changes to Command or Pipeline needed."

---

**Q46. "How would you add command history (Up/Down arrow key navigation)?"**

> "Two approaches: (1) Link against GNU readline (`-lreadline`) and replace `std::getline(std::cin, line)` with `readline(prompt)` and `add_history(line)`. Readline handles all terminal control sequences automatically. (2) Implement a minimal version manually: intercept raw terminal input using `termios` to put stdin into raw mode (`cfmakeraw`), detect escape sequences (`\033[A` for Up, `\033[B` for Down), maintain a `std::vector<std::string> history` and an index, and reprint the current history entry on arrow key press.
>
> The first approach (readline) is what production shells use. The manual approach teaches terminal programming — raw mode, escape sequences, and cursor movement."

---

**Q47. "How would you add support for `&&` (run second command only if first succeeds)?"**

> "Three changes:
> 1. **Parser:** Remove `&&` from `rejectUnsupported`. Tokenizer already produces two consecutive `&` tokens. In the parser, detect `t == '&' && tokens[i+1] == '&'`, emit a single `AND_IF` token (or just re-use `&&` as a string token), and advance by 2.
> 2. **Data structures:** Add a new type `ConditionalChain` with `vector<pair<Pipeline, ConnectorType>>` where `ConnectorType` is `AND`, `OR`, or `NONE`. Or simply detect `&&` at the top level and split into multiple pipelines with connectors.
> 3. **Executor:** After each pipeline executes, check the exit status. If connector is `AND` and status != 0, skip the next pipeline. If connector is `OR` and status == 0, skip. Chain the last status through the entire conditional sequence."

---

**Q48. "Your shell has no `stderr` redirection (`2>` or `2>&1`). How would you add it?"**

> "In the `Command` struct, add a field `std::string error_file` and `bool stderr_to_stdout` (for `2>&1`). In the parser, detect `2>` or `2>&1` as tokens (requires tokenizer update to handle `2>` as a two-character token starting with a digit). In `applyRedirections`, add:
> ```cpp
> if (cmd.stderr_to_stdout) {
>     dup2(STDOUT_FILENO, STDERR_FILENO); // 2>&1
> } else if (!cmd.error_file.empty()) {
>     int fd = open(cmd.error_file.c_str(), O_WRONLY | O_CREAT | O_TRUNC, 0644);
>     dup2(fd, STDERR_FILENO);
>     close(fd);
> }
> ```
> The crucial ordering issue: `2>&1` must be processed AFTER stdout redirection. If `cmd > out.txt 2>&1` is processed as `2>&1` first, stderr gets fd 1 (terminal, not file), then `> out.txt` redirects stdout. Result: stdout to file, stderr to terminal — wrong. If `> out.txt` is processed first, fd 1 is now `out.txt`, then `2>&1` copies fd 1 to stderr — both go to the file. Correct."

---

# ══════════════════════════════════════
# CMakeLists.txt — Build System Questions
# ══════════════════════════════════════

---

**Q49. "What does `CMAKE_EXPORT_COMPILE_COMMANDS ON` do, and why is it useful?"**

> "It generates a `compile_commands.json` file in the build directory. This JSON file contains an entry for each source file listing the exact compiler invocation: the compiler binary, all flags, include paths, defines, and the source file path. Tools like `clangd` (LSP server for editors), `clang-tidy` (static analyzer), and `include-what-you-use` read this file to understand exactly how the project is built. Without it, `clangd` makes guesses about include paths and may produce incorrect diagnostics. It's effectively the machine-readable build description that enables tooling integration."

---

**Q50. "What's the difference between `target_compile_options` and `add_compile_options` in CMake?"**

> "`add_compile_options(...)` applies to ALL targets in the CMakeLists.txt and all subdirectories from that point. It's a global modifier. `target_compile_options(shellx PRIVATE ...)` applies ONLY to the `shellx` target. The `PRIVATE` keyword means these flags are only used when compiling `shellx`'s own source files — not propagated to targets that link against it.
>
> For a binary executable like ShellX, the distinction doesn't matter (executables don't propagate compilation flags). But for library targets, using `PRIVATE` vs `PUBLIC` vs `INTERFACE` precisely controls which flags are part of the library's interface contract."

---

# ══════════════════════════════════════
# MASTER CHEAT CARD — Print Before Interview
# ══════════════════════════════════════

```
EXACT CODE LOCATIONS TO KNOW:

  main.cpp:13    → installSignalHandlers() must be before REPL & fork
  main.cpp:20-22 → clear flag BEFORE reapFinishedJobs (not after)
  main.cpp:32    → std::flush (not endl) for prompt
  main.cpp:56    → static_cast<unsigned char> for isspace()
  main.cpp:76-77 → -1 sentinel for exit command detection
  main.cpp:87    → ternary prevents returning -1 as shell exit code

  executor.cpp:25-29 → child resets SIGINT to SIG_DFL after fork
  executor.cpp:33    → _exit() not exit() (double-flush prevention)
  executor.cpp:38    → argv.reserve() pre-allocation
  executor.cpp:45    → const_cast for execvp C API bridge
  executor.cpp:49-53 → ENOENT→127, EACCES→126 exit codes
  executor.cpp:75-77 → EINTR retry loop (belt+suspenders with SA_RESTART)
  executor.cpp:103   → isBuiltin check BEFORE fork

  pipeline.cpp:13    → 2*(n-1) fds for N-1 pipes
  pipeline.cpp:16    → pipe(&pipe_fds[2*i]) — flat array indexing
  pipeline.cpp:19-21 → partial cleanup: close already-opened pipes on failure
  pipeline.cpp:38-44 → partial cleanup: waitpid already-forked children on fork failure
  pipeline.cpp:60    → dup2(pipe_fds[2*(i-1)], STDIN_FILENO) — stdin from prev pipe
  pipeline.cpp:68    → dup2(pipe_fds[2*i+1], STDOUT_FILENO) — stdout to next pipe
  pipeline.cpp:74-77 → close ALL pipe fds AFTER dup2 (safe: dup2'd fds still live)
  pipeline.cpp:104-107 → parent closes ALL pipe fds (prevents EOF hang)
  pipeline.cpp:126   → only last command's status reported (bash default)

  signals.cpp:5-6   → volatile sig_atomic_t (atomic + no-register-cache)
  signals.cpp:11-13 → flag-only handler (async-signal-safe)
  signals.cpp:28    → SA_RESTART + SA_NOCLDSTOP flags

  jobs.cpp:36        → waitpid(..., WNOHANG) — non-blocking poll
  jobs.cpp:32        → erase() returns next valid iterator
  jobs.cpp:72-74     → killAllJobs sends SIGTERM, doesn't waitpid (orphan adoption)

  parser.cpp:38-42  → >> before > in tokenizer (greedy/maximal munch)
  parser.cpp:66-107 → rejectUnsupported: &&, ||, ;, $(), backticks

  redirection.cpp:17 → close(fd) after dup2 (refcount drops 2→1, not 0)
  redirection.cpp:22-27 → O_TRUNC vs O_APPEND
  redirection.cpp:29 → 0644 mode (rw-r--r-- modified by umask)
```
