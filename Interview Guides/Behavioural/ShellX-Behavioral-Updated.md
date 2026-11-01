# 🎯 Behavioral Interview Prep — ShellX (Unix Shell in C++)
### Amazon SDE-1 Intern Edition — merged and corrected

You built **ShellX**, a Unix shell from scratch in C++20 using raw POSIX system calls.

Use the **STAR format**: Situation → Task → Action → Result (+ a one-line reflection where it helps).

**Speaking guidelines**
- Keep each answer under **2 minutes** out loud.
- Say **"I"**, not "we". Amazon interviewers score *your* actions.
- Amazon interviewers press on vague answers. Every story below has a mechanism, a number, or a file/function name you can defend.
- Amazon currently has **16** Leadership Principles (two were added in 2021). The tags on each story use that list.

---

## 🏗️ Your Role — Quick Intro

*Use this for: "Tell me about your project." / "What did YOU specifically build?" / "Why should I care?"*

> "I built **ShellX**, a Unix shell from scratch in C++20 on Linux using only raw POSIX system calls. No libraries, no frameworks. It runs external programs via `fork()` and `execvp()`, supports multi-stage pipelines like `cmd1 | cmd2 | cmd3`, I/O redirection (`>`, `>>`, `<`), background jobs (`sleep 5 &`), and signal handling, using `pipe()`, `dup2()`, `waitpid()` and `sigaction()`.
>
> A shell sits at the intersection of everything an OS does: process lifecycle, file descriptors, IPC, and asynchronous signals. The hardest parts weren't any single feature. They were the concurrency details: a `SIGCHLD` race between background and foreground reaping, and a pipeline hang caused by the parent leaking pipe file descriptors.
>
> It ships with 22 automated tests, a sanitizer-clean debug build (ASan + UBSan), and documented known limitations. I'm proud of it because the decisions about what to build, what to defer, and why are all written down and defensible."

---

## ⭐ Story 1 — The Pipeline Hang Bug
**LPs:** Dive Deep · Bias for Action · Ownership
**Question types:** "Tell me about a bug you debugged." / "Tell me about a time you dug into the root cause." / "Tell me about acting without complete information."

- **S — Situation:** After implementing multi-stage pipelines, I ran `cat file.txt | grep foo | wc -l` and the shell froze. No output, no error, and the prompt never came back. Pipelines are a core shell feature, so this was a critical bug.
- **T — Task:** Find exactly why it hung and fix it without breaking the single-command and two-stage pipes that already worked.
- **A — Action:** I didn't start by reading docs. I added `fprintf(stderr, ...)` after every `fork()`, every `close()`, and at each `waitpid()`. Within minutes the picture was clear: the parent was stuck in `waitpid()`, all three children were alive, and `wc` was blocked in `read()`. Since nothing had crashed, this wasn't an exec or fork failure. Something was keeping the pipe open. I ran `lsof -p <parent_pid>` while it was hung and saw the parent still holding the pipe **write-end** descriptors it got from `pipe()`. The kernel only delivers EOF to a reader when *every* write-end reference is closed, and the parent's copies kept that from ever happening. The fix was 4 lines: right after the fork loop, the parent closes every pipe fd.
- **R — Result:** The hang was gone and all multi-stage pipes terminate. From first seeing the hang to knowing the root cause took about 25 minutes. I wrote the rule into the code as an invariant: *"Every process closes every fd it doesn't own."* A pipeline test (`cat | sort | uniq`) is now in the 22-test suite so it can't regress.

> 💡 **If they ask: "Show me the fix."**
> ```cpp
> // Parent: close ALL pipe fds immediately after all forks
> for (int j = 0; j < 2 * (n - 1); ++j) {
>     close(pipe_fds[j]);
> }
> ```
> 💡 **If they ask: "Why does it hang every time, not just sometimes?"**
> "Because the parent always holds the write ends, so the last reader never gets EOF. It's deterministic. What varies is only which stage you see blocked."
> 💡 **If they ask: "Why fork all children before waiting on any?"**
> "If I waited on stage 1 before forking stage 2, stage 1 could fill the pipe buffer and block on write with no reader yet. Fork everything first, then wait."

---

