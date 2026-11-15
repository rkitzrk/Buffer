# Amazon Behavioral Guide (Final): 25 Questions
**Kumaresh R** | VIT Amaravati, B.Tech CSE (June 2027), CGPA 9.09
Built from your resume and the Auth Service, LexiCore and ShellX project contexts.

**Designed for "no more code changes."** Every script says only what is true of your code and results today. Anything not yet done is stated as a **next step**, never as a completed fix. Scripts are ~140-165 words (60-70 seconds at a normal pace). No special notation, so you can read them aloud.

---

## 0. Read this first (10 minutes)

### Tags used on each question
- ✅ **Evidenced**: the facts are visible in your code, tests or results.
- ⚠ **Confirm**: the *fix* is visible in code, but the *story of how you found it* comes from your own notes. Tell it only if it really happened that way. If it didn't, use the fallback line given.
- 🧩 **Template**: no story in your files. Fill in a true one, or use the fallback.

### Claims to avoid or soften (nothing here needs a code change)
| Don't say | Why | Say instead |
|---|---|---|
| "10,423 req/s, 97.67% success" as a result | No result file in the repo. Your JMeter tests call the gateway, not Auth. | Only if you have the run output: "JMeter on the full pipeline showed about 10.4K req/s; I'd want to re-measure Auth alone." Otherwise: "I load-tested through the gateway; I don't have an isolated Auth number." |
| "Median of 5 trials" | `benchmark.cpp` times each strategy once per size. | "1,000 seeded queries per configuration." (Edit the resume line too.) |
| "POSIX-compliant", "standard ENOENT/EACCES exit codes" | Observed exit 126 for a missing command, not 127. | "Built on POSIX system calls." Mention the 126 bug yourself (Q11). |
| "100% confidence", "zero bugs" | Overclaim. Release builds disable `assert`. | "Tested against an oracle; here's what the tests don't cover." |
| "Unshuffled BK-tree depth 100+" | Measured: 27 unshuffled, 19 shuffled. | "27 down to 19." |
| "Validated by the oracle when it caught the sentinel bug" | Appears only in prep notes, not git history. | Present it as reasoning (Q16). |
| Any "visits X% of the tree" figure | Never measured. | "I haven't counted nodes visited." (Q8) |
| "Exactly-once" or "at-least-once" for Auth | Neither is provided. | "Loss and duplicates are both possible today." (Q6) |

### Optional 15-minute checks (running commands, not editing code)
1. Run LexiCore `prefix a 5` and `exact Apple`. Q19 uses both results.
2. Run `./test_shellx.sh` once. Q3 and Q15 quote "22 tests".
3. Find your JMeter output if you have one. It decides whether Q12/Q25 may quote RPS.
4. Say your role in Auth out loud: "I built ___; the team built ___." Your Auth file's role line is blank.

### Delivery rules
- Speak as **I** for what you wrote, **we** for team work. Expect "what was *your* part?" on every story.
- Order: situation (10s), action (30s), result (15s), lesson (5s).
- End with one honest limit plus the next step. Amazon rewards that.
- One main story per principle. Don't reuse a story unless asked.

### Story map
| LP | Question(s) | Main story |
|---|---|---|
| Intro / motivation | 1, 2 | profile, Why Amazon |
| Dive Deep | 3, 8 | ShellX pipe hang; fuzzy-search result |
| Ownership | 4, 25 | gateway integration; end-to-end Auth |
| Customer Obsession | 5 | amount bug + clear failure reasons |
| Earn Trust / failure | 6, 19, 21 | delivery guarantees; self-audit |
| Highest Standards | 7 | oracle and property tests |
| Invent and Simplify | 9 | single LEFT JOIN |
| Are Right, A Lot | 10 | SIGCHLD handler |
| Learn and Be Curious | 11 | POSIX and stdio buffers |
| Think Big | 12 | scaling Auth |
| Bias for Action | 13 | consumer/DB decoupling |
| Frugality | 14 | rolling rows, STL only, connection caps |
| Handle ambiguity | 15 | ShellX scope |
| Have Backbone | 16, 20 | bounded distance; disagreement |
| Deliver Results | 17, 23 | LexiCore delivery; priorities |
| Broad responsibility | 18 | risk audit |
| Hire and Develop | 22 | mentoring |
| Self-improvement | 24 | competitive programming |

