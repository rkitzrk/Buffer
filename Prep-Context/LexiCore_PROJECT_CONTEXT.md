# PROJECT_CONTEXT: LexiCore

Sources: all files in LexiCore.zip. "(verified by run)" marks facts I confirmed by compiling and running code.

## 1. Summary
LexiCore is a single-process C++20 CLI that answers three query types over a word list: exact match, prefix autocomplete, and fuzzy (edit-distance) search. Each uses a different structure: `unordered_set`, Trie, BK-tree. A benchmark harness compares each against a linear-scan baseline. Stack: C++20, STL only, CMake >= 3.20 (CMakeLists.txt), g++ 13.3.0 on Ubuntu 24.04 (RESULTS.md). Dictionary: `data/words.txt` has 104,334 raw lines and loads to 88,344 unique words (verified by run). No latency SLA is defined; the goal is per-query-type speedup evidence. Solo project; 7 commits, 2026-09-09 to 2026-10-05; one implementation commit (2cb19b3).

## 2. Architecture
- `Dictionary` (dictionary.cpp): loads the file, `normalize()` (lowercase, strip non-alpha), dedupes via `wordSet_`, provides `getSubset(n, seed)` and `getShuffledWords(seed)` (`std::mt19937` + `std::shuffle`).
- `editDistance` / `editDistanceBounded` (edit_distance.cpp): Levenshtein with two rolling rows. The bounded variant uses a diagonal band, an early length-difference exit and a row early-exit, and returns `maxDist+1` as a sentinel.
- `Trie` (trie.cpp): `TrieNode{unordered_map<char, unique_ptr<TrieNode>>, isEndOfWord}`. Has `insert`, `search`, `startsWith`, `autocomplete(prefix, limit=20)`, `getNodeCount`.
- `BKTree` (bktree.cpp): `BKNode{word, unordered_map<int, unique_ptr<BKNode>>}`. Has `insert`, `search(query, maxDistance)`, `getNodeCount`, `getMaxDepth`. Recursive.
- `rankResults` (ranking.cpp): distance ascending, then lexicographic.
- `runBenchmark` + `BenchmarkConfig` (benchmark.cpp): sizes {1000, 10000, 0=full}, 1000 queries, `numTrials=5`, `fuzzyMaxDistance=2`, seed 42.
- `app/main.cpp`: REPL: `exact`, `prefix <p> [limit]`, `fuzzy <w> [maxDist]`, `benchmark`.

Data flow: `main` loads the dictionary, then builds the Trie (load order) and the BK-tree (`getShuffledWords(42)`). `exact` queries `getWordSet()`. `prefix` calls `trie.autocomplete`. `fuzzy` calls `bktree.search`, then `rankResults`, and prints at most 20 results. Everything is in memory. `lexicore_lib` is linked by `lexicore` and four test executables.

## 3. Key design decisions
1. Structure per query type. Hash set for exact, Trie for prefix, BK-tree for fuzzy. Linear scan is kept as benchmark baseline and test oracle (README). Trade-off: three indexes in memory, no unified index.
2. Trie children as `unordered_map<char, unique_ptr>`. Rejected: `array<Node*,26>` (README). Reason: flexible alphabet, sparse-node memory. Trade-off: slower lookup. No measurement in the repo [unverified].
3. `unique_ptr` ownership for all nodes, no manual `delete` (README). Trade-off: recursive destruction.
4. Seeded shuffle (seed 42) before BK insertion (`Dictionary::getShuffledWords`). Rejected: sorted-order insertion, which README says creates chain-shaped trees. Trade-off: tree shape depends on the seed.
5. BK search uses true `editDistance()`, not `editDistanceBounded()` (bktree.cpp comment "CRITICAL"). Reason: pruning interval `[d-maxDistance, d+maxDistance]` needs the real `d`. Trade-off: `editDistanceBounded` is unused outside tests.
6. Normalization: lowercase, strip every non-letter, drop blanks, first duplicate wins (dictionary.hpp). Trade-off: "dog's" becomes "dogs"; non-ASCII letters are dropped (inferred from `std::isalpha` on bytes) [unverified at runtime].
7. Ranking: distance, then lexicographic, no frequency data (README: deterministic output).
8. Benchmark subsets are seeded random samples, not first-N (README).