## ⭐ Story 2 — The Double-Flush Bug (`exit()` vs `_exit()`)
**LPs:** Are Right, A Lot · Dive Deep · Insist on the Highest Standards
**Question types:** "Tell me about a subtle detail you caught." / "Tell me about a time you made a technical decision from first principles." / "Tell me about fixing the cause, not the symptom."

- **S — Situation:** Whenever a command didn't exist (a typo, say), the shell printed the error and then `shellx> shellx> ` with the prompt doubled on one line. The shell kept working, so it was easy to dismiss.
- **T — Task:** Understand *why* the prompt doubled and fix the cause. A quick patch, like printing `\r` before the prompt, would have hidden it.
- **A — Action:** Because it only happened after `execvp()` failed, I knew the bug was in what the failing child did. The child was calling `exit(1)`. The man page says `exit()` flushes all open stdio streams. After `fork()`, the child inherits a **copy** of the parent's user-space `stdout` buffer, which still held the unflushed `"shellx> "`. The child flushed its copy, then the parent flushed its own copy later, so the text printed twice. To confirm rather than assume, I reproduced it deliberately with `exit()` and a forced exec failure, then switched every child-side exit to `_exit()`, which terminates immediately with no stdio flushing.
- **R — Result:** The doubled prompt never came back across hundreds of runs. I put a comment at each call site, *"`_exit()`, not `exit()`: avoid double-flushing stdio buffers"*, so nobody "cleans it up" later.

> 💡 **If they ask: "What's the difference between `exit()` and `_exit()`?"**
> "`exit()` is a C library call. It runs `atexit` handlers and flushes stdio buffers. `_exit()` is the direct syscall: immediate termination, no cleanup. In a forked child that never successfully exec'd, you want `_exit()` so you don't touch buffers you inherited."
> 💡 **Prep note (check against your code):** be ready for "why was the prompt still in the buffer at fork time?" The prompt has no trailing newline, and stdout on a terminal is line-buffered, so it can sit unflushed. Know whether your code also calls `fflush` after the prompt.

---

## ⭐ Story 3 — The SIGCHLD Race Condition
**LPs:** Ownership · Invent and Simplify · Dive Deep
**Question types:** "Tell me about a concurrency problem you solved." / "Tell me about a hard-to-reproduce problem you owned end to end." / "Tell me about finding a simpler solution."

- **S — Situation:** While building background jobs (`cmd &`), I initially had the `SIGCHLD` handler call `waitpid()` directly. Then, roughly once in every 10 runs, the shell printed `waitpid: No child processes`.
- **T — Task:** Nobody assigned this. It was my project, and the failure was non-deterministic, which is the hardest kind. I had to reproduce it, find the root cause, and fix it without adding new races.
- **A — Action:** I logged a timestamp, the PID, the return value and `errno` at every `waitpid()` call and ran the failing case repeatedly. The pattern: the error only appeared when a background child exited at nearly the same moment a foreground command finished. The cause was my handler's `waitpid(-1, ..., WNOHANG)`. The `-1` matches *any* child, including the foreground one. If the handler ran between the foreground child exiting and the executor's own `waitpid(fg_pid)`, the handler reaped it first and the executor got `ECHILD`. I made the handler do one thing: set `volatile sig_atomic_t g_sigchld_pending = 1`. Reaping now happens at exactly two places: `executor.cpp` for foreground (with an `EINTR` retry loop) and `reapFinishedJobs()` in `jobs.cpp` at the top of each REPL iteration for background. `reapFinishedJobs()` calls `waitpid(it->pid, &status, WNOHANG)` — specific PID, not `-1` — so it can never accidentally reap a foreground child.
- **R — Result:** I ran 500 consecutive background launches with overlapping foreground commands and saw zero `ECHILD` errors. The handler is two lines, the loop check is four, and anyone can audit which code path owns which PID. Adding a new signal later follows the same pattern: add a flag, check it in the loop.

> 💡 **If they ask: "Show me the code."**
> ```cpp
> static void sigchld_handler(int /*signo*/) { g_sigchld_pending = 1; }
> ```
> ```cpp
> do { result = waitpid(pid, &status, 0); }
> while (result == -1 && errno == EINTR);
> ```
> 💡 **If they ask: "Why not call `waitpid` in the handler?"**
> "Handlers must be async-signal-safe, and more importantly the wildcard reap races with the foreground wait. A flag makes ownership of each PID deterministic."

