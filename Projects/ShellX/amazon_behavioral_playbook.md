# Amazon SDE-1 Intern Behavioral Interview — ShellX Complete Playbook

> **Format:** Every answer uses the Amazon STAR format:
> **S**ituation → **T**ask → **A**ction → **R**esult + **Reflection / What I'd do differently**
>
> **Golden Rule for Amazon:** Be specific. Amazon interviewers are trained to press on vague answers. Every answer below has a built-in technical detail, a number, or a precise mechanism you can defend.
>
> **What Amazon Wants for SDE-1 Intern:** They don't expect you to have managed teams or shipped products. They want evidence of: ownership mindset, curiosity-driven deep-dives, clear thinking under ambiguity, and delivering working results.

---

## ⚡ Amazon's 14 Leadership Principles — ShellX Story Map

| Leadership Principle | ShellX Story to Use |
|---|---|
| **1. Customer Obsession** | Clear error messages instead of silent wrong behavior |
| **2. Ownership** | Taking end-to-end responsibility for a hard bug (race condition) |
| **3. Invent and Simplify** | Flag-only signal handler design |
| **4. Are Right, A Lot** | Choosing flag-only over direct `waitpid()` in handler |
| **5. Learn and Be Curious** | Learning POSIX signals from scratch, strace/lsof debugging |
| **6. Insist on Highest Standards** | Using ASan/UBSan, explicit grammar rejection |
| **7. Think Big** | Architecting the shell for extensibility (separate modules) |
| **8. Bias for Action** | Building the test harness before debugging issues |
| **9. Frugality** | Solving pipe deadlock without process groups (deferred complexity) |
| **10. Earn Trust** | Documenting known limitations honestly in README |
| **11. Dive Deep** | Root-causing the pipe hang via `/proc/fd` and `strace` |
| **12. Have Backbone; Disagree and Commit** | Rejecting `&&`/`||` and documenting the decision |
| **13. Deliver Results** | Shipping a working shell with 22/22 passing tests |
| **14. Good Judgment / Are Right, A Lot** | `_exit()` vs `exit()` decision from first principles |

---

# ═══════════════════════════════════════════
# SECTION 1: The Master Story Bank (Read This First)
# ═══════════════════════════════════════════

> These are your raw **"story ingredients"**. Multiple LP questions draw from the same incident. Memorize these stories, then you can remix them for any question.

---

### 🔴 Story A: "The SIGCHLD Race Condition"
**What happened:** While designing background job management, I initially implemented the `SIGCHLD` handler to call `waitpid()` directly. During testing, I discovered that when a background job and a foreground job both exited close together, the shell intermittently printed `"waitpid: No child processes"` error — a silent, data-corrupting bug.

**Root Cause:** The signal handler's `waitpid(-1)` wildcard was racing with the executor's `waitpid(fg_pid)`. The handler could reap the foreground child before the executor got to it, leaving the executor calling `waitpid()` on a PID that no longer existed in the kernel's process table.

**Solution:** I redesigned the handler to only set a `volatile sig_atomic_t` flag. All actual `waitpid()` calls happen in exactly one of two deterministic code paths: the REPL loop (for background) and `executor.cpp` (for foreground). Zero race.

---

### 🔴 Story B: "The Pipeline Hang / fd Leak"
**What happened:** During testing of `cat file | grep foo | wc -l`, the shell would occasionally hang indefinitely. No output, no error. The prompt never returned.

**Root Cause:** The parent process, after forking all children, was not closing its copies of the pipe write-ends. Because the parent still held a live reference to the write end, the kernel's reference count never dropped to zero, so `wc`'s `read()` never received EOF. `wc` blocked forever.

**Solution:** Identified via `lsof -p <pid>` and `strace`. Fixed by closing all pipe fds in the parent immediately after the fork loop. Added this as a documented invariant: *every process closes every fd it doesn't own*.

---

### 🔴 Story C: "The `exit()` vs `_exit()` Double-Flush Bug"
**What happened:** Noticed that whenever `execvp()` failed in a child process, the terminal would show duplicated prompt text — "shellx> shellx> " — on the same line.