---

# Part 1: Introduction

## Q1. Tell me about yourself ✅
**Script (~60s)**
"I'm a final-year Computer Science student at VIT Amaravati with a 9.09 CGPA, and I focus on backend and systems programming in C++. From February to July 2026 I was in the HPE Career Preview Program, where I built the Authorization Service for a payment gateway. It consumes Kafka events, validates each payment against PostgreSQL, and publishes authorized or failed events. Alongside that I built two systems projects. ShellX is a Unix shell with pipes, redirection, background jobs and signal handling. LexiCore is a search engine with a hash set, a trie and a BK-tree, tested against a brute-force oracle and benchmarked. I also compete on Codeforces, where I reached Specialist and have solved over a thousand problems. I want a role where I own backend systems and think hard about how they fail."

**Learn**
- Be ready to go one level deeper on any item you mention.
- Know your graduation date (June 2027) and availability.

---

## Q2. Why Amazon? 🧩
**Script (~45s)**
"Three reasons. First, [team or product you target] runs systems where reliability decisions like delivery guarantees, backpressure and idempotency directly affect customers. Those were the problems I enjoyed most at HPE. Second, ownership: I learn most when I own a problem including its failure modes, and Amazon expects that from engineers. Third, the bar: I want to work with people who ask 'how do you know?' about every claim, because that habit has shaped how I test and benchmark my own work."

**Fallback:** keep it to these three reasons; fill the bracket with one real team or product.

**Learn**
- Research your target team and one Amazon service you can discuss.
- Pick 3 Leadership Principles you genuinely connect with, each with a story from this guide.

---

# Part 2: Technical depth

## Q3. Hardest bug / deep dive ⚠
*Merges: "hardest bug", "troubleshooting with no obvious answer", "a small oversight that caused a freeze or leak".*
**Script (~65s)**
"While building ShellX, a three-stage pipeline like `cat file | grep foo | wc -l` hung forever, while single commands and two-stage pipes worked. I reasoned about why a reader blocks: it only sees end-of-file when every write end of the pipe is closed. So I traced every process and every file descriptor it held. The parent kept write ends open after forking, and children held copies of pipes they didn't use, so the last process waited forever. I fixed it with one rule: every process closes every pipe descriptor it doesn't use. Children close all of them right after `dup2`, and the parent closes all of them after the fork loop. The shell supports up to 16 stages, and my tests cover two- and three-stage pipelines. The lesson: model resource ownership instead of adding prints. The gap is that my test script has no timeout, so a regression would hang instead of fail."

**Confirm:** the hang symptom comes from your behavioural notes. If it didn't happen, say "this is the failure mode my design prevents" and keep the rest.

**Learn**
- Why EOF needs all write ends closed. What `dup2` does to the descriptor table.
- Why the parent closes after forking, not before.
- If `fork()` fails mid-pipeline: parent closes all fds and waits for the already-forked children.
- How to test: add a timeout; check `/proc/<pid>/fd` for leaks.
- Related edge case you can mention: a tokenizer that checks `>>` before `>` (`echo hi>>out.txt`).

---

## Q4. Ownership across service boundaries ✅
*Merges: "cross-team issue", "took ownership of an integration problem".*
**Script (~65s)**
"In our payment gateway, a Java API Gateway published payment events and my C++ Authorization Service consumed them. The Java side sent fields in camelCase, like `paymentIntentId`, and sent `amount` as either a string or a number, while my service expected snake_case. Rather than wait for the other side to change its contract, I took ownership of making my ingestion layer tolerant. I wrote a custom JSON deserializer that accepts both naming styles with fallback lookups and parses `amount` either way. For messages that are genuinely malformed, the worker catches the exception and publishes the raw payload to a dead-letter topic, so one bad message can't stall the consumer. On the output side I emit camelCase so the downstream Clearing service can read it directly. I'd add contract tests next, because today this fix has none."