---

## ⭐ Story 4 — Building the Test Suite and Sanitizer Build
**LPs:** Insist on the Highest Standards · Ownership · Deliver Results
**Question types:** "Tell me about a time you went beyond what was asked." / "Tell me about raising the quality bar." / "What would you do differently?"

- **S — Situation:** The shell's features worked when I tried them by hand, so I could have called it done. But I kept re-discovering bugs I thought I'd already fixed. Manual testing wasn't scaling as the code grew.
- **T — Task:** No one required tests. I decided the project needed an automated safety net before I added more features.
- **A — Action:** I wrote `tests/test_shellx.sh`, a 214-line Bash harness that feeds input to the shell binary, captures stdout and stderr together, and checks substrings and exit codes. It has 22 tests across 6 categories: basic commands, builtins, redirection, pipes, background jobs, and error rejection. Each test runs in a fresh `mktemp -d` directory so files don't leak between tests. I also ran the project with `-fsanitize=address,undefined` manually via a separate debug build directory.
- **R — Result:** 22/22 pass in under 3 seconds, and the code compiles with `-Wall -Wextra -Wpedantic` (set in `CMakeLists.txt`) with zero warnings. The suite caught the `>>` append bug and a missing-whitespace validation case. UBSan flagged a real undefined behaviour: I was passing plain `char` to `std::isspace()` in `main.cpp`. Where `char` is signed (e.g. x86 Linux), bytes above 127 become negative values, which is UB. The fix was `static_cast<unsigned char>(c)`.

> 💡 **If they ask: "What would you do differently?"**
> "Write the test harness on day one, not after the third regression. Tests give you the confidence to change code without fear."

---

## ⭐ Story 5 — Learning POSIX From Scratch
**LPs:** Learn and Be Curious · Bias for Action
**Question types:** "Tell me about learning something quickly." / "Tell me about stepping outside your comfort zone."

- **S — Situation:** I'd spent two years in competitive programming, but had never written a systems program: no `fork()`, no file descriptors, no signals. ShellX started from zero POSIX experience.
- **T — Task:** Build a working shell on syscalls I'd never used, and make it robust enough to pass automated tests.
- **A — Action:** I learned in phases and built as I learned. First `fork()` + `execvp()` + `waitpid()` to run one command. Then builtins and redirection. Then pipes and signals, the hard part. When the pipeline hung, I taught myself `strace -f` and `lsof`. I could literally see `read(0, ...)` blocked in `wc` and the parent's leaked descriptors. When the SIGCHLD race appeared, I read the POSIX specs for `sigaction()` and `waitpid()` to learn exactly what they guarantee. I also built with ASan/UBSan so my learning mistakes would surface early.
- **R — Result:** I delivered a working shell with 22 passing tests and documented design decisions. I can now read man pages and specs comfortably, debug with `strace`/`lsof`, and reason about processes that interact. That's something competitive programming never taught me.

> 💡 **Key takeaway line:** "Systems bugs are usually not in the code you think is broken. They're in the interaction between components. The pipeline hang wasn't in my pipe creation; it was in the parent's cleanup."

---

## ⭐ Story 6 — Known Limitations: Scoping and Honesty
**LPs:** Earn Trust · Frugality · Deliver Results
**Question types:** "Tell me about being transparent about a limitation or mistake." / "Tell me about avoiding unnecessary complexity." / "Tell me about a trade-off you made."