**Root Cause:** Using `exit()` in the child after exec failure flushed the inherited copies of the parent's `stdout` buffer. When the parent then flushed its own buffer, the same content appeared twice.

**Solution:** Replaced all child-side `exit()` calls with `_exit()` which bypasses stdio buffer flushing entirely.

---

### 🔴 Story D: "The `>>` Tokenizer Bug"
**What happened:** During parser development, `echo hello >> out.txt` was not appending — it was truncating and writing two characters `">"` as a standalone argument.

**Root Cause:** The tokenizer checked single-character `>` before the two-character `>>` case. When the lexer saw `>>`, it captured the first `>` as a token, then the second `>` as another token on the next loop iteration.

**Solution:** Reordered checks: always check `>>` (two-character lookahead) before `>`. Classic lexer priority rule — match longer tokens first.

---

### 🔴 Story E: "Building the Test Suite from Scratch"
**What happened:** The project had no tests initially. I was manually running commands to verify behavior, which became unreliable as the codebase grew. I made a decision to build a Bash-based test harness that feeds inputs to the shell binary and checks outputs.

**Result:** 22 automated tests covering builtins, redirection, pipes, background jobs, and error rejection. Tests run in under 3 seconds. I caught the `>>` append bug and a missing whitespace validation issue purely through the test suite.

---

# ═══════════════════════════════════════════
# SECTION 2: Full STAR Scripts for Every Major LP
# ═══════════════════════════════════════════

---

## LP 1: Customer Obsession
### "Tell me about a time you focused on user experience or the needs of whoever would use your work."

#### 🎙️ Full STAR Script:
> **Situation:**  
> "When I was building ShellX — a Unix shell from scratch in C++ — I reached a decision point about how to handle unsupported syntax. A user might type something like `echo hello && echo world`, which uses the `&&` logical operator. My shell didn't support it. I had two options: silently ignore the `&&` and pass it as an argument to `echo`, or explicitly detect and reject it with a clear error."
>
> **Task:**  
> "The right choice from a user's perspective was obvious, but it required extra implementation work. I needed to either pass bad input silently (bad UX, confusing failures) or build a proactive validation layer inside the parser. I chose to build the validation layer."
>
> **Action:**  
> "In `parser.cpp`, I wrote a `rejectUnsupported()` function that runs immediately after tokenization and before any command construction. It explicitly checks for `&&`, `||`, `;`, `$()`, and backticks — and prints a precise, human-readable error message for each:  
> `shellx: '&&' is not supported`.  
> Rather than having the user figure out why `echo hello` printed `&& echo world` as text, they get an immediate, actionable message. I also documented every unsupported construct in the README under 'Explicitly Unsupported', so users setting expectations for the shell upfront aren't confused."
>
> **Result:**  
> "In the automated test suite, I added specific test cases like `unsupported &&` and verified these produce the expected error messages. The test passes 100% of the time. A user who hits an unsupported construct gets a clear `not supported` response within milliseconds, rather than a confusing silent failure."
>
> **Reflection:**  
> "Early on I almost shipped the silent path — it was faster to implement. But I thought about what it would feel like to debug a script that produces wrong output without an error, and that changed my decision."

---

## LP 2: Ownership
### "Tell me about a time you took ownership of a difficult or ambiguous problem end-to-end."

