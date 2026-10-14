# 📚 LexiCore Study Guide — Phases 2 + 3 + 4
### *Glue Layer · Performance Engineering · Build, Testing & Docs*

---

## 🗺️ Files in This Document

### Phase 2 — Glue Layer (The "connective tissue")
| Order | File | Role |
|---|---|---|
| 1 | [`include/dictionary.hpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/include/dictionary.hpp) | Contract for loading & normalizing words |
| 2 | [`src/dictionary.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/src/dictionary.cpp) | File I/O, normalization, deduplication, sampling |
| 3 | [`include/ranking.hpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/include/ranking.hpp) | Contract for sorting fuzzy results |
| 4 | [`src/ranking.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/src/ranking.cpp) | Comparator-based sort |
| 5 | [`app/main.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/app/main.cpp) | CLI entry point, query dispatch |

### Phase 3 — Performance Engineering
| Order | File | Role |
|---|---|---|
| 6 | [`include/benchmark.hpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/include/benchmark.hpp) | BenchmarkConfig struct |
| 7 | [`src/benchmark.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/src/benchmark.cpp) | Timing harness, query generation, paired comparisons |
| 8 | [`RESULTS.md`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/RESULTS.md) | Empirical results & observations |

### Phase 4 — Build, Testing & Docs
| Order | File | Role |
|---|---|---|
| 9  | [`CMakeLists.txt`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/CMakeLists.txt) | Build system configuration |
| 10 | [`tests/test_edit_distance.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/tests/test_edit_distance.cpp) | Unit + property-based tests |
| 11 | [`tests/test_trie.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/tests/test_trie.cpp) | Trie edge-case tests |
| 12 | [`tests/test_bktree.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/tests/test_bktree.cpp) | BK-tree unit tests |
| 13 | [`tests/test_correctness.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/tests/test_correctness.cpp) | BK-tree vs linear oracle |
| 14 | [`README.md`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/README.md) | Architecture, design decisions, complexity table |

---

---

# 📦 Phase 2 — Glue Layer

---

## File 1+2: `dictionary.hpp` / `dictionary.cpp`

### What it does in one line
Reads `words.txt`, normalizes every word (lowercase, strip non-alpha), deduplicates, and exposes the word list in three ways: ordered vector, hash set, and random subsets.

---

### Header Walkthrough

```cpp
class Dictionary {
public:
    bool loadFromFile(const std::string& filepath);           // returns false on failure
    const std::vector<std::string>& getWords() const;         // ordered, deduplicated list
    const std::unordered_set<std::string>& getWordSet() const;// O(1) exact membership
    size_t size() const;

    std::vector<std::string> getSubset(size_t n, unsigned seed) const;    // seeded sample
    std::vector<std::string> getShuffledWords(unsigned seed) const;        // full shuffle

private:
    static std::string normalize(const std::string& raw);    // static = no 'this' needed
    std::vector<std::string>   words_;     // load-order, deduplicated
    std::unordered_set<std::string> wordSet_; // for O(1) lookup
};
```

**Key design points:**
- Two parallel structures (`words_` and `wordSet_`) serve different purposes — vector for ordered iteration (Trie build, benchmark subsets), set for O(1) exact lookup (CLI `exact` command).
- `normalize` is `static` — it doesn't need or use any class state (`this`). It's a pure function: input string → output string.
- `getWords()` and `getWordSet()` return `const&` — zero-copy access. Callers get a read-only view of the internal data.

---

### Implementation Walkthrough

#### `normalize()` — String Sanitization
```cpp
static std::string Dictionary::normalize(const std::string& raw) {
    std::string result;
    result.reserve(raw.size());           // pre-allocate to avoid reallocations
    for (char c : raw) {
        if (std::isalpha(static_cast<unsigned char>(c))) {
            result += static_cast<char>(
                std::tolower(static_cast<unsigned char>(c))
            );
        }
        // non-alpha characters are silently dropped
    }
    return result;
}
```

**Why `result.reserve(raw.size())`?**
Worst case, every character is alphabetic → result has same length as raw. `reserve` pre-allocates that capacity upfront, avoiding repeated heap reallocations as the string grows. This is O(1) extra allocation instead of O(log n) amortized.

**Why `static_cast<unsigned char>`?**
`isalpha()` and `tolower()` take `int` but are undefined if passed a negative value. On platforms where `char` is signed, characters above ASCII 127 (like accented letters) are negative when widened to int. Casting to `unsigned char` guarantees a value in [0,255], which is always safe.

**What 'aple → apple' normalization doesn't do:**
It strips non-alpha but does NOT fix typos. Normalization is about cleaning raw file input, not spell-correction. That's the BK-tree's job.

---

#### `loadFromFile()` — The Loading Pipeline
```cpp
bool Dictionary::loadFromFile(const std::string& filepath) {
    std::ifstream file(filepath);
    if (!file.is_open()) return false;       // caller checks this!

    std::string line;
    while (std::getline(file, line)) {
        std::string word = normalize(line);
        if (word.empty()) continue;           // blank lines, punctuation-only lines → skip

        if (wordSet_.insert(word).second) {   // .second == true if newly inserted
            words_.push_back(word);           // only add to vector if not a duplicate
        }
    }
    return true;
}
```

**The dual-insert idiom:**
`wordSet_.insert(word)` returns `std::pair<iterator, bool>`. The `.second` bool is `true` if the word was newly inserted (not a duplicate). This lets us deduplicate in a single pass without a separate `count()` check — two lookups instead of one.

**Loading pipeline visualization:**
```
words.txt (104,334 raw lines)
        │
        ▼
  std::getline()       ← reads one line at a time
        │
        ▼
  normalize()          ← lowercase + strip non-alpha
        │
        ▼
  word.empty()?  ──YES──► skip (blank/punctuation lines)
        │ NO
        ▼
  wordSet_.insert()
        │
   .second == true?  ──NO──► skip (duplicate)
        │ YES
        ▼
  words_.push_back()   ← add to ordered vector

Result: 88,344 unique words
```

---

#### `getSubset()` — Seeded Random Sampling
```cpp
std::vector<std::string> Dictionary::getSubset(size_t n, unsigned seed) const {
    std::vector<std::string> copy = words_;          // copy the full word list
    std::mt19937 rng(seed);                          // Mersenne Twister, seeded
    std::shuffle(copy.begin(), copy.end(), rng);     // random permutation
    if (n < copy.size()) copy.resize(n);             // take first n of shuffled
    return copy;
}
```

**Why not just take `words_[0..n]`?**
`words_` preserves load order, which for `/usr/share/dict/words` is alphabetical. Taking the first 1000 words would give you mostly 'a' words — an unrepresentative slice. After shuffling, the first 1000 are a random sample across the entire alphabet.

**Why seeded (not `std::random_device`)?**
`std::random_device` gives truly random seeds — different every run. Seeded `mt19937` gives the same shuffle every run. Same seed → same subset → benchmark results are reproducible and comparable across machines and time.

---

#### `getShuffledWords()` — Full Shuffle for BK-tree
```cpp
std::vector<std::string> Dictionary::getShuffledWords(unsigned seed) const {
    std::vector<std::string> copy = words_;
    std::mt19937 rng(seed);
    std::shuffle(copy.begin(), copy.end(), rng);
    return copy;
}
```

This is almost identical to `getSubset()` but skips the `resize`. Used in `main.cpp` to get the full shuffled list for BK-tree insertion. Seed 42 is hardcoded in main — the shuffle is deterministic, so the tree shape is always the same.

---

## File 3+4: `ranking.hpp` / `ranking.cpp`

### What it does in one line
Takes the raw `vector<pair<string, int>>` output from BK-tree search (word + distance, unordered) and sorts it into a deterministic order: ascending distance, then alphabetical tie-break.

---

### Header Walkthrough

```cpp
struct RankedResult {
    std::string word;
    int distance;
};

std::vector<RankedResult> rankResults(
    const std::vector<std::pair<std::string, int>>& results);
```

**Design note:** `RankedResult` is a named struct instead of using `pair<string,int>` directly. This is intentional — `result.word` and `result.distance` are far more readable than `result.first` and `result.second`. It's a small thing but reflects good API design: make intent explicit.

---

### Implementation Walkthrough

```cpp
std::vector<RankedResult> rankResults(
    const std::vector<std::pair<std::string, int>>& results) {

    std::vector<RankedResult> ranked;
    ranked.reserve(results.size());             // pre-allocate

    for (const auto& [word, dist] : results) {  // structured binding
        ranked.push_back({word, dist});          // aggregate initialization
    }

    std::sort(ranked.begin(), ranked.end(),
              [](const RankedResult& a, const RankedResult& b) {
                  if (a.distance != b.distance) {
                      return a.distance < b.distance;  // closer first
                  }
                  return a.word < b.word;              // lex tie-break
              });

    return ranked;
}
```

**The comparator logic:**
```
Fuzzy search("aple", maxDist=2) raw results (unordered):
  ("able", 2), ("apple", 1), ("ale", 1), ("ample", 2)

After rankResults():
  ("ale",   1)  ← dist 1, lex smallest
  ("apple", 1)  ← dist 1
  ("able",  2)  ← dist 2, lex smallest
  ("ample", 2)  ← dist 2
```

**Why lex tie-break?**
Without it, two words at the same distance could appear in any order — output would be non-deterministic across runs (unordered_map iteration order is unspecified). The lex tie-break makes results reproducible regardless of tree traversal order or hardware. This is documented explicitly in README: "ranking is scientifically clean."

---

## File 5: `app/main.cpp`

### What it does in one line
Loads the dictionary, builds all three index structures, then runs an interactive REPL loop dispatching commands to the appropriate structure.

---

### Architecture of `main()`

```
main()
  │
  ├─ Parse argv for dictionary path (default "data/words.txt")
  ├─ dict.loadFromFile() → 88K words
  ├─ Build wordSet  (from dict.getWordSet()  — O(1) reference, already built)
  ├─ Build Trie    (insert each word in load order)
  ├─ Build BKTree  (insert shuffled words, seed=42)
  ├─ printBanner() + printHelp()
  │
  └─ while(true) REPL loop
        ├─ std::getline(cin, line)
        ├─ parse command token from line via istringstream
        └─ dispatch:
             "exact"     → wordSet.count(word)
             "prefix"    → trie.autocomplete(prefix, limit)
             "fuzzy"     → bktree.search(query, maxDist) → rankResults()
             "benchmark" → runBenchmark(dict)
             "help"      → printHelp()
             "quit/exit" → break
```

---

### Key Code Sections

#### Building the Index Structures
```cpp
// wordSet: already built inside Dictionary — just get a reference
const auto& wordSet = dict.getWordSet();  // O(1), no copy

// Trie: insert in load order (alphabetical for this dict)
Trie trie;
for (const auto& w : dict.getWords()) {
    trie.insert(w);
}

// BK-tree: MUST be shuffled first
BKTree bktree;
auto shuffled = dict.getShuffledWords(42);  // fixed seed for reproducibility
for (const auto& w : shuffled) {
    bktree.insert(w);
}
```

**Why 3 separate structures?**
Each answers a different query type. `wordSet` doesn't help with prefix or fuzzy. Trie doesn't help with fuzzy. BK-tree doesn't help with exact or prefix efficiently. You pay the memory cost once at startup, then get O(1)/O(k)/practical performance at query time.

---

#### The REPL Loop Pattern
```cpp
std::string line;
while (true) {
    std::cout << "LexiCore> ";
    if (!std::getline(std::cin, line)) break;  // EOF (Ctrl+D) → exit

    std::istringstream iss(line);
    std::string command;
    if (!(iss >> command)) continue;            // empty line → loop again

    // dispatch by command string...
}
```

**`std::istringstream`** parses `line` as a stream. `iss >> command` reads the first whitespace-delimited token. Then `iss >> word`, `iss >> limit`, etc., read subsequent tokens. If a token isn't present, `iss >> x` returns false/falsy, which is how missing arguments are detected.

---

#### The Prefix Command — Subtle Trick
```cpp
if (command == "prefix") {
    size_t limit = 20;
    iss >> limit;   // may or may not be present

    // Fetch more than limit to know total count
    auto results = trie.autocomplete(prefix, limit + 1000);
    size_t total = results.size();
    size_t shown = std::min(total, limit);

    for (size_t i = 0; i < shown; ++i) cout << "  " << results[i] << "\n";

    if (total > limit) {
        cout << "  ...and " << (total - limit) << " more\n";
    }
}
```

The trie is asked for `limit + 1000` results so we know the real total. Then only `limit` are displayed but the "...and N more" message shows the actual count. This is a UI decision: let the user know there are more results without flooding the terminal.

---

#### Error Handling Philosophy
```cpp
if (!dict.loadFromFile(dictPath)) {
    std::cerr << "Error: Could not open dictionary file: " << dictPath << "\n";
    return 1;   // non-zero = failure
}
if (dict.size() == 0) {
    std::cerr << "Error: Dictionary is empty after loading.\n";
    return 1;
}
```

Two separate error cases: file not found vs file opened but all lines were blank/invalid. Checking both gives precise error messages instead of a cryptic crash later when the structures are empty.

---

#### The `printBanner()` Alignment Trick
```cpp
std::cout << "║  Dictionary: " << wordCount << " words loaded"
          << std::string(std::max(0, 6 - (int)std::to_string(wordCount).size()), ' ')
          << "     ║\n";
```
Pads with spaces so the box border `║` always aligns regardless of whether wordCount is 1000 (4 digits) or 88344 (5 digits). A small but real attention to output quality.

---

---

# ⚡ Phase 3 — Performance Engineering

---

## File 6: `benchmark.hpp`

```cpp
struct BenchmarkConfig {
    std::vector<size_t> dictSizes = {1000, 10000, 0};  // 0 = full
    size_t numQueries  = 1000;
    size_t numTrials   = 5;          // median of 5 runs
    int    fuzzyMaxDistance = 2;
    unsigned seed = 42;
};

void runBenchmark(const Dictionary& dict, const BenchmarkConfig& config = {});
```

The `= {}` default argument means callers can call `runBenchmark(dict)` with zero configuration — it uses sensible defaults. Power users can override specific fields: `runBenchmark(dict, {.numQueries=500})` (C++20 designated initializer syntax).

---

## File 7: `src/benchmark.cpp`

### Architecture Overview

```
runBenchmark(dict, config)
   │
   └─ for each dictSize in {1K, 10K, 88K}:
         │
         ├─ Get word subset (seeded random sample)
         ├─ Generate 3 query sets (same across all strategies in this run):
         │     exactQueries   ← sample from subset
         │     prefixQueries  ← prefixes of random words (length 1-5)
         │     fuzzyQueries   ← mutated words (substitution/insertion/deletion)
         │
         ├─ Build structures: hashSet, Trie, BKTree (timed)
         │
         ├─ Warm-up pass (1 full run, untimed)
         │
         └─ Paired comparisons (each timed with timeQueries()):
               [Exact]  linear scan  vs  hash lookup
               [Prefix] linear scan  vs  trie autocomplete
               [Fuzzy]  linear scan  vs  BK-tree search
```

---

### `mutateWord()` — Generating Realistic Fuzzy Queries

```cpp
std::string mutateWord(const std::string& word, std::mt19937& rng) {
    std::uniform_int_distribution<int> opDist(0, 2);  // pick operation
    int op = opDist(rng);
    size_t pos = posDist(rng);   // random position

    switch (op) {
        case 0: result[pos] = char(charDist(rng));          // substitution
        case 1: result.insert(pos, 1, char(charDist(rng))); // insertion
        case 2: if (result.size() > 1) result.erase(pos,1); // deletion
    }
}
```

This simulates a real typo — exactly the operations edit distance measures. Feeding these mutated words as queries ensures the fuzzy benchmark tests the actual use case (typo-tolerant search), not arbitrary strings.

---

### `generatePrefixQueries()` — Why Length 1–5?

```cpp
std::uniform_int_distribution<size_t> lenDist(1, std::min(word.size(), size_t(5)));
queries.push_back(word.substr(0, lenDist(rng)));
```

Short prefixes (1-5 chars) are realistic — a user typing "app" is already 3 characters in. Also, short prefixes produce many results (broad queries), which stress-tests autocomplete more than a 10-character prefix that matches one word. The benchmark is designed to reveal the structures under realistic load.

---

### `TimingResult` Struct — What's Measured

```cpp
struct TimingResult {
    double totalMs;       // wall-clock time for all N queries
    double avgUs;         // totalMs * 1000 / N  (microseconds per query)
    double queriesPerSec; // N / (totalMs / 1000)
    double p50Us;         // median latency
    double p95Us;         // 95th percentile
    double p99Us;         // 99th percentile (tail latency)
};
```

---

### `timeQueries()` — The Timing Harness

```cpp
template <typename Func>
TimingResult timeQueries(const std::vector<std::string>& queries, Func&& func) {
    std::vector<double> latencies;
    latencies.reserve(queries.size());

    auto totalStart = std::chrono::high_resolution_clock::now();

    for (const auto& q : queries) {
        auto start = std::chrono::high_resolution_clock::now();
        func(q);                   // ← the actual operation being timed
        auto end   = std::chrono::high_resolution_clock::now();
        double us  = std::chrono::duration<double, std::micro>(end - start).count();
        latencies.push_back(us);
    }

    auto totalEnd = std::chrono::high_resolution_clock::now();
    // ... compute percentiles from sorted latencies
}
```

**Why `Func&&` (forwarding reference)?**
Accepts any callable — lambda, function pointer, function object. The template `Func` is deduced as the exact closure type of the lambda, enabling inlining. No virtual dispatch overhead.

**Why `std::chrono::high_resolution_clock`?**
The highest-resolution clock available on the platform — typically nanosecond resolution on modern Linux. For operations in the microsecond range (Trie: ~4µs, hash: ~0.1µs), you need this precision.

**Percentile computation:**
```cpp
std::sort(latencies.begin(), latencies.end());   // sort all per-query latencies
auto percentile = [&](double p) -> double {
    size_t idx = static_cast<size_t>(p * (n - 1));
    return latencies[std::min(idx, n - 1)];
};
// p50 = median, p95 = 95th percentile, p99 = 99th percentile
```

---

### `timeBuildMs()` — Measuring Construction Cost

```cpp
template <typename Func>
double timeBuildMs(Func&& func) {
    auto start = std::chrono::high_resolution_clock::now();
    func();   // the build operation (e.g., all trie inserts in a loop)
    auto end  = std::chrono::high_resolution_clock::now();
    return std::chrono::duration<double, std::milli>(end - start).count();
}
```

Used as:
```cpp
double trieBuildMs = timeBuildMs([&]() {
    for (const auto& w : words) trie.insert(w);
});
```

The lambda captures `trie` and `words` by reference — the build work happens inside the lambda, cleanly bracketed by the clock reads.

---

### `volatile bool found` — The Anti-Optimization Guard

```cpp
auto linearResult = timeQueries(exactQueries, [&](const std::string& q) {
    volatile bool found = std::find(words.begin(), words.end(), q) != words.end();
    (void)found;
});
```

Without `volatile`, the compiler may detect that `found` is never read after assignment and eliminate the entire computation as dead code — you'd be benchmarking a no-op. `volatile` forces the computation to materialize. `(void)found` suppresses the unused-variable compiler warning.

---

### Fairness Principles in the Benchmark Design

1. **Same queries for all strategies** — `exactQueries`, `prefixQueries`, `fuzzyQueries` are generated once and reused. Linear scan and hash lookup see identical inputs.

2. **Seeded random subsets** — not alphabetical prefix slices.

3. **Warm-up before timing** — cold cache effects excluded.

4. **Median of 5 trials** — reduces measurement noise (LexiCore notes this in benchmark config but the inner `timeQueries` doesn't repeat trials — the 5 trials design is noted for production extension).

5. **Word-level limit=20 for prefix trie** — matches real UI behavior, not an unlimited scan.

---

## File 8: `RESULTS.md` — Reading the Numbers

### Structure Diagnostics Table
```
Dict Size | Trie Nodes | Trie Build | BK Nodes | BK Build  | BK Max Depth
1,000     | 5,815      | 1.09ms     | 1,000    | 1.00ms    | 9
10,000    | 42,994     | 9.48ms     | 10,000   | 12.21ms   | 12
88,344    | 185,264    | 39.42ms    | 88,344   | 150.10ms  | 19
```

**What these numbers tell you:**
- Trie nodes grow sub-linearly with words because adjacent words share prefixes (85,264 << 88,344 × avg_word_len ≈ 618,408)
- BK-tree nodes = word count exactly (each word = one node)
- BK-tree build (150ms) >> Trie build (39ms) because every insertion in BK-tree computes editDistance at each level — O(m) per level × O(log n) levels × n words. Trie insertion is O(k) per word, simpler.
- Max depth 19 confirms the shuffle worked — a healthy tree.

---

### Exact Match Results — Hash vs Linear
```
Dict 88K | Linear scan: 121.80ms total | 8,211 q/s
Dict 88K | Hash lookup:   0.45ms total | 2,229,674 q/s  → 271× faster
```
Hash is 271× faster at 88K — scaling advantage grows with dictionary size (7× at 1K, 271× at 88K). This is O(1) vs O(n) in action.

---

### Prefix Search — Trie vs Linear
```
Dict 10K | Linear: 31.78ms | Trie: 4.51ms
Dict 88K | Linear: 214.71ms | Trie: 4.39ms  ← nearly IDENTICAL to 10K!
```
The killer number: Trie goes from 4.51ms (10K) to 4.39ms (88K) — essentially flat. Linear goes from 31ms to 214ms — 6.75× worse. This is the empirical proof of O(k) independence.

---

### Fuzzy Search — BK-tree vs Linear
```
Dict 88K | Linear: 13,818ms | BK-tree: 5,729ms → 2.4× faster
```
Only 2.4× — much less dramatic than Trie's 49×. Why? Edit distance itself is O(n·m) and must still be computed for every visited node. The BK-tree prunes subtrees but still visits a significant fraction of nodes for maxDist=2 with English words. The pruning is less effective than prefix restriction.

---

---

# 🏗️ Phase 4 — Build, Testing & Docs

---

## File 9: `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.20)
project(LexiCore LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)      # error if C++20 not available
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)    # generates compile_commands.json for IDEs/linters
```

**`CMAKE_EXPORT_COMPILE_COMMANDS ON`** — generates a `compile_commands.json` file that tells tools like `clangd`, `clang-tidy`, and `bear` exactly how each file was compiled. This enables accurate code navigation and lint in VSCode/CLion.

---

```cmake
# Core library — compiled ONCE, linked by both app and tests
add_library(lexicore_lib
    src/dictionary.cpp
    src/edit_distance.cpp
    src/trie.cpp
    src/bktree.cpp
    src/ranking.cpp
    src/benchmark.cpp
)
target_include_directories(lexicore_lib PUBLIC include)
```

**Why a library target?**
Without a library, each test executable would compile all 6 source files independently — redundant work and potential inconsistency. With `lexicore_lib`, the 6 files compile once into a static library (`.a` file), and both the app and all 4 test executables just link against it.

`PUBLIC include` — the `include/` directory is exported to all consumers of `lexicore_lib`. When the app and tests link against `lexicore_lib`, they automatically get the include path — no need to specify it again per target.

---

```cmake
add_executable(lexicore app/main.cpp)
target_link_libraries(lexicore PRIVATE lexicore_lib)