- **S — Situation:** After shipping background jobs, I found that pressing `Ctrl+C` also sent `SIGINT` to background jobs and killed them. In bash, `Ctrl+C` only hits the foreground command. This was a genuine flaw.
- **T — Task:** Decide whether to fix it now with POSIX process groups, or ship and document it.
- **A — Action:** Proper isolation needs `setpgid()` and `tcsetpgrp()`, and it's coupled to another gap: my `Job` struct tracks a single `pid_t`, so backgrounded pipelines can't be supported either. Implementing half of it would create more bugs than none. So I deferred both on purpose and wrote a **Known Limitations** section in the README: what doesn't work, exactly why, and what the combined fix involves (process groups plus a `std::vector<pid_t>` per job). I also recorded the decision in `docs/implementation_plan.md`. Everything unsupported (`&&`, `||`, `;`, `$()`, backticks) is listed under "Explicitly Unsupported", and the parser rejects it with a clear error.
- **R — Result:** The core shell shipped correct for everything it claims to support, with no half-built feature cluttering the codebase. Anyone reading the repo knows what's missing and why. In interviews I raise this limitation myself before being asked.

> 💡 **If they ask: "How would you fix it?"**
> "In the child after `fork()`, call `setpgid(0, 0)` to start a new process group. For foreground jobs, hand over the terminal with `tcsetpgrp(STDIN_FILENO, child_pgid)`. Then `Ctrl+C` only reaches the foreground group. I'd change `Job` to hold a vector of PIDs at the same time."

---

## ⭐ Story 7 — Modular Architecture
**LPs:** Think Big · Invent and Simplify
**Question types:** "Tell me about a design decision you made." / "Tell me about designing for the future."

- **S — Situation:** I could have put everything in one `main.cpp`, as many simple shells do. It's faster at first, but I knew the project would grow and a monolith gets painful quickly.
- **T — Task:** Structure it so parsing, execution, signals, job tracking and redirection can each be changed and understood independently.
- **A — Action:** I split ShellX into 8 modules with clear contracts: `shellx.hpp` (shared types `Command` and `Pipeline` only), `parser` (input → pipeline structure, no execution knowledge), `executor` (dispatch), `pipeline` (multi-process wiring), `signals` (handler installation and flags only), `jobs` (background tracking), `builtins` (commands that must run in the parent, like `cd`), and `redirection` (fd manipulation). Each module includes only what it needs, and the shared header is the common vocabulary.
- **R — Result:** The structure paid off immediately. Redesigning the SIGCHLD handler touched only `signals.cpp` and 4 lines in `main.cpp`. Adding `>>` touched only `parser.cpp` and `redirection.cpp`. Features like history or tab completion would slot in without disturbing the core.

---

## ⭐ Story 8 — The `>>` Tokenizer Bug (Attention to Detail + Standing by a Design)
**LPs:** Dive Deep · Insist on the Highest Standards · Have Backbone (design stance)
**Question types:** "Tell me about a small detail that caused a big problem." / "Tell me about a time you pushed for the harder, cleaner approach."

- **S — Situation:** `echo hello > out.txt` worked, but `echo hello >> out.txt` wasn't appending. It was treating `>>` as two separate `>` tokens, so the redirection was parsed wrongly.
- **T — Task:** Find out why `>>` broke when `>` worked, and fix it in the right layer.
- **A — Action:** I traced it to the tokenizer in `parser.cpp`. The single-character `>` check ran before the two-character `>>` check, so the first `>` matched alone and the second matched on the next iteration. The fix is the classic lexer rule of *maximal munch*: check longer tokens first. If the current char is `>` and the next is also `>`, emit `">>"` and advance by 2. I had a design choice here. The quicker hack was to let the *parser* spot two consecutive `>` tokens. I held to the cleaner design, where the tokenizer owns multi-character operators, because it keeps the parser free of operator-recognition logic. It was slightly more code, but any future operator (like `<<`) follows the same pattern.
- **R — Result:** `>>` works. I added a regression test (`echo hello >> f; echo world >> f; cat f` checks both lines exist), and it's in the 22-test suite.

> ⚠️ **Honest note on Backbone:** this is a disagreement with *myself* about a design, not a disagreement with a person. If the question is "tell me about disagreeing with a teammate or manager", use a real story from your own experience (see *Gaps to Fill* below). Don't stretch this one.

---

# ➕ Extras

## 🗺️ Leadership Principle → Story Map (all 16)