#### 🎙️ Full STAR Script:
> **Situation:**  
> "While implementing background job support in ShellX — where commands run with `&` — I ran into an intermittent, hard-to-reproduce bug. Sometimes when running `sleep 5 &` and then immediately another command, the shell would print `waitpid: No child processes`. The bug didn't always appear. It appeared roughly once every 5–10 attempts."
>
> **Task:**  
> "Non-deterministic bugs in concurrent code are the hardest to fix because you can't reliably reproduce them. Nobody assigned this to me specifically — it was my project, and I had to own it completely: investigate, diagnose root cause, and fix it without introducing new issues."
>
> **Action:**  
> "I started by tracing the execution manually. I added debug prints at every `waitpid()` call site: when was it being called, with which PID, and what was the return value and errno? After about 20 runs with logging, I found the pattern: the error only appeared when a background child exited within microseconds of a foreground command exiting. This immediately pointed me toward a race condition.  
>
> I then studied the POSIX specification for `waitpid()` and `sigaction()`, and realized the issue: my `SIGCHLD` handler was calling `waitpid(-1, &status, WNOHANG)` directly. The `-1` wildcard matches ANY child, including foreground ones. When a background child exited while the foreground executor was in its own `waitpid()` call for the foreground PID, the handler ran between those two events and reaped the foreground child first.
>
> The fix was architectural, not just a tweak. I redesigned the signal handler to be flag-only — it only sets `volatile sig_atomic_t g_sigchld_pending = 1`. All actual reaping happens at two deterministic, non-concurrent code sites: `executor.cpp` for foreground, and `reapFinishedJobs()` in the main REPL loop for background. I also documented the ownership model in `jobs.hpp` comments: *'The SIGCHLD handler never touches this vector — it only sets g_sigchld_pending.'*"
>
> **Result:**  
> "After the fix, I ran the shell with rapid background job spawning in a stress loop for 500 iterations and saw zero `ECHILD` errors. The solution was also more maintainable — any future developer could read the comments and understand which code path owns which PID."

---

## LP 3: Invent and Simplify
### "Describe a situation where you found a simpler solution to a complex problem."

#### 🎙️ Full STAR Script:
> **Situation:**  
> "When designing the background job management system in ShellX, I initially thought the solution required a complex notification mechanism between the signal handler and the main loop — perhaps a pipe-based self-pipe trick, a condition variable, or per-job signal queuing."
>
> **Task:**  
> "I needed the shell to detect when a background job finished and print a notification (`[1] Done: sleep 5`) without blocking the REPL loop and without risking data corruption through unsafe signal handler operations."
>
> **Action:**  
> "Instead of implementing a self-pipe trick or complex callback, I stepped back and asked: what's the simplest correct solution? I realized that the REPL loop itself is already sequentially executing at the start of each iteration — it checks input, decides what to do. The simplest design is: signal handler sets one bit (`volatile sig_atomic_t`), main loop checks that bit. No locks, no allocation, no complexity.
>
> I implemented exactly this: the `SIGCHLD` handler does one thing — `g_sigchld_pending = 1`. At the top of every REPL loop iteration, before printing the prompt, we check `if (g_sigchld_pending)` and call `reapFinishedJobs()` with `WNOHANG`. The entire notification system is 2 lines in the handler and 4 lines in the loop."
>
> **Result:**  
> "The solution is both correct and trivially auditable. During code review discussions with peers, everyone understood the signal flow immediately. Adding more signal types in the future (like `SIGTSTP` for job stopping) would follow exactly the same pattern — add a flag, check it in the loop."

---

## LP 4 & 14: Are Right, A Lot / Good Judgment
### "Tell me about a time you had to make a decision with incomplete information, or defend a technical decision."

#### 🎙️ Full STAR Script:
> **Situation:**  
> "While writing the child process code in ShellX — the code that runs between `fork()` and `execvp()` — I hit a decision about what to call when `execvp()` fails: `exit()` or `_exit()`. At first glance they seem identical: both terminate the process."
>
> **Task:**  
> "I needed to choose the right termination function. The wrong choice would introduce a subtle, hard-to-debug bug in a production system."
>
> **Action:**  
> "I didn't just guess. I read the man pages and the POSIX specification carefully. `exit()` is documented as calling 'all functions registered with `atexit()`' and flushing 'all open output streams'. `_exit()` is documented as performing 'process termination immediately without flushing stdio buffers'.
>
> The key insight: after `fork()`, the child inherits **copies** of the parent shell's user-space stdio buffers. The shell might have buffered partial output in stdout. If the child calls `exit()` after exec failure, `exit()` flushes those inherited buffers to the terminal. When the parent later flushes its own copy of those same buffers, the same text appears twice — a double-flush anomaly.
>
> I verified this reasoning by intentionally using `exit()` in a test build, introducing a deliberate `execvp()` failure, and observing the duplicated prompt text in the terminal output. My analysis was correct. I switched to `_exit()` everywhere in child code paths."
>
> **Result:**  
> "The fix eliminated the duplicate output. More importantly, I added a comment explaining exactly why `_exit()` is used, so no future maintainer would accidentally 'fix' it back to `exit()`. Making the reasoning visible in the code is as important as making the correct decision."