## 4. Hard problems and bugs
1. Sentinel in BK pruning. Symptom: pruning with the bounded distance would silently skip valid subtrees. Cause: `editDistanceBounded` returns `maxDist+1` when the true distance is larger, so the interval is wrong. Fix: `searchImpl` in bktree.cpp calls `editDistance`; the comment and `edit_distance.hpp` warn about it. Tests: `test_correctness.cpp` (oracle) and `testBoundedBailout` in test_edit_distance.cpp, which asserts the sentinel (e.g. "cat","dog",1 returns 2). That this bug actually occurred and was caught by the oracle appears only in docs/prep notes; the git history has no such change [unverified].
2. Degenerate BK-tree from ordered input. Fix: `getShuffledWords(42)`. Measured on `data/words.txt` (verified by run): max depth 27 unshuffled, 19 shuffled (matches RESULTS.md: 19 at 88,344). Tests: none asserts depth on the real dictionary (`testNodeCountAndDepth` uses 4 words).
3. Proving pruning correctness (silent drops). Fix: `linearFuzzySearch` oracle in `test_correctness.cpp`. It runs 11 small-dictionary queries x maxDist 0-3, a randomized pass (500 random words, 200 queries, maxDist 0-3, seed 99999), and edge cases (empty query, query longer than any word, maxDist=0). A mismatch calls `std::abort()` in the first two tests.
4. Benchmark skew. First-N slices of a sorted list would bias results; `getSubset(n, seed)` samples randomly, and query sets are seeded (`seed+1/+2/+3`) and shared by both strategies. No tests cover the harness.

## 5. Concurrency, errors, failure behaviour
- Single-threaded; no threads or locks in src/app. Structures are built once and read-only afterwards, so no shared-state protection is needed.
- Missing or empty dictionary: prints an error and exits 1. Path comes from `argv[1]`, default `data/words.txt` (relative to the working directory).
- CLI validation: `fuzzy` rejects negative maxDist; `prefix` limit 0 becomes 20; unknown commands print a hint.
- `exact`, `prefix` and `fuzzy` do not normalize the query (main.cpp). `exact Apple` is not found.
- Non-numeric `limit` or `maxDist` is not checked; a failed extraction yields 0 [unverified at runtime].
- No retries, timeouts, persistence or crash recovery; the index is rebuilt each start. Recursion depth is bounded by the longest word (Trie) and BK depth (19 here).

## 6. Testing and quality
- Four `add_test` executables (CMakeLists.txt): `test_edit_distance` (9 test functions: basic cases, bounded variant, symmetry and triangle inequality over 1,000 random cases each), `test_trie` (8), `test_bktree` (9), `test_correctness` (3). Total 29 (counted by run). Plain `<cassert>`, no framework.
- ASan/UBSan come only from a manual `-DCMAKE_CXX_FLAGS` (README), not CMakeLists.txt. `build-debug/Testing/Temporary/LastTest.log` shows pass. I rebuilt with `g++ -fsanitize=address,undefined` (no cmake available); all four exit 0 (verified by run).
- Not present: CI config, `-Wall`/`-Werror`, fuzzing, coverage, tests for `Dictionary`, `rankResults`, benchmark code or CLI.
- `assert` is a no-op under `-DNDEBUG`, which CMake's Release build sets (`build-release/.../flags.make`: `-O3 -DNDEBUG`). Release test runs check little beyond the `std::abort` paths in test_correctness.

## 7. Measured results (RESULTS.md only)
Conditions: x86_64, Ubuntu 24.04, g++ 13.3.0, C++20, 1,000 queries per config, fuzzy maxDist=2, seeded queries, subsets by random sample. Times are avg µs per query. Doc says 5 trials, median (see §10).