**Learn**
- Why you accepted both formats (speed, decoupling) and the trade-off (looser contract).
- Missing IDs aren't exceptions: they become `""`, which fails as `uuid` in SQL and becomes a `FAILED` event with a database-error reason, not a DLT message.
- Which parts you own (Auth only) vs gateway, Clearing and the persistence worker.

---

## Q5. Customer Obsession ✅
*Merges: "put the customer first", "found a root cause that affected users", "designed for actionable errors".*
**Script (~65s)**
"My Authorization Service sat in a payment path, so its customers were the downstream services, support engineers and ultimately the people paying. Two things mattered to them. First, a rejected payment has to say why. I validate in a fixed order and each failure publishes a specific reason, such as 'Payment method expired' or 'Insufficient funds in account balance', instead of a generic failure. Second, valid payments must not be rejected. I found that amounts like 13644.96 were being compared as 13644.959999, because converting a `double` with `std::to_string` produces binary rounding noise. I changed it to compare against the database's own text value, using fixed two-decimal formatting on the C++ side. The honest follow-up: that fix has no automated test, and money should be stored as integer minor units or a decimal type, not a `double`."

**Learn**
- Why not an epsilon compare: money needs exact decimals. `numeric(15,2)` is only in a code comment (unverified in the schema).
- The seven reason strings in check order: user not found; intent not found or mismatch; amount mismatch; method not found or not the user's; method expired; account not found; insufficient funds.
- Database errors become `FAILED` (permanent), not a retry. A transient DB outage would fail real payments, so retries with backoff are a next step.

---

## Q6. Failure / mistake / what you'd do differently ✅
*Merges: "a time you failed", "biggest mistake", "something you'd redo".*
**Script (~65s)**
"My biggest technical miss on the Authorization Service was building the fast path before defining failure semantics. When I reviewed it critically, I couldn't honestly claim at-least-once delivery. Offsets commit through librdkafka defaults before the worker finishes, so a crash with queued tasks loses those messages. A failed publish to the dead-letter topic is still logged as success, because the publish function never throws and I don't register a delivery-report callback. And a replay re-authorizes, because there's no idempotency key. I didn't hide that: I documented the exact behaviour and the plan. Commit offsets only after a successful publish, handle delivery reports, bound the queue, and make processing idempotent on transaction ID. The lesson is to decide how a system fails before optimizing how fast it runs, especially for payments."

**Learn**
- Auto-commit vs manual commit; at-most-once vs at-least-once vs exactly-once.
- Why a successful `produce()` doesn't mean the broker has the message.
- The gateway dedupes with Redis, so Auth isn't the only dedupe layer (a script note wrongly says Auth dedupes).
- Don't say "I fixed it." Say "this is my documented next step."

---

## Q7. Insist on the Highest Standards ✅
*Merges: "ensured quality", "raised the bar", "how did you know it was right".*
**Script (~65s)**
"In LexiCore, bugs in a BK-tree don't crash, they silently drop valid results. A handful of hand-picked words wasn't enough to trust it. So I built layers of verification. A brute-force linear-scan oracle compares the BK-tree against ground truth across 500 random words, 200 queries and four distance thresholds, and aborts on any mismatch. Property tests check that my edit distance behaves as a true metric, with symmetry and the triangle inequality over 1,000 random cases each, because the tree's pruning depends on that. I compiled and ran the tests under AddressSanitizer and UndefinedBehaviorSanitizer, and every node is owned by a unique pointer, with no manual delete. I'm also precise about limits: plain asserts disappear in Release builds, and there are no tests for the benchmark harness or the CLI."

**Learn**
- 29 test functions across four executables, plain `<cassert>` with no framework. Only the first two correctness tests call `std::abort()`, which survives Release.
- Sanitizers come from manual compile flags, not CMake.
- Recursive destruction of `unique_ptr` trees is safe here because depth is bounded (BK depth 19, Trie by word length).
- Why metric axioms matter: if the triangle inequality fails, pruning is invalid.