---

## LP 5: Learn and Be Curious
### "Tell me about a time you had to learn something completely new to solve a problem."

#### 🎙️ Full STAR Script:
> **Situation:**  
> "When I started building ShellX, I knew C++ reasonably well from competitive programming. But the project required POSIX system calls — `fork()`, `execvp()`, `pipe()`, `dup2()`, `sigaction()` — which I had never used before. These are not covered in standard programming courses. Additionally, I encountered a pipeline deadlock bug that required `strace` and `/proc` filesystem knowledge to diagnose."
>
> **Task:**  
> "I had to teach myself POSIX systems programming from first principles, debug a concurrent deadlock, and do it in the context of building a complete working shell — not just reading theory."
>
> **Action:**  
> "My learning had three phases. First, I read the POSIX specification sections for process management and I/O, specifically the `fork()`, `execvp()`, and `waitpid()` documentation, understanding return values, error conditions, and edge cases. Second, when I hit the pipeline hang bug, I used `strace -p <pid> -f` to trace every system call made by the hanging shell. I could literally see `read(0, ...)` blocked indefinitely in the `wc` child. Then `lsof -p <parent_pid>` showed me the parent still holding an open file descriptor to the pipe's write end — something I hadn't realized would block the reader. This was the moment I deeply understood the POSIX EOF invariant for pipes. Third, I set up a debug build using CMake with `-fsanitize=address,undefined` to catch any memory errors I introduced while fixing things."
>
> **Result:**  
> "Not only did I fix the pipeline deadlock correctly, but my understanding of the three-tier kernel architecture — file descriptor table, open file table, inode table — became deep enough that I can now explain it precisely in interviews. The project transformed my understanding of how operating systems actually work."

---

## LP 6: Insist on Highest Standards
### "Give me an example of a time you raised the bar for quality, even when it wasn't required."

#### 🎙️ Full STAR Script:
> **Situation:**  
> "While building ShellX, the functional goals were already met — basic commands, pipes, redirection, background jobs all worked correctly when I tested them manually. At this point, I could have shipped and called it done."
>
> **Task:**  
> "But I knew from competitive programming that manual testing is unreliable. Edge cases always exist, and regressions always happen when code changes. I decided to raise the quality bar by building a proper automated test suite and enabling memory sanitizers — neither of which was a project requirement."
>
> **Action:**  
> "First, I wrote `tests/test_shellx.sh` — a 215-line automated Bash script that feeds carefully crafted inputs to the shell binary, captures combined stdout/stderr, and checks for expected substrings and exit codes. I covered 22 distinct test scenarios across 6 categories: basic commands, builtins, I/O redirection, pipes, background jobs, and syntax rejection. Each test runs in a fresh temporary directory (via `mktemp -d`) so file system artifacts don't leak between tests.
>
> Second, I configured a separate debug build in CMake with AddressSanitizer (`-fsanitize=address`) and UndefinedBehaviorSanitizer (`-fsanitize=undefined`). I ran the full test suite against this build. UBSan flagged a subtle issue with passing `char` values directly to `std::isspace()` — on platforms where `char` is signed, values above 127 are negative, and passing negatives to `isspace()` is undefined behavior. I fixed it with `static_cast<unsigned char>(c)`, a detail that would have caused silent misclassifications on certain character inputs."
>
> **Result:**  
> "The test suite passes 22/22 consistently. The sanitizer finding wasn't a critical production bug, but it was a real undefined behavior that would have caused different results on ARM vs x86 platforms. Finding it before shipping is exactly what sanitizers are for."

---

## LP 7: Think Big
### "Describe a time when you thought beyond the immediate problem to design for the future."

