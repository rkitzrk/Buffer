# 🎯 Behavioral Interview Prep — LexiCore (C++20)

You built **LexiCore** — a high-performance Fuzzy Search & Autocomplete Retrieval Engine in modern C++20 using pure STL (zero external dependencies).

Use the **STAR format**: Situation → Task → Action → Result
**Speaking guideline**: 90 seconds to 2 minutes per answer. Spend ~20% on S+T, ~60% on Action (what *you* did), ~20% on a quantified Result.
**How to practice**: say each story out loud 3 times, close the doc, say it from memory, record yourself, fix the parts that sound scripted.

> ✅ This version merges both earlier docs into **8 stories + extras** and has been **checked against your `RESULTS.md`** (all benchmark numbers below match it). Details that `RESULTS.md` can't confirm (code-level facts, debugging timeline) are listed in **"Verify Before You Interview"** at the end.

---

# PART 1: The Core 8 Stories

*Each story is tagged with the Amazon Leadership Principles (LPs) it can answer. One story usually covers 3–4 LPs — pick the angle the question asks for.*

---

## 🏗️ Your Role — Quick Intro

*Use when asked: "What did YOU specifically work on?"*

> "I designed and built **LexiCore** from scratch in C++20 — a multi-strategy search engine that answers exact, prefix, and typo-tolerant queries over an 88,344-word dictionary. I matched each query type to the data structure suited to it: an `unordered_set` for O(1) exact lookup, a Trie for O(k) autocomplete, and a BK-Tree with triangle-inequality pruning for fuzzy search. I also built a correctness oracle to verify the BK-Tree never misses valid results, a two-row O(m)-memory Levenshtein implementation, and a benchmark suite showing speedups of up to 271× over linear scan."

---

## ⭐ Story 1 — The Sentinel Early-Exit Bug

**LPs:** Dive Deep · Ownership · Are Right, A Lot · Earn Trust (admitting a mistake)
**Question type:** "Tell me about a subtle bug you found and fixed." / "Tell me about a mistake." / "Tell me about a time you went deep to find a root cause."

- **S — Situation:** While optimizing BK-Tree fuzzy search, I swapped the full edit-distance call for my optimized `editDistanceBounded()`, which exits early and returns a sentinel (`maxDist + 1`) once the distance exceeds the threshold. Nothing crashed, the code compiled, and small unit tests passed — but my correctness oracle showed the BK-Tree returning *fewer* results than the brute-force linear scan.
- **T — Task:** Find out why an early-exit optimization was silently causing false negatives.
- **A — Action:** I traced the traversal node by node. BK-Tree pruning relies on the triangle inequality: a child edge labeled `w` can only lead to a match if `w ∈ [d − maxDist, d + maxDist]`, where `d` is the distance from the query to the *current node*. Concrete example with `maxDist = 2`: if the true distance to a node is 7, the bounded function returns the sentinel 3. The window becomes `[1, 5]` instead of the correct `[5, 9]` — so a child on edge 6, which can hold a word within distance 2 of the query, is skipped. The sentinel was masquerading as a real distance. The fix: use the true `editDistance()` for `d` (pruning window), and use bounded only where a yes/no threshold check is enough. I documented the trap in the header and README.
- **R — Result:** The oracle now agrees with linear scan on all 800 query × threshold combinations. The README carries a permanent warning so nobody "optimizes" it back. **Lesson I now apply:** a sentinel value is not a measurement — never feed it into math that assumes a real value.

> 💡 **If they ask: "Show me the fix."**
>
> ```cpp
> // src/bktree.cpp
> // MUST use true editDistance(), NOT editDistanceBounded().
> // Bounded returns a sentinel (maxDist + 1) on bail-out, which corrupts the
> // pruning interval [d - maxDist, d + maxDist] and silently skips valid subtrees.
> int d = editDistance(query, node->word);
>
> if (d <= maxDistance) {
>     results.emplace_back(node->word, d);
> }
>
> int low  = d - maxDistance;
> int high = d + maxDistance;
> for (const auto& [childDist, childNode] : node->children) {
>     if (childDist >= low && childDist <= high) {
>         searchImpl(childNode.get(), query, maxDistance, results);
>     }
> }
> ```

---

## ⭐ Story 2 — The Degenerate Tree & Deterministic Shuffle

**LPs:** Invent and Simplify · Dive Deep · Bias for Action
**Question type:** "Tell me about a time you optimized performance." / "A simple solution to a complex problem." / "Diagnosing unexpected behavior."