enable_testing()
foreach(test_name test_edit_distance test_trie test_bktree test_correctness)
    add_executable(${test_name} tests/${test_name}.cpp)
    target_link_libraries(${test_name} PRIVATE lexicore_lib)
    add_test(NAME ${test_name} COMMAND ${test_name})
endforeach()
```

**`PRIVATE` vs `PUBLIC` in `target_link_libraries`:**
- `PRIVATE` — the linked library is an implementation detail. Consumers of `lexicore` don't need to know it links `lexicore_lib`. Correct for executables.
- `PUBLIC` — propagates to consumers. Correct for libraries that expose types from their dependencies in their own headers.

**`foreach` loop** — creates 4 test executables and 4 CTest entries with a single loop instead of repeating 3 lines × 4 = 12 lines of CMake. Maintainable: add a new test file to the list and it's automatically registered.

---

### Build Commands Decoded

```bash
# Debug build with sanitizers
cmake -S . -B build-debug -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -g"
cmake --build build-debug
cd build-debug && ctest --output-on-failure

# Release build (O2 optimized)
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
./build/lexicore
```

- `-S .` — source directory is current dir
- `-B build` — build artifacts go into `build/` (out-of-source build — keeps source tree clean)
- `-DCMAKE_BUILD_TYPE=Release` — enables `-O2` optimization flags
- `-DCMAKE_CXX_FLAGS=...` — appends extra compiler flags (sanitizers)
- `ctest --output-on-failure` — runs all registered tests, shows output only on failures

---

## Files 10–13: The Test Suite

### `test_edit_distance.cpp` — Three Testing Tiers

**Tier 1 — Concrete unit tests:**
```cpp
assert(editDistance("kitten", "sitting") == 3);
assert(editDistance("saturday", "sunday") == 3);
assert(editDistance("intention", "execution") == 5);
```
These are famous Levenshtein examples. Good to memorize — "kitten→sitting = 3" is a classic interview example.

**Tier 2 — Bounded variant sentinel tests:**
```cpp
// true dist("cat","dog") = 3, maxDist=1 → returns sentinel = maxDist+1 = 2
assert(editDistanceBounded("cat", "dog", 1) == 2);
```
This is the tricky one — the assertion tests that the sentinel (2) is returned, NOT the true distance (3).

**Tier 3 — Property-based randomized tests:**
```cpp
// Symmetry: dist(a,b) == dist(b,a) over 1000 random pairs
// Triangle inequality: dist(a,c) <= dist(a,b)+dist(b,c) over 1000 random triples
std::mt19937 rng(12345);  // seeded → reproducible even when testing random inputs
```
If either property fails on any random pair, `std::abort()` terminates the test with a detailed error message showing the violating inputs.

---

### `test_trie.cpp` — Edge Cases That Reveal Design Intent

| Test | What it reveals |
|---|---|
| `search("app") == false` with "apple" inserted | `isEndOfWord` must be checked, not just node existence |
| `startsWith("") == true` | Empty prefix always matches — node existence check returns root |
| `getNodeCount() == 1` for empty trie | Root node pre-created in constructor |
| `autocomplete("app", 20)` returns sorted | `std::sort` called on DFS results |
| `autocomplete("ap", 2)` returns exactly 2 | Limit is enforced during DFS, not after |

---

### `test_bktree.cpp` — Key Coverage

| Test | What it reveals |
|---|---|
| `search("book", maxDist=1)` returns 4 results | Insert order matters — "hook" is at depth 2 under "cook" |
| `search("book", maxDist=0)` returns only "book" | maxDist=0 = exact match via BK-tree |
| Duplicate insertion → `getNodeCount() == 1` | dist==0 → return without creating child |
| Large radius `maxDist=10` → returns all 4 words | When radius exceeds all distances, everything is visited |
| `search("zzz", maxDist=1)` → empty | Triangle inequality correctly prunes all subtrees |

---

### `test_correctness.cpp` — The Oracle Test

This is the most important test in the suite. Structure:

```
testSmallDictionary():
  17 curated words (book/cook/hook/..., apple/apply/ample/...)
  11 queries × 4 maxDist thresholds (0,1,2,3) = 44 comparisons
  Assert: BK-tree result set == linear scan result set for every case