---

## Q8. Dive Deep: a result that surprised you ✅
*Merges: "unexpected metrics", "analysed data to find the cause".*
**Script (~65s)**
"When I benchmarked LexiCore on 88,344 words, exact lookup came out 271 times faster than a linear scan and prefix search 49 times faster. But BK-tree fuzzy search at distance 2 was only 2.4 times faster, 5,729 microseconds versus 13,818. I wanted to understand that rather than gloss over it. The speedup was about 2 times at 1,000 words and about 2.5 times at 10,000, so it didn't improve with scale. My explanation is that at distance 2 the pruning window is wide, so many branches survive, and each visited node still costs a full edit-distance computation. I should be clear that this is a hypothesis: I haven't counted nodes visited per query. The next steps are to add that counter and try a SymSpell-style deletion index, which trades memory for speed."

**Learn**
- Numbers: exact 0.45 vs 121.80 µs; prefix 4.39 vs 214.71 µs; fuzzy 5,729.39 vs 13,818.35 µs (p99 8,797).
- How BK pruning works and why a wider window means more nodes.
- SymSpell in one sentence: precompute deletions at index time so a lookup becomes hash probes plus verification.
- Harness honesty: each strategy is timed once per size, partial warm-up; "5 trials" is documented but not implemented.
- Benchmarks use seeded random subsets, because the first N words of a sorted file would all start with 'a'.

---

## Q9. Invent and Simplify ✅
*Merges: "simplified a complex process", "found a simpler way".*
**Script (~60s)**
"Authorizing a payment needs five checks: the user exists, the payment intent exists and belongs to the user, the amount matches, the payment method belongs to the user, and the wallet balance covers it. The obvious design is several sequential queries per message. I replaced that with one parameterised query that LEFT JOINs four tables, `users`, `payment_intents`, `payment_methods` and `accounts`, anchored on the user. A missing row comes back as null, so my code can tell exactly which check failed in one pass. Card expiry needs one more query using the database's own clock, so each authorization takes one or two queries. Parameters are bound server-side, so no string building and no SQL injection. The trade-off is that the query is tied to the schema, so schema changes need coordination."

**Learn**
- Why LEFT JOIN (to identify which check failed) rather than INNER JOIN.
- Indexes you'd want and how to confirm with `EXPLAIN ANALYZE` (no plan is in the repo).
- What the query does *not* check: intent status value, currency, duplicates, `transaction_id`.
- Easy next step: fold the date into the first query to get strictly one query.

---

## Q10. Are Right, A Lot: a design decision ✅
*Merges: "technical decision you're proud of", "how you choose between options".*
**Script (~65s)**
"ShellX has background jobs, so I had to handle SIGCHLD. The common approach is a handler that calls `waitpid` with no PID filter. I rejected it because it could reap the foreground child the shell is blocked on, causing an ECHILD error and a lost exit status. Instead the handler only sets a `sig_atomic_t` flag, which is async-signal-safe, and the main loop reaps background jobs. That gives each PID exactly one `waitpid` call site: foreground in the executor, background in the job reaper. I added retry loops for EINTR and used SA_RESTART. The trade-off is that background completion is only noticed when the loop next runs, after the next input line. I accepted that for correctness, and my next step would be masking signals around the flag check."

**Learn**
- Which functions are async-signal-safe; why `printf`/`malloc` in a handler is unsafe.
- EINTR vs SA_RESTART; why you use both.
- The remaining race: flag check-then-clear isn't masked with `sigprocmask`, so a signal in between can be lost.
- Zombies: on `exit` jobs get SIGTERM but aren't reaped (left defunct in a container with no init).
- Not tested: SIGCHLD reaping, SIGINT, `jobs` output.

---