- **S — Situation:** On the full 88,344-word dictionary, BK-Tree fuzzy search was barely beating a brute-force scan — far from the sub-linear behavior the theory promised.
- **T — Task:** Find out why the tree wasn't pruning and fix it without adding heavy machinery.
- **A — Action:** I wrote diagnostics — `getMaxDepth()` and node counts — and found the tree depth was over 100. `/usr/share/dict/words` is alphabetically sorted, so consecutive words (`apple`, `application`, `apply`) are within tiny edit distances of each other and the tree grew into long skewed chains instead of branching. Research suggested rebalancing schemes, but I wanted the simplest thing that worked: shuffle the words before insertion with a **fixed seed** (`std::mt19937 rng(42)`). One `std::shuffle` call, and the seed makes tree shape and benchmarks fully reproducible.
- **R — Result:** Max depth dropped from 100+ to **19** at 88K words, and fuzzy search reached a consistent **2.4× speedup** over linear scan (5.7 ms vs 13.8 ms per query). The fix was one function and one line — no balancing code.

> 💡 **If they ask: "Why a fixed seed?"**
>
> ```cpp
> // src/dictionary.cpp
> std::vector<std::string> copy = words_;
> std::mt19937 rng(42);  // deterministic: same tree topology on every run
> std::shuffle(copy.begin(), copy.end(), rng);
> ```
> *"A random seed would give a different tree each run, making benchmarks irreproducible and bugs hard to replay."*

---

## ⭐ Story 3 — The Correctness Oracle (Testing Rigor)

**LPs:** Insist on the Highest Standards · Ownership · Dive Deep
**Question type:** "How do you test complex algorithms?" / "A time you refused to accept 'good enough'." / "Ensuring quality."

- **S — Situation:** Pruning bugs in metric trees don't crash — they silently drop results. Unit tests on a handful of toy words give false confidence; "it seems to work" isn't enough for an algorithm whose correctness rests on a mathematical argument.
- **T — Task:** Build a test that *guarantees* BK-Tree pruning never produces false negatives.
- **A — Action:** In `tests/test_correctness.cpp` I built a **differential oracle**: an exhaustive linear scan, obviously correct, as ground truth, compared against the BK-Tree on **500 dictionary words × 200 randomized queries × 4 thresholds (0–3) = 800 query × threshold combinations**. Because the two return results in different orders, I wrote `resultSetsMatch()` — sort both result lists with the same comparator (word, then distance) and compare with `==`, which makes it an order-independent set-equality check in about a dozen lines. Around it: property-based tests of the metric axioms (symmetry, triangle inequality over random pairs), and everything runs under AddressSanitizer + UBSan. When the oracle failed, I fixed the code — I didn't loosen the test.
- **R — Result:** The oracle caught the sentinel bug (Story 1) before it could ship. Final state: zero mismatches across all combinations, zero sanitizer errors. Three testing tiers — unit, property-based, oracle — each catching a different class of bug.

---

## ⭐ Story 4 — The Benchmark Sampling-Bias Trap

**LPs:** Dive Deep · Insist on the Highest Standards · Are Right, A Lot
**Question type:** "Tell me about a time data misled you." / "How do you design benchmarks?" / "A time you questioned your own results."

- **S — Situation:** Benchmarking at 1K, 10K, and 88K words, the smaller subsets gave odd results — prefix hit rates were unnaturally high and fuzzy pruning behaved erratically compared to the full dictionary.
- **T — Task:** Find the cause and make the methodology honest enough that I could stand behind every number.
- **A — Action:** I inspected how subsets were built and found I'd sliced the *first N words* of an alphabetically sorted file — so my 1K "dictionary" was essentially words starting with "a". I replaced it with a seeded random sample (`getSubset(n, seed)`) spanning all 26 letters and word lengths. I also locked down the rest of the method: **same query set for both strategies being compared**, a warm-up pass before timing, seeded query generation, 1,000 queries per trial, **5 trials per configuration with the median reported**, p50/p95/p99 latency, and `volatile` sinks so the compiler can't optimize measured work away.
- **R — Result:** Results became representative and reproducible. They also exposed real structure: Trie autocomplete latency is flat — **4.51 µs at 10K vs 4.39 µs at 88K** — the O(k) property shown empirically, while linear scan degrades from 31.78 µs to 214.7 µs (≈6.75×). The Trie's advantage therefore *grows* with scale: ~1.9× at 1K, ~7× at 10K, **49×** at 88K. Hash lookup shows the same pattern: 7× → 53× → **271×**.

