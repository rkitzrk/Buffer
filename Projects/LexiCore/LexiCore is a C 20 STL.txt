**LexiCore** is a C++20, STL-only retrieval engine: exact match (`unordered_set`), prefix/autocomplete (Trie), fuzzy/typo search (BK-tree + Levenshtein), plus a benchmark vs linear scan. Phase 0 is the CS + C++ you need before opening source files.

Later file phases (names only; we go deep after you confirm):

| Phase | Focus | Files (importance order) |
|---|---|---|
| **1** | Core algorithms | `edit_distance.hpp/.cpp`, `trie.hpp/.cpp`, `bktree.hpp/.cpp` |
| **2** | System around them | `dictionary.hpp/.cpp`, `ranking.hpp/.cpp`, `app/main.cpp` |
| **3** | Proof | `benchmark.hpp/.cpp`, `tests/*`, `RESULTS.md` |
| **4** | Build + docs | `CMakeLists.txt`, `README.md`, `docs/*` |

---

# Phase 0 — Foundations

Think of this as: *if an interviewer never lets you open the repo, you still explain the product and every structure it uses.*

## 0.1 What problem this project solves

Three query types, **one dictionary**, **different structures**:

1. **Exact:** “Is `apple` in the dictionary?”
2. **Prefix:** “Words starting with `app`?” (autocomplete)
3. **Fuzzy:** “Words within edit distance 2 of `aple`?” (typos)

Naive answer: scan all ~88K words every time. Exact scan is cheap-ish; fuzzy scan runs Levenshtein on every word → **O(n · L²)** and is slow. LexiCore’s claim: pick the structure that matches the query, then **measure** it.

That last part is what placements care about: not “I implemented a trie,” but “I chose per query type and proved the trade-off.”

## 0.2 Mental model (end-to-end)

```text
words.txt → normalize (lower, strip non-alpha, drop blanks, first-wins dedup)
         → vector (order) + unordered_set (exact)
         → Trie (prefix)
         → BK-tree from shuffled words (fuzzy)
CLI: exact | prefix | fuzzy | benchmark
fuzzy results → rank by (distance ↑, then lex)
```

**Mismatch vs older docs:** some guides mention frequency ranking. The **code** ranks by distance then lexicographic order. No frequencies. Say that in interviews.

## 0.3 Strings and “distance”

- A **word** is a sequence of characters. Operations: insert, delete, substitute (each cost 1 here). Transpose (Damerau) is **not** used.
- **Levenshtein distance** `d(a,b)` = min operations to turn `a` into `b`.
- It is a **metric** on strings:
  - `d(x,x) = 0`
  - `d(x,y) = d(y,x)` (symmetry)
  - `d(x,z) ≤ d(x,y) + d(y,z)` (triangle inequality)

The third property is **why BK-trees prune**. If you cannot state triangle inequality, you cannot explain the fuzzy index.

Classic DP table `dp[i][j]` = distance between first `i` chars of `a` and first `j` of `b`:

- `dp[0][j] = j`, `dp[i][0] = i`
- `dp[i][j] = min(delete, insert, substitute-or-match)`

This project uses **two rows** (`prev`/`curr`) → **O(min(n,m))** extra space, still **O(n·m)** time.

**Bounded** variant: if you only care whether `d ≤ maxDist`, you can:

- reject if `|n-m| > maxDist`
- only fill a **diagonal band** of width `2·maxDist+1`
- return sentinel `maxDist+1` if it must exceed the bound

Sentinel ≠ true distance. Using it to compute BK-tree prune intervals `[d − R, d + R]` is a **correctness bug**.

## 0.4 Hash tables (exact path)

`unordered_set<string>`: average **O(1)** membership, worst **O(n)** if everything collides.

You should be able to say:

- hash → bucket
- chaining vs open addressing (libstdc++ typically chaining)
- load factor / rehash
- why hashing **cannot** do prefix or “near” queries without scanning keys

That last sentence is the design justification for Trie and BK-tree.

## 0.5 Tries (prefix path)