## Q11. Learning something new quickly ⚠
*Merges: "unfamiliar technology", "steep learning curve".*
**Script (~65s)**
"My background was competitive programming, mostly STL and algorithms. ShellX forced me to learn the Linux process model: `fork`, `execvp`, `dup2`, `sigaction`. A good example is what happens when `exec` fails in a forked child. The child inherits a copy of the parent's buffered stdio, and calling `exit()` would flush that buffer a second time, so output can appear twice when redirected. I learned to use `_exit()` in every child failure path. I also found a flaw in my own code: I call `perror` before checking `errno`, and `perror` can overwrite it, so a missing command can exit with 126 instead of 127. The fix is to save `errno` first. I learn by building a small piece, breaking it, and reading the man pages for why."

**Confirm:** the 126 observation is from a run in a container (stderr not a terminal). Run it once yourself before telling this. Also fix your resume wording.

**Learn**
- `fork` copies user-space buffers; `_exit` skips cleanup handlers.
- Exit codes: 127 not found, 126 not executable, 128+signal for signal death.
- Same flaw shape in the pipeline code path.
- Related: `std::isspace` on a plain `char` (parser) is undefined for negative values; cast to `unsigned char`.

---

## Q12. Think Big: scale it 10x ✅
*Merges: "what would break first", "how would you redesign it".*
**Script (~65s)**
"First I'd measure, because I don't have a trusted throughput number for Auth alone. Then I'd fix three things I already know cap it. One: the input topic has 3 partitions in the gateway config, so only 3 consumers in the group receive messages even though the autoscaler allows 15 to 60 pods. I'd raise partitions and keep the transaction ID as the key. Two: the work queue is unbounded, so a slow database grows memory until the pod is killed and queued messages are lost. I'd bound it and pause the consumer for backpressure. Three: the balance check reads a balance that's debited later by another service, so concurrent payments from one wallet could all pass. I'd reserve funds atomically in the database. With manual offset commits and idempotency on transaction ID, that closes the main gaps."

**Learn**
- Max parallelism in a consumer group equals the partition count; repartitioning affects key ordering.
- `pause()`/`resume()` on the consumer for backpressure.
- Atomic reservation, for example a conditional `UPDATE` with a balance check, or a ledger with holds.
- How to test Auth in isolation: produce straight to Kafka, watch consumer lag.
- Config oddities: deployment sets 2 replicas but the HPA minimum is 15; 100 threads vs 20 connections.

---

## Q13. Bias for Action ✅
*Merges: "unblocked a bottleneck", "built something quickly", "speed vs safety".*
**Script (~65s)**
"The Authorization Service needed to work end to end before I knew every failure mode, so I shipped a thin slice first: consume, check, publish. Real problems showed up quickly. The first was that doing database work inside the consumer loop would stall consumption, so I added a thread pool, and the consumer callback now copies the message, enqueues it and returns immediately. The second was database connections: I built a connection pool with a guard object that returns the connection in its destructor, even if an exception is thrown, and pointed it at PgBouncer. Each fix was small because the slice was small. I consciously deferred delivery guarantees to keep moving, and I wrote that down as a known limitation with a plan, so speed didn't turn into hidden debt."

**Learn**
- Why 100 threads but 20 connections: up to 80 workers wait on the pool's condition variable. The wait has no timeout, which is a known gap.
- A failed reconnect returns the dead connection, so the request becomes `FAILED`.
- PgBouncer transaction mode and what it assumes about sessions.
- Startup behaviour: the pool opens all connections eagerly; if the DB is down the process exits 1 and Kubernetes restarts it.
- Why detached tasks create the offset/loss risk (ties to Q6).

---

## Q14. Frugality ✅
*Merges: "did more with less", "avoided unnecessary complexity".*
**Script (~60s)**
"Both systems projects use no external libraries: ShellX is raw POSIX calls and LexiCore is only the C++ standard library. That keeps builds simple and means I understand every line. In LexiCore, my edit-distance function uses two rolling rows instead of a full table, so memory per comparison is small, and I added a length-difference early exit. I also picked the cheapest structure for each query: a hash set for exact match, a trie for prefixes, a BK-tree for fuzzy search, and I kept the linear scan only as a baseline. On the HPE project the deployment caps each pod at 20 database connections even though it runs 100 workers, with PgBouncer in front, because database connections are expensive. The trade-off is that extra workers wait, so I'd tune that ratio from real measurements."