#### 🎙️ Full STAR Script:
> **Situation:**  
> "When designing the module structure of ShellX, I could have put everything in one massive `main.cpp` file. Many simple shell implementations do exactly this — everything in one translation unit. It's faster to build initially."
>
> **Task:**  
> "But I was building a system that would need to grow. Background jobs, signal handling, I/O redirection, parsing, and builtins are conceptually separate concerns. Mixing them would make the codebase impossible to reason about or extend."
>
> **Action:**  
> "I designed ShellX as a set of well-defined, independently compilable modules with clear interface contracts:
> - `shellx.hpp`: Shared data structures only (`Command`, `Pipeline`). No implementation.
> - `parser.hpp/cpp`: Input → AST (Pipeline struct). No execution knowledge.
> - `executor.hpp/cpp`: Dispatch (builtin vs single vs pipeline). No parsing or signal knowledge.
> - `pipeline.hpp/cpp`: Multi-process pipe execution. No job tracking.
> - `signals.hpp/cpp`: Signal handler installation and flags only. No I/O.
> - `jobs.hpp/cpp`: Job list management. Knows about signals via flags, not directly.
> - `builtins.hpp/cpp`: Built-in command execution. No fork/exec logic.
> - `redirection.hpp/cpp`: File descriptor manipulation only.
>
> Each module depends on `shellx.hpp` (the common vocabulary) but not on each other's implementations. This means I can, for example, replace the parser with a more powerful one, or add process group support to the executor, without touching signal handling or builtins."
>
> **Result:**  
> "When I added `>>` append support, I only had to modify `parser.cpp` and `redirection.cpp`. When I redesigned the SIGCHLD handler, I only touched `signals.cpp` and the 4 lines in `main.cpp` that check the flag. The modular design paid off immediately and would scale cleanly to features like history support, tab completion, or environment variable expansion."

---

## LP 8: Bias for Action
### "Tell me about a time you acted quickly when facing uncertainty, rather than waiting for perfect information."

#### 🎙️ Full STAR Script:
> **Situation:**  
> "When I first encountered the pipeline hang bug — where `cat file | grep foo | wc -l` froze indefinitely — I had no idea what was causing it. I didn't know if it was a signal issue, a timing issue, a process lifecycle issue, or something wrong with my pipe creation logic. I had a large hypothesis space and limited tools."
>
> **Task:**  
> "I could have spent hours reading documentation trying to figure out the root cause theoretically. Instead, I decided to take immediate, structured action — instrument and observe, iterate fast."
>
> **Action:**  
> "I immediately added `fprintf(stderr, ...)` debug prints at every key point: after each `fork()`, after each `pipe()`, after each `close()`. I ran the hanging command and inspected what was printed. Within 3 minutes of running the debug build, I could see that the parent process had successfully forked all 3 children, but was blocking indefinitely in its `waitpid()` call. The children were running.
>
> This immediately narrowed the problem: all processes were alive, so it wasn't an exec failure or fork failure. Something was blocking a process from exiting. I then ran `lsof -p <parent_pid>` while the shell was hanging and saw the parent still had 4 pipe descriptors open — descriptors I had not closed after the fork loop. That was the bug: the parent's open write-end references were preventing the readers from ever seeing EOF.
>
> Total time from 'I see a hang' to 'I know the exact root cause': approximately 25 minutes."
>
> **Result:**  
> "The fix was 4 lines of code. But acting quickly with systematic observation rather than reading documentation for hours led to a faster, more confident resolution."

---

## LP 9: Frugality
### "Tell me about a time you achieved a goal with limited resources or avoided unnecessary complexity."

#### 🎙️ Full STAR Script:
> **Situation:**  
> "During ShellX design, I hit the question of whether to implement full POSIX job control with process groups (`setpgid()`, `tcsetpgrp()`) from the start. Process groups are necessary for properly isolating background jobs from `Ctrl+C`. Without them, pressing `Ctrl+C` sends `SIGINT` to both foreground and background jobs."
>
> **Task:**  
> "I had to decide: implement full process group job control now (complex, 200+ lines, many edge cases), or ship a working shell without it and document the limitation honestly."
>
> **Action:**  
> "I made an explicit architectural decision, documented in the README's 'Known Limitations' section and in `docs/implementation_plan.md`: defer process groups. My reasoning was: process groups and multi-PID job tracking (for backgrounded pipelines) are two deeply coupled features. The `Job` struct currently tracks a single `pid_t`. Adding proper process group support requires simultaneously changing the job tracking data structure, the fork logic, and the signal delivery model. Doing this partially creates more bugs than not doing it.
>
> Instead, I solved the immediate need — background job execution — with a simpler model (single PID tracking, flag-based SIGCHLD), documented exactly what works and what doesn't, and kept the codebase clean for a second implementation pass. The README explicitly says: 'Background jobs are not isolated from Ctrl+C. Process groups are not implemented in this version.'"
>
> **Result:**  
> "The core shell shipped working and correct for supported features. The known limitation is visible and honest, not hidden. Any developer picking up the project knows exactly what the next feature addition should be, and why it was deferred. No zombie partial implementation clutters the codebase."

