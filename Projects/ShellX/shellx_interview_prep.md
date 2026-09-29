# ShellX — SDE Interview Prep (50 Questions)

**Project in one line:** ShellX is a Unix shell written from scratch in C++20, using raw POSIX syscalls (`fork`, `execvp`, `waitpid`, `pipe`, `dup2`, `sigaction`) — no shell libraries, no readline, no libc shell helpers. It supports external commands, 4 builtins (`cd`, `pwd`, `exit`, `jobs`), I/O redirection (`>`, `>>`, `<`), N-stage pipelines, background jobs (`&`), and correct signal handling (`SIGCHLD`, `SIGINT`).

**Why this project is interview gold:** it is the single best CS-fundamentals project you can put on a resume, because it forces you to actually use — not just define — process creation, virtual memory/fd tables, IPC, signals, and race-condition reasoning. Every OS course topic (fork/exec/wait, pipes, signals, zombies, fd inheritance) shows up as *working code with edge cases handled*, which is exactly what interviewers probe for in SDE-2/backend/infra interviews at product-based companies.

**How to use this doc:** Each entry gives the question(s) as an interviewer might realistically phrase them (sometimes merged, since interviewers often combine 2–3 related ideas into one), a polished spoken-style answer you can give, a TL;DR one-liner for a rushed recap, and a "key mappings" line pointing to the exact file/function in the repo so you can pull up code if asked to point at it.

---

## Codebase map (memorize this shape before the interview)

```
main.cpp (REPL loop)
   -> parser.cpp: parse(line) -> Pipeline{ vector<Command>, background }
   -> executor.cpp: executePipeline(Pipeline)
         size==1 & isBuiltin -> builtins.cpp: executeBuiltin()      [runs in shell process, no fork]
         size==1 & external  -> executor.cpp: executeSingle()       [fork + redirection.cpp + execvp + waitpid]
         size>1              -> pipeline.cpp: executePipelineChain()[N-1 pipes, N forks, dup2 wiring]
   -> signals.cpp: SIGCHLD/SIGINT handlers (flag-only)
   -> jobs.cpp: g_jobs vector, addJob/reapFinishedJobs/printJobs/killAllJobs
```

Structs (`shellx.hpp`):
```cpp
struct Command  { vector<string> args; string input_file; string output_file; bool append_mode; };
struct Pipeline { vector<Command> commands; bool background; }; // background lives ONLY here
```

---

# TOP 10 — Must-nail fundamentals

### Q1. Walk me through exactly what your shell does when I type `ls -l | grep foo > out.txt`. (Merges: "explain your architecture" + "trace a command end-to-end")

**Polished answer:** The REPL in `main.cpp` reads the line and hands it to `parse()`. The tokenizer in `parser.cpp` splits it into tokens (`ls`, `-l`, `|`, `grep`, `foo`, `>`, `out.txt`), recognizes `|` and `>` as special tokens, and splits on `|` into two token lists — one per command. Each list is walked to pull out redirection operators into `Command::output_file`/`input_file` and leave the rest as `args`. The result is a `Pipeline` with two `Command`s, the second one carrying `output_file = "out.txt"`.

`executePipeline()` in `executor.cpp` sees `commands.size() == 2`, so it's not a builtin/single-command path — it calls `executePipelineChain()` in `pipeline.cpp`. That function creates `N-1 = 1` pipe with `pipe()`, then forks twice. In the first child, stdout is `dup2`'d onto the pipe's write end; in the second child, stdin is `dup2`'d onto the pipe's read end, and *then* `applyRedirections()` (from `redirection.cpp`) opens `out.txt` and `dup2`'s it onto stdout — so the redirection is applied on top of the pipe wiring, meaning `grep`'s output goes to the file, not back to the pipe. Every child closes **all** pipe fds (both ends, all pipes) after wiring, and so does the parent, right after forking, to avoid fd leaks and hangs. Each child then resets `SIGINT` to default and calls `execvp()`. The parent `waitpid()`s on both children in order and reports the *last* command's exit status as the pipeline's status.

**TL;DR:** parse → split into 2 Commands on `|` → 1 pipe, 2 forks → dup2 wiring (pipe first, then redirection) → every process closes fds it doesn't own → exec → parent waits on all, reports last child's status.

**Key mappings:** `parser.cpp::parse`, `pipeline.cpp::executePipelineChain`, `redirection.cpp::applyRedirections`.

---

### Q2. Explain `fork()`, `exec()`, and `wait()`/`waitpid()` — why do you need all three, and what does each actually do at the OS level?

**Polished answer:** `fork()` duplicates the calling process: same code, same open file descriptors (they now share file *offsets* via the underlying open file description), same memory contents (copy-on-write, not a real duplicate — pages are only copied when written to). It returns twice: 0 in the child, the child's PID in the parent, -1 on failure. `exec*()` family (I use `execvp`) *replaces* the calling process's image — code, data, stack, heap — with a new program, but keeps the same PID and the same open file descriptor table. That's the whole trick shells rely on: fork to get a *new process*, then rearrange fds in that clone (via `dup2`) before exec so the new program inherits exactly the stdin/stdout/stderr you want, without the new program needing to know anything about pipes or files. `waitpid()` is how the parent reaps the child's exit status and removes it from the process table — without it, a terminated child becomes a *zombie* (a process table entry with no address space, kept around so the parent can eventually collect the status).

In `executeSingle()`, this is literally the three-step dance: `fork()` → child does `applyRedirections()` + `execvp()` → parent does a blocking `waitpid()` (foreground) or defers it to `reapFinishedJobs()` (background).

**TL;DR:** `fork` = clone process; `exec` = replace image in place; `wait`/`waitpid` = reap exit status and prevent zombies. Together they let a shell run other programs without leaking its own state into them.

**Key mappings:** `executor.cpp::executeSingle` (lines with `fork()`, `execvp()`, `waitpid()`).

---

### Q3. How do pipes actually work under the hood, and why does your pipeline creator do `N-1` pipes for `N` commands and close every single fd it isn't using? (Merges: "explain pipe()/dup2()" + "why the aggressive fd closing")

**Polished answer:** `pipe(fd[2])` creates a unidirectional kernel buffer with two file descriptors: `fd[0]` for reading, `fd[1]` for writing. A pipeline of N commands needs data to flow between each adjacent pair, so you need exactly N-1 pipes — `cmd1|cmd2|cmd3` needs 2 pipes, one between 1&2 and one between 2&3.

`dup2(oldfd, newfd)` makes `newfd` become a copy of `oldfd` pointing at the same underlying open file description, closing whatever `newfd` used to point to first. In `executePipelineChain()`, child `i` (0-indexed) does `dup2(pipe_fds[2*(i-1)], STDIN)` if it's not the first command (read from the previous pipe) and `dup2(pipe_fds[2*i+1], STDOUT)` if it's not the last (write to the next pipe).

The closing discipline is the part people get wrong and where real shells (and real interview candidates) fail: **every process must close every pipe fd it doesn't need**, including the ends it just `dup2`'d away from (the original numbers, not just the STDIN/STDOUT slots) and the ends belonging to *other* pipes it's not part of. If you forget this, the classic bug is: a downstream reader never sees EOF because some other process — often the shell itself — still holds the write end open, so `read()` blocks forever even after the real writer has finished and exited. That's why the code closes **all** `2*(n-1)` pipe fds in every child right after wiring, and the parent closes all of them immediately after forking everyone, before it even starts calling `waitpid()`.

**TL;DR:** `pipe()` = kernel ring buffer with read/write fd pair; N commands need N-1 pipes; `dup2` rewires stdin/stdout; every process must close fds it doesn't own or a reader can hang forever waiting for an EOF that never comes because a write end is still open somewhere.

**Key mappings:** `pipeline.cpp::executePipelineChain` — pipe creation loop, dup2 wiring block, the two "close ALL pipe fds" loops (child and parent).

---

### Q4. What are zombie and orphan processes? How does ShellX avoid creating zombies, both for foreground and background commands? (Merges classic "what is a zombie" + "how do you avoid it in your project")

**Polished answer:** A **zombie** is a process that has called `exit()` (or was killed) but whose parent hasn't yet called `wait()`/`waitpid()` on it — the kernel keeps a minimal process-table entry (PID, exit status) around so the parent can retrieve that status later; it's not consuming memory/CPU, but it's a leaked table slot, and if enough accumulate you can hit the OS's PID/process-table limits. An **orphan** is the opposite problem: a *parent* dies before its child does; the child gets re-parented to `init`/PID 1 (or a subreaper), which is expected to reap it automatically.

ShellX guarantees "exactly one `waitpid()` call site per PID," which is the core invariant that prevents both zombies and double-reap races. For a **foreground** command, `executeSingle()` blocks on `waitpid(pid, &status, 0)` right after forking — the moment the child exits, the parent reaps it. For a **background** command (`cmd &`), the parent doesn't block; instead the PID is registered in `g_jobs` via `addJob()`, and reaping happens later, in `reapFinishedJobs()`, called from the REPL loop whenever `g_sigchld_pending` is set (i.e., a `SIGCHLD` arrived). That function calls `waitpid(pid, &status, WNOHANG)` per still-running job so it never blocks the shell. On `exit`, `killAllJobs()` sends `SIGTERM` to any remaining background jobs but *deliberately does not wait for or reap them* — the shell process is about to terminate anyway, so any children it leaves running become orphans and are reparented to init, which reaps them. That's a documented, intentional tradeoff, not an oversight.

**TL;DR:** Zombie = child exited, parent hasn't waited yet; orphan = parent died first, child reparented to init. ShellX reaps foreground children with a blocking `waitpid()` and background children with `WNOHANG` polling in `reapFinishedJobs()`, guaranteeing one waitpid call per PID — no double-reap races, no leaked zombies.

**Key mappings:** `executor.cpp::executeSingle` (blocking wait), `jobs.cpp::reapFinishedJobs` (WNOHANG), `jobs.cpp::killAllJobs`.

---

### Q5. Why does your `SIGCHLD` handler only set a flag instead of calling `waitpid()` directly? What could go wrong if it called `waitpid()` itself? (Merges "async-signal-safety" + "signal handler race conditions")

**Polished answer:** POSIX signal handlers can only safely call a small, documented set of **async-signal-safe** functions — things like `write()`, `_exit()`, `signal()` itself. `waitpid()` *is* technically on that safe list, but the real danger isn't the syscall — it's everything a naive handler tends to do around it: printing with `std::cout`/`printf` (not signal-safe: they can lock internal buffers, and if the signal interrupts the shell mid-`cout` in `main()`, you deadlock on the same lock from inside the handler), using STL containers or `new`/`malloc` (allocators use locks internally — same reentrancy hazard), or racing with the *foreground* `waitpid()` in `executeSingle()`: if the handler itself reaped a background child while the main thread is also inside `waitpid()` for a foreground child, you could get two independent `waitpid()` call sites, which risks reaping the wrong child's status or double-freeing job-list bookkeeping.