**Learn**
- The three indexes' sizes: Trie 185,264 nodes, BK-tree 88,344 nodes. No memory figures were measured.
- Why a Postgres connection is costly (a backend process each).
- Why the unordered_map-per-node trie saves memory on sparse nodes but loses ordering and speed. This was chosen by reasoning and not benchmarked against `array<26>`.

---

## Q15. Handling ambiguity / scope ✅
*Merges: "unclear requirements", "said no to features".*
**Script (~60s)**
"ShellX had no spec beyond 'build a Unix shell', and a real shell is enormous. So I defined the scope on purpose: external commands, four builtins, redirection, pipes up to 16 stages, single-command background jobs, and signal handling. For everything outside it, I decided the shell must fail loudly rather than behave wrongly. It explicitly rejects semicolons, `&&`, `||`, command substitution and backgrounded pipelines with an error message. I wrote the limits into the README, including that there are no process groups yet. That let me test what I did include: 22 black-box tests pass. I'd rather ship a small correct core than a wide flaky one, and my next steps are process groups for proper job control and tests for signals and zombie reaping."

**Learn**
- Known quirks you can name yourself: job numbers are vector positions; the `Done` branch in `printJobs` is dead code; a mid-line `&` becomes a literal argument; unterminated quotes are accepted; repeated redirects keep only the last.
- Not tested: SIGINT, zombie reaping, `jobs`, exit codes 126/127, limits, pipelines over 16.
- How you'd add `cmd1 | cmd2 &`: a job with multiple PIDs or a process group.

---

## Q16. Have Backbone: challenging my own shortcut ⚠
*Merges: "challenged a quick optimization", "proved an assumption wrong".*
**Script (~60s)**
"For fuzzy search I had a fast bounded edit-distance function that stops early and returns `maxDist + 1` when the true distance is larger. It's tempting to use it inside the BK-tree too. But the tree needs the exact distance to decide which children to visit. For example, with maxDist 2, if the true distance is 7 but the bounded function returns 3, the search would only look at edges 1 to 5 instead of 5 to 9, silently skipping valid matches. So I used the exact distance in the tree, put a warning comment in the code, and wrote a test that asserts the sentinel behaviour. The oracle test checks the tree against a linear scan. The cost is that the bounded function is only used in tests, and fuzzy search stays at 2.4 times faster. Correctness came first."

**Confirm:** your prep notes say the oracle caught it. Git has no such history. Tell it as reasoning, as written above, unless you're sure it was found by a failing test.

**Learn**
- Prove the pruning rule from the triangle inequality: if d(q,u)=d, matches in a child at edge weight w need |w − d| ≤ maxDist.
- Where bounded distance *can* be used: linear scan, or a filter-then-verify step like SymSpell.
- Note the bounded function re-assigns a full row each iteration, so its cost isn't strictly proportional to the band.

---

# Part 3: Results, risk and self-awareness

## Q17. Deliver Results ✅
*Merges: "delivered a project end to end", "fixed hidden flaws in how you measured".*
**Script (~65s)**
"LexiCore went from nothing to a tested, benchmarked search engine between September 9 and October 5, 2026, working alone. It supports exact, prefix and fuzzy search over 88,344 words, with a hash set, a trie and a BK-tree. Two things made the results trustworthy. First, the dictionary is sorted, and inserting sorted words into a BK-tree makes it deep, so I insert in a seeded shuffled order. That cut maximum depth from 27 to 19. Second, benchmarking the first N words of a sorted list would only test words starting with 'a', so I sample subsets randomly and use the same seeded query sets for every strategy. Results: 271 times faster for exact, 49 for prefix, 2.4 for fuzzy, reported as they are. I know the harness still times each strategy once, so I'd add repeated trials next."