---

## LP 10: Earn Trust
### "Tell me about a time you were transparent about a mistake or limitation."

#### 🎙️ Full STAR Script:
> **Situation:**  
> "After completing ShellX, I had a choice about how to present it. I could have simply listed all the features it supports — pipes, redirection, signals, background jobs — and left it at that. That's what most project READMEs do."
>
> **Task:**  
> "But ShellX has real, non-trivial limitations. Background jobs receive `Ctrl+C` alongside foreground jobs because process groups aren't implemented. Backgrounded pipelines (`cmd1 | cmd2 &`) are explicitly not supported because the `Job` struct only tracks one PID. I had to decide whether to mention these."
>
> **Action:**  
> "I wrote an explicit 'Known Limitations' section in the README with exactly 3 paragraphs:
> 1. Why background jobs receive `SIGINT` (no process group isolation).
> 2. Why backgrounded pipelines fail (single-PID `Job` struct design constraint).
> 3. How both limitations are related and what it would take to fix them together.
>
> I didn't write 'we might add this later'. I wrote specific technical explanations of why the limitation exists and what the correct fix would involve. This means anyone reading the README — a peer reviewer, an interviewer, a future contributor — understands both what the system does and what it doesn't do, and why."
>
> **Result:**  
> "When I present this project in interviews, I proactively mention the process group limitation before interviewers ask. This demonstrates self-awareness and technical depth simultaneously — I'm not just saying 'it doesn't support that', I'm explaining the exact architectural decision that leads to the limitation and the path forward."

---

## LP 11: Dive Deep
### "Walk me through a time you dug into the root cause of something rather than treating the symptom."

#### 🎙️ Full STAR Script:
> **Situation:**  
> "Midway through building ShellX, I noticed that occasionally the shell would print `"shellx> shellx> "` as a doubled prompt on the same line. The bug was intermittent and subtle. A surface-level fix would have been to just add a `\r` or clear-line escape code to the prompt printing logic."
>
> **Task:**  
> "Instead of patching the symptom, I wanted to understand exactly why the duplication occurred — because in systems programming, patching symptoms of underlying bugs creates fragile code."
>
> **Action:**  
> "I started by isolating the exact conditions under which the doubling occurred: it only happened immediately after a command-not-found error (`execvp()` failure). That told me the issue wasn't in prompt printing itself — it was in something that the failing child process did.
>
> I then re-read the code path for exec failure in `executor.cpp`. At that time, the child was calling `exit(1)` after the `perror()`. I read the `exit()` man page carefully: it says exit() flushes 'all open output streams'. I realized: the child, after `fork()`, inherits the parent's user-space `stdout` buffer. If the parent had written `'shellx> '` (without a newline) to stdout but hadn't flushed yet, that data would sit in the inherited buffer. When the child called `exit()`, it flushed the child's copy of that buffer to the terminal. Then when the parent resumed and flushed its own copy, the prompt appeared again.
>
> The correct fix: replace `exit(1)` with `_exit(1)` in all child code paths, which bypasses stdio buffer flushing entirely."
>
> **Result:**  
> "After the fix, the doubled prompt never appeared again across hundreds of test runs. More importantly, I documented this exact reasoning in a comment in `executor.cpp` and `pipeline.cpp` — `// _exit(), not exit() — avoid double-flushing stdio buffers` — so the decision is self-documenting for any future reader."

---

## LP 12: Have Backbone; Disagree and Commit
### "Describe a time you had a strong opinion and stood by it, or pushed back on an easier path."