---

## ⭐ Story 5 — Learning BK-Trees from First Principles

**LPs:** Learn and Be Curious · Dive Deep · Deliver Results
**Question type:** "Tell me about a time you learned something quickly." / "Something unfamiliar you had to master."

- **S — Situation:** I needed typo-tolerant search. I knew edit distance, but I'd never seen a metric-space tree. My competitive-programming background had covered Tries, segment trees, and graph algorithms — not this.
- **T — Task:** Understand Burkhard–Keller trees and their pruning math well enough to implement a production-quality version in days, not just paste pseudocode.
- **A — Action:** I started from the math instead of the code: the four metric axioms (non-negativity, identity, symmetry, triangle inequality). **I wrote property tests proving Levenshtein distance satisfies them before writing any tree code.** Then I built incrementally — insertion, recursive search, and only then pruning — testing each layer.
- **R — Result:** A tested, benchmarked implementation, and — more importantly — the mathematical clarity that let me spot the sentinel bug (Story 1) quickly. That depth also lets me answer "what would you do differently?" immediately: SymSpell precomputes single-deletion variants into a hash map and avoids per-node edit distance entirely. Learning first principles first saved days of debugging.

---

## ⭐ Story 6 — Trie Node Design: Making and Defending a Trade-off

**LPs:** Have Backbone; Disagree and Commit · Are Right, A Lot · Think Big
**Question type:** "A technical decision you stood by." / "A decision with incomplete information." / "A time you made a different choice than the obvious one."

- **S — Situation:** Designing the Trie, I had to choose child storage: `std::array<std::unique_ptr<TrieNode>, 26>` (O(1) direct index, cache-friendly, but 26 slots per node) or `std::unordered_map<char, std::unique_ptr<TrieNode>>` (allocates only for existing children, but hashing overhead).
- **T — Task:** Decide before I had profiling data, and be able to defend the decision.
- **A — Action:** I reasoned from the structure of the data. The full dictionary produces **185,264 Trie nodes**, and most are deep — near-leaf nodes with 0–2 children. An array costs 26 × 8 = 208 bytes of pointers per node (~38 MB total, mostly null); a map pays only for children that exist. I chose `unordered_map`, but I was explicit that **array is faster per lookup** and would be the right call for a latency-critical, dense-alphabet service. I recorded both options and the reasoning in the README so anyone who disagrees can see the exact trade-off — and swap it in a single class.
- **R — Result:** Autocomplete runs at **4.39 µs** (~227K queries/sec) at 88K words — more than fast enough — with memory proportional to actual children. And because the decision is documented rather than buried, it's a decision that can be revisited with data instead of argued from opinion.

> 💡 **Follow-up trap: "Doesn't your normalization strip non-letters, so the 26-array would work fine?"**
> *"Yes — after normalization the alphabet is a–z, so the flexibility argument is secondary. The main reason is memory on sparse nodes. Flexibility only matters if I later keep apostrophes or hyphens. If memory weren't a concern I'd benchmark the array version — I'd expect it to be faster per lookup."*

---

## ⭐ Story 7 — When the Result Wasn't What I Expected (Only 2.4×)

**LPs:** Earn Trust · Learn and Be Curious · Are Right, A Lot · Deliver Results
**Question type:** "Tell me about a mistaken assumption." / "Adapting to unexpected results." / "Being honest about limitations."

- **S — Situation:** After hash lookup came out **271×** faster and the Trie **49×** faster than linear scan at 88K words, I assumed the BK-Tree would show a similarly dramatic win. It came out at only **2.4×** (5.7 ms vs 13.8 ms per query at maxDist = 2). And unlike the other two, its advantage didn't grow with scale — it stayed flat at roughly 2.0×, 2.5×, and 2.4× at 1K, 10K, and 88K words.
- **T — Task:** Decide whether this was a bug, a bad benchmark, or a real property of the problem — and report it honestly instead of inflating it.
- **A — Action:** I ruled out a bug first (the oracle confirmed the results were correct — Story 3) and a bad benchmark (same seeded queries, warm-up, median of 5 trials — Story 4). Then I reasoned about why. For the exact and prefix paths, the data structure removes almost all the work. For fuzzy search, the BK-Tree still has to compute a full edit distance at every node it visits, and edit-distance space has high intrinsic dimensionality, so at radius 2 the pruning rings overlap and a large share of the tree is still visited. The cost is dominated by edit-distance computation, not by traversal — which is why the speedup is bounded no matter how large the dictionary gets. I documented this in the results and named the production fix: SymSpell-style deletion indexing, which avoids per-node edit distance altogether.
- **R — Result:** An honest, defensible evaluation instead of a cherry-picked one, and a sharper conclusion: *the right structure depends on the query type and scale, and fuzzy search is fundamentally harder than exact or prefix search.* Even 2.4× matters at scale: 13.8 ms → 5.7 ms per query. If I had more time, I'd add a counter for edit-distance calls per query to measure exactly how much pruning achieves, rather than inferring it.