A **trie** is a tree where each edge is a character, a path from root is a prefix, `isEndOfWord` marks a complete word.

- Insert / exact / “does prefix exist”: **O(k)** in word/prefix length **k**, independent of dictionary size **n**.
- Autocomplete: walk **k** edges, then DFS (or BFS) the subtree. Output size can be large → this project **caps** with `limit`.

Children here: `unordered_map<char, unique_ptr<TrieNode>>` (sparse, any alphabet) vs `array<26>` (faster, fatter, a–z only). Interviewers love this trade-off.

Space: many nodes. Full dict here: **185,264 nodes for 88,344 words**. Compressed / radix trie is the usual “production next step,” not implemented.

## 0.6 Metric trees and BK-trees (fuzzy path)

A **BK-tree** (Burkhard–Keller): each node stores a word; an edge to a child is labeled with **integer edit distance** from parent to child. At most one child per distance (recurse if that slot is taken).

**Search** for query `q` with radius `R`:

1. `d = editDistance(q, node.word)` — **true** distance.
2. If `d ≤ R`, collect the word.
3. Recurse **only** children with edge key in `[d − R, d + R]`.

Why: if child is at distance `e` from parent, triangle inequality implies you cannot be within `R` of the query unless `e` lies in that interval. Whole subtrees skipped **without** computing distance to those words.

Insertion **order** changes shape. Sorted dictionaries → similar consecutive distances → **chains**. This repo **shuffles** with seed **42**. Depth ~19 at 88K vs potentially 100+ unshuffled.

BK-tree is **not** asymptotically “always O(log n).” Pruning depends on metric, radius, and data. Benchmarks here: about **2–2.5×** vs linear at `maxDist=2`, not 100×. Honest numbers beat fake log n.

## 0.7 C++ you must be fluent in (to read Phase 1)

| Topic | Why it appears |
|---|---|
| `std::unique_ptr` / RAII | Trie and BK nodes; no raw `new`/`delete` |
| `std::unordered_map` / `unordered_set` | children maps; exact index |
| `std::vector`, `std::swap` | DP rows; word lists |
| `std::move` | `BKNode` constructor |
| `std::shuffle` + `std::mt19937` | BK insert order; subsets |
| `static_cast<unsigned char>` with `tolower`/`isalpha` | avoid UB on signed `char` |
| C++20, CMake library + exe + ctest | how the binary is built |
| ASan/UBSan | tests run with sanitizers |

If `unique_ptr` vs `shared_ptr` vs raw pointer is fuzzy, fix that before Phase 1.

## 0.8 Complexity and “when fancy structures lose”

| Structure | Search | Insert | Space | Role |
|---|---|---|---|---|
| Linear | exact O(n); prefix O(n·k); fuzzy O(n·L²) | O(1) append | O(n) | baseline |
| Hash set | O(1) avg exact | O(1) avg | O(n) | exact only |
| Trie | O(k) + output | O(k) | O(nodes) | prefix |
| BK-tree | data-dependent | O(depth × L²) | O(n) | fuzzy |

Small **n**: extra pointer chasing can lose to a tight scan. That is a **valid** empirical result, not a failure.

## 0.9 Ranking, fairness, testing (conceptual)

- Rank: distance ascending, then lexicographic — **deterministic**, no hidden frequency.
- Benchmarks: same query set, seeded subsets (not first N of an alphabetical file), median of trials, warmup.
- Correctness: BK-tree vs linear oracle; edit-distance **symmetry** and **triangle inequality** on random strings.

## 0.10 How this maps to CP (your background)

You already have: DP on strings, trees, hashing, Big-O.

New “SDE” layer:

- **Index vs query** (build once, query many times)
- **Metric pruning** (not just “run DP n times”)
- **Ownership and UB** (sanitizers, `unsigned char`)
- **Measurement** (p50/p95, paired comparison)
- **API design** (limit, maxDist, sentinel documented)

---

# Phase 0 — Interview questions and answers

Grouped. Answers match **this repo** unless marked as general/extension.

### A. Product and design