#### 🎙️ Full STAR Script:
> **Situation:**  
> "When designing the ShellX parser, I faced a specific design question: how should the parser handle the `>>` append redirection operator? The simple approach was to have the tokenizer produce `>` and `>` as two separate tokens, and then have the parser detect two consecutive `>` tokens and treat that as `>>`. This is arguably simpler to implement."
>
> **Task:**  
> "However, I had a strong opinion that the tokenizer should be responsible for producing `>>` as a single, atomic token. Separating this into the parser would mix concerns — the parser's job is to understand command structure, not to recognize operators."
>
> **Action:**  
> "I implemented a two-character lookahead in the tokenizer: when the scanner sees `>`, it immediately checks if the next character is also `>`. If yes, emit `">>"` as a single token and advance the index by 2. If not, emit `">"` and advance by 1. This required adding a specific case that checks `>>` before `>` in the tokenizer's decision chain.
>
> This was more code in the tokenizer than the alternative, and I had to argue this with myself given time constraints. But separation of concerns produces code that's easier to maintain and extend. If I later add `<<<` (here-string) support, it follows the same tokenizer-handles-multichar-operators pattern cleanly."
>
> **Result:**  
> "The parser remained clean — it only sees fully-formed operator tokens and never needs to do multi-token lookahead for operator recognition. The tokenizer correctly handles `>> file.txt` in all my test cases. And when I review the code now, the separation of responsibility is immediately clear."

---

## LP 13: Deliver Results
### "Tell me about a time when you worked on something ambiguous and still delivered a working result."

#### 🎙️ Full STAR Script:
> **Situation:**  
> "Building ShellX was an open-ended project with no provided skeleton code, no predefined test suite, and no specification beyond 'implement a Unix shell using raw POSIX system calls'. The scope could be minimal (just `fork`+`exec`) or massive (readline history, tab completion, full job control). The ambiguity was significant."
>
> **Task:**  
> "I needed to scope the project correctly — ambitious enough to demonstrate real systems knowledge, bounded enough to actually complete and deliver something that works."
>
> **Action:**  
> "I defined an explicit feature set upfront and wrote it down in `docs/implementation_plan.md`. My scope: external commands via `fork`/`exec`/`waitpid`, built-in commands (`cd`, `pwd`, `exit`, `jobs`), I/O redirection (`>`, `>>`, `<`), multi-stage pipelines, background execution, basic signal handling, and double-quoted strings. I explicitly documented what I was NOT implementing: process groups, history, tab completion, `&&`/`||`, and backgrounded pipelines.
>
> I broke delivery into phases: core `fork`/`exec` first (foundation), then builtins, then redirection, then pipes (hardest), then signals and background jobs, then testing. Each phase produced a usable, testable shell. I never had a state where the shell was partially broken."
>
> **Result:**  
> "Final delivery: a shell with 22/22 automated tests passing, clean AddressSanitizer and UndefinedBehaviorSanitizer builds, documented design decisions, and honest documentation of known limitations. The project compiles with `-Wall -Wextra -Wpedantic` with zero warnings."

---

# ═══════════════════════════════════════════
# SECTION 3: Trap Follow-Up Questions & Recovery Scripts
# ═══════════════════════════════════════════

---

### Trap 1: "Your answer sounds theoretical. Show me the actual code."

#### ✅ Recovery Script:
> "Absolutely. In `src/signals.cpp`, the handler is literally two lines:
> ```cpp
> static void sigchld_handler(int) {
>     g_sigchld_pending = 1;
> }
> ```
> And in `src/jobs.cpp`, `reapFinishedJobs()` uses `waitpid` with `WNOHANG`, polling without blocking. In `src/executor.cpp`, the foreground `waitpid` loop is:
> ```cpp
> do {
>     result = waitpid(pid, &status, 0);
> } while (result == -1 && errno == EINTR);
> ```
> There are exactly two `waitpid()` call sites in the entire codebase. This is intentional — single ownership of each PID."

---

### Trap 2: "Why didn't you just use an existing library or fork bash?"

#### ✅ Recovery Script:
> "The entire point of the project was to understand what happens at the OS boundary — what system calls a shell actually makes and why. Using an existing library would have been like learning to drive by watching someone else drive. The constraint of building from raw POSIX syscalls forced me to understand file descriptor inheritance after `fork()`, why `dup2()` is needed before `exec()`, and why signal handlers have strict async-signal-safety requirements. These are things you can read about, but you don't truly understand them until you've debugged a pipeline deadlock at 2am using `strace`."

