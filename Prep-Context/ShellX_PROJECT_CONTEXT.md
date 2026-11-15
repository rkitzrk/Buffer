# PROJECT_CONTEXT: ShellX

Sources: all files in `ShellX.zip` (src, include, tests, CMakeLists.txt, README.md, docs/, git log). "Observed" = compiled and run. Unconfirmable items are marked [unverified].

## 1. Summary
ShellX is a Unix shell written in C++20 for Linux using raw POSIX calls (`fork`, `execvp`, `waitpid`, `pipe`, `dup2`, `sigaction`). README.md calls it a "hands-on systems programming exercise, not a Bash clone"; a learning project, not production. Author per git: R Kumaresh. Size: 8 `.cpp` files = 761 lines, 8 headers = 159 lines, one 214-line bash test script. Build: CMake >= 3.20, `-Wall -Wextra -Wpedantic`, README says g++ 13+ (g++ 13.3.0 used for my build). No external libraries. No performance or scale target stated anywhere.

Feature set: external commands, builtins `cd pwd exit jobs`, `>` `>>` `<`, multi-stage pipes (max `MAX_PIPELINE_LENGTH = 16`), single-command background `&`, double-quoted strings, SIGCHLD/SIGINT handling. Limits in `shellx.hpp`: `MAX_INPUT_LENGTH = 4096`, `MAX_BACKGROUND_JOBS = 64`.

## 2. Architecture
Modules (header in `include/`, impl in `src/`):
- `main.cpp`: REPL. Installs signals, prints `shellx> `, `std::getline`, length/blank checks, `parse`, `executePipeline`, reaps jobs.
- `parser.cpp`: `tokenize()`, `rejectUnsupported()`, `parse()` → `Pipeline`. Empty `Pipeline` = error/empty input.
- `executor.cpp`: `executePipeline()` dispatch; `executeSingle()` does fork/exec/wait or registers a background job.
- `pipeline.cpp`: `executePipelineChain()`: N-1 pipes, N forks, fd wiring, wait for all.
- `redirection.cpp`: `applyRedirections()` (open/dup2/close, run in the child).
- `builtins.cpp`: `isBuiltin`, `executeBuiltin` (run in parent; `exit` returns sentinel -1).
- `jobs.cpp`: global `std::vector<Job> g_jobs`, `addJob`, `reapFinishedJobs`, `printJobs`, `killAllJobs`.
- `signals.cpp`: flag-only handlers, `installSignalHandlers()`.

Data flow: line → `tokenize` → `rejectUnsupported` → strip trailing `&` → split on `|` → per-command redirection extraction → `Pipeline{commands, background}` → one command: builtin in parent, else `executeSingle`; 2+ commands: `executePipelineChain` (rejected if background) → exit status returned to REPL.
Key types (`shellx.hpp`): `Command{args, input_file, output_file, append_mode}`, `Pipeline{commands, background}`; `Pipeline::background` is the sole background flag (comment in header).

## 3. Key design decisions
1. SIGCHLD handler only sets `volatile sig_atomic_t g_sigchld_pending` (`signals.cpp`). Alternative rejected (implementation_plan.md): handler calling `waitpid(-1, WNOHANG)`, which could steal the foreground child and yield `ECHILD`. Trade-off: background completion is noticed only when the REPL loop runs (see section 8).
2. One `waitpid` call site per PID: foreground in executor/pipeline, background in `reapFinishedJobs()` (README table).
3. Builtins run in the parent without fork, so `cd` persists (`executor.cpp` lines 103-106). Trade-off: no redirection/background/pipe-aware behaviour for builtins (section 8).
4. `_exit()` not `exit()` in children after failed redirect/exec, to avoid double-flushing inherited stdio buffers (`executor.cpp`, `pipeline.cpp`).
5. Every process closes every pipe fd it doesn't use (README: "single rule preventing all pipeline hangs and fd leaks"); `pipeline.cpp` closes all in child after dup2 and all in parent after forking.
6. Pipeline status = last command's status, no `pipefail` (README; `pipeline.cpp`).
7. SIGINT: shell handler sets a flag and survives; children reset to `SIG_DFL` (`executor.cpp`, `pipeline.cpp`).
8. Unsupported syntax is rejected explicitly: `;`, `&&`, `||`, `$(`, backtick, backgrounded pipelines. Process groups/`setpgid` deferred (README Known Limitations).
9. `exit`: `SIGTERM` to running jobs, no reap, rely on init (`killAllJobs`).

