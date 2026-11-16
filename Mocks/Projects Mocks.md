# Projects Mocks

## Mock - 1

SESSION STATE
- Mock type: projects (HPE Authorization Service, LexiCore, ShellX); first session, summary already given once
- Rounds done: 7 (7th question unanswered)
- Questions already asked:
  1. Auth: what the 10,423 req/s and 97.67% JMeter figures measured
  2. Auth: bad-JSON message and offset fate, can it be lost
  3. Auth: 100 threads vs 20 DB connections at 10x DB latency
  4. Auth: double-spend with concurrent same-wallet payments
  5. LexiCore: why fuzzy is only 2.4x faster
  6. LexiCore: "median of 5 trials" vs benchmark.cpp
  7. ShellX: pipe fd ownership and hang in 3-stage pipeline
  8. ShellX: SIGCHLD flag race while REPL blocks in getline (asked, not answered)
- Score trend: depth 2.4 | ownership 2.6 | trade-offs 1.7 | clarity 2.0 (out of 5)
- Top 3 weaknesses: (1) resume claims not defensible; (2) wrong or vague failure-mode consequences and fixes; (3) fragmented, hedged delivery, answer not led directly
- Handled well: pipe EOF/fd hang (ShellX); camelCase/snake_case parsing not yet tested
- Facts to fix:
  - 10,423 req/s is API Gateway ingest throughput (JMeter hit the gateway), not Auth throughput; a teammate ran the test; failure breakdown unknown
  - LexiCore timed each strategy once per size; numTrials only printed, so "median of 5" is false until code is fixed
  - Offsets auto-stored on consume(), before the worker runs; loss and duplicates are possible
  - Slow DB causes unbounded queue growth and OOM, not consumer lag
  - BK shuffle (seed 42) is already in place: depth 19 vs 27
- Next focus: ShellX SIGCHLD/sigprocmask and job reaping; HPE behavioral stories (not yet covered); LexiCore Trie autocomplete bug

---