> 💡 **If they ask: "How do you know edit distance is the bottleneck?"**
> *"It's my explanation of why the speedup is flat and modest, and it's consistent with the numbers — but I haven't instrumented distance-call counts yet. That's the first thing I'd add to confirm it."* (Honest and shows rigor.)

---

## ⭐ Story 8 — Zero Dependencies, Ruthless Scoping, Shipped

**LPs:** Frugality · Deliver Results · Bias for Action · Think Big · Ownership
**Question type:** "Tell me about working under constraints." / "Delivering under a deadline." / "Prioritization." / "Thinking beyond the immediate problem."

- **S — Situation:** I set a self-imposed deadline to finish LexiCore in time to discuss it in internship interviews. The scope was ambitious: three search structures, a CLI, a test suite, a benchmark harness, and docs — and I imposed one more constraint: **STL only**, no Boost, Google Benchmark, Catch2, or CLI libraries.
- **T — Task:** Ship a complete, defensible system without dependencies, by prioritizing what interviewers and users would actually care about.
- **A — Action:** I ordered the work by value: (1) core algorithms — edit distance, Trie, BK-Tree; (2) a working CLI so the project is demonstrable live; (3) correctness tests (oracle + property); (4) benchmarks that quantify the trade-offs; (5) documentation last. I built the supporting pieces myself — a `std::chrono::steady_clock` timing harness, percentile calculators, a seeded RNG for repeatable queries, an interactive REPL — and used `std::unique_ptr` throughout so ownership is RAII-clean, with **no manual `new`/`delete` anywhere**. I deliberately cut frequency-based ranking, a radix trie, SymSpell, and a REST layer — and recorded each cut in "Known Trade-offs" with a production path. I kept the architecture modular (Dictionary, Trie, BKTree, Ranking) so those can be added without rewrites.
- **R — Result:** A working 88K-word CLI with exact, prefix, and fuzzy search; a test suite under ASan/UBSan with zero errors; benchmark results with documented methodology; and a project that builds anywhere with CMake and a C++20 compiler — no dependency management at all. The cuts are documented decisions, not accidents.

---

# PART 2: Extras

---

## 🎤 "Tell Me About Yourself" (60 sec)

> "I'm a [year] CS student and a competitive programmer — I reached Specialist on Codeforces — who enjoys turning algorithmic knowledge into real engineering. My main project is **LexiCore**, a fuzzy search and autocomplete engine I built in C++20 with no external libraries. It answers exact, prefix, and typo-tolerant queries across 88,000 words, using a hash set, a Trie, and a BK-Tree respectively. I benchmarked each against linear-scan baselines, and I verified the BK-Tree with a correctness oracle against brute force — which caught a subtle bug where an early-exit optimization silently broke the tree's pruning. Building something from scratch, measuring whether my decisions were right, and being honest when the numbers surprised me — that's the kind of work I want to do at Amazon."

## 🎤 90-Second Project Pitch ("Tell me about a project")

> "LexiCore is a search engine I built in C++20 using only the STL. The problem: developers often grab a generic search library without knowing which data structure fits which query. I wanted to show it empirically.
>
> I built three strategies — a hash set for O(1) exact match, a Trie for O(k) prefix search, and a BK-Tree for typo-tolerant search using triangle-inequality pruning — each benchmarked against a linear scan on up to 88,344 words.
>
> Results: hash lookup is 271× faster, the Trie is 49× faster, and its latency stays flat from 10K to 88K words — O(k) in practice. The BK-Tree is only 2.4× faster, which taught me edit-distance computation, not traversal, is the bottleneck.
>
> The most interesting engineering detail: the BK-Tree must use the *true* edit distance for pruning, because the bounded variant returns a sentinel that silently shrinks the search window. I only found it because I'd built a correctness oracle comparing the tree against brute force on 800 combinations. The project has unit, property-based, and oracle tests, all under AddressSanitizer and UBSan."

---

## 🗺️ Leadership Principle → Story Map (all 16)