## 4. Hard problems and bugs
Git has one implementation commit (`8b5a3d0`, 2026-09-20), so debugging histories are not in git. The only narrative source is `docs/prep/ShellX-Behavioral.md` (author's own STAR notes). Treat symptoms below as [unverified] unless stated; the fixes are visible in code.
1. Pipeline hang (Behavioral.md Story 1: `cat file | grep foo | wc -l` froze). Root cause: parent kept pipe write ends open, so readers never saw EOF. Fix: `pipeline.cpp` lines 104-107 (parent closes all fds after forks) plus lines 74-77 (child closes all after dup2). Tests: `two-stage pipe`, `multi-stage pipe (cat | sort | uniq)`. The script has no timeout, so a regression would hang rather than fail.
2. Reap race (foreground `waitpid` vs SIGCHLD). Fix: flag-only handler (`signals.cpp` 11-13); `EINTR` retry loops (`executor.cpp` 75-77, `pipeline.cpp` 41-43, 116-118); `SA_RESTART | SA_NOCLDSTOP`. Tested only indirectly: the one background test checks the `[1] <pid>` output; no test asserts reaping, no zombie check.
3. Double flush after failed exec: `_exit` (see decision 4). Only the error message is tested.
4. Tokenizer `>>` vs `>`: `parser.cpp` 37-42 checks `>>` first (Behavioral.md Story 8). Tested by `echo >> file (append)`.
5. Signal handlers installed before first fork (`main.cpp` comment: avoids missed SIGCHLD and zombies). No test.

## 5. Concurrency, errors, failure behaviour
- Shared state: `g_jobs` is touched only from main-thread code; handlers touch only two `sig_atomic_t` flags. Single-threaded; no locks. The `g_sigchld_pending` check-then-clear in `main.cpp` is not masked with `sigprocmask`, so a signal arriving between the check and the clear can be lost (not reproduced).
- Exec failure: `perror` then `_exit(127)` if `ENOENT` else 126. See Discrepancies: observed 126 for a missing command.
- Redirect failure: `perror(filename)`, `_exit(1)`. `fork`/`pipe`/`dup2`/`waitpid` failures: `perror`, return 1; on fork failure mid-pipeline the parent closes all fds and waits for already-forked children.
- Signal death: status `128 + WTERMSIG`.
- Bad input: parse errors print `shellx: ...` to stderr and re-prompt; input over 4096 chars rejected; EOF (Ctrl+D) calls `killAllJobs` and exits.
- Background: foreground-waiting parent is interrupted by SIGCHLD safely (EINTR loop). On `exit`/EOF, running jobs get `SIGTERM` and are not reaped (observed: left `<defunct>` in my container, which has no init reaper).
- Exit code of shell: `main` returns last status on EOF; the `exit` builtin ignores arguments and the shell returns 0 (observed `false` then `exit 5` → rc 0).

## 6. Testing and quality
- `tests/test_shellx.sh`: bash, black-box, one process per case via `echo | shellx`, substring grep on combined output. 22 tests, 0 failures when I ran it (compiled with direct `g++ -std=c++20 -Wall -Wextra -Wpedantic`: no warnings). Categories: basic 3, builtins 4, redirection 4, pipes 3, background 1, errors 6, quotes 1.
- Sanitizers: README documents an ASan/UBSan debug build (`-DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -g"`); `build-debug/CMakeCache.txt` has those flags. No sanitizer run logs in the repo [unverified that it was clean].
- No CI config, no unit tests, no linter beyond compiler warnings, no fuzzing, no valgrind, no benchmarks.
- NOT tested: SIGINT behaviour, SIGCHLD/zombie reaping, `jobs` output, `exit` killing jobs, exit codes 126/127, `;` and `$(` rejection, `MAX_*` limits, pipeline > 16 commands, `cd` with `HOME` unset, `>`/`<` combined with pipes, unterminated quotes, fd leaks.

## 7. Measured results
None in the repo (no benchmark, load or timing files). Only measured fact: 22/22 tests pass (my run). `Behavioral.md` claims "under 3 seconds" for the suite [unverified].

## 8. Known limitations / next steps
From README: no process groups, so Ctrl+C can reach background jobs; no backgrounded pipelines (`Job` holds one PID). From code:
- Job numbers are vector positions, recomputed after erase: observed two jobs both reported `[1] Done`. `Job::running` is never set false, so the `Done` branch in `printJobs` and the `!running` erase are dead.
- `MAX_BACKGROUND_JOBS` only warns; the job is still added (`jobs.cpp` 12-15).
- Background completion is reported only after the next input line (REPL blocks in `getline`).
- Builtins ignore redirections/`&`; `pwd | cat` forks and execs an external `pwd`.
- Mid-line `&` becomes a literal arg (observed `echo a & echo b` prints `a & echo b`).
- Quotes: adjacent text/quote splits into separate tokens, unterminated quote accepted silently, no single quotes/escapes. No `$VAR`, globbing, `~`, `2>`, history, line editing.
- Repeated redirects: last wins, earlier file not created (observed `> f1 > f2` makes only `f2`).
- `std::isspace(line[i])` on plain `char` (`parser.cpp` 17, 53) is UB for negative values.
Next: `setpgid`/`tcsetpgrp` job control, `sigprocmask` around flag handling, `waitpid(-1)` loop for jobs, stable job IDs, fix errno clobbering, add tests for signals/zombies.

## 9. Likely interview questions
1. Why must the parent close pipe fds, and what hangs if not? → `pipeline.cpp` 74-77, 104-107.
2. Why is the SIGCHLD handler flag-only? → `signals.cpp` 11-13; implementation_plan.md "Locked decision".
3. Why are `cd`/`exit` builtins and not forked? → `executor.cpp` 103-106; `builtins.cpp`.
4. Why `_exit` and not `exit` in the child? → `executor.cpp` 33, 50-52.
5. What are `EINTR` and `SA_RESTART`; why both? → `signals.cpp` 28-30; `executor.cpp` 75-77.
6. What does `dup2` do in redirection order vs pipe wiring? → `pipeline.cpp` 58-82; `redirection.cpp`.
7. What is a zombie; where could one occur here? → `jobs.cpp` `reapFinishedJobs`, `killAllJobs`.
8. Why does Ctrl+C possibly kill background jobs? → README Known Limitations; no `setpgid`.
9. How is exit status computed for a pipeline and for signals? → `pipeline.cpp` 125-132 (128 + signal).
10. How would you add `cmd1 | cmd2 &`? → `executor.cpp` 113-118; `jobs.hpp` single `pid_t`.

## 10. Discrepancies and unverified claims
- README/executor: `ENOENT → 127`, but `perror()` runs before the `errno` check (`executor.cpp` 48-50, `pipeline.cpp` 96-97). Observed `nosuchcmd` returns 126 when stderr is a pipe, file or /dev/null; TTY case [unverified].
- `Behavioral.md` says the `isspace` signed-char UB was fixed; `parser.cpp` still passes uncast `char` (only `main.cpp` casts).
- `Behavioral.md` claims sanitizer-clean build and "under 3 seconds" tests [unverified]; its bug stories (pipeline hang, `>>` tokenizer, double flush) have no commit history.
- `implementation_plan.md` says CMake has "Debug/Release + sanitizer support"; `CMakeLists.txt` has neither option, sanitizers come only from `CMAKE_CXX_FLAGS`.
- `builtins.hpp` comment lists builtins as (cd, pwd, exit), omitting `jobs`.
- CMake was unavailable in my environment; the build cache references CMake 3.28.3 and 4.2.3 [versions used by author unverified]. I built with g++ directly.