| Leadership Principle | Best story | Notes |
|---|---|---|
| **Customer Obsession** | Extra A1 (clear errors) | Frame "customer" as whoever uses the shell |
| **Ownership** | 3 (SIGCHLD), 4 (tests) | |
| **Invent and Simplify** | 3 (flag-only handler), 7 | |
| **Are Right, A Lot** | 2 (`_exit()` from first principles) | |
| **Learn and Be Curious** | 5 | |
| **Hire and Develop the Best** | ⚠️ none | Prepare a non-ShellX story (mentoring, helping a peer) |
| **Insist on the Highest Standards** | 4, 8 | |
| **Think Big** | 7 | |
| **Bias for Action** | 1 (instrument first, then read) | |
| **Frugality** | 6 (deferred process groups) | |
| **Earn Trust** | 6 | |
| **Dive Deep** | 1, 2 | |
| **Have Backbone; Disagree and Commit** | 8 (weak, design stance only) | ⚠️ Prepare a story involving a real person |
| **Deliver Results** | 4, Extra A2 | |
| **Strive to be Earth's Best Employer** | ⚠️ none | Prepare a non-ShellX story (team, inclusion, helping others) |
| **Success and Scale Bring Broad Responsibility** | ⚠️ none | Prepare a non-ShellX story |

> The last three plus *Hire and Develop* are rarely asked of interns, but "Tell me about a time you helped someone else succeed" does come up.

---

## 🅰️ Extra A — Two 60-Second Mini-Stories

**A1. Customer Obsession — Clear errors instead of silent wrong behaviour**
> "ShellX doesn't support `&&`. I could have let `echo hello && echo world` quietly pass `&&` as a normal argument, which prints the word `&&` and confuses the user. I wrote a `rejectUnsupported()` check in `parser.cpp` that runs right after tokenizing and rejects `&&`, `||`, `;`, `$()` and backticks with a precise message: `shellx: '&&' is not supported`. It was extra work, but I thought about debugging a script that produces wrong output with no error, and that decided it. There's a test for it, and the README lists every unsupported construct up front."

**A2. Deliver Results — Scoping an open-ended project**
> "ShellX had no spec beyond 'build a shell with raw syscalls', so the scope could have been tiny or endless. I wrote the scope in `docs/implementation_plan.md` first: external commands, builtins, redirection, pipelines, background jobs, basic signals, double-quoted strings. And I wrote what was *out*: process groups, history, tab completion, `&&`/`||`, backgrounded pipelines. I delivered in phases: `fork/exec` first, then builtins, then redirection, then pipes (hardest), then signals and background jobs, then tests. At every phase the shell was usable, never half-broken. Final result: 22/22 tests, sanitizer-clean, zero warnings."

---

## 🪤 Extra B — Trap Follow-Ups and Recovery Scripts

**"This sounds theoretical. Show me the code."**
> "In `src/signals.cpp` the handler is two lines: it sets `g_sigchld_pending = 1`. `reapFinishedJobs()` in `src/jobs.cpp` uses `waitpid` with `WNOHANG`. The foreground wait in `src/executor.cpp` retries on `EINTR`. There are exactly two `waitpid()` call sites in the codebase, on purpose: single ownership of each PID."

**"Why not use a library or copy bash?"**
> "The goal was to understand the OS boundary: what syscalls a shell actually makes. Building from raw POSIX forced me to learn fd inheritance across `fork()`, why `dup2()` must happen before `exec()`, and why signal handlers have strict async-signal-safety rules. You don't really understand those until you've debugged a pipeline deadlock with `strace`."

**"What would you do with 3 more months?"**
> "First, process groups (`setpgid`, `tcsetpgrp`) so `Ctrl+C` only hits the foreground. Second, change `Job` from one `pid_t` to a `std::vector<pid_t>` for backgrounded pipelines. These two are coupled, so I'd do them together. Third, command history with readline. All three are already in the README's Known Limitations, so the roadmap is explicit."

**"What would you do differently?"**
> "Tests on day one, and the sanitizer build from the first commit instead of after the first bug."

---

## 🧩 Extra C — Gaps to Fill (Important)

Every story above is a **solo project**. Amazon's behavioural round often asks about other people. Prepare **one real story each**, from a class project, hackathon, club, internship, or open source:

| Likely question | Your story (fill in) |
|---|---|
| Disagreed with a teammate or mentor, and what happened | |
| Made a real mistake with consequences, and how you owned it | |
| Received tough feedback, and what you changed | |
| Helped someone else learn or succeed | |
| Worked under a tight deadline with competing priorities | |