testRandomizedOracle():
  500 random words (generated programmatically)
  200 queries (half dictionary words, half mutated) × 4 thresholds = 800 comparisons
  Assert: zero mismatches

testEdgeCases():
  Empty query ("")
  Query longer than any word ("abcdefgh" vs {"a","ab","abc","abcd"})
  maxDist=0 exact match
```

**`resultSetsMatch()` — order-independent set comparison:**
```cpp
bool resultSetsMatch(vector<pair<string,int>> a, vector<pair<string,int>> b) {
    auto cmp = [](const pair<string,int>& x, const pair<string,int>& y) {
        return x.first < y.first || (x.first == y.first && x.second < y.second);
    };
    std::sort(a.begin(), a.end(), cmp);
    std::sort(b.begin(), b.end(), cmp);
    return a == b;     // vector operator== compares element by element after sort
}
```

Both vectors sorted by the same comparator → any ordering differences disappear → `a == b` is a true set equality check.

---

## File 14: `README.md` — Design Decisions Worth Knowing

The README has a **Design Decisions** section that's denser than it looks. Key points:

**"Dictionary is shuffled before BK-tree insertion"**
Not buried in a comment inside the code — it's a documented architectural decision with a reason. Alphabetically-sorted input → chain-shaped degenerate trees. Shuffling produces max depth 19 for 88K words.

**"The BK-tree search uses the TRUE `editDistance()` not `editDistanceBounded()`"**
This is the sentinel bug documented as a known trap. The README explains it so future maintainers don't 'optimize' the wrong thing.

**"All words are lowercased, non-alphabetic characters stripped"**
Normalization policy is explicit. If you change this, prefix and fuzzy search behavior changes. Document policies, don't leave them as implicit behavior.

**Complexity Summary Table:**
```
Structure   | Search    | Insert   | Space
Linear scan | O(n·m)    | O(1)     | O(n)
Hash table  | O(1) avg  | O(1) avg | O(n)
Trie        | O(k)      | O(k)     | O(alphabet × nodes)
BK-tree     | Practical | O(depth) | O(n)
```

**Known Trade-offs section** — this is gold for interviews. It shows the author understands the limitations:
- Trie memory: `unordered_map` per node has overhead. Radix trie is the production fix.
- BK-tree may underperform on small dictionaries (overhead not worth it at small scale).
- Edit distance is O(n·m) — acceptable for word-length strings.

---

## Complete Data Flow — End to End

```
data/words.txt (104K raw lines)
      │
      ▼