**Q1. What is LexiCore in one minute?**  
A multi-strategy dictionary search engine: hash set for exact membership, trie for prefix autocomplete, BK-tree for Levenshtein-bounded fuzzy search, with a linear-scan baseline and a benchmark suite on ~88K words.

**Q2. Why not one structure for everything?**  
A hash set cannot enumerate prefixes or neighbors. A trie does not give edit-distance neighborhoods cheaply. A BK-tree is a poor exact/prefix index. Matching structure to query is the architecture.

**Q3. Why keep a linear scan if it is slower?**  
It is the **oracle** (correctness) and the **baseline** (performance). Without it you cannot claim speedup or prove BK-tree completeness.

**Q4. Is this a search engine like Google?**  
No. No crawling, ranking web pages, inverted indexes, or BM25. It is the **retrieval core** of autocomplete / spell-check.

**Q5. Why C++20 and STL only?**  
Fits systems/DSA interviews: you own every structure, no dependency theater. C++20 is a clear standard for `unique_ptr`, chrono, etc.

**Q6. Docs mention frequency ranking. Does the code?**  
No. `rankResults` sorts by distance then lexicographic order. If asked, say frequency was a planned stretch and you chose deterministic ranking without synthetic frequencies.

**Q7. What is the normalization policy?**  
Lowercase, keep only alphabetic characters, skip empty lines, **first occurrence wins** on duplicates.

**Q8. Why strip non-alpha instead of keeping hyphens/apostrophes?**  
Simplicity and a uniform alphabet for the trie. Cost: `don't` and `o'clock` collapse. Call it a deliberate policy, not an accident.

**Q9. How would you design “search suggestions” at Google/Amazon scale?**  
(General.) Prefix: trie / ternary search tree / finite automaton / n-gram DB. Fuzzy: BK-tree, metric trees, Levenshtein automata, symmetric delete, Elasticsearch fuzziness. Rank: frequency, recency, personalization, click-through. Shard the dictionary. LexiCore is the **in-memory single-node core**.

**Q10. Why CLI not HTTP?**  
Keeps scope on algorithms. A REST wrapper does not change Trie/BK-tree complexity.

---

### B. Hashing and exact match

**Q11. Complexity of exact lookup?**  
Average O(1) for `unordered_set`; worst O(n) under pathological hashes/collisions. Linear exact is O(n) string compares.

**Q12. Why `unordered_set` not `unordered_map<string,int>`?**  
No frequency payload. Set is the right ADT for membership.

**Q13. Why not `std::set` (tree)?**  
O(log n) and ordered. You do not need order for exact; hash is faster on average.

**Q14. Can a hash table do prefix search?**  
Not without scanning all keys (or extra indexes). Hashing destroys prefix locality.

**Q15. What is a load factor?**  
`size / bucket_count`. Above a threshold, rehash (allocate more buckets, reinsert). Rehash is amortized into O(1).

**Q16. Chaining vs open addressing?**  
Chaining: list/vector per bucket, simple deletes. Open addressing: probe sequence, better cache if load is moderate, clustering issues. Know that `unordered_*` in libstdc++ is typically node-based chaining.

**Q17. How does `string` hashing work at a high level?**  
Bytes mixed into an integer (implementation-defined quality). Interview bar: collision resistance is not cryptographic; remaining-string attacks exist in theory; for a dictionary it is fine.

**Q18. Hash vs trie for exact match?**  
Hash usually wins on exact (see RESULTS: millions of QPS). Trie exact is O(k) with more pointer chasing. Trie still wins **prefix**.

---

### C. Levenshtein / DP

**Q19. Recurrence?**  
`dp[i][j] = dp[i-1][j-1]` if `a[i-1]==b[j-1]`, else `1 + min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])` (del, ins, sub).

**Q20. Time and space of naive table?**  
O(n·m) time and space.

**Q21. How does this repo save space?**  
Two rows of length m+1; `swap(prev, curr)`. Space O(m). (You can swap roles of n,m to use the shorter string as columns.)

**Q22. Why is `kitten` → `sitting` distance 3?**  
k→s, e→i, insert g (classic). Know 2–3 textbook examples: `saturday`/`sunday` = 3, `intention`/`execution` = 5.