**Learn**
- Build times: Trie 39.42 ms, BK-tree 150.10 ms. Startup index build is under 200 ms total.
- Normalization trade-offs: "dog's" becomes "dogs"; non-letters are dropped; queries are *not* normalized.
- 7 commits, one implementation commit; don't claim an iterative git history.
- Release flags are `-O3 -DNDEBUG` (RESULTS.md says -O2).

---

## Q18. Spotting risks before they became problems ✅
*Merges: "anticipated an issue", "security or reliability concern".*
**Script (~60s)**
"After building the Authorization Service, I audited it end to end for risks nobody had asked about. The most serious: the balance check isn't atomic with the debit, which happens later in another service, so concurrent payments from one wallet could all pass. Second, I handle money as a `double`, which risks rounding errors. Third, the database password is plaintext in the deployment file instead of a Kubernetes Secret. Fourth, there are no health probes, metrics or structured logs, so failures would be hard to see. I ranked them by customer impact, put double-spending first, and wrote a remediation plan: atomic reservation, a decimal or integer type, proper secrets management, then observability. Finding these in review is far cheaper than finding them in an incident. I haven't fixed them yet, and I say so."

**Learn**
- Race conditions and double-spend prevention: conditional updates, row locks, idempotency keys.
- Kubernetes Secrets, liveness and readiness probes.
- How you ranked them: impact times likelihood.

---

## Q19. Auditing your own work ⚠
*Merges: "found a flaw in work you thought was done", "found a discrepancy".*
**Script (~60s)**
"In LexiCore my unit tests for prefix autocomplete passed on tiny dictionaries. When I ran it on the full 88,344-word dictionary, `prefix a 5` returned 'a', 'ay', 'ays', 'ayyubid' and 'ayyubids', not the first five in alphabetical order. The cause is that trie children live in an `unordered_map`, so traversal order is arbitrary. The code stops after `limit` words, then sorts those. So it sorts the wrong five. I also noticed the CLI doesn't normalize queries, so `exact Apple` isn't found even though the dictionary is lowercase. I haven't changed the code yet; I've documented both. The fix is to collect all completions and take the top few, or use an ordered child container, and normalize queries in the CLI. The lesson: tiny test fixtures hide order-dependent bugs."

**Confirm:** run `prefix a 5` and `exact Apple` yourself first. Only claim you found it if you did.

**Learn**
- Header comment says "sorted lexicographically": the code and docs disagree.
- Trade-offs: ordered `std::map` vs `array<26>` vs radix trie.
- CLI `prefix` asks for `limit+1000` results so the "and N more" count is capped.
- Other docs vs code mismatches you can name: README "fraud rules" (none exist); `docs/prep` files describe a different project.

---

# Part 4: People questions 🧩

*These aren't in your project files. Use a true story. If none exists, use the fallback.*

## Q20. Disagreement / conflict / Disagree and Commit 🧩
**Script skeleton (~65s)**
"On [project], [teammate or mentor] and I disagreed about [specific technical decision]. I wanted [your position] because [data or reasoning]. They preferred [their position] because [their reasoning, stated fairly]. I [brought evidence: a test, a measurement, a written comparison]. We decided on [outcome]. Once decided, I committed fully by [action]. The result was [outcome]. I learned to disagree with data, not opinion, and to commit completely once the decision is made."

**Fallback (honest):** use the Q4 decision (accept both JSON formats vs ask the gateway team to change), and say plainly it was a trade-off you weighed, not an argument with a person.

**Learn**
- Expect: "What if they still said no?" (Disagree and commit.)
- Show respect for the other view. Never blame.

---

## Q21. Receiving critical feedback 🧩
**Script skeleton (~60s)**
"During [program or project], [reviewer or mentor] told me [specific criticism]. My first reaction was [honest reaction]. After thinking it through, I realised [what was right]. I changed [specific action], and the result was [concrete improvement]. Since then I [habit change]."

**Fallback (honest):** "The feedback came from my own review." Use Q6 or Q19 and say so.

**Learn**
- Don't be defensive. Name the action and the result.

---