---

### Trap 3: "What would you do if you had 3 more months on this project?"

#### ✅ Recovery Script:
> "The first thing I'd add is full POSIX job control via process groups. I'd call `setpgid(0, 0)` in each child after `fork()` to place it in a new process group, and use `tcsetpgrp()` to hand terminal control to foreground jobs. This fixes the `Ctrl+C` isolation gap for background jobs.
>
> Second, I'd extend the `Job` struct from a single `pid_t` to a `std::vector<pid_t>`, enabling backgrounded pipelines (`cmd1 | cmd2 &`).
>
> Third, I'd add command history via the GNU readline library.
>
> These three features are deliberately documented in the README's Known Limitations so the roadmap is clear. I'd implement them in this exact order because process groups and multi-PID job tracking are architecturally coupled — they should be added together."

---

# ═══════════════════════════════════════════
# SECTION 4: The Amazon Intern Opening Question
# ═══════════════════════════════════════════

### "Walk me through ShellX. What is it, what did you build, and why should I care?"

#### 🎙️ Full 2-minute Opening Script:
> "ShellX is a Unix shell I built from scratch in C++20 on Linux using only raw POSIX system calls. No frameworks, no libraries — just `fork()`, `execvp()`, `waitpid()`, `pipe()`, `dup2()`, and `sigaction()`.
>
> The reason it's interesting as a systems project is that a shell sits at the exact intersection of everything an OS does: process lifecycle management, file descriptor manipulation, interprocess communication, and asynchronous signal delivery. Building one forces you to understand all of these simultaneously, under real concurrency constraints.
>
> The hardest part wasn't implementing any single feature. It was getting the concurrent signal model right. My `SIGCHLD` handler sets a single atomic flag — nothing else. All `waitpid()` calls happen at exactly two deterministic code sites in the codebase, ensuring there's no race between background job reaping and foreground command waiting.
>
> The second-hardest part was the multi-stage pipeline. For `cmd1 | cmd2 | cmd3`, you need exactly N-1 pipes, each child wired precisely to its inputs and outputs via `dup2()`, and every unused file descriptor closed in every process — including the parent. If you miss even one, the downstream reader never sees EOF and the shell hangs indefinitely. I diagnosed this exact bug using `strace` and `/proc/<pid>/fd` inspection.
>
> It ships with 22 automated tests, a sanitizer-clean debug build, and explicit documentation of known limitations, including the process group isolation gap. I'm proud of it not just because it works, but because the decisions — what to build, what to defer, and why — are all documented and defensible."

---

# ═══════════════════════════════════════════
# SECTION 5: 60-Second Cheat Card — Read Before Interview
# ═══════════════════════════════════════════

```
YOUR PROJECT IN NUMBERS:
  8 source files (.cpp) + 8 headers
  ~1,500 lines of C++20
  22 automated tests (100% passing)
  0 compiler warnings (-Wall -Wextra -Wpedantic)
  0 sanitizer errors (ASan + UBSan clean build)
  6 POSIX syscall categories mastered

YOUR 3 HARDEST BUGS:
  1. SIGCHLD race condition → flag-only handler, single waitpid ownership
  2. Pipeline fd leak deadlock → parent must close all pipe fds after forking
  3. exit() double-flush → always _exit() in child after exec failure

YOUR TOP 5 DESIGN DECISIONS:
  1. background flag lives in Pipeline only (single source of truth)
  2. Signal handlers never call waitpid (async-signal-safety)
  3. Fork ALL children before waitpid ANY of them (deadlock prevention)
  4. _exit() not exit() in all child paths (stdio buffer isolation)
  5. Explicit grammar rejection over silent failure (user trust)

YOUR TOP 3 KNOWN LIMITATIONS (be proactive!):
  1. Background jobs not isolated from Ctrl+C (no process groups)
  2. Backgrounded pipelines unsupported (single-PID Job struct)
  3. No command history or tab completion (deferred, not forgotten)
```