| Leadership Principle | Best story | Backup / short answer |
|---|---|---|
| Customer Obsession | *Short answer below* | Deterministic ranking + `…and N more` CLI output |
| Ownership | Story 1 | Story 8 |
| Invent and Simplify | Story 2 | Story 3 (`resultSetsMatch` — sort-and-compare) |
| Are Right, A Lot | Story 6 | Story 1, Story 4 |
| Learn and Be Curious | Story 5 | Story 7 |
| Hire and Develop the Best | *No LexiCore story — prepare from another experience (mentoring, team, club)* | — |
| Insist on the Highest Standards | Story 3 | Story 4 |
| Think Big | Story 8 | Story 7 (SymSpell path) |
| Bias for Action | Story 8 | Dual-structure Dictionary decision (short answer below) |
| Frugality | Story 8 | — |
| Earn Trust | Story 7 | Story 1 (documenting the mistake) |
| Dive Deep | Story 1 | Story 4 |
| Have Backbone; Disagree and Commit | Story 6 | — |
| Deliver Results | Story 8 | Story 5 |
| Strive to be Earth's Best Employer | *No LexiCore story — prepare separately* | — |
| Success and Scale Bring Broad Responsibility | *No LexiCore story — prepare separately* | — |

> Amazon now lists **16** Leadership Principles (two were added in 2021). The old guide titled itself "14" and only covered 13. For an SDE intern loop the heaviest weight is usually on the first ~12 above; have at least one non-LexiCore story ready for the three marked "prepare separately".

### 30-Second Short Answers (for LPs LexiCore only partly covers)

**Customer Obsession:**
> "In LexiCore's CLI, `prefix a` would match thousands of words and flood the terminal, so I show the top results and print how many more exist. For fuzzy results, BK-Tree traversal order is non-deterministic, so I built a separate ranking module — sort by edit distance, then lexicographically — so the same query always returns the same order. Small things, but they're what make a tool feel trustworthy."

**Bias for Action:**
> "For the Dictionary class I needed both O(1) exact lookup and ordered indexed access. No single STL container does both, so I made the call in minutes: keep a `vector` and an `unordered_set`, accept the extra memory, document the trade-off, and revisit only if profiling says it matters. Reversible decision, so I moved."

**Think Big:**
> "I kept Dictionary, Trie, BKTree, and Ranking as independent components so I could describe a production path: swap the Trie for a radix trie, BK-Tree for SymSpell, and add frequency-based ranking — without rewriting the system."

---

## 📊 Numbers Cheat Sheet (from your RESULTS.md — memorize these)

**Setup:** g++ 13.3.0, `-O2`, C++20, Ubuntu 24.04. 88,344 unique words. 5 trials per config (median), 1 warm-up pass, 1,000 seeded queries per trial, same queries for all strategies. Subsets are seeded random samples. All per-query times are **averages in µs**.

| Query type | 1K words | 10K words | 88,344 words | Speedup at 88K |
|---|---|---|---|---|
| **Exact:** linear → hash | 0.86 → 0.12 µs (7×) | 12.67 → 0.24 µs (53×) | **121.80 → 0.45 µs** | **271×** |
| **Prefix:** linear → Trie | 3.64 → 1.94 µs (1.9×) | 31.78 → 4.51 µs (7×) | **214.71 → 4.39 µs** | **49×** |
| **Fuzzy (maxDist=2):** linear → BK-tree | 163.58 → 82.71 µs (2.0×) | 1,639.65 → 663.81 µs (2.5×) | **13,818 → 5,729 µs (13.8 → 5.7 ms)** | **2.4×** |

| Structure | 1K | 10K | 88,344 |
|---|---|---|---|
| Trie nodes | 5,815 | 42,994 | **185,264** |
| BK-tree nodes | 1,000 | 10,000 | **88,344** |
| BK-tree max depth (after shuffle) | 9 | 12 | **19** |
| Trie build (ms) | 1.09 | 9.48 | 39.42 |
| BK-tree build (ms) | 1.00 | 12.21 | 150.10 |

| Other | Value |
|---|---|
| Throughput at 88K | Hash 2.23M q/s · Trie 228K q/s · BK-tree 175 q/s · linear fuzzy 72 q/s |
| Tail latency, Trie at 88K | p50 4.55 µs · p95 6.90 µs · p99 9.45 µs |
| Trie latency, 10K → 88K | 4.51 → 4.39 µs (flat — O(k)); linear prefix scan 31.78 → 214.71 µs (≈6.75×) |
| Oracle test | 500 words × 200 queries × 4 thresholds = 800 comparisons |
| Array-per-node cost | 26 × 8 B = 208 B/node (~38 MB at 185K nodes) |