Template (60–90 seconds): **S** one sentence of context, **T** what *you* were responsible for, **A** 3 specific things *you* did, **R** a number or concrete outcome, then one line on what you learned.

> I haven't written these for you because I don't know your real experiences, and Amazon interviewers will probe the details. Invented stories fall apart under follow-up questions.

---

## ❓ Quick Q&A Cheat Sheet

| Question | Key Answer |
|---|---|
| "Why C++?" | Full control over processes, file descriptors and signals. No GC pauses. Direct POSIX syscall access. |
| "What syscalls does your shell use?" | `fork()`, `execvp()`, `waitpid()`, `pipe()`, `dup2()`, `open()`, `close()`, `sigaction()`, `kill()`, `chdir()`, `getcwd()`. |
| "Hardest part of building a shell?" | Signals and pipeline fd management. The SIGCHLD race and the pipe write-end leak were both subtle and took systematic debugging. |
| "What does `dup2()` do?" | Makes the second fd refer to the same open file as the first. Used to wire pipe ends to stdin/stdout before `exec`. |
| "Why close pipe fds in the parent?" | EOF is only delivered when *all* write-end references are closed. A leaked parent copy keeps readers blocked forever. |
| "What is a zombie process?" | A finished process whose exit status hasn't been collected with `waitpid()`. ShellX reaps background jobs with `WNOHANG` each REPL iteration. |
| "What is async-signal safety?" | Handlers may only call functions safe to run when interrupting arbitrary code. Mine only set a `volatile sig_atomic_t`: no `malloc`, no `printf`, no STL. |
| "Why do builtins like `cd` run in the parent?" | A child can't change the parent's working directory. |
| "What would you add next?" | Process groups, multi-PID jobs for backgrounded pipelines, readline history. |
| "Most important lesson?" | Bugs often live in the interaction between components, not the code that looks broken. |

---

## ⚡ Quick One-Liners for Common Follow-ups

| Follow-up | Answer |
|---|---|
| "What would you do differently?" | "Write the test harness on day one, not after the third regression." |
| "How did you handle not knowing something?" | "Man pages, the POSIX spec, and `strace`. When theory wasn't enough, I reproduced the bug in isolation and instrumented it until I could see exactly what happened." |
| "How did you decide what to include vs exclude?" | "I scoped to what taught me the most (`fork`/`exec`, pipes, signals) and documented everything I skipped and why." |
| "Biggest takeaway?" | "Competitive programming teaches you to solve problems. Systems programming teaches you to solve problems that interact with other problems." |
| "Describe yourself in one line." | "A competitive programmer who built a shell from scratch to understand what actually happens when you press Enter in a terminal." |

---

## 📇 60-Second Cheat Card (Read Before the Interview)

```
PROJECT IN NUMBERS
  8 source files (.cpp) + 8 headers, ~1,500 lines of C++20
  22 automated tests, all passing, run in < 3 s
  0 compiler warnings (-Wall -Wextra -Wpedantic)
  0 sanitizer errors (ASan + UBSan build)

3 HARDEST BUGS
  1. Pipeline hang   → parent must close all pipe fds after forking
  2. exit() flush    → use _exit() in every child path
  3. SIGCHLD race    → flag-only handler, exactly 2 waitpid sites

TOP 5 DESIGN DECISIONS
  1. Handlers only set flags (async-signal-safety)
  2. Fork ALL pipeline children before waiting on ANY
  3. _exit() not exit() in children
  4. Tokenizer owns multi-char operators (maximal munch: >> before >)
  5. Explicit rejection of unsupported syntax over silent failure

KNOWN LIMITATIONS (say these first)
  1. No process groups → background jobs get Ctrl+C
  2. Backgrounded pipelines unsupported (single-PID Job struct)
  3. No history / tab completion (deferred on purpose)

STORY → LP QUICK PICK
  1 Dive Deep / Bias for Action     5 Learn and Be Curious
  2 Are Right, A Lot                6 Earn Trust / Frugality
  3 Ownership / Invent & Simplify   7 Think Big
  4 Highest Standards               8 Attention to detail
```