## Q22. Helping or mentoring someone 🧩
**Script skeleton (~60s)**
"[Person] was struggling with [topic, for example Kafka offsets, pipes and file descriptors, a contest problem]. I [walked through it with a diagram / paired / wrote a short guide], and I focused on [helping them reach the answer themselves]. As a result, [concrete outcome]. Explaining it also exposed gaps in my own understanding."

**Learn**
- Pick a topic you know deeply, such as pipe fds or Kafka offsets, so follow-ups are safe.
- Be ready to teach a concept in 60 seconds.

---

# Part 5: Priorities, growth and flagship

## Q23. Competing priorities ⚠ (verify dates)
**Script (~60s)**
"From February to July 2026 I was in the HPE program while keeping up with my coursework and competitive programming, and from September I built two more projects. I handle that by ranking work on impact and deadline and by keeping commitments small and visible. At HPE I broke the service into slices that each worked end to end, so I always had something demonstrable. I made scope cuts deliberately: ShellX has no job control or `;` and `&&`, and LexiCore has no persistence or multi-threading. Both limits are documented, so what I shipped was complete and tested rather than half-built. My CGPA stayed at 9.09 throughout."

**Learn**
- Know your real schedule; don't overclaim hours.
- Be ready for "what did you drop, and why?"

---

## Q24. Self-improvement through competitive programming ✅
*Merges: "pressure", "learning outside class", "a goal you set".*
**Script (~60s)**
"Competitive programming is how I train under pressure. A Codeforces round gives about two hours for several problems, so I read everything first, solve the easy ones fast, and set a time limit per problem so one problem can't eat the contest. When I'm stuck, I write a brute-force solution to check my idea against, which is the same oracle idea I later used in LexiCore. The results are a peak rating of 1455, Specialist, on Codeforces and 951 on AtCoder, with ranks like 877th, top 2.1%, in a Division 3 round and 2194th, top 10.9%, in Division 2. I've solved over a thousand problems. The difference from engineering is that contest code is thrown away, while production code needs readability, failure handling and tests."

**Learn**
- Walk through one problem you solved and one you failed, with the reason.
- Know your numbers: also 1305th (top 4%) in a Div 3 round.
- Review core patterns (graphs, DP, binary search) in case of a live problem.

---

## Q25. Proudest accomplishment / end-to-end walkthrough ✅
*Merges: "most complex system you built", "walk me through a project".*
**Script (~70s)**
"My proudest work is the Authorization Service in a payment gateway at HPE. A Java gateway publishes `payment_initiated`. My C++ service consumes it, and a thread pool does the work so the consumer never waits on the database. A worker parses the JSON, borrows a pooled connection and runs one validation query covering user, intent, amount, method, expiry and balance. It publishes `payment_authorized` or `payment_failed` with a specific reason, and unparseable messages go to a dead-letter topic. Clearing consumes authorized payments, and a persistence worker debits the balance. I built the validation query, both pools, the JSON handling and the fixes I described. I'm also candid about the gaps: offsets commit before processing completes, queued work can be lost on a crash, and there's no balance reservation, and I know exactly how I'd close them."

**Learn**
- Draw the flow: `payment_initiated` → Auth → `payment_authorized` / `payment_failed` / `payment_initiated.DLT`.
- Shutdown: SIGINT/SIGTERM stop the loop, the pool drains, the producer flushes. This is not tested. The flag is `volatile bool`, not `sig_atomic_t`.
- Key tech: C++17, librdkafka, libpqxx, nlohmann/json, Ubuntu 22.04 container.
- Which parts are yours vs the team's. Be exact.
- Only quote an RPS figure if you have the output.

---

# Final checklist
1. Mark every ⚠ story true or false, and use the fallback where it's false.
2. Fill the 🧩 brackets with real stories (or use fallbacks).
3. Fix the resume lines in the "Don't say" table (RPS, median of 5, POSIX-compliant).
4. Run the optional checks in section 0 (15 minutes).
5. Practise each script aloud with a timer, 60-70 seconds each.
6. For every story, write one sentence starting "My part was…".
7. For each project prepare one "I don't know yet, here's how I'd find out" answer.