**Q23. Is Levenshtein a metric?**  
Yes, with unit costs on ins/del/sub. Tests check symmetry and triangle inequality on random strings.

**Q24. Hamming vs Levenshtein vs Damerau–Levenshtein?**  
Hamming: substitutions only, equal length. Levenshtein: ins/del/sub. Damerau: also adjacent transposition (common typo). This project is plain Levenshtein.

**Q25. Why `|len(a)-len(b)|` is a lower bound?**  
You need at least that many insertions or deletions. Bounded DP uses it as an O(1) reject.

**Q26. What is `editDistanceBounded`?**  
Band DP around the diagonal; early abort; returns true `d` if `d ≤ maxDist`, else **`maxDist+1` sentinel**.

**Q27. Why must BK-tree search use full `editDistance` for `d`?**  
Prune range is `[d−R, d+R]`. A sentinel `R+1` is not `d`, so the interval is wrong and you can skip valid children.

**Q28. Can you use bounded DP inside the tree at all?**  
You could use it to **decide membership** (`d ≤ R`) **after** you already have true `d` for pruning, or only if you still compute true `d` another way. The comment in headers: do not feed the sentinel into the interval.

**Q29. Wagner–Fischer vs Myers vs Ukkonen?**  
Wagner–Fischer = standard DP. Ukkonen-style: threshold / diagonal. Myers: bit-parallel for small alphabets. LexiCore is Wagner–Fischer + a banded/threshold variant.

**Q30. DP on words of length 5 vs 5000?**  
O(n·m) is fine for dictionary words; terrible for documents. Scaling limit, not a bug.

**Q31. Recursive Levenshtein with memo vs iterative?**  
Same complexity; iterative is cache-friendlier and has no recursion depth issue. This code is iterative.

**Q32. Can edit distance be computed in O(n+m)?**  
Not in general for Levenshtein (essentially quadratic under SETH-type conjectures). Approximate / embedding tricks exist; out of this project’s scope.

---

### D. Trie

**Q33. What does a TrieNode store here?**  
`unordered_map<char, unique_ptr<TrieNode>> children` and `bool isEndOfWord`.

**Q34. Why `isEndOfWord`?**  
`app` may be a prefix of `apple` and also a word. The flag distinguishes.

**Q35. Insert complexity?**  
O(k). Creates missing nodes.

**Q36. Autocomplete steps?**  
`findNode(prefix)` O(k); if null, empty; else DFS collecting paths where `isEndOfWord`, stop at `limit`. Results intended lexicographic — depends on map iteration + DFS order; `unordered_map` does **not** iterate alphabetically. **Interview-honest:** header says “sorted lexicographically”; if collect order is not a sort, say you’d `std::sort` before return (verify in Phase 1). Flag this when you read `trie.cpp`.

**Q37. Why `limit`?**  
Prefix `a` can dump thousands of words and flood the terminal.

**Q38. `array<unique_ptr<TrieNode>, 26>` vs `unordered_map`?**  
Array: O(1) child, more memory per node, a–z only. Map: sparse, Unicode/flexible, extra hash overhead. README chooses map for sparse nodes and flexible alphabet.

**Q39. Memory problem of tries?**  
One heap node per character along unique prefixes. 88K words → ~185K nodes here. Fix: radix/Patricia trie, LOUDS, succinct tries.

**Q40. Trie vs suffix tree vs suffix array?**  
Trie: prefixes of a **set of words**. Suffix tree/array: all suffixes of **one** (or concatenated) string — different problem (substring search).

**Q41. Ternary search tree?**  
Three-way trie (lo/eq/hi), less memory than 256-wide nodes, slower than hash-child. Alternative, not used.

**Q42. Why trie search independent of n?**  
You only walk `k` characters. **n** affects memory and branching, not the walk length. Collecting all matches still depends on subtree size.

**Q43. How do you delete a word from a trie?**  
Walk the path, unset `isEndOfWord`, optionally prune nodes with no children and not end-of-word. Not implemented here.