Dictionary::loadFromFile()
  normalize() → lowercase, strip non-alpha
  deduplicate via unordered_set
      │
      ├── words_ (vector, 88K, ordered)
      │     │
      │     ├── Trie build (O(n·k), 39ms)
      │     └── Benchmark subsets (seeded shuffle)
      │
      └── wordSet_ (unordered_set, 88K)
            └── exact command (O(1))

getShuffledWords(42)
      │
      ▼
BKTree build (shuffled, O(n·m·log n), 150ms)

───────────────────────────────────────
User types: "fuzzy aple 2"
      │
      ▼
main.cpp REPL parses command="fuzzy", query="aple", maxDist=2
      │
      ▼
bktree.search("aple", 2)
  → editDistance at each visited node (true, not bounded)
  → triangle inequality prunes subtrees
  → returns vector<pair<string,int>>
      │
      ▼
rankResults()
  → converts to RankedResult structs
  → std::sort (dist asc, lex tie-break)
  → returns vector<RankedResult>
      │
      ▼
main.cpp prints top 20, "...and N more"
```

---

## Complete File Dependency Graph

```
main.cpp ──────────────────────────────────────────┐
    │                                               │
    ├── dictionary.hpp/cpp                          │
    │       (words_, wordSet_, shuffle)             │
    │                                               │
    ├── trie.hpp/cpp                                │
    │       (insert, autocomplete)                  │
    │                                               ▼
    ├── bktree.hpp/cpp                         benchmark.hpp/cpp
    │       (insert, search)                        │
    │           │                                   │
    │           └── edit_distance.hpp/cpp ◄─────────┘
    │               (editDistance,
    │                editDistanceBounded)
    │
    └── ranking.hpp/cpp
            (rankResults, RankedResult)