> 💡 **One-sentence takeaway:** *"Exact and prefix speedups grow with dictionary size — 7→271× and 1.9→49× — while fuzzy stays around 2–2.5× at every size, because edit-distance computation dominates."*

---

## ❓ Quick Q&A Cheat Sheet

| Question | Key Answer |
|---|---|
| **"Why C++20?"** | Direct control over memory layout and ownership, zero-overhead abstractions, no garbage-collector pauses polluting benchmarks, and the STL had every structure I needed. It also forced me to learn RAII, const-correctness, and move semantics. |
| **"Why not use a Trie for fuzzy search?"** | A Trie can do fuzzy search by branching DFS, but branching explodes for arbitrary edits. A BK-Tree exploits the triangle inequality to skip whole distance rings. |
| **"How does BK-Tree pruning work?"** | If the query is at distance `d` from a node, any word within `k` of the query must sit on a child edge `w` with `d − k ≤ w ≤ d + k`. Every other branch is skipped without computing a distance. |
| **"Biggest mistake?"** | Using `editDistanceBounded()` for pruning. It returns a sentinel (`maxDist + 1`), not a real distance, so the pruning window was wrong and valid subtrees were skipped. Caught by the oracle, fixed, documented. |
| **"Hardest part?"** | Realizing the sentinel bug: it's invisible until you see it, because the sentinel masquerades as a real distance. Once seen, obvious — which is what makes it dangerous. |
| **"How did you test BK-Tree correctness?"** | Three tiers: unit tests (known cases), property tests (metric axioms), and an oracle test against brute-force linear scan on 800 combinations — plus ASan/UBSan on everything. |
| **"How did you measure performance?"** | `std::chrono::steady_clock` timing, 1,000 seeded queries per run, same query set for both strategies, warm-up pass, p50/p95/p99, measured at 1K/10K/88K words. |
| **"Why shuffle before BK-Tree insertion?"** | Sorted input makes near-identical words adjacent → degenerate chains (depth 100+). Seeded shuffle gives depth 19 and a reproducible tree. |
| **"How did you handle memory safety?"** | All tree ownership is `std::unique_ptr` — RAII frees whole subtrees automatically; no manual `new`/`delete`; ASan + UBSan clean. |
| **"How is Levenshtein memory-optimized?"** | Row *i* depends only on row *i−1*, so two rolling `vector<int>` rows (`prev`, `curr`) replace the full table: O(n·m) → O(m) space. `std::swap` on vectors exchanges internal pointers (O(1)), no element copying. A bounded variant adds a diagonal band and early exit — but only for yes/no threshold checks. |
| **"Why `static_cast<unsigned char>` before `isalpha`/`tolower`?"** | If `char` is signed, bytes above 127 become negative ints; passing those to `isalpha` is undefined behavior. The cast makes it well-defined. |
| **"What would you improve next?"** | (1) SymSpell / deletion-neighborhood index for sub-millisecond fuzzy search; (2) radix (Patricia) trie to cut single-child chains; (3) frequency-based ranking so "the" outranks "thee". |

---

## ⚡ Quick One-Liners for Follow-ups

| Follow-up | Crisp spoken answer |
|---|---|
| **"What would you do differently starting over?"** | "I'd build a SymSpell index alongside the BK-Tree to compare precomputation against metric-tree pruning, and add word-frequency ranking." |
| **"Biggest takeaway?"** | "Structure choice matters differently by query type: 271× for exact, 49× for prefix, but only 2.4× for fuzzy — because edit-distance computation, not traversal, dominates. Measuring beats assuming." |
| **"How did you verify no memory leaks?"** | "Ownership is `std::unique_ptr` with RAII, and the whole suite runs under AddressSanitizer and UBSan with zero errors." |
| **"How do you handle ambiguous requirements?"** | "I pick an explicit policy, test it, and document it — e.g., normalization lowercases, strips non-letters, and dedupes on first occurrence." |
| **"How did competitive programming help — and what did you unlearn?"** | "It gave me algorithmic fluency: DP and Trie traversal came easily. I had to unlearn throwaway-code habits — global arrays, raw pointers, terse names — and replace them with RAII, modular headers, tests, and documentation." |
| **"Describe yourself in one sentence."** | "A competitive programmer with strong algorithmic intuition who builds disciplined, production-minded systems in modern C++." |

---