**Q44. Can a trie do fuzzy search?**  
Yes: DFS with remaining edit budget (trie + DP / automaton). More common in spellcheck than BK-trees at huge scale. LexiCore uses BK-tree instead — you should be able to compare both.

---

### E. BK-tree and metrics

**Q45. What is a BK-tree?**  
A tree for discrete metrics: children keyed by distance to parent; search uses triangle inequality to skip branches.

**Q46. Insert algorithm?**  
Empty → root. Else `e = d(word, node)`; if child `e` exists, recurse; else attach.

**Q47. Why at most one child per distance?**  
By construction the map is `distance → one node`. Ties recurse down that unique edge.

**Q48. Search pruning formula?**  
Let `d = dist(query, node)`, radius `R`. Visit children with keys in `[d−R, d+R]` (and valid keys).

**Q49. Prove pruning (sketch).**  
Let parent `p`, child `c`, query `q`. `|d(q,p) − d(p,c)| ≤ d(q,c)` (rearrange triangle). If `d(q,c) ≤ R`, then `d(p,c)` lies in `[d(q,p)−R, d(q,p)+R]`. If the edge length is outside, **no** point in that subtree can be within `R`? **Careful:** the child **is** the subtree root; descendants are further. The standard BK-tree argument applies at **each** node independently using that node as the pivot: you only skip children whose **edge label** cannot lead to a close point given the metric. You should redraw this on paper until it is muscle memory.

**Q50. Why shuffle before insert?**  
Alphabetical order correlates distances → skinny tree → weaker pruning, more like a list.

**Q51. Why seed 42?**  
Reproducible tree shape, benchmarks, depth stats.

**Q52. Is BK-tree always faster?**  
No. Small n, large R, clustered distances → pruning fails. RESULTS still ~2× at R=2 on this word list. Report that; do not invent log n.

**Q53. Complexity of insert?**  
O(depth × cost of edit distance). Depth is data-dependent.

**Q54. VP-tree / cover tree / HNSW?**  
Other metric/ANN indexes. VP-tree: vantage point + radius split, often for continuous metrics. HNSW: graph-based ANN for embeddings. BK-tree is the discrete, explainable choice for edit distance.

**Q55. Why not k-d tree on strings?**  
k-d trees need coordinate axes and typically Euclidean-like splits. Edit distance is not Euclidean in character coordinates.

**Q56. What happens if `maxDist` is huge (e.g. 10)?**  
Interval covers almost every child → near-linear scan + tree overhead. Radius must stay small (typos: 1–2).

**Q57. Empty tree / empty query?**  
Empty tree: no results. Empty string vs words: distance = word length; may match short words within R.

**Q58. Does BK-tree store each word once?**  
Yes, one node per inserted word (duplicates should already be removed by Dictionary).

**Q59. How do you verify BK-tree correctness?**  
Oracle: for many (query, R), compare BK result set to `{ w | editDistance(q,w) ≤ R }` from the vector. This project has that test.

---

### F. C++ / memory / correctness

**Q60. Why `unique_ptr` not raw pointers?**  
Exclusive ownership, destructor of parent destroys children, exception-safe, no double-free.

**Q61. Why not `shared_ptr`?**  
No shared ownership. Shared_ptr costs atomic refcounts.

**Q62. RAII?**  
Resource tied to object lifetime. `unique_ptr`, `ifstream` in `loadFromFile`.

**Q63. Why `static_cast<unsigned char>` before `isalpha`/`tolower`?**  
`char` may be signed; passing negative to ctype is UB.

**Q64. `std::move` in `BKNode(std::string w)`?**  
Avoid copying the string into the member.

**Q65. Rule of five?**  
If you define destructor/copy/move, define all. Here nodes use `unique_ptr` so the class is move-only by default — good.

**Q66. What do ASan and UBSan catch?**  
Use-after-free, leaks (ASan leak mode), OOB, signed overflow, invalid ctype, etc. README: tests run with both.

**Q67. Why CMake 3.20 and a static-ish library target?**  
`lexicore_lib` linked into CLI and each test — compile core once, test without the CLI.