| Query (88,344 words) | Linear | Structure | Speedup |
|---|---|---|---|
| Exact | 121.80 | hash 0.45 (p99 1.04) | 271x |
| Prefix (limit 20) | 214.71 | Trie 4.39 (p99 9.45) | 49x |
| Fuzzy | 13,818.35 | BK-tree 5,729.39 (p99 8,797.34) | 2.4x |

Other numbers (RESULTS.md):
- Trie at 10K words: 4.51 µs; at 88K: 4.39 µs.
- BK-tree vs linear: 82.71 vs 163.58 µs (1K); 663.81 vs 1,639.65 µs (10K).
- Build at 88,344 words: Trie 39.42 ms, 185,264 nodes (node count verified by run); BK-tree 150.10 ms, 88,344 nodes, depth 19.
- BK depth at 1K and 10K: 9 and 12.
No memory or CLI latency figures exist.

## 8. Known limitations and next steps
- Fuzzy gain is only 2.4x; README attributes it to edit-distance cost. Docs/prep notes propose a SymSpell-style index, radix trie and frequency ranking [not implemented].
- `Trie::autocomplete` stops after `limit` words in `unordered_map` iteration order, then sorts. It does not return the lexicographically first N (see §10).
- CLI `prefix` requests `limit+1000` results, so "...and N more" is capped.
- `editDistanceBounded` re-assigns a full row each iteration, so cost is not strictly proportional to the band; it is unused in production code.
- No index persistence or multi-threading.

## 9. Likely interview questions
1. Why must BK search use the true distance? bktree.cpp `searchImpl`; edit_distance.hpp comment.
2. Prove the triangle-inequality pruning interval. bktree.cpp `low`/`high`.
3. Why is fuzzy only 2.4x faster, and how would you improve it? RESULTS.md; §8 above.
4. Why shuffle before BK insertion, and what depth results? `getShuffledWords`; depth 19 vs 27 (§4).
5. Trie as `unordered_map` vs `array<26>` vs radix trie? trie.hpp; README "Memory Ownership".
6. How do you know the BK-tree is correct? test_correctness.cpp oracle; test_edit_distance.cpp property tests.
7. How does `editDistance` work and what is its space cost? edit_distance.cpp, two rows, O(m).
8. Are your benchmark numbers trustworthy? benchmark.cpp `timeQueries`; §10 items 2-3.
9. Why can asserts silently pass in Release? CMake `-DNDEBUG`; tests use `<cassert>`.
10. Is `autocomplete("a", 5)` correct? trie.cpp `collectWords`; §10 item 1.

## 10. Discrepancies and unverified claims
1. Trie.hpp says autocomplete returns words "sorted lexicographically" up to `limit`. The code collects the first `limit` words in hash-map order and then sorts them. `autocomplete("a", 5)` returned `a ay ays ayyubid ayyubids` (verified by run). Unit tests use tiny dictionaries and miss it.
2. RESULTS.md and README say 5 trials, median reported, and 1 full warm-up pass. In benchmark.cpp `config.numTrials` is only printed in the banner. Each strategy is timed once per size. Warm-up is partial: 10 queries for fuzzy, one pass over the query list for exact and prefix.
3. RESULTS.md says Release `-O2`. The build directories use `-O3 -DNDEBUG` (CMake default for Release; CMakeLists.txt sets no flags).
4. README and docs/prep say unshuffled BK depth is "100+". Measured on `data/words.txt` in load order: 27 (verified by run).
5. README architecture shows a "Normalization" stage before queries. main.cpp normalizes only the dictionary, not queries.
6. Unsupported in the repo: Trie memory efficiency vs array; that the sentinel bug occurred and the oracle caught it.
7. docs/prep/Authorization.md and HPE-F.md describe a different project (librdkafka, libpqxx). They are not LexiCore. docs/prep is listed in .gitignore; four study files were added and then deleted in commits 81946bd and 7442355.
8. Not confirmed: CPU model and provenance of `/usr/share/dict/words`. Benchmarks were not re-run.