So the handler (`sigchld_handler` in `signals.cpp`) does the minimum: `g_sigchld_pending = 1;` — one assignment to a `volatile sig_atomic_t`. `sig_atomic_t` is guaranteed by the C standard to be read/written atomically even if a signal arrives mid-write, and `volatile` tells the compiler not to cache it in a register or reorder/eliminate reads across the signal boundary (the compiler can't otherwise "see" that a signal handler running asynchronously might change it). The actual reaping happens later, from ordinary (non-signal) code in the REPL loop, which checks the flag and calls `reapFinishedJobs()` — safely, with full access to `cout`, the STL, and the job list.

**TL;DR:** Signal handlers can only safely touch a tiny set of reentrant primitives — no `cout`, no STL, no `malloc`. The handler just flips a `volatile sig_atomic_t` flag; the actual `waitpid()`/bookkeeping runs later in normal code, which also avoids a reap race with the foreground `waitpid()`.

**Key mappings:** `signals.cpp::sigchld_handler`, `signals.hpp` (flag declarations), `main.cpp` (flag check → `reapFinishedJobs()`).

---

### Q6. Why do builtins like `cd` have to run in the parent shell process, and not in a forked child?

**Polished answer:** `cd` calls `chdir()`, which changes the **calling process's** current working directory — a piece of per-process kernel state. If ShellX forked a child and ran `chdir()` there, the child's CWD would change, but the child would then just `exit()`, and the *parent shell* (the one actually running the REPL) would still be sitting in its old directory. The change would be completely invisible to the user's next command. This is exactly the same reason real shells implement `cd`, `export`, and similar state-mutating commands as builtins rather than external binaries — a subprocess fundamentally cannot mutate its parent's process state (different address space, different fd table beyond what was inherited at fork time, different CWD).

`isBuiltin()` in `builtins.cpp` checks the command name (`cd`, `pwd`, `exit`, `jobs`) *before* `executePipeline()` decides whether to fork at all — for a single-command pipeline it's checked first, so builtins never pay the fork/exec cost and, more importantly, their side effects land in the right process.

**TL;DR:** A child process can't mutate its parent's CWD, environment, or process state — those effects die with the child. So `cd`/`pwd`/`exit`/`jobs` are detected up front and run directly in the shell's own process, no fork.

**Key mappings:** `builtins.cpp::isBuiltin`, `builtins.cpp::executeBuiltin`, `executor.cpp::executePipeline` (the `isBuiltin` check before any fork for `size()==1`).

---

### Q7. Explain the exit code convention you used — 0, 1, 126, 127, and `128+signal`. Where do these come from and how do you compute them?

**Polished answer:** These aren't arbitrary — they're the POSIX/shell convention that tools like `make`, CI systems, and `$?` checks in scripts rely on. `0` means success. Ordinary command failure is whatever the program itself returns (surfaced via `WEXITSTATUS(status)` after `WIFEXITED(status)` is true — these macros decode the packed `int status` that `waitpid` fills in, since a single int has to encode both *how* the child died and *what* it returned/was killed by). `126` conventionally means "found the file but couldn't execute it" (e.g. permission denied, `EACCES`, or it's not actually executable) — in ShellX, that's the `else` branch after `execvp` fails when `errno != ENOENT`. `127` means "command not found" — that's `errno == ENOENT` from `execvp`, i.e., PATH search found nothing. `128+N` means the process was terminated by signal number N — decoded via `WIFSIGNALED(status)` / `WTERMSIG(status)`; e.g. a command killed by `SIGINT` (signal 2) exits with 130, `SIGKILL` (9) with 137. This is why `128+signum` never collides with a normal 0–125 program return code range, and scripts can distinguish "program chose to fail" from "program was killed."

**TL;DR:** 0 = success; program's own code = normal failure; 126 = found but not executable; 127 = command not found (`execvp` → `ENOENT`); `128+signal` = killed by that signal, decoded via `WIFSIGNALED`/`WTERMSIG`.

**Key mappings:** `executor.cpp::executeSingle` (the `perror`+`_exit(127/126)` branch on exec failure, and the `WIFEXITED`/`WIFSIGNALED` decode after `waitpid`), same pattern in `pipeline.cpp::executePipelineChain`.

---

### Q8. What's the difference between `_exit()` and `exit()`, and why does the child process use `_exit()` after a failed `fork()`/`exec()` rather than `exit()` or `return`?

**Polished answer:** `exit()` (from `<cstdlib>`) does "full" process termination: it flushes and closes all open C `stdio` streams (`FILE*` buffers), runs functions registered with `atexit()`, runs static/global C++ destructors, and *then* makes the `_exit` syscall. `_exit()` skips all of that and terminates immediately at the syscall level.

The reason it matters here: after `fork()`, both parent and child share *copies* of the same buffered `stdio` state (e.g., anything already sitting in `std::cout`'s or `stdout`'s internal buffer at the moment of `fork()` exists in both processes' memory, unflushed). If the child called `exit()` (or just returned from `main`), it would flush its copy of those buffers — which can duplicate output the parent will *also* eventually flush, producing garbled or doubled terminal output. `_exit()` avoids this entirely by skipping the flush. It's also simply the more honest description of what a failed-exec child should do: it isn't part of the shell's normal lifecycle anymore, it just needs to report an error code to its parent as fast and cleanly as possible.

**TL;DR:** `exit()` flushes stdio buffers + runs cleanup handlers; `_exit()` terminates immediately with no flushing. A post-fork child shares copies of the parent's buffered output, so using `exit()` there risks double-flushing/duplicated output — hence `_exit()` everywhere a child bails out (fork/exec failure, redirection failure, dup2 failure).

**Key mappings:** every `_exit(...)` call in `executor.cpp::executeSingle` and `pipeline.cpp::executePipelineChain`.

---

### Q9. How does background execution (`cmd &`) work end to end — what data structure tracks it, and how does the shell know when it finishes?

**Polished answer:** The parser strips a trailing `&` token and sets `Pipeline::background = true` (this is the *only* place background state lives — see Q29 on why). In `executeSingle()`, if `background` is true, the parent does **not** call a blocking `waitpid()`. Instead it builds a display string of the command's args, calls `addJob(pid, cmdString)` which pushes a `Job{pid, command, running=true}` onto the global `g_jobs` vector and returns a 1-based job number, then immediately prints `"[N] <pid>"` and returns control to the REPL — the user gets their prompt back right away.

Completion is detected asynchronously: when *any* child terminates, the kernel delivers `SIGCHLD` to the shell, the handler sets `g_sigchld_pending`, and the next time the REPL loop's top-of-loop check runs (in `main.cpp`, also re-checked right after `executePipeline` returns, since a background job could finish *during* some other foreground command), it calls `reapFinishedJobs()`. That function walks `g_jobs` and calls `waitpid(pid, &status, WNOHANG)` on each still-`running` job — `WNOHANG` means "check but don't block if it hasn't exited yet." For each job that *has* exited, it prints a `Done`/`Killed` message with the job number and erases it from `g_jobs`.

**TL;DR:** `&` sets `Pipeline::background`; the parent skips blocking `waitpid`, registers the PID in `g_jobs`, and returns immediately. `SIGCHLD` sets a flag; the REPL polls `g_jobs` with non-blocking `waitpid(..., WNOHANG)` in `reapFinishedJobs()` and reports/removes finished jobs.

**Key mappings:** `parser.cpp::parse` (strip trailing `&`), `executor.cpp::executeSingle` (background branch), `jobs.cpp::addJob`/`reapFinishedJobs`/`printJobs`.

---

### Q10. Give me the high-level architecture of your shell — what are the layers/modules and why did you split it that way?

**Polished answer:** The pipeline is: **read** (REPL in `main.cpp`) → **parse** (`parser.cpp` turns a raw string into a `Pipeline` of `Command`s — pure data, no syscalls) → **dispatch** (`executor.cpp::executePipeline` decides: single builtin → run in-process; single external → `executeSingle`; N≥2 → `executePipelineChain`) → **execute** (the actual `fork`/`dup2`/`execvp`/`waitpid` work, split further into `redirection.cpp` for file-fd wiring and `pipeline.cpp` for pipe-fd wiring) → **support systems** that run orthogonally to all of the above: `signals.cpp` (installed once at startup, before the REPL loop and before any `fork()`, so a background child can't finish and deliver `SIGCHLD` before the handler exists) and `jobs.cpp` (the background-job bookkeeping consulted from the REPL loop).

The module boundaries map directly to *separate concerns that would otherwise tangle*: parsing is pure and testable without touching the OS at all; redirection and pipe-wiring are separate because a command can need *either or both* (a pipeline stage can also redirect its own stdin/stdout on top of its pipe connection — see Q1); builtins are separate because they explicitly *don't* fork, which is a different execution model from everything else. This separation is also why the header files read almost like a spec — each one documents its single responsibility and its ownership rules (e.g. `jobs.hpp` explicitly states the signal handler never touches `g_jobs` directly).

**TL;DR:** read → parse (data only) → dispatch (builtin vs single vs pipeline) → execute (fork/dup2/exec/wait, split into redirection vs pipe-wiring) → signals/jobs run orthogonally. Split by *execution model* (in-process vs forked) and by *which fd wiring concern* (file vs pipe), which keeps each module independently reasoned-about and testable.

**Key mappings:** whole repo; especially `include/*.hpp` comments, which double as the architecture doc.

---

# 11–25 — Strong follow-ups & mechanism depth

### Q11. Walk me through your tokenizer/parser. How does it handle quoted strings, and what does it do with malformed input like a trailing `|`?

**Polished answer:** `tokenize()` in `parser.cpp` is a single left-to-right scan over the raw line: it skips whitespace, and for each remaining character either (a) if it's `"`, it consumes everything up to the next `"` as one token (a very basic quoting model — no escape sequences, no nesting), (b) if it's the two-character sequence `>>`, emits that as one token (checked *before* the single-`>` case so append-mode isn't misparsed as truncate followed by a stray `>`), (c) if it's one of `| < > &`, emits it as a single-character token, or (d) otherwise reads a maximal run of non-special, non-whitespace characters as a "word" token.

After tokenizing, `rejectUnsupported()` scans the token list for constructs the grammar explicitly doesn't support — `;`, `&&` (two adjacent `&` tokens), `||` (two adjacent `|` tokens), `$(...)`, backticks — and bails with a clear stderr message rather than silently mis-executing them (see Q12 for why that matters). Then `parse()` strips a trailing `&` into `Pipeline::background`, splits the remaining tokens into per-command token lists on `|` (rejecting empty segments — a leading, trailing, or doubled `|` — as a syntax error), and for each segment walks it once more to pull `<`/`>`/`>>` plus their filename argument out into `Command::input_file`/`output_file`, leaving everything else as `args`. Empty `args` after that (e.g. `ls | | grep`) is also rejected. Every failure path returns an empty `Pipeline{}`, which `main.cpp` treats as "already printed an error, just loop again."

**TL;DR:** Single-pass tokenizer (whitespace + `"..."` + `| < > >> &` recognized) → reject explicitly-unsupported constructs → split on `|` → per-segment redirection extraction into `Command` fields → empty `Pipeline` return value doubles as the error signal.

**Key mappings:** `parser.cpp::tokenize`, `parser.cpp::rejectUnsupported`, `parser.cpp::parse`.

---

### Q12. Why explicitly reject `;`, `&&`, `||`, `$()`, and backticks instead of just not implementing them (i.e., letting them fall through as regular words)?

**Polished answer:** If those tokens were left unrecognized, they'd silently become literal *arguments* to whatever command precedes them — e.g. `echo hi; rm -rf /tmp/x` would run `echo` with literal args `"hi;"` and `"rm"`... actually it would try to run `echo` with those as arguments and never touch `rm` at all, which is a *silent, confusing wrong behavior*, not a crash. That's the worst kind of bug for a shell: the user thinks they chained two commands, but only the first one (with garbage extra args) ran, and there's no error to tell them so. Explicitly detecting and rejecting these tokens converts "silently wrong" into "loudly and immediately wrong," which is strictly better UX and much easier to debug — and it's honest about the grammar the shell actually supports (documented in the README as "Explicitly Unsupported"), rather than pretending compatibility it doesn't have.

**TL;DR:** Unrecognized operators wouldn't error — they'd become literal argv words and silently do the wrong thing. Explicit rejection turns a silent correctness bug into an immediate, clear error message.

**Key mappings:** `parser.cpp::rejectUnsupported`.

---

### Q13. What does `dup2()` actually do at the file-descriptor-table level? Why not just reassign the variable in your code?

**Polished answer:** Every process has a small per-process table of file descriptors (integers 0, 1, 2, 3, ...), each entry pointing to an *open file description* in a kernel-wide table, which in turn points to the actual file/pipe/socket and tracks things like the current read/write offset and access mode. `dup2(oldfd, newfd)` makes `newfd` point at the *same open file description* as `oldfd` — if `newfd` was already open, it's closed first automatically. This is fundamentally different from "reassigning a variable" because file descriptor `1` (stdout) is a slot in the *process's* table, not a variable in your program; every C library function that writes to stdout writes to whatever open file description is sitting in slot 1 of that table. `dup2(pipe_write_fd, STDOUT_FILENO)` makes slot 1 point at the pipe instead of the terminal, so *anything the program later writes to stdout* — including code inside `execvp`'d programs that know nothing about pipes — transparently goes to the pipe instead. That's the entire mechanism that lets an unmodified `ls` binary participate in a pipeline: it just writes to fd 1 like always, and the shell rearranged what fd 1 points to before calling `execvp`.

**TL;DR:** `dup2(old,new)` makes fd `new` point at the same underlying open file description as `old` (closing whatever `new` pointed to first) — it rewires a slot in the process's fd table, which is why unmodified programs that just write to fd 1 transparently participate in redirection/pipes without knowing it.

**Key mappings:** `redirection.cpp::applyRedirections`, `pipeline.cpp::executePipelineChain` (dup2 wiring block).

---

### Q14. What's the practical difference between `>` and `>>`, and what open() flags implement each?

**Polished answer:** Both open the target file for writing, creating it if it doesn't exist (`O_WRONLY | O_CREAT`). `>` additionally passes `O_TRUNC`, which truncates the file to zero length at open time — so if the file existed with content, that content is discarded before the first write. `>>` instead passes `O_APPEND`, which doesn't truncate; instead it makes the kernel atomically seek to end-of-file before *every* write — important detail: it's atomic per-write at the kernel level, not "seek once at open then write," which is what makes `>>` safe even with concurrent writers (each write always lands at the then-current end of file). `applyRedirections()` computes this flag set from `Command::append_mode` (set by the parser depending on whether it saw `>` or `>>`) and passes `0644` as the creation permission mode (rw-r--r--) when `O_CREAT` causes a new file to be made.

**TL;DR:** `>` = `O_WRONLY|O_CREAT|O_TRUNC` (wipe then write); `>>` = `O_WRONLY|O_CREAT|O_APPEND` (kernel atomically appends at end-of-file on every write, safe under concurrent writers).

**Key mappings:** `redirection.cpp::applyRedirections`.

---

### Q15. Why does your shell install signal handlers before the REPL loop starts and before any `fork()`? What race condition would exist if you installed them later, or lazily on first background job?

**Polished answer:** If a background job's process could finish and the kernel could try to deliver `SIGCHLD` *before* `installSignalHandlers()` has run, the default disposition for `SIGCHLD` (ignore-and-let-kernel-auto-reap on some systems, or just "no handler installed" on others) would apply instead — meaning either the shell never learns the child finished, or worse, the timing window between "child forked" and "handler installed" is a genuine race: a *very* fast-exiting background child (imagine `true &`) could complete before the parent even finishes setting up the job-list entry and installing handlers, if those were reordered relative to the first possible fork. Installing handlers once, unconditionally, at the very top of `main()` before the REPL loop begins and before any `fork()` call anywhere in the program removes the race entirely by construction — there is no code path that can fork a child while handlers are not yet installed.

**TL;DR:** Any window between "a child could be forked" and "the SIGCHLD handler exists" is a real race that can lose a completion notification for a fast-exiting child — so handlers are installed unconditionally before the REPL loop and before the first possible `fork()`.

**Key mappings:** `main.cpp` (`installSignalHandlers()` is the first line of `main()`).

---

### Q16. What does `SA_RESTART` do, and why use it here instead of the default `sigaction` behavior?

**Polished answer:** By default, when a signal is delivered while a process is blocked in certain "slow" syscalls (`read`, `write`, `waitpid`, etc.), the syscall is interrupted and returns `-1` with `errno == EINTR`, forcing the caller to explicitly check for and retry it. `SA_RESTART` tells the kernel to automatically restart the interrupted syscall after the handler returns, for most (not all) syscalls. ShellX sets `SA_RESTART` for both `SIGCHLD` and `SIGINT` handlers as a convenience — but the code *doesn't rely on it exclusively*, it still wraps `waitpid()` calls in an explicit `do { ... } while (result == -1 && errno == EINTR)` retry loop (belt-and-suspenders, as the code comments literally say). That's the right defensive posture: `SA_RESTART` doesn't cover every syscall/every platform guarantee, so the explicit retry loop is the actual correctness guarantee, and `SA_RESTART` just reduces how often it's needed in practice.

**TL;DR:** `SA_RESTART` auto-restarts interrupted slow syscalls after a handler returns instead of forcing an `EINTR` return; ShellX uses it for convenience but *also* explicitly retries on `EINTR` around `waitpid()` since `SA_RESTART` isn't a universal guarantee.

**Key mappings:** `signals.cpp::installSignalHandlers`; the `do {...} while (EINTR)` loops in `executor.cpp` and `pipeline.cpp`.

---

### Q17. The shell installs a custom `SIGINT` handler so it survives Ctrl+C — but the code explicitly resets `SIGINT` to `SIG_DFL` in every forked child before `exec()`. Why?

**Polished answer:** Signal dispositions are inherited across `fork()` (the child starts with the same handler table as the parent) but are reset to default across `exec()` *only* for handlers that were set to a custom function pointer — because the new program image doesn't have that function anymore, `exec()` is forced to reset those to `SIG_DFL` automatically... except that's actually not fully reliable to depend on, and more importantly, there's a **window between `fork()` and `exec()`** where the child is still running the shell's own code with the shell's own signal disposition. If the user hits Ctrl+C during that window (e.g. while `applyRedirections()` is running in the child before `execvp()`), and the child still had the shell's SIGINT handler (which just sets a flag and returns — it wouldn't terminate the child), the child would *survive* a Ctrl+C the user expected to kill it, and worse, once it execs into a real program, the exec-triggered reset means the exec'd program does get default behavior anyway — but only *after* that gap. So resetting to `SIG_DFL` explicitly, immediately after `fork()`, closes that gap deterministically and matches the semantics every real shell provides: foreground commands terminate on Ctrl+C, the shell itself never does.

**TL;DR:** The shell's own `SIGINT` handler is only appropriate for the shell process itself; the child inherits it across `fork()`, so it's explicitly reset to `SIG_DFL` right after forking (before `exec()`) so the child — including during the pre-exec redirection setup window — responds to Ctrl+C like a normal program, and the shell process itself never dies from it.

**Key mappings:** the `sigaction(SIGINT, &sa, nullptr)` block right after `fork()` in both `executor.cpp::executeSingle` and `pipeline.cpp::executePipelineChain`.

---

### Q18. What is `sig_atomic_t`, and why is `volatile` required alongside it for the flags shared between signal handlers and the main loop?

**Polished answer:** `sig_atomic_t` (from `<csignal>`) is a type the C standard guarantees can be read and written as a single indivisible operation even if a signal arrives in the middle of an access — so you'll never observe a torn/partial value. That solves the *atomicity* half of the problem. `volatile` solves a different, compiler-level problem: without it, the compiler is free to assume a variable never changes underneath the currently-executing code (since from the compiler's single-threaded-control-flow point of view, nothing in the visible instruction stream modifies it), and could legally cache `g_sigchld_pending` in a register and never re-read memory, or even eliminate the "empty" check entirely if it can't see the value changing — because a signal handler running asynchronously isn't part of that visible control flow. `volatile` forces every access to actually hit memory, which is exactly what's needed since the "writer" (the signal handler) is invisible to the compiler's optimizer. Together, `volatile sig_atomic_t` is the minimum correct primitive for a signal-to-mainline communication flag in C/C++ — note this is *not* the same as thread-safety (it says nothing about memory ordering across cores/threads), it's specifically about signal-handler reentrancy on a single thread.

**TL;DR:** `sig_atomic_t` guarantees the read/write itself can't be torn by an interrupting signal; `volatile` stops the compiler from caching/eliminating reads of a variable it can't see being written by the (invisible-to-it) signal handler. Together they're the correct, minimal tool for signal→mainline flag communication — not a general thread-safety mechanism.

**Key mappings:** `signals.hpp` (declarations), `signals.cpp` (handler writes), `main.cpp` (mainline reads/clears).

---

### Q19. Trace the full lifecycle of every file descriptor involved in a 2-command pipeline, from `pipe()` to the last `close()`.

**Polished answer:** Say `cmd1 | cmd2`. `pipe(&pipe_fds[0])` creates fds, call them `r` (read end) and `w` (write end), both open *in the parent* at this point. `fork()` for child 0 (`cmd1`): the child inherits copies of `r` and `w` (same open file descriptions, new fd-table entries in the child). Child 0 has no previous stage, so it skips the stdin dup2; it does `dup2(w, STDOUT_FILENO)` so its stdout now points at the pipe's write end (fd 1 is closed-then-repointed by dup2). It then closes **both** `r` and `w` (its original copies — the dup2'd fd 1 stays open since dup2 gives it a fresh reference, distinct from the original `w` fd number if they differ, or a no-op if they happened to coincide). Then `execvp("cmd1", ...)`. `fork()` for child 1 (`cmd2`): also inherits `r`/`w`; does `dup2(r, STDIN_FILENO)` (skips the stdout dup2 since it's the last stage), then closes both original `r` and `w`, then `execvp("cmd2", ...)`. Back in the **parent**, immediately after both forks return, it closes both `r` and `w` too — critically, the parent never needs either end, but if it *didn't* close them, the parent (which lives on as the shell) would hold the write end open even after `cmd1` exits, so `cmd2`'s `read()` on the pipe would never see EOF and would block forever waiting for more data that's never coming. Finally the parent `waitpid()`s both children in order.

**TL;DR:** Parent creates pipe → forks both children (each inherits both ends) → each child dup2's the end it needs onto stdin/stdout, then closes *both* original ends → parent closes both ends right after forking (before waiting) — otherwise the parent's own lingering write-end reference prevents EOF and hangs the pipeline.

**Key mappings:** `pipeline.cpp::executePipelineChain` end to end.

---

### Q20. What happens if a child process in a pipeline forgets to close the read end of a pipe it doesn't need (or the parent forgets to close its copies)? Describe the exact failure mode.

**Polished answer:** A pipe's read side gets `EOF` (a `read()` returning 0) only when **every** file descriptor referencing its write end, across **every** process, has been closed. If any process — including the shell itself — still has the write end open, even if that process never writes to it again, the kernel has no way to know no more data is coming, so a reader blocked in `read()` on the other end just... blocks forever. This manifests as a program that looks "stuck" with no error message: e.g., `cat file | grep foo` would hang indefinitely after `grep` finishes reading available data, because it's still waiting for possible more input, since some fd somewhere (say, the parent shell process that forgot to close its copy of the write end) is technically still capable of writing. This is precisely why `executePipelineChain()` closes *all* `2*(n-1)` pipe fds in *every* child (not just the ones it dup2'd) and in the parent, immediately — it's not paranoia, it's the single rule that prevents every pipeline-hang class of bug.

**TL;DR:** A pipe reader only sees EOF once *every* fd referencing the write end (in every process) is closed; leaving even one stray copy open — commonly in the shell process itself — causes the reader to block forever with no error, which is why the code closes every pipe fd in every process that doesn't need it, immediately after wiring.

**Key mappings:** the "close ALL pipe fds" loops in `pipeline.cpp::executePipelineChain` (child block and parent block).

---

### Q21. What's the difference between `WNOHANG` and a plain blocking `waitpid()` call, and where does your code use each?

**Polished answer:** `waitpid(pid, &status, 0)` blocks the calling process until the specified child changes state (exits or is killed) — the caller does nothing else until then. `waitpid(pid, &status, WNOHANG)` returns immediately regardless: it returns the child's PID if that child has already exited (collecting its status), returns 0 if the child is still running, and returns -1 on error. ShellX uses the blocking form for **foreground** commands in `executeSingle()` — the shell genuinely has nothing useful to do until the foreground command finishes, so blocking is correct and simple. It uses `WNOHANG` in `reapFinishedJobs()` for **background** jobs — the shell must keep running the REPL (accept new input, show the prompt) while background jobs are still executing, so it can only ever *check* on them non-blockingly, from a spot in the loop it controls, never sit and wait.

**TL;DR:** Blocking `waitpid()` (flag `0`) is correct when the shell has nothing else to do (foreground commands); `WNOHANG` is correct when the shell must remain responsive while children run in parallel (background jobs), returning immediately either way.

**Key mappings:** `executor.cpp::executeSingle` (blocking), `jobs.cpp::reapFinishedJobs` (`WNOHANG`).

---

### Q22. What is a process group, and why does the README call out "no process group control" as a known limitation? What concretely breaks without it?

**Polished answer:** A process group is a kernel-level grouping of related processes (typically a pipeline) that lets the terminal driver and job-control mechanisms treat them as one unit — most importantly, terminal-generated signals like `SIGINT` (Ctrl+C) and `SIGTSTP` (Ctrl+Z) are delivered to the *foreground process group* of the controlling terminal as a whole, not to an individual PID. Real shells call `setpgid()` to put each pipeline into its own new process group and use `tcsetpgrp()` to tell the terminal which group is currently "foreground," so that background jobs are deliberately left in a *different* group and don't receive Ctrl+C at all.

ShellX doesn't do this — every child, foreground or background, stays in the shell's own process group (the default at fork time, since it never calls `setpgid`). Concretely this means: if you background a long-running job (`sleep 100 &`) and then hit Ctrl+C while typing your next command, the terminal delivers `SIGINT` to the whole foreground process group, which — because there's no separation — can include that "background" job too, killing something the user explicitly wanted to keep running detached. This is documented as an intentional, known gap rather than a bug, deferred until the core fork/exec/pipe/signal machinery (everything else in this doc) was solid.

**TL;DR:** Process groups let the terminal target `SIGINT`/`SIGTSTP` at "the current foreground pipeline" as a unit and exclude backgrounded jobs; without `setpgid()`/`tcsetpgrp()`, background jobs share the shell's group and can be killed by a Ctrl+C the user intended only for the foreground — a documented, deferred limitation, not an oversight.

**Key mappings:** README.md "Known Limitations" section (no code implements this yet — good opportunity to describe as a "here's what I'd build next" answer).

---

### Q23. How would you extend ShellX to support `fg`/`bg` and Ctrl+Z (job control), given the current architecture?

**Polished answer:** Three layered additions. First, process groups (Q22): when forking a new pipeline, call `setpgid(pid, pipeline_leader_pid)` in *both* parent and child (redundantly, to avoid a race over which runs first) so the whole pipeline shares one group ID distinct from the shell's own. For foreground pipelines, call `tcsetpgrp(STDIN_FILENO, pipeline_pgid)` to hand the terminal to that group, and `tcsetpgrp(STDIN_FILENO, shell_pgid)` to take it back once the pipeline finishes or is stopped — this is what makes Ctrl+C/Ctrl+Z land on the right group. Second, handle `SIGTSTP` similarly to how `SIGCHLD` is handled now — a flag-setting handler plus `WUNTRACED` passed to `waitpid` so a *stopped* (not just exited) child is detected, and extend the `Job` struct with a tri-state status (`Running`/`Stopped`/`Done`) instead of the current boolean. Third, implement `fg`/`bg` as new builtins: `fg %N` calls `tcsetpgrp()` to give that job's group the terminal, sends `SIGCONT` if it was stopped, and does a blocking `waitpid(..., WUNTRACED)` on it (now it's "foreground" again); `bg %N` just sends `SIGCONT` without taking the terminal, leaving it running in the background. The existing `g_jobs` vector, `reapFinishedJobs()` polling loop, and the SIGCHLD-flag pattern all stay — this is additive, not a rewrite.

**TL;DR:** Add `setpgid`/`tcsetpgrp` per pipeline, handle `SIGTSTP` with the same flag-based pattern as `SIGCHLD` plus `WUNTRACED`, extend `Job` to a tri-state status, and add `fg`/`bg` builtins that toggle terminal ownership and send `SIGCONT` — built on top of, not replacing, the existing job-list/signal-flag architecture.

**Key mappings:** extension of `jobs.hpp`/`jobs.cpp`, `signals.cpp`, new builtins in `builtins.cpp`.

---

### Q24. Your code says backgrounded pipelines (`cmd1 | cmd2 &`) are explicitly unsupported. Why, and how would you add support?

**Polished answer:** The `Job` struct only tracks a single `pid_t`. A pipeline has N processes, so "is this background job still running" isn't a single-PID question anymore — you'd need to track a *set* of PIDs (or at minimum the group leader's PID, if process groups were implemented per Q22/23) and decide what "done" means: all N exited? Just the last one (matching the pipeline's own exit-status convention of "last command wins")? `executePipeline()` currently detects this case explicitly and prints an error rather than mishandling it (consistent with the general "loud, immediate error over silent wrong behavior" philosophy from Q12), rather than, say, only tracking the last command's PID and silently leaking the earlier stages as untracked children.

To add support: extend `Job` to hold `vector<pid_t> pids` (or adopt process groups and store just the `pgid`, checking liveness via `waitpid(-pgid, ...)` semantics or by tracking each member), have `executePipelineChain()` register the whole set with `addJob()` instead of returning synchronously when `pipeline.background` is true (currently it just refuses), and update `reapFinishedJobs()` to poll every PID in the set with `WNOHANG` and only mark the job "Done" once all members have been reaped — printing the *last* command's exit status to match the existing pipeline exit-status convention.

**TL;DR:** `Job` currently tracks one PID; a backgrounded pipeline has N processes, so it needs either a PID set or (better, combined with process-group work) a single `pgid` to track/reap as a unit — currently refused explicitly rather than silently only tracking one stage and leaking the rest.

**Key mappings:** `executor.cpp::executePipeline` (the explicit refusal), `jobs.hpp::Job` (single-`pid_t` limitation).

---

### Q25. Why does `Pipeline` store `background` as a single field on the whole pipeline, rather than each `Command` having its own background flag? Isn't that less flexible?

**Polished answer:** It's less flexible in the sense that the *grammar itself* doesn't allow `cmd1 & | cmd2` — but that's a deliberate, correct restriction, not a missing feature: in this shell's grammar (matching real shells), `&` only ever applies to the trailing end of an entire pipeline, never to an individual stage mid-pipe. Given that, storing a `bool` on every `Command` would introduce a state that's *always* either unused (for every command except conceptually "the last one") or, worse, could be set inconsistently by a bug (e.g., two different commands in the same pipeline disagreeing about background-ness, which is meaningless but is a state the type system would otherwise allow). Putting `background` only on `Pipeline` makes the invalid state unrepresentable — there's no way for the code to ever ask "is this individual command backgrounded" and get a different answer depending on which command in the pipeline you asked, because that's not even a question the data model permits. It's a small example of a broader principle: model your data so that inconsistent states can't be constructed, rather than validating consistency after the fact. The `shellx.hpp` comment on the struct spells this out explicitly.

**TL;DR:** `&` only ever applies to a whole pipeline in this grammar, never to one stage — storing `background` once on `Pipeline` (rather than duplicated per-`Command`) makes "commands in the same pipeline disagree about background-ness" an unrepresentable state instead of a bug you have to guard against.

**Key mappings:** `shellx.hpp` (`Command`/`Pipeline` struct comments — this is explicitly called out in the code).

---

# 26–50 — Depth, edge cases, tradeoffs & "extend this" questions

### Q26. Trace exact fd numbers/operations for a 3-stage pipeline `cmd1 | cmd2 | cmd3`. How many pipes, and what does the middle process do differently from the ends?

**Polished answer:** N=3 commands need N-1=2 pipes — call them pipe A (between cmd1/cmd2) and pipe B (between cmd2/cmd3), giving 4 fds total: `A_r, A_w, B_r, B_w`, all initially open in the parent, all inherited by all 3 forked children. Child 0 (`cmd1`, `i=0`): no stdin dup2 (it's first — `i>0` is false), `dup2(A_w, STDOUT)` since `i < n-1`, then closes all 4 original fds, execs. Child 1 (`cmd2`, `i=1`) is the interesting middle case: `dup2(A_r, STDIN)` because `i>0`, **and** `dup2(B_w, STDOUT)` because `i<n-1` — it's the only stage that rewires *both* ends, reading from the previous pipe and writing to the next one, and note it uses two *different* pipes for its two ends, not the same one. Then it closes all 4 original fds too, execs. Child 2 (`cmd3`, `i=2`): `dup2(B_r, STDIN)` since `i>0`, no stdout dup2 since `i == n-1` (last), closes all 4, execs. The parent closes all 4 immediately after the fork loop, then `waitpid()`s all three in order, reporting child 2's (the last one's) exit status.

**TL;DR:** 2 pipes for 3 commands; the middle stage is the only one that dup2's *two different pipes* (previous pipe's read end → stdin, next pipe's write end → stdout); every process still closes all 4 fds regardless of how many it actually used.

**Key mappings:** `pipeline.cpp::executePipelineChain`, generalizing the `i>0`/`i<n-1` conditionals for `i` in the middle.

---

### Q27. `isBuiltin()` is a linear chain of string comparisons over 4 names. Is that a problem? How would you scale it to, say, 30 builtins?

**Polished answer:** For 4 fixed short strings, a linear `==` chain is essentially free — string comparison with early length mismatch is O(1)-ish in practice for distinct-length/prefix strings, and 4 comparisons is nothing compared to the cost of even a single `fork()`, so optimizing this specific function would be solving a non-problem at this scale; the honest answer in an interview is "this is fine here, and rewriting it would be premature optimization." That said, if I were scaling to dozens of builtins with actual dispatch logic (not just a membership check), I'd restructure to a `std::unordered_map<std::string, int(*)(const Command&)>` (function pointer, or `std::function` if state capture is needed) built once at startup — `isBuiltin()` becomes `map.count(name)`, and `executeBuiltin()` becomes `map.at(name)(cmd)` instead of a growing if/else chain, which is both O(1) average dispatch and — more valuably than the perf gain — much easier to read and extend without touching a giant switch statement.

**TL;DR:** 4 fixed strings: a linear chain is genuinely fine, not a real bottleneck next to `fork()` cost. At real scale (dozens of builtins with actual per-command logic), I'd switch to a `name → handler function` map built once at startup for O(1) dispatch and better extensibility.

**Key mappings:** `builtins.cpp::isBuiltin`, `builtins.cpp::executeBuiltin`.

---

### Q28. What are `MAX_PIPELINE_LENGTH`, `MAX_INPUT_LENGTH`, and `MAX_BACKGROUND_JOBS` for? Why bound these at all?

**Polished answer:** These are defensive limits (`shellx.hpp`: 16 commands per pipeline, 4096 chars per input line, 64 background jobs) against unbounded resource consumption from a single bad or malicious input line — without them, a pathological input like a pipeline with thousands of stages would try to `fork()` thousands of processes and create thousands of pipe fd pairs in a tight loop, potentially exhausting the process's fd table (`RLIMIT_NOFILE`) or the system's process table before any useful error is reported, and would degrade with a confusing low-level `fork: Resource temporarily unavailable` error instead of a clear "pipeline too long" message. `MAX_INPUT_LENGTH` matters because `std::getline` itself has no built-in length cap — main.cpp explicitly checks `line.size() > MAX_INPUT_LENGTH` after reading, since an attacker (or just a pasted huge blob) could otherwise hand the tokenizer an unbounded string. `MAX_BACKGROUND_JOBS` caps the `g_jobs` vector from growing unboundedly if a user backgrounds far more jobs than intended — note the code still tracks the job past the limit but warns, rather than silently dropping job tracking (a "fail loud, don't fail silent" choice again).

**TL;DR:** Fixed, generous-but-finite limits on pipeline length, input line length, and tracked background jobs — defensive bounds that turn "resource exhaustion with a cryptic OS-level error" into "clear, immediate shellx error," at negligible cost to real usage.

**Key mappings:** `shellx.hpp` (constants), `parser.cpp::parse` (pipeline-length check), `main.cpp` (input-length check), `jobs.cpp::addJob` (background-jobs check).

---

### Q29. Why bother distinguishing 126 vs 127 vs a generic "fork failed" `1`? Doesn't the user just see "it didn't work" either way?

**Polished answer:** These distinct codes matter enormously to anything that *scripts* against the shell, even though a human staring at a single failed interactive command might not care. `$?` (or equivalent) is how shell scripts branch on failure type — `command -v foo || echo "not installed"` patterns, CI scripts checking for 127 specifically to detect a missing tool versus 126 to detect a permissions problem versus some other code meaning the *program itself* reported an application-level error, are all real, common patterns. A generic `1` for every failure would make it impossible for any calling script to distinguish "your PATH is broken" from "the file exists but isn't executable" from "the program ran and decided to fail" — all meaningfully different problems requiring different fixes. This is also just adherence to POSIX/shell convention, which matters for interoperability even in a from-scratch educational shell like this one.

**TL;DR:** Distinct exit codes let *calling scripts*, not just humans, programmatically distinguish "not found" vs "not executable" vs "ran and application-level failed" — collapsing them to a generic `1` would make the shell impossible to script against reliably.

**Key mappings:** exit-code logic in `executor.cpp::executeSingle` and `pipeline.cpp::executePipelineChain`.

---

### Q30. Explain the `do { result = waitpid(...); } while (result == -1 && errno == EINTR);` idiom. Why is it needed even with `SA_RESTART` set?

**Polished answer:** This is the standard idiom for making a blocking syscall robust against spurious interruption by *any* signal, not just the ones this program explicitly installs handlers for — `SA_RESTART` is a per-signal-handler flag, and its restart guarantee (a) only applies to signals for which *you* control the handler's flags, and (b) is documented as not covering every syscall on every platform/kernel version consistently (`waitpid` restart behavior has historically had platform quirks). Rather than trust that guarantee to hold everywhere, the code treats `EINTR` from `waitpid()` as "try again" unconditionally: loop back and re-call `waitpid()` on the *same* target until it either succeeds or fails with a real error. This is a well-known, idiomatic C pattern (appears constantly in real systems code — `read()`/`write()` loops guard the same way) precisely because relying purely on `SA_RESTART` is considered fragile practice; the explicit retry is the actual correctness guarantee, `SA_RESTART` is just an optimization that makes the retry loop rarely trigger in practice.

**TL;DR:** `EINTR` means "a signal interrupted this blocking call, nothing went wrong, just try again" — the retry loop is the portable, robust guarantee; `SA_RESTART` is a nice-to-have that reduces how often the loop actually has to retry, not a substitute for it.

**Key mappings:** every `EINTR` retry loop in `executor.cpp` and `pipeline.cpp`.

---

### Q31. `execvp` vs `execv`, `execl`, `execve` — what's the difference, and why `execvp` here?

**Polished answer:** The `exec` family differs on two independent axes. Axis one: argument passing style — `l` variants take arguments as a variadic C list (`execl(path, arg0, arg1, ..., NULL)`), `v` variants take a `char* const argv[]` array — `v` is the natural fit here since ShellX already has arguments in a `std::vector<std::string>` that it needs to convert to a null-terminated C array anyway (which is exactly what the code does: builds a `vector<const char*>`, ending with `nullptr`). Axis two: PATH resolution and environment — plain `exec(v/l)` requires an exact path to the executable and does not search `$PATH`; the `p`-suffixed variants (`execvp`, `execlp`) *do* search `$PATH` the same way a shell normally resolves bare command names like `ls`; and `e`-suffixed variants (`execve`, `execle`) let you pass an explicit custom environment array instead of inheriting the caller's. ShellX uses `execvp` specifically because it needs both: argv-array-style arguments (matches its existing data) *and* PATH search (so users can type `ls` instead of `/usr/bin/ls`), while inheriting the shell's own environment is exactly the desired behavior (no need for `execve`'s custom-environment feature). The code comment even states the reasoning directly: "`execvp` searches PATH — don't reimplement path resolution."

**TL;DR:** `v` = array-style args (matches the existing `vector<string>`), `p` = searches `$PATH` for a bare command name, no `e` needed since inheriting the shell's environment is exactly what's wanted — `execvp` is the one combination that needs zero manual PATH-resolution or environment-building code.

**Key mappings:** `executor.cpp::executeSingle`, `pipeline.cpp::executePipelineChain` (both call `execvp`).

---

### Q32. If `fork()` fails partway through building a 5-stage pipeline (say child 3 of 5 fails to fork), how does the code clean up the pipes and the 2 children that already forked successfully?

**Polished answer:** `executePipelineChain()` handles this explicitly rather than leaking. If `fork()` returns -1 for child `i`, the code: (1) `perror("fork")`, (2) closes **all** `2*(n-1)` pipe fds in the parent immediately (nothing left half-open), (3) loops over children `0..i-1` — the ones that *did* successfully fork — and `waitpid()`s each of them (with the same `EINTR` retry pattern used everywhere else) so they don't become zombies even though the overall pipeline is being aborted, and (4) returns `1` (generic failure) without ever calling `executePipeline`'s multi-command success path. Note those already-forked children will likely fail on their own — e.g. they may already have dup2'd into a pipe expecting a downstream partner that will never exist and exec'd into a program that gets unexpected EOF/broken-pipe behavior — but that's an orthogonal concern; the cleanup code's job is specifically to not leak fds or zombies from the partial pipeline, which it does correctly by reusing the exact same reap logic as the success path.

**TL;DR:** On a mid-pipeline `fork()` failure, the code closes all pipe fds and `waitpid()`s every already-forked child before returning an error — reusing the same close-everything/reap-everything discipline as the success path, so a partial failure leaks neither fds nor zombies.

**Key mappings:** `pipeline.cpp::executePipelineChain` (the `if (pid == -1)` branch inside the fork loop).

---

### Q33. `killAllJobs()` sends `SIGTERM` to background jobs on exit but explicitly does *not* wait for or reap them. Isn't that leaving zombies?

**Polished answer:** No — and this is a subtle but important distinction from "leaving zombies" versus "leaving orphans," which the design decisions table calls out explicitly. A zombie requires a *parent that's still alive but hasn't reaped*. Here, the shell process itself is about to call `exit`/terminate (`executeBuiltin`'s `exit` branch returns `-1`, which `main.cpp` treats as "break the REPL loop," after which the process ends). Once the shell process itself terminates, any children it hasn't reaped yet are automatically re-parented by the kernel to `init` (or the nearest subreaper) — and `init`'s entire job includes calling `wait()` on all its children indefinitely, specifically to clean up exactly this situation. So the "unreaped" children don't stay zombies forever; they briefly become zombies (if they've already exited) or keep running (if `SIGTERM` hasn't taken effect yet) and then get reparented and reaped by `init` moments later. Explicitly *not* waiting here is correct and even necessary — if `killAllJobs()` tried to block waiting for every background job to actually terminate before the shell could exit, a job that ignores `SIGTERM` or takes a while to clean up would hang the *shell's own exit*, which is worse UX than a very brief zombie window that the kernel handles automatically anyway.

**TL;DR:** Not reaping on exit is correct, not a leak — the shell process is terminating anyway, so any unreaped children get re-parented to `init`, whose entire purpose includes reaping orphans; blocking `exit` on every background job's termination would be strictly worse UX for no correctness benefit.

**Key mappings:** `jobs.cpp::killAllJobs`, README "Design Decisions" table row on "Exit behavior."

---

### Q34. Why does a pipeline report the *last* command's exit status, even if an earlier stage failed? Isn't that misleading?

**Polished answer:** It's exactly bash's default behavior (without `set -o pipefail`), and ShellX deliberately matches it rather than inventing different semantics — `pipeline_status` in `executePipelineChain()` is only ever assigned from the loop iteration where `i == n-1`, i.e., the final command. It can be misleading in exactly the classic case people bring up in interviews: `grep pattern nonexistent_file | wc -l` — `grep` fails (file not found, or no match), but `wc -l` still successfully counts 0 lines and exits 0, so the whole pipeline reports success even though the first stage failed. This is a genuinely known, debated shell design point — it's why `bash`'s `set -o pipefail` exists as an opt-in override (report failure if *any* stage failed, not just the last). ShellX doesn't implement `pipefail`-style tracking (it doesn't even track intermediate exit statuses beyond waiting on them), but that would be a small, natural addition: track every stage's `WIFEXITED`/`WEXITSTATUS` result in the wait loop, not just the last, and take the max/first-nonzero if a `pipefail`-equivalent flag were ever added.

**TL;DR:** Deliberately matches bash's non-`pipefail` default — only the last stage's status is reported, meaning an earlier failing stage can be masked by a later successful one (`grep missing | wc -l` "succeeding"); this is a known, documented shell semantics point, not a bug, and adding `pipefail` support would be a natural, small extension.

**Key mappings:** `pipeline.cpp::executePipelineChain` (`if (i == n - 1)` guard on status assignment), README "Design Decisions" table.

---

### Q35. What exactly is an "async-signal-safe" function, and give concrete examples of functions used elsewhere in this codebase that would be unsafe to call from inside a signal handler.

**Polished answer:** Async-signal-safety means a function can be safely called from inside a signal handler that might interrupt *any* point of the program's normal execution — including, critically, a point where the program was already in the middle of calling that very same function (or one that shares internal state with it, like a heap allocator lock or a `stdio` buffer lock). POSIX publishes an explicit list of guaranteed-safe functions: things like `write()`, `_exit()`, `signal()`, a handful of others — notably a short list, and notably *not* including `printf`/`cout` (which buffer internally and may hold a lock), `malloc`/`new` (the heap allocator uses internal locks/data structures that could be mid-mutation when the signal arrives, risking corruption or deadlock if the handler also tries to allocate), or most STL container operations (which may allocate). In this codebase specifically: `std::cout <<` (used throughout `main.cpp`, `jobs.cpp` for job-completion messages) is unsafe to call from a handler; `g_jobs.push_back()`/`.erase()` (uses the heap allocator internally) would be unsafe; `waitpid()` itself is technically on the safe list, but reaping from *within* the handler is still avoided here for the race-condition reasons in Q5, not purely a safety-list technicality. This is exactly *why* the actual handlers (`sigchld_handler`, `sigint_handler` in `signals.cpp`) do nothing but a single assignment to a `sig_atomic_t` — everything else (the `cout` calls, the `g_jobs` mutations) is deliberately deferred to ordinary mainline code.

**TL;DR:** Async-signal-safe = safe to call even if it interrupts itself mid-execution; `cout`/`printf` (buffering/locks) and `malloc`/`new`/STL containers (allocator locks) are *not* on that list — which is exactly why every handler in this codebase does nothing but flip a `sig_atomic_t` flag and defers all real work (printing, container mutation) to the REPL's mainline code.

**Key mappings:** `signals.cpp` (both handlers), contrasted with `jobs.cpp`/`main.cpp` (where the unsafe operations actually happen, safely, outside signal context).

---

### Q36. Explain "double-flushing of stdio buffers" concretely — what would you actually observe on screen if a child used `exit()` instead of `_exit()` after a failed `exec()`?

**Polished answer:** Say the shell had printed something to its buffered `std::cout` (or the underlying C `stdout`) that hadn't been flushed to the terminal yet at the moment `fork()` ran — both parent and child now have their own *copy* of that unflushed buffer content in their respective memory (fork copies the whole address space, buffers included). If the child then fails to `exec()` (say, command not found) and calls `exit()` instead of `_exit()`, `exit()`'s cleanup path flushes `stdout`, writing that leftover buffered content to the terminal — but the *parent* will also, at some later point in its own normal execution, flush that same unflushed content (its own copy of it) to the terminal too. The visible symptom is the same partial output line (or block of output) appearing twice, interleaved unpredictably with everything else being printed, which is a genuinely confusing bug to chase down because it looks like a logic error in *what* is being printed rather than a process-lifecycle issue in *how many times* the buffer gets flushed. `_exit()` sidesteps this by never running the flush path at all in the never-going-to-exec child.

**TL;DR:** Fork copies unflushed buffer contents into both processes; `exit()` in a child that's about to abort (not exec) would flush that copy too, duplicating whatever output was pending at fork time — `_exit()` skips the flush entirely, avoiding visibly duplicated terminal output.

**Key mappings:** every `_exit()` call site in `executor.cpp`/`pipeline.cpp`; README design-decisions table row "Child termination."

---

### Q37. Your REPL is a single-threaded loop that checks flags at the top (`g_sigchld_pending`, `g_sigint_received`) rather than using a multi-threaded design. What are the tradeoffs versus, say, a dedicated signal-handling thread?

**Polished answer:** The flag-check pattern (sometimes called a "self-pipe"-adjacent or simple polling pattern, though this specific version is even simpler — no actual pipe, just a checked global) keeps the entire program single-threaded, which sidesteps an entire category of bugs: no need for mutexes/atomics beyond the signal-safe primitives already required anyway, no risk of the job list or REPL state being read/written concurrently from two different threads, and no thread-creation/joining complexity around `fork()` (which is notoriously awkward with threads — only the forking thread survives into the child, so a multi-threaded parent that forks has to be very careful about what state/locks the child inherits in a possibly-inconsistent state). The tradeoff is responsiveness: the flags are only checked at specific points (top of the REPL loop, and once more right after `executePipeline()` returns) — so a background job finishing *while the shell is blocked inside a long-running foreground command's `waitpid()`* won't get its "Done" message printed until the *next* time control returns to those check points, not the instant it actually finishes. In practice this is a minor, cosmetic delay (the job did still get reaped correctly by the eventual check — nothing is lost, it's purely a "when do we print the notification" question), and the massive simplicity win of staying single-threaded is a good trade for a project like this. A dedicated signal-thread approach exists in real systems (e.g., blocking signals on all threads and having one thread call `sigwait()` in a loop) but adds real complexity for a benefit (near-instant background-completion notification) that most users never notice.

**TL;DR:** Single-threaded flag-polling avoids an entire class of concurrency bugs and plays nicely with `fork()`'s single-thread-survives semantics, at the cost of background-job-completion messages appearing slightly late (only at the next REPL checkpoint) rather than the instant they happen — a good trade for a project at this scope.

**Key mappings:** `main.cpp` (the two flag-check points in the REPL loop).

---

### Q38. How would you unit-test fork-based logic like this, given that `fork()`/`waitpid()` are hard to isolate in a normal unit test? What did the actual test script do instead?

**Polished answer:** `tests/test_shellx.sh` takes the pragmatic, correct approach for this kind of program: black-box, process-level integration testing rather than trying to unit-test individual functions that fork. It pipes a sequence of shellx commands into the actual compiled `./build/shellx` binary via stdin, captures combined stdout+stderr, and asserts either a substring appears in the output or the process's exit code matches an expectation — testing observable behavior (did `ls | grep foo` actually filter, did `cmd &` actually print a job number, did exiting after backgrounding actually send SIGTERM) rather than internal implementation. This is the right level for this kind of code because the actual correctness properties that matter (no hangs, no zombies, correct fd wiring) are fundamentally properties of *process interaction*, not of any single pure function — the tokenizer/parser is the one part of the codebase that's naturally pure/testable in isolation (`parse()` takes a string, returns a `Pipeline` struct, no syscalls), so if I were extending test coverage I'd add narrower unit tests specifically for `parser.cpp`'s edge cases (quoting, malformed redirections, `rejectUnsupported` cases) since those don't need a subprocess at all, while keeping everything fork/exec/pipe/signal-related as black-box integration tests exactly like the existing script does. I'd also reach for the debug build with ASan/UBSan (already wired into `CMakeLists.txt`/README) run under the same test script — sanitizers catch fd/memory issues that black-box output-matching alone would miss.

**TL;DR:** The project uses black-box process-level integration tests (feed input via stdin, assert on output substrings/exit codes) — the right approach since correctness here is about process interaction, not pure functions; `parser.cpp` is the one genuinely unit-testable module in isolation, and running the same integration suite under ASan/UBSan would catch memory/fd issues output-matching alone can't.

**Key mappings:** `tests/test_shellx.sh`, `CMakeLists.txt`/README debug-build-with-sanitizers instructions.

---

### Q39. Why compile with `-Wall -Wextra -Wpedantic`, and why bother with `-fsanitize=address,undefined` for a project like this?

**Polished answer:** `-Wall -Wextra -Wpedantic` turns on broad warning categories (`-Wall`/`-Wextra` catch things like unused variables, sign-compare mismatches, potentially uninitialized reads, suspicious implicit conversions; `-Wpedantic` enforces strict ISO C++ standard conformance rather than accepting compiler-specific extensions) — cheap, compile-time-only, zero-runtime-cost bug detection that's especially valuable in raw-syscall systems code where a wrong-signedness comparison or an uninitialized `struct sigaction` field is exactly the kind of subtle bug that's otherwise invisible until it manifests as a rare crash. `-fsanitize=address` (ASan) instruments memory accesses to catch use-after-free, heap/stack buffer overflows, and double-frees at the point they happen (not just when they eventually corrupt something visibly) — genuinely important here because this code does manual fd-array indexing (`pipe_fds[2*i]`) and manages raw C arrays (`argv` built as `vector<const char*>`) where an off-by-one is very plausible. `-fsanitize=undefined` (UBSan) catches undefined behavior like signed integer overflow, misaligned pointer access, or invalid enum values — the kind of bug that "works by accident" on one compiler/platform and breaks on another. For a project whose entire value proposition is "I understand what's actually happening at the syscall/memory level," building and testing under both sanitizers is exactly the due diligence that backs that claim up, and it's a strong thing to mention proactively in an interview even if not asked directly.

**TL;DR:** `-Wall -Wextra -Wpedantic` catches a broad class of bugs at zero runtime cost, at compile time; ASan/UBSan catch memory-safety and undefined-behavior bugs at the exact instant they occur during test runs — both are exactly the right due-diligence tools for hand-rolled syscall/fd/pointer-heavy systems code like this.

**Key mappings:** `CMakeLists.txt` (`-Wall -Wextra -Wpedantic`), README "Build" section (sanitizer debug build instructions).

---

### Q40. Does fd-closing order matter — parent closing pipe fds *before* vs *after* it starts calling `waitpid()`? Walk through why the code closes them where it does.

**Polished answer:** Yes, order matters, and the code closes all pipe fds in the parent *immediately after the fork loop finishes*, strictly before entering the `waitpid()` loop. The reason connects directly to the EOF mechanism from Q20: if the parent kept its copies of the pipe write-ends open *while* waiting on children, and — hypothetically — an earlier-stage child in the pipeline finished and closed its own write-end copy, the *later*-stage child reading from that pipe still wouldn't see EOF, because the parent's copy of that same write-end fd is still open, even though the parent process itself never writes to it. The reader would then block in its own `read()` call indefinitely, and since the parent is meanwhile sitting in `waitpid()` for that very child, you get a genuine deadlock: parent waiting for a child that's waiting for EOF that only the parent (which isn't going to provide it) can trigger by closing its fd. Closing everything *before* waiting removes any possibility of this — by the time any `waitpid()` call happens, the *only* processes that can possibly still hold pipe fds open are the pipeline's own children, so EOF propagation is guaranteed to work correctly based purely on the children's own lifecycle.

**TL;DR:** The parent closes all pipe fds *before* it starts `waitpid()`-ing, not after — because if it waited first while still holding write-end copies open, a downstream reader could never see EOF (the parent's lingering fd prevents it), creating a genuine deadlock between "parent waiting on a child" and "child waiting on an EOF only the parent can now trigger."

**Key mappings:** `pipeline.cpp::executePipelineChain` — note the parent's close-loop is physically positioned before the wait-loop in the function body.

---

### Q41. Trace what happens for `ls | fakecmd_that_does_not_exist`. Does the pipeline hang, error immediately, or something else?

**Polished answer:** It runs to completion cleanly, just with an error printed and a "failure" exit status — it doesn't hang. Both children fork successfully and get their fd wiring set up identically to any other pipeline (the parser/pipeline-setup code has no idea in advance whether a command name will resolve — that's only discovered at `execvp()` time inside each child, independently). Child 0 execs `ls` successfully and writes its output to the pipe as normal. Child 1 calls `execvp("fakecmd_that_does_not_exist", ...)`, which fails and returns -1 with `errno == ENOENT`; the code path is `perror(argv[0])` (prints something like `fakecmd_that_does_not_exist: No such file or directory` to stderr) followed by `_exit(127)` — child 1 terminates immediately without ever reading from its stdin (the pipe). Because child 1 exits (closing its copy of the pipe's read end as part of normal process termination), and `ls` (child 0) will eventually finish writing and exit too — if `ls`'s output is large enough to fill the pipe's kernel buffer before child 1 exits, `ls` could actually get `SIGPIPE` (default action: terminate) or an `EPIPE` write error once it tries to write to a pipe whose reader has gone away, though for a typical small `ls` listing this usually isn't hit before child 1 has already exited on its own. Either way, the parent's `waitpid()` loop collects both exit statuses normally (no hang, since normal process exit closes fds and unblocks everything), and the pipeline's *reported* exit status is child 1's — the 127 from the exec failure, since that's the last command.

**TL;DR:** No hang — each child discovers exec success/failure independently at `execvp()` time; the failing stage `_exit(127)`s immediately (closing its fds as part of normal exit, which lets the pipeline finish normally), and the reported pipeline exit status is 127 since it's the last command's status; the earlier stage may occasionally see `SIGPIPE`/`EPIPE` if it tries to write after the reader has already exited, but that's independent, expected pipe behavior, not a hang.

**Key mappings:** `pipeline.cpp::executePipelineChain` exec-failure branch; general POSIX pipe/`SIGPIPE` semantics (good to mention even though not explicitly handled in this codebase).

---

### Q42. How would you add shell variable support (`$VAR`, `export FOO=bar`) on top of this architecture? Where exactly would substitution happen?

**Polished answer:** I'd add it as a distinct pass between tokenizing and the rest of parsing, keeping the layering intact: after `tokenize()` produces raw word tokens but before `rejectUnsupported`/pipe-splitting/redirection-extraction runs, walk each non-quoted word token and perform `$VAR`/`${VAR}` substitution by looking up an in-shell variable table (a new `std::unordered_map<std::string,std::string> g_shell_vars`, separate from the process environment) — this mirrors how quoting already gets special-cased in the tokenizer (double-quoted tokens are typically substitution-eligible; single-quoted, if I added single-quote support, would be literal — a good detail to mention, since it shows awareness of real shell semantics even though this shell currently only has double quotes). `export FOO=bar` would become a new builtin (alongside `cd`/`pwd`/`exit`/`jobs` in `builtins.cpp`) that both updates `g_shell_vars` *and* calls `setenv()` so exported vars actually propagate to forked children via the environment `execvp` inherits — non-exported assignment (`FOO=bar` alone) would only touch `g_shell_vars`, not the process environment, matching bash's distinction between shell variables and environment variables. The substitution pass staying separate from tokenizing keeps `tokenize()` itself simple and testable (raw lexical structure only), while variable expansion becomes its own clearly-scoped, independently-testable function — same design philosophy as the existing module boundaries (Q10).

**TL;DR:** Add a substitution pass between tokenizing and grammar-parsing that resolves `$VAR` tokens against a shell-variable table; `export` becomes a new builtin that also calls `setenv()` so exported variables actually reach forked children via the environment `execvp` inherits — kept as its own pass to preserve the existing clean module boundaries.

**Key mappings:** would sit between `parser.cpp::tokenize` and the rest of `parser.cpp::parse`; new builtin alongside `builtins.cpp`.

---

### Q43. What's missing to fully support Ctrl+Z / suspend-and-resume, beyond what you described for `fg`/`bg` in Q23?

**Polished answer:** Beyond the process-group/`SIGTSTP`-handling/`fg`-`bg` builtins already covered, two more pieces: first, terminal-mode considerations — a fully job-control-capable shell also needs to save/restore terminal attributes (`tcgetattr`/`tcsetattr`) per job in some implementations, so a suspended interactive program (like an editor mid-raw-mode) doesn't leave the terminal in a broken state when control returns to the shell; ShellX currently does none of this since it never suspends anything. Second, the REPL's `waitpid()` calls need `WUNTRACED` (and ideally `WCONTINUED`) added to their flags so a *stopped* child (via `SIGTSTP`/`SIGSTOP`) is reported as a distinct state transition rather than being invisible to `waitpid` until it actually exits — currently every `waitpid()` call site in this codebase only distinguishes "still running" vs "exited/killed," with no third "stopped" state at all, which is consistent with the codebase honestly not attempting job-control suspend/resume yet (again, documented as a known gap rather than a silent bug).

**TL;DR:** Beyond process groups and a `SIGTSTP` handler, you'd need `WUNTRACED`/`WCONTINUED` added to the `waitpid()` calls to detect stopped/resumed state transitions (currently only exited/running exist) and terminal-attribute save/restore per job to avoid leaving the terminal in a broken mode after suspending a program that changed it.

**Key mappings:** builds on Q22/Q23; current `waitpid()` calls in `executor.cpp`/`jobs.cpp` lack `WUNTRACED`.

---

### Q44. Explain why `cd`'s effect (changing directory) is scoped to the shell process only, and doesn't affect already-running external commands.

**Polished answer:** Current working directory is per-process kernel state (part of the process's entry in the kernel's process table, alongside things like its fd table and process group). It's established at process creation and only ever changed by that process itself calling `chdir()`/`fchdir()` — there's no mechanism for one process to reach into another already-running process and change its CWD. So when ShellX's `cd` builtin calls `chdir()`, it only ever changes the shell's own CWD. Any external command already running (in the background, say) keeps whatever CWD it had when it was `fork()`'d — fork copies the parent's CWD at that instant as a snapshot, it doesn't create an ongoing link back to the parent. This is exactly why `cd`, as established in Q6, has to be a builtin: if it were external (its own binary, forked and exec'd), it would change *its own* CWD in its own process, then exit, having no observable effect on the shell that spawned it — a subprocess fundamentally cannot mutate parent process state like CWD, environment variables, or open shell-level file descriptors, only its own.

**TL;DR:** CWD is per-process kernel state, set at process creation (via fork's snapshot) and changed only by a process calling `chdir()` on itself — no process can change another's CWD, which is precisely why `cd` must run in the shell's own process (Q6) rather than as a forked external command.

**Key mappings:** `builtins.cpp` (`cd` calling `chdir()` directly, no fork).

---

### Q45. Are there any security concerns with how `execvp` resolves commands via `$PATH` here? What's the classic PATH-related vulnerability class, and does this code have it?

**Polished answer:** The classic vulnerability is a **relative-path / current-directory PATH injection**: if `$PATH` contains `.` (the current directory) — especially if it appears *before* trusted system directories like `/usr/bin` — then running a common command name like `ls` inside a directory that happens to contain an attacker-planted file also named `ls` executes the attacker's file instead of the real system binary, since `execvp`'s PATH search finds it first. This isn't a bug in ShellX's own code — `execvp`'s PATH-search semantics are exactly what a shell is supposed to provide (Q31: "don't reimplement path resolution") — it's a property of *how the user's `$PATH` environment variable is configured*, which is inherited from whatever shell/environment launched ShellX in the first place, not something ShellX sets or validates. It's still worth knowing and mentioning proactively in an interview: a production-grade shell *could* choose to warn or refuse to search `.`/relative-path entries in `$PATH` by default (some real shells and distros patch/configure this defensively), but that would be an explicit policy decision layered on top of `execvp`'s default behavior, not something this codebase currently does or claims to do.

**TL;DR:** The PATH-search-finds-attacker's-file-in-cwd class of vulnerability is a property of how `$PATH` is configured (e.g. containing `.`), not a bug in ShellX's `execvp` usage — which correctly delegates to standard PATH search rather than reimplementing it — but it's a real, worth-knowing security consideration for any shell.

**Key mappings:** general POSIX knowledge tied to `executor.cpp`/`pipeline.cpp`'s `execvp` calls.

---

### Q46. Is `g_jobs` (a global `std::vector`) thread-safe? Why or why not, and why is that acceptable here?

**Polished answer:** `g_jobs` is not thread-safe in any general sense — plain `std::vector` operations like `push_back`/`erase` have zero built-in synchronization, and concurrent mutation from two threads would be a data race. It's acceptable here specifically because the *entire program is single-threaded* (Q37) — there is exactly one thread of control ever running normal (non-signal-handler) code, so there's never a scenario of two threads both calling `addJob`/`reapFinishedJobs` concurrently. The one thing that genuinely runs "asynchronously" relative to the main thread's flow is the signal handler — but, critically, `jobs.hpp`'s own comment states the invariant explicitly: the `SIGCHLD` handler *never touches `g_jobs`* at all, it only sets a flag; every actual read/write of `g_jobs` happens from ordinary mainline REPL code. So the safety argument isn't "vector operations happen to be safe here" (they wouldn't be, under concurrency) — it's "there is provably only ever one execution context that touches this data structure, by construction," which is a stronger and simpler guarantee than trying to add locks around a genuinely concurrent design would provide.

**TL;DR:** `std::vector` itself provides no synchronization, but that's fine here because the program is single-threaded and the signal handler is deliberately barred from ever touching `g_jobs` (only setting a flag) — so there is provably exactly one execution context that ever reads/writes it, making the "is it thread-safe" question moot by construction rather than by locking.

**Key mappings:** `jobs.hpp` (comment stating the SIGCHLD-handler-never-touches-this-vector invariant), `jobs.cpp`.

---

### Q47. How does the shell distinguish "command not found" (127) from "found but can't execute it" (126), and why would a script care about that distinction?

**Polished answer:** Both are detected at the same point — the `execvp()` call failing and returning -1 — but disambiguated by inspecting `errno` immediately after: `errno == ENOENT` means the PATH search found no file by that name anywhere, mapped to 127; anything else (most commonly `EACCES`, permission denied, but also things like `ENOEXEC` for a malformed executable) is mapped to 126, meaning *something* was found at that name but couldn't actually be executed. A script cares because the fix is completely different depending on which one occurred: 127 usually means a typo, a missing install, or a broken `$PATH`, and the fix is "install the tool" or "check PATH"; 126 usually means the file exists but lacks the executable bit (a very common mistake — forgetting `chmod +x` on a script) or, less commonly, a corrupted/wrong-architecture binary, and the fix is entirely different ("check permissions," not "install something"). CI pipelines and deployment scripts frequently branch on exactly this distinction to produce actionable error messages rather than a generic "something failed."

**TL;DR:** Both come from `execvp()` failing; `errno==ENOENT` → 127 (nothing found by that name → fix PATH/install it), anything else (typically `EACCES`) → 126 (found it, can't execute it → fix permissions) — genuinely different root causes and fixes, which is why scripts branch on the distinction.

**Key mappings:** the `errno == ENOENT ? _exit(127) : _exit(126)` logic in `executor.cpp::executeSingle` and `pipeline.cpp::executePipelineChain`.

---

### Q48. Where would you plug in heredoc (`<<EOF`) or wildcard globbing (`*.txt`) support, given the current architecture?

**Polished answer:** Two very different layers, which is itself a good thing to point out. **Globbing** (`*.txt` expanding to matching filenames) is purely a *parsing/pre-exec* concern — no new syscall machinery needed beyond what the shell already has: after tokenizing (and ideally after any variable substitution, per Q42, since expansion order matters — bash expands variables before globbing), walk each word token, check whether it contains glob metacharacters (`*`, `?`, `[...]`), and if so replace that single token with zero-or-more tokens by calling the POSIX `glob()` library function (or hand-rolling directory matching against `readdir()`), inserting the results in place into `Command::args` before the command is dispatched — the executor/redirection/pipeline code downstream never needs to know globbing happened at all, since by the time they see it, it's just already-expanded `args`. **Heredocs** are architecturally closer to existing redirection than to parsing: `<<EOF` needs the parser to recognize the `<<` token and then, unusually, *keep reading subsequent lines from the REPL* (not run the command yet) until a line matching the exact delimiter is seen, accumulating those lines as the command's stdin content — then, instead of `open()`ing a real file the way `<` does in `redirection.cpp::applyRedirections`, you'd write that accumulated text to one end of an anonymous `pipe()` (or a temp file) and `dup2` that onto the child's stdin, essentially synthesizing an input source with the exact same wiring mechanism the code already uses for real files and pipes — so heredoc support is much more "reuse the dup2 plumbing with a different fd source" than "new subsystem."

**TL;DR:** Globbing is a pure pre-exec token-expansion pass (replace a `*`-containing token with matched filenames via `glob()`, entirely before the executor ever sees it — no new syscall wiring needed). Heredocs need the parser to consume extra input lines until a delimiter, then feed the accumulated text to the child via the exact same `dup2`-onto-stdin mechanism already used for real file/pipe redirection — reusing plumbing, not building new subsystems.

**Key mappings:** would extend `parser.cpp` (globbing pass, heredoc line-consumption) and reuse `redirection.cpp::applyRedirections`'s dup2 pattern (heredoc).

---

### Q49. This codebase uses `std::string`/`std::vector` (which allocate, and can throw) right alongside raw syscalls and `fork()`. Is that a safe combination? What's the tradeoff versus writing it in pure C with manual memory management?

**Polished answer:** It's safe here specifically because of *where* the STL usage is concentrated versus where the raw-syscall/fork-sensitive code runs. The genuinely fork-and-signal-sensitive regions — the actual signal handlers (Q35) — touch zero STL and zero allocation, by design; that's the one place where "STL might allocate/lock internally" would be a real correctness hazard, and the code correctly avoids it there. Everywhere else — building a `Command`'s `args` vector during parsing, formatting a background-job's command string with `ostringstream` before printing it — runs in perfectly ordinary, non-signal, non-time-critical context, where `std::string`/`std::vector`'s conveniences (automatic memory management, bounds-safe growth, no manual `malloc`/`free`/`strcpy` bookkeeping) are a straightforward win over hand-rolled C string/array code, with essentially no downside: even a thrown exception from, say, `std::bad_alloc` on OOM would propagate up through ordinary C++ stack unwinding exactly like in any other C++ program, since none of that code is running inside a signal handler where exceptions would be genuinely unsafe to unwind through. The one place worth double-checking in an interview is *immediately post-fork, pre-exec, in the child* — the code there does build a `std::vector<const char*> argv` from `cmd.args` (`.c_str()` calls, `.push_back()`) before calling `execvp` — this is fine because it's still ordinary non-signal-handler code running in a freshly forked, single-threaded child, not signal context, so normal STL/allocator behavior is exactly as safe as in the parent. The overall tradeoff versus pure C: modern C++ RAII containers meaningfully reduce a whole class of manual-memory bugs (leaks, double-frees, buffer overflows in string handling) that are exactly the kind of bug systems-C code is historically prone to, at the cost of needing to reason carefully about *where* the line between "ordinary C++ code" and "signal-handler code" is — which this codebase does correctly and explicitly.

**TL;DR:** STL usage is fine everywhere it appears here because none of it touches actual signal-handler code (which stays allocation-free and STL-free, correctly) — the fork-then-build-argv-then-exec code in the child runs as ordinary single-threaded C++ before `execvp`, not signal context, so RAII/exceptions behave normally; the real discipline is knowing exactly where the "no allocation, no STL" boundary has to be (signal handlers only) and enforcing it there, which this code does.

**Key mappings:** contrast `signals.cpp` (zero STL) against `executor.cpp`/`pipeline.cpp`'s post-fork argv-building (STL used freely, safely, since it's non-signal context).

---

### Q50. Give me your 60-second pitch for this project to a recruiter/interviewer who's never seen it — what OS concepts can you speak to fluently because of it?

**Polished answer:** "I built a Unix shell from scratch in C++20 using only raw POSIX syscalls — no shell libraries — that supports pipelines, I/O redirection, background jobs, and proper signal handling. The reason I built it wasn't to reinvent bash, it was to force myself to actually *use*, not just define, the core OS concepts every systems/backend interview probes: process creation and lifecycle (`fork`/`exec`/`wait`, avoiding zombies and orphans correctly for both foreground and background execution), inter-process communication via pipes and file descriptor manipulation (`pipe`/`dup2`, and specifically the fd-closing discipline that prevents pipeline hangs — the bug class that trips up almost everyone who first implements this), and asynchronous signal handling done correctly (async-signal-safety, why handlers can only set a flag and defer real work to mainline code, and the race conditions that appear if you get that wrong). I also made and documented explicit design tradeoffs — like matching bash's last-command-wins pipeline exit status, or deliberately not implementing process-group-based job control yet and explaining exactly why and what it would take — because I think being able to articulate *why* you made a choice, and what its known limitations are, is more valuable in an interview than a feature list." This framing works well because it front-loads the *transferable* skill (OS fundamentals you can discuss in the abstract, for any systems/backend/infra role) rather than the specific artifact, while still giving you a concrete, code-backed example for literally any "tell me about a challenging project" or "explain X OS concept" follow-up.

**TL;DR:** Frame it as: "I implemented, not just studied, process lifecycle/fork-exec-wait, pipes/fd management, and signal handling correctly — with documented, deliberate tradeoffs — which is why I can go deep on any OS-fundamentals question, not just describe the project." Lead with the transferable OS knowledge, use the project as concrete proof.

**Key mappings:** the whole repo — this is your closing answer, tie it back to Q1–Q49 as needed depending on what the interviewer digs into.

---

## Quick-reference cheat sheet (last-minute skim before walking in)

| Concept | One-liner | File |
|---|---|---|
| Zombie avoidance | Exactly one `waitpid()` per PID: blocking for fg, `WNOHANG` polling for bg | `executor.cpp`, `jobs.cpp` |
| Pipe hang prevention | Every process closes every pipe fd it doesn't own, before waiting | `pipeline.cpp` |
| SIGCHLD safety | Handler only sets `volatile sig_atomic_t`; real work deferred to REPL | `signals.cpp` |
| SIGINT survival | Shell installs custom handler; child resets to `SIG_DFL` post-fork | `signals.cpp`, `executor.cpp` |
| Builtins in-process | `cd`/`pwd`/`exit`/`jobs` mutate parent state → must not fork | `builtins.cpp` |
| Exit code convention | 126 = found, can't exec; 127 = not found; 128+N = killed by signal N | `executor.cpp` |
| `_exit` vs `exit` | `_exit` skips flushing copied stdio buffers → no duplicated output | everywhere post-fork |
| Pipeline status | Last command's exit status wins (bash non-pipefail default) | `pipeline.cpp` |
| Known limitation | No process groups → bg jobs not isolated from Ctrl+C | README |
| Known limitation | No backgrounded pipelines — `Job` tracks 1 PID only | `jobs.hpp`, `executor.cpp` |

Good luck — you don't need to memorize these answers verbatim. Understand *why* each design decision was made, since a good interviewer will ask "what if you did it differently" as a follow-up, and the README's own "Design Decisions" table is essentially a pre-built answer key for exactly that.

---

## Pronunciation guide

Say these correctly out loud and you'll sound like you've actually used them, not just read about them. Grouped by category; syllables to stress are in CAPS.

### Syscalls / library functions

| Term | Pronunciation |
|---|---|
| `fork()` | "fork" |
| `exec()` (the family in general) | "EX-eck" |
| `execvp()` | "EX-eck-vee-pee" (spell out V-P, don't blend it) |
| `execv()` | "EX-eck-vee" |
| `execl()` | "EX-eck-el" |
| `execve()` | "EX-eck-vee-ee" |
| `wait()` | "wait" |
| `waitpid()` | "wait-pid" (PID as one syllable, rhymes with "kid") |
| `pipe()` | "pipe" |
| `dup2()` | "dupe-two" (like "duplicate," not "dup-two" with a short u) |
| `sigaction()` | "sig-ACK-shun" |
| `chdir()` | usually said "change-dir" or letter-by-letter "C-H-D-I-R"; either is fine, "change-dir" is more common in speech |
| `getcwd()` | "get-C-W-D" (spell out C-W-D) or "get current working directory" |
| `kill()` | "kill" |
| `open()` / `close()` | "open" / "close" (plain English) |
| `perror()` | "P-error" or "print error" |
| `_exit()` | "underscore exit" or just "exit" with emphasis that it's the raw syscall version (context usually makes clear vs. library `exit()`) |
| `glob()` | "glob" (rhymes with "blob") |
| `readdir()` | "read-DIR" (read, then dir) |
| `setpgid()` | "set-P-G-I-D" (spell out P-G-I-D) |
| `tcsetpgrp()` | "T-C-set-P-group" or spelled "T-C-set-P-G-R-P" — commonly just said as "tc-set-pgrp," spelling the "pgrp" part |
| `tcgetattr()` / `tcsetattr()` | "T-C-get-attributes" / "T-C-set-attributes" |

### Signals

| Term | Pronunciation |
|---|---|
| `SIGCHLD` | "sig-CHILD" (the CHLD is just pronounced "child") |
| `SIGINT` | "sig-INT" (INT rhymes with "hint," not spelled out) |
| `SIGTERM` | "sig-TERM" (TERM as in "terminate") |
| `SIGKILL` | "sig-KILL" |
| `SIGSTOP` | "sig-STOP" |
| `SIGTSTP` | "sig-T-stop" (say "T" as the letter, then "stop") |
| `SIGCONT` | "sig-CONT" (rhymes with "front," short for "continue") |
| `SIGPIPE` | "sig-PIPE" |
| `SA_RESTART` | "S-A restart" (spell S-A, then say "restart" normally) |
| `SA_NOCLDSTOP` | "S-A no-child-stop" (say it as words: "no child stop") |
| `SIG_DFL` | "sig-D-F-L" (spell out D-F-L) or "sig-default" |

### Macros / flags on `status` from `wait`/`waitpid`

| Term | Pronunciation |
|---|---|
| `WIFEXITED` | "W-if-exited" (say "W," then "if exited" as two words) |
| `WEXITSTATUS` | "W-exit-status" |
| `WIFSIGNALED` | "W-if-signaled" |
| `WTERMSIG` | "W-term-sig" (term, then sig) |
| `WNOHANG` | "W-no-hang" (say it as three words: W, no, hang) |
| `WUNTRACED` | "W-untraced" (un-TRAYST) |
| `WCONTINUED` | "W-continued" |

### `open()` flags

| Term | Pronunciation |
|---|---|
| `O_RDONLY` | "O-R-D-only" or "O-read-only" (both used; "O-read-only" is more natural in speech) |
| `O_WRONLY` | "O-write-only" (don't try to sound out "WRONLY" — say "write-only") |
| `O_CREAT` | "O-cree-AT" or "O-create" (CREAT is missing the final E, but most people just say "create") |
| `O_TRUNC` | "O-trunk" (rhymes with "trunk," short for "truncate") |
| `O_APPEND` | "O-append" |

### errno values

| Term | Pronunciation |
|---|---|
| `errno` | "AIR-no" or "err-no" (either is common; "AIR-no" is more standard) |
| `EINTR` | "E-int-er" or "E-interrupt" — commonly said "E-I-N-T-R" spelled out, or "E-intr" |
| `ENOENT` | "E-no-ent" (say "E," then "no," then "ent" — short for "no entry") |
| `EACCES` | "E-access" (the missing S is silent in speech — just say "access") |

### C++ / language terms

| Term | Pronunciation |
|---|---|
| `sig_atomic_t` | "sig-atomic-tee" (say "atomic" normally, then the trailing "_t" as "tee") |
| `volatile` | "VOL-uh-tile" (like the English word) |
| `pid_t` | "P-I-D-tee" (spell PID, then "tee" for `_t`) |
| `argv` | "AR-jay-vee" (like "arg-vee," short for "argument vector") |
| `argc` | "AR-jay-see" |
| `RAII` | "R-A-I-I" spelled out letter by letter (no natural word form; some say "raw-ee" informally but spelling it out is safest in an interview) |
| `STL` | "S-T-L" spelled out ("Standard Template Library") |
| `stdin` / `stdout` / `stderr` | "STAN-dard-in" / "STAN-dard-out" / "STAN-dard-err" (or informally "std-in," "std-out," "std-err" said quickly) |
| `STDIN_FILENO` | "STD-in file number" (say "std-in," then "file number" — don't try to sound out FILENO as one word) |
| `STDOUT_FILENO` | "STD-out file number" |

### General systems/shell terms

| Term | Pronunciation |
|---|---|
| POSIX | "PAH-six" (rhymes with "Pontiac six," two syllables) |
| REPL | "REP-ul" (rhymes with "steeple" minus the ee — most people say it like the word "ripple" with an e) |
| PID | "pid" (one syllable, rhymes with "kid") |
| PGID | "P-G-I-D" spelled out, or "P-group-ID" |
| fd / "file descriptor" | say "file descriptor" in full the first time, "F-D" (spelled) afterward |
| EOF | "E-O-F" spelled out ("end of file") |
| PATH (the env var) | just say "path," but it's common to say "the PATH variable" to disambiguate from a filesystem path |
| CMake | "SEE-make" (like "see" + "make") |
| ASan | "AY-san" (short for AddressSanitizer — say it like "a-san") |
| UBSan | "U-B-san" (spell U-B, then "san" — short for UndefinedBehaviorSanitizer) |
| `malloc` | "MAL-lock" |
| glob / globbing | "glob" / "GLOB-ing" (rhymes with "blob"/"robbing") |
| heredoc | "HERE-dock" (as in "here" + "doc(ument)") |
| zombie process | plain English, "ZOM-bee" |
| orphan process | plain English, "OR-fun" |

**Tip for the interview itself:** if you're ever unsure how to say an acronym-heavy macro like `WIFSIGNALED`, it's completely normal and safe to just say "the wait status macro that checks if it was killed by a signal" instead of forcing the pronunciation — interviewers care that you know *what it does*, not that you can say it smoothly.