**Q68. Header vs `.cpp`?**  
Headers: types and contracts. `.cpp`: algorithms. `#pragma once` include guard.

**Q69. `namespace lexicore`?**  
Avoid global name clashes (`Trie`, `Dictionary`).

**Q70. Why not exceptions for missing file?**  
`loadFromFile` returns `bool`; `main` prints and exits 1. Simple CLI policy.

---

### G. Benchmarking and results literacy

**Q71. Why not time one query?**  
Noise. Use many queries, several trials, **median**.

**Q72. Why warmup?**  
Caches, branch predictors, page faults. First run is biased.

**Q73. Why random subset, not first 1000 lines?**  
Word lists are alphabetical; first 1000 ≈ only `a…`. Distorts prefix and edit-distance distributions.

**Q74. Paired comparison?**  
Exact: linear vs hash. Prefix: linear vs trie. Fuzzy: linear vs BK. Same queries, same dict slice.

**Q75. What did they actually measure (order of magnitude)?**  
Exact at 88K: hash ~0.45µs vs linear ~122µs. Prefix: trie ~4.4µs vs linear ~215µs (trie ~flat in n). Fuzzy R=2: BK ~5.7ms vs linear ~13.8ms per query. Memorize the **shape**, not every digit.

**Q76. Why is trie latency similar at 10K and 88K?**  
O(k) + similar prefix fan-out for those random prefixes; n grew but walk length did not.

**Q77. Why is fuzzy still milliseconds?**  
Each visited node pays **full Levenshtein**. Pruning helps but remaining nodes × O(L²) is still large. 175 QPS vs 72 QPS at 88K.

**Q78. p95 vs average?**  
Tail latency. Interviews (SRE/backend) care that p99 is not 10× p50.

**Q79. Compiler `-O2` vs Debug for benches?**  
Always Release for performance numbers. Debug + sanitizers for tests.

---

### H. Ranking and CLI

**Q80. Sort key for fuzzy results?**  
Primary: smaller distance. Secondary: `word` lexicographic.

**Q81. Why lex tie-break?**  
Stable, reproducible demos and tests. No hidden RNG in output order.

**Q82. Default `maxDist`?**  
CLI default 2 (typical typo budget).

**Q83. Default autocomplete limit?**  
20.

**Q84. How is the CLI parsed?**  
Command + args (`exact`, `prefix`, `fuzzy`, `benchmark`, `help`, `quit`). Dictionary path optional `argv[1]`.

---

### I. Testing theory

**Q85. Unit vs property vs oracle?**  
Unit: known pairs (`kitten`/`sitting`). Property: ∀ random pairs, `d(a,b)==d(b,a)`; ∀ triples, triangle inequality. Oracle: BK ⊆ and ⊇ linear brute force.

**Q86. Why 1000 random pairs is not a proof?**  
It is **evidence**, not a formal proof. Good enough for regression; say that.

**Q87. What would you add?**  
(Extension.) Empty prefix, Unicode, huge `limit`, `maxDist=0` equals exact, insertion of duplicates, unbalanced tree fixture.

---

### J. Systems / placement “design” extras (beyond repo, with answers)

**Q88. How would you persist the trie?**  
Serialize nodes, or rebuild from word list (this project rebuilds at startup — 88K is cheap).

**Q89. Concurrent queries?**  
After build, Trie/BK/hash are read-only → many readers, no lock. Rebuild needs a swap of indexes.

**Q90. Memory of 88K English words?**  
Strings + hash set + trie nodes + BK nodes: multiple copies of each word (vector, set, trie path, BK node). Interview: “we duplicate for speed; interning would save RAM.”

**Q91. How does a real spellchecker work (e.g. hunspell)?**  
Affix rules + dictionary + edit/try lists, not a pure BK-tree of every form.

**Q92. How does Elasticsearch `fuzziness` work (high level)?**  
Often term expansion via bounded edits / automata, then inverted-index lookup — not a giant BK-tree of all terms in one process necessarily.

**Q93. LRU cache of queries?**  
Repeated prefixes/typos; cache `(type, query, params) → results`. Easy extra.