Tests:
    test_edit_distance.cpp  →  edit_distance.hpp/cpp
    test_trie.cpp           →  trie.hpp/cpp
    test_bktree.cpp         →  bktree.hpp/cpp + edit_distance.hpp/cpp
    test_correctness.cpp    →  bktree.hpp/cpp + dictionary.hpp/cpp + edit_distance.hpp/cpp
```

`edit_distance` is the lowest-level dependency — everything that does fuzzy operations depends on it. `dictionary` is second-lowest — it feeds all structures. `ranking` is a leaf — nothing depends on it.

---

## ✅ Phases 2+3+4 Mastery Checklist

- [ ] Explain why Dictionary maintains BOTH `words_` and `wordSet_` simultaneously
- [ ] What does `wordSet_.insert(word).second` return, and how is it used for dedup?
- [ ] Why does `getSubset()` shuffle before taking first N?
- [ ] What does `static std::string normalize(...)` mean — why `static`?
- [ ] Why does `RankedResult` exist instead of just using `pair<string,int>`?
- [ ] Explain the REPL's use of `istringstream` to parse command tokens
- [ ] Why is `volatile bool found` needed in benchmarks?
- [ ] What does `Func&&` mean in `timeQueries<Func>`?
- [ ] What does `CMAKE_EXPORT_COMPILE_COMMANDS ON` generate and why is it useful?
- [ ] Why is `lexicore_lib` a library target and not part of the executable directly?
- [ ] What is `PRIVATE` vs `PUBLIC` in `target_link_libraries`?
- [ ] What three testing tiers exist in `test_edit_distance.cpp`?
- [ ] What does `resultSetsMatch()` do and why does it sort before comparing?
- [ ] From RESULTS.md: why is Trie at 88K nearly identical in latency to 10K?
- [ ] From RESULTS.md: why is BK-tree only 2.4× faster (not 49× like Trie)?

---

> ✅ **Phases 2+3+4 complete. All files covered.**
>
> Say **"give the SDE prep guide"** for the intense interview Q&A across all Phases 2-4 topics.