**Q94. Thread the linear baseline?**  
Embarrassingly parallel over words; would change speedup vs BK. Fair bench should document threads=1.

**Q95. Integer overflow in DP?**  
Word lengths tiny; `int` is fine. Mention you’d use wider types for huge strings.

**Q96. Locale and `tolower`?**  
`std::tolower` is not Unicode-aware. Turkish `I`/`i` etc. Policy: ASCII-oriented dictionary.

**Q97. Binary search on sorted vector for exact?**  
O(log n) comparisons. Hash still typically faster. Prefix: lower_bound + scan while prefix matches — a strong **linear-index** alternative to a trie; mention in interviews.

**Q98. Why still a trie if sorted vector can prefix-scan?**  
Trie: O(k) to the node then subtree enumeration without scanning unrelated words. Sorted vector: O(log n + output) and excellent cache. RESULTS show trie wins vs **naive** full scan, not vs `lower_bound`. Strong follow-up: “I compared to naive; production might use sorted array or FM-index.”

**Q99. Cache locality: trie vs vector scan?**  
Trie: pointer chasing, poor cache. Vector: sequential. Explains small-n surprises.

**Q100. How do you talk about this as a Specialist on Codeforces?**  
“Same DP and trees as contests, but I treated them as indexes with measurable trade-offs, sanitizers, and an oracle — that’s the engineering delta.”

---

### K. Rapid-fire definitions

**Q101. Alphabet?**  
Set of characters; here effectively `a–z` after normalize.

**Q102. Dictionary size n vs query length k vs word length L?**  
Keep these symbols distinct in every complexity sentence.

**Q103. Metric space?**  
Set + distance with identity, symmetry, triangle inequality.

**Q104. Discrete metric for BK-tree?**  
Distance takes integer values so edges can be keyed by `int`.

**Q105. Sentinel value?**  
A reserved integer meaning “failed / unknown,” here `maxDist+1`.

**Q106. Dedup first-wins?**  
`wordSet_.insert(word).second` then `push_back`.

**Q107. Degenerate tree?**  
Height Θ(n); BK-tree from sorted input.

**Q108. Fan-out?**  
Number of children. Trie: up to alphabet. BK: up to max observed distance (word-length scale).

**Q109. Build time vs query time?**  
Trie ~40ms, BK ~150ms at 88K (from RESULTS). Build once per process.

**Q110. What is CMake `add_test`?**  
Registers binaries with CTest.

---

### L. Questions you should drill on paper (answers included)

**Q111. Compute `dp` for `cat` vs `cut`.**  
Distance 1 (substitution). Table: you should fill 4×4 including empty prefixes.

**Q112. Prefix `ca` in `{cat, car, dog, cab}`.**  
`cat, car, cab`.

**Q113. BK insert `cat`, then `bat` (d=1), then `cats` (d=1 from cat — child 1 exists → go to `bat`, d(cats,bat)=?).**  
Work it by hand: `cats` vs `bat` = 2 (example of recurse on occupied edge). Do this until insert is automatic.

**Q114. If `d(q,p)=3`, `R=1`, which child keys?**  
`[2,4]`.

**Q115. Hash lookup misses, fuzzy finds `apple` at dist 1. Is that consistent?**  
Yes. Exact and fuzzy are different predicates.

---

## How to use Phase 0 (placement)

1. Explain the **three-query / three-structure** diagram without notes.  
2. Write Levenshtein DP and two-row optimization from memory.  
3. State BK prune interval and why sentinel must not be used.  
4. Contrast hash vs trie vs sorted `lower_bound` honestly.  
5. Quote **qualitative** RESULTS (hash >> linear; trie ~flat in n; BK ~2× not 100×).  
6. C++: `unique_ptr`, ctype UB, shuffle seed.

Do **not** start `trie.cpp` / `bktree.cpp` until those six are easy.

---

When this is solid, reply **proceed to Phase 1** and we will read the must-read sources in importance order (edit distance → trie → BK-tree), with file-by-file walkthrough and SDE questions tied to the actual code.
