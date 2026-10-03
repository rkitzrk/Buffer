# 📚 LexiCore Study Guide — Phase 0
### *ELI5 + All Foundations + Intense SDE Interview Prep with Scripted Answers*

---

## 🗺️ Full Reading Plan (All 5 Phases)

| Phase | Files | Theme |
|---|---|---|
| **Phase 0** | *(this doc)* | ELI5 + Foundations + Interview Prep |
| **Phase 1** | `edit_distance.hpp/.cpp` → `bktree.hpp/.cpp` → `trie.hpp/.cpp` | ⭐ Core Algorithms |
| **Phase 2** | `dictionary.hpp/.cpp` → `ranking.hpp/.cpp` → `app/main.cpp` | Glue Layer + CLI Design |
| **Phase 3** | `benchmark.cpp`, `benchmark.hpp`, `RESULTS.md` | Performance Engineering |
| **Phase 4** | `CMakeLists.txt`, `tests/*.cpp`, `README.md`, `docs/` | Build, Testing & Docs |

---

## 🧒 Part 1 — Explain Like I'm 5

### What is LexiCore?

Imagine you have a **giant book of 88,000 English words**. Your friend types something into a search box. LexiCore has to answer three kinds of questions really fast:

1. **"Is this word real?"** → Exact lookup. Like checking if "apple" is in the book.
2. **"What words start with 'app'?"** → Prefix / autocomplete. Like your phone keyboard.
3. **"I typed 'aple' by mistake — what did I mean?"** → Fuzzy / typo-tolerant search. Like spell-check.

No single data structure is best at all three. So LexiCore uses a **different filing cabinet** for each:

| Question | Filing Cabinet | Speed |
|---|---|---|
| Exact match | Hash Table (`unordered_set`) | O(1) — instant |
| Prefix / autocomplete | Trie | O(k) — only walks the prefix |
| Fuzzy / typo | BK-Tree | Prunes branches mathematically |

The **glue** that makes fuzzy search work is **Edit Distance** — a Dynamic Programming algorithm that counts the minimum number of edits (insert / delete / substitute) to turn one word into another.

---

### How the BK-Tree Magic Works (Triangle Inequality)

```
Query: "aple", maxDist = 1
Root word: "sample" → dist("aple","sample") = 4

BK-tree children of "sample":
  edge 2 → "maple"
  edge 5 → "example"

Pruning: only recurse into children where edge ∈ [4-1, 4+1] = [3, 5]
→ edge 2 is out of [3,5] → skip "maple" entirely without computing!
→ edge 5 is in [3,5] → recurse into "example"
```

This is the **triangle inequality**: if node N is distance `d` from query Q, and child C is distance `k` from N, then by triangle inequality, dist(Q,C) ≥ |d-k|. If |d-k| > maxDist, it's mathematically impossible for C to be a match.

---

## 📐 Part 2 — All Foundational Topics

### Foundation 1: C++ Memory & Ownership
### Foundation 2: STL Containers Deep Dive
### Foundation 3: Dynamic Programming (Edit Distance)
### Foundation 4: Trie Data Structure
### Foundation 5: BK-Tree & Metric Spaces
### Foundation 6: Hash Tables Internals
### Foundation 7: Algorithm Complexity
### Foundation 8: C++ Language Features
### Foundation 9: File I/O & String Processing
### Foundation 10: Build Systems & Testing

*(All covered in detail inside the Interview Q&A sections below)*

---

---

# 🔥 Part 3 — INTENSE SDE Interview Q&A
### *Scripted answers — say this almost verbatim*

> Each section has **Concept Questions**, **Code Questions**, and **Design Questions** exactly as interviewers ask them.

---

## 🔷 Section 1: Dynamic Programming & Edit Distance

---

**Q1: What is Edit Distance? Explain from scratch.**

> *"Edit distance, also called Levenshtein distance, is the minimum number of single-character operations needed to transform one string into another. The three allowed operations are: insertion, deletion, and substitution — each costing 1. For example, turning 'cat' into 'bat' requires one substitution, so the edit distance is 1. Turning 'kitten' into 'sitting' requires 3 substitutions and 1 insertion, so the distance is 3. It's computed using a classic 2D dynamic programming table."*

---

**Q2: Write the DP recurrence for Edit Distance.**

> *"We define dp[i][j] as the edit distance between the first i characters of string A and the first j characters of string B.*
>
> *Base cases: dp[0][j] = j (insert j characters), dp[i][0] = i (delete i characters).*
>
> *Recurrence:*
> ```
> cost = 0 if A[i-1] == B[j-1], else 1
> dp[i][j] = min(
>     dp[i-1][j] + 1,        // delete from A
>     dp[i][j-1] + 1,        // insert into A
>     dp[i-1][j-1] + cost    // substitute
> )
> ```
> *Final answer is dp[n][m] where n = len(A), m = len(B). Time complexity is O(n·m), space is O(n·m) naive or O(m) with rolling rows."*

---

**Q3: How do you optimize edit distance to O(m) space? (The rolling row trick)**

> *"Instead of storing the full n×m table, I observe that each row dp[i] only depends on dp[i-1] — the previous row. So I maintain just two arrays: 'prev' and 'curr'. After computing each row into 'curr', I swap them using std::swap which is O(1) since it just swaps internal pointers. This reduces space from O(n·m) to O(m). The time complexity remains O(n·m). LexiCore does exactly this in edit_distance.cpp."*

---

**Q4: What is the bounded/threshold edit distance optimization and when do you use it?**

> *"If you only care whether the distance is ≤ maxDist (not the exact value), you can optimize in two ways: First, if |len(A) - len(B)| > maxDist, return early — the length difference alone exceeds the threshold, so it's impossible. Second, instead of computing the full table, only compute cells within a diagonal band of width 2*maxDist+1 around the main diagonal. Cells outside this band can never contribute to a solution within the threshold. This makes it faster in practice but the key caveat is: the return value when distance > maxDist is a sentinel (maxDist+1), NOT the true distance. You must never use this sentinel for further mathematical computations like BK-tree pruning intervals."*

---

**Q5: What are other DP problems similar to Edit Distance? (Classic follow-up)**

> *"Edit distance is a member of the string DP family. Closely related problems are: LCS — Longest Common Subsequence, where dp[i][j] is the LCS of prefixes; LIS — can be solved with DP and binary search; String interleaving; Regular expression matching. All of these use the same 'prefix of A vs prefix of B' sub-problem structure. The key pattern is: the answer to a prefix-pair depends on the answers to smaller prefix-pairs."*

---

**Q6: What is the time and space complexity of edit distance? Can you do better than O(n·m)?**

> *"Standard Levenshtein is O(n·m) time and O(m) space with rolling rows. For exact edit distance you cannot fundamentally do better than O(n·m) in the worst case — it's provably tight for the general case. However, there are approximations and specialized algorithms: the Ukkonen algorithm runs in O(k·min(n,m)) where k is the actual edit distance — great when you expect small distances. The Bit-Parallel algorithm runs in O(n·m/w) where w is the word size (64 bits), using bitwise operations. For LexiCore's use case — short English words, typically 3-10 characters — the standard O(n·m) is perfectly fine, and the bounded variant gives practical speedups."*

---

**Q7: Prove that edit distance satisfies the triangle inequality.**

> *"I need to show that dist(A, C) ≤ dist(A, B) + dist(B, C) for any strings A, B, C. Proof by construction: suppose we have an optimal sequence of edits turning A into B (cost = dist(A,B)), and an optimal sequence turning B into C (cost = dist(B,C)). Concatenating these two sequences gives us a valid (not necessarily optimal) sequence turning A into C. Since a valid sequence is an upper bound on the optimal, dist(A,C) ≤ dist(A,B) + dist(B,C). This triangle inequality is what makes edit distance a metric, and it's the mathematical foundation that makes BK-trees work."*

---

## 🔷 Section 2: Trie Data Structure

---

**Q8: What is a Trie? Explain the structure and operations.**

> *"A Trie, or prefix tree, is a tree where each node represents a character, and the path from root to a node spells out a prefix. Each node contains: a map from characters to child nodes, and an isEndOfWord boolean flag. To insert 'apple', you walk from the root, creating nodes for 'a', 'p', 'p', 'l', 'e' if they don't exist, then set isEndOfWord = true on the last node. To search for a word, you traverse the path and check isEndOfWord. For autocomplete with prefix 'app', you walk to the 'app' node, then do a DFS collecting all words reachable from there. The critical property is that insert and search are both O(k) where k is the word length — completely independent of how many words are in the trie."*

---

**Q9: Write Trie insert and search code.**

> ```cpp
> struct TrieNode {
>     unordered_map<char, unique_ptr<TrieNode>> children;
>     bool isEndOfWord = false;
> };
>
> void insert(const string& word) {
>     TrieNode* curr = root.get();
>     for (char c : word) {
>         if (!curr->children.count(c))
>             curr->children[c] = make_unique<TrieNode>();
>         curr = curr->children[c].get();
>     }
>     curr->isEndOfWord = true;
> }
>
> bool search(const string& word) {
>     TrieNode* curr = root.get();
>     for (char c : word) {
>         auto it = curr->children.find(c);
>         if (it == curr->children.end()) return false;
>         curr = it->second.get();
>     }
>     return curr->isEndOfWord;
> }
> ```
> *"I use unordered_map for the children map for memory efficiency on sparse nodes. The alternative is array<Node*,26> which gives O(1) character lookup instead of O(1) average but uses 26 pointers per node regardless of how many children exist."*

---

**Q10: How does autocomplete work in a Trie?**

> *"Autocomplete has two phases. Phase 1 — navigation: starting from root, traverse the path corresponding to the prefix character by character. If any character is not found, there are no matching words. Phase 2 — collection: from the node you reached (representing the full prefix), do a DFS collecting every node where isEndOfWord is true. I pass the accumulated prefix string down the recursion and append each character as I go deeper. I also pass a limit to avoid flooding output for very broad prefixes like 'a'. In LexiCore's implementation, the collected results are then sorted lexicographically for deterministic output."*

---

**Q11: What is the memory trade-off: unordered_map vs array[26] per node?**

> *"Array of 26 gives O(1) child lookup (direct index by 'c' - 'a') and is cache-friendly — the array sits contiguously in memory. But it wastes 26 pointers per node even if only 2-3 children exist. For a large, sparse dictionary, this is wasteful. unordered_map allocates only what's needed per node, so memory is proportional to actual children. The downside is indirection and worse cache behavior. A middle ground used in production is a compressed trie or Patricia trie, which merges single-child chains into single edges storing the full substring. LexiCore chose unordered_map for flexibility and memory efficiency."*

---

**Q12: What is a Compressed Trie / Patricia Trie and when would you use it?**

> *"A compressed trie merges chains of single-child nodes into one edge labeled with a full substring instead of individual characters. For example, if the only word starting with 'xyz' is 'xylophone', instead of nodes x→y→l→o→p→h→o→n→e, you store a single edge labeled 'xylophone'. This reduces node count dramatically for sparse tries with long shared prefixes. It's used in production systems like IP routing tables (longest prefix match), text search, and suffix trees. The trade-off is more complex insertion and deletion logic."*

---

**Q13: Compare Trie vs Hash Map for autocomplete.**

> *"Hash map gives O(1) exact lookup but cannot do prefix search at all — you'd have to scan every key, which is O(n). Trie gives O(k) prefix search and O(k) exact lookup. For autocomplete specifically, Trie is the right tool because its structure inherently groups words by shared prefixes. A hash map has no structural relationship between 'apple' and 'application' — they hash to completely different buckets. A trie naturally puts them under the same 'appl' prefix node."*

---

## 🔷 Section 3: BK-Tree & Fuzzy Search

---

**Q14: What is a BK-Tree and what problem does it solve?**

> *"A BK-Tree is a tree data structure designed for fuzzy search in metric spaces — spaces where a distance function satisfies the triangle inequality. In LexiCore, the metric is edit distance. The tree lets us find all words within a given edit distance of a query word without comparing against every word in the dictionary. The core insight is: by knowing the distance from the query to a node, we can use the triangle inequality to mathematically prove that certain entire subtrees cannot contain any matches, and prune them without ever computing their distances."*

---

**Q15: Explain BK-Tree construction. How do you insert words?**

> *"The root is the first word inserted. To insert a new word W starting at node N: compute d = editDistance(N.word, W). If d is 0, it's a duplicate — skip. If node N already has a child on edge d, recurse into that child. Otherwise, create a new child node containing W and attach it to N with edge label d. The key property maintained is: every node on edge k from its parent is exactly k edit distance away from that parent. After building the tree, each node's position encodes its metric relationship to its ancestors."*

---

**Q16: Explain BK-Tree search with the triangle inequality. This is the core algorithmic insight.**

> *"To find all words within maxDistance of query Q: At node N, compute d = editDistance(Q, N.word). If d ≤ maxDistance, add N.word to results. Then for each child of N with edge label k, I need to decide: is it possible that the child's word is within maxDistance of Q?*
>
> *By the triangle inequality: dist(Q, child) ≥ |dist(Q, N) - dist(N, child)| = |d - k|.*
>
> *So if |d - k| > maxDistance, it is mathematically impossible for the child to be a match — I can skip it entirely. Equivalently, I only recurse into children where k ∈ [d - maxDistance, d + maxDistance].*
>
> *This pruning is what makes BK-tree faster than linear scan. For small maxDistance relative to typical inter-word distances, most subtrees get pruned."*

---

**Q17: Why must BK-Tree use true editDistance() for pruning, not editDistanceBounded()?**

> *"This is a subtle but critical correctness issue in LexiCore. The bounded variant returns maxDist+1 as a sentinel — not the true distance — when the actual distance exceeds maxDist. The pruning interval formula is [d - maxDistance, d + maxDistance] where d must be the TRUE distance from query to the current node. If I substitute the sentinel value, say maxDist+1 = 3, when the real distance is 7, my pruning interval would be [1, 5] instead of [5, 9]. I would incorrectly recurse into children that should be pruned, AND possibly skip children I should visit — silently producing wrong results. So for the pruning calculation, I always use the full editDistance(), which returns the exact value. The bounded variant is only safe for the leaf-level decision of whether to include a word in results."*

---

**Q18: Why shuffle the dictionary before inserting into a BK-Tree?**

> *"If words are inserted in alphabetical order, adjacent words tend to have very similar edit distances from any given root word — for example 'able', 'abled', 'abler', 'ables' all have distance 1 from each other. This creates a degenerate chain-shaped tree where the root has one child, that child has one child, and so on — essentially a linked list with depth equal to dictionary size. Search on such a tree degrades to O(n), defeating the entire purpose.*
>
> *Shuffling with a fixed seed (42 in LexiCore) ensures random ordering, which produces a well-branched tree. RESULTS.md shows the 88K-word shuffled BK-tree has max depth 19 — meaning at most 19 levels to traverse in the worst case — versus potentially thousands without shuffling."*

---

**Q19: What is the time complexity of BK-Tree search?**

> *"There is no clean closed-form complexity for BK-tree search. It's empirically practical for small maxDistance values. The fraction of nodes visited depends on: the distribution of edit distances in the dictionary, the maxDistance threshold, and the tree's branching factor. For small radius (maxDist ≤ 2) relative to typical word lengths (5-10 chars), only a small fraction of nodes are visited. LexiCore's benchmarks show BK-tree is 2-2.5× faster than linear scan at 88K words for maxDist=2. For large maxDistance approaching word length, the pruning becomes ineffective and performance degrades toward O(n)."*

---

## 🔷 Section 4: Hash Tables & `unordered_set` / `unordered_map`

---

**Q20: How does a hash table work internally?**

> *"A hash table stores key-value pairs in an array of buckets. When inserting key K: compute hash(K), take modulo array size to get bucket index, store the pair in that bucket. For lookup: compute the same hash, go to that bucket, compare keys. When two keys hash to the same bucket — called a collision — the standard technique is chaining: each bucket holds a linked list of entries. The average-case complexity is O(1) for insert and lookup assuming a good hash function distributes keys uniformly. Worst case is O(n) if all keys collide into one bucket.*
>
> *std::unordered_set and unordered_map in C++ use open addressing or chaining depending on the implementation, with a load factor threshold (default 1.0) that triggers rehashing — resizing the bucket array and reinserting all elements — when exceeded."*

---

**Q21: What is a hash collision and how is it handled?**

> *"A collision occurs when two different keys produce the same hash value modulo the bucket count. Two common resolution strategies: Chaining — each bucket is a linked list; multiple elements with the same hash live in the same bucket's list. Lookup requires scanning the list. Open addressing — if a bucket is occupied, probe sequentially (linear probing) or with a different formula (quadratic probing, double hashing) until an empty bucket is found. C++ STL typically uses chaining. The worst-case impact of collisions is O(n) per operation if all keys collide, which is why a good hash function and appropriate load factor matter."*

---

**Q22: What is the load factor and why does it matter?**

> *"Load factor = number of elements / number of buckets. As load factor increases, collision probability increases, degrading performance toward O(n). std::unordered_map has a default max_load_factor of 1.0 — when exceeded, it rehashes: allocates a new, larger bucket array (typically 2× size) and reinserts all elements. This rehash is O(n) amortized. For performance-critical code, you can call reserve(n) upfront to pre-size the table and avoid rehashing. In LexiCore's dictionary, 88K words are inserted, so the hash set does several rehashes during loading."*

---

**Q23: How does `unordered_set::insert` return a pair and how is it used for deduplication?**

> *"insert() returns a std::pair<iterator, bool>. The second element is true if insertion succeeded (key was new) and false if the key already existed. LexiCore's dictionary.cpp uses this idiom:*
> ```cpp
> if (wordSet_.insert(word).second) {
>     words_.push_back(word);  // only add to vector if truly new
> }
> ```
> *This is a common C++ pattern for 'insert and check if it's a duplicate' in one operation, avoiding a separate contains() call which would be two hash table lookups instead of one."*

---

**Q24: When would you use unordered_map vs map?**

> *"std::map is a red-black tree giving O(log n) insert and lookup with keys always sorted. std::unordered_map is a hash table giving O(1) average insert and lookup but keys are unordered. Use map when you need ordered iteration, range queries, or the key type doesn't have a good hash function. Use unordered_map when you want fastest lookup and don't need ordering. In LexiCore, the Trie uses unordered_map<char, unique_ptr<TrieNode>> for children — char is hashable and we don't need ordered children (we sort results at the end anyway)."*

---

## 🔷 Section 5: C++ Memory Management

---

**Q25: What is RAII? Why is it important?**

> *"RAII stands for Resource Acquisition Is Initialization. The core idea is: tie resource lifetime to object lifetime. When you acquire a resource — allocate memory, open a file, lock a mutex — do it in a constructor. Release it in the destructor. Because C++ guarantees destructors run when objects go out of scope — even if an exception is thrown — the resource is always released. This eliminates entire classes of bugs: memory leaks, file handle leaks, deadlocks from unreleased mutexes.*
>
> *std::unique_ptr is RAII for heap memory. std::ifstream is RAII for file handles. std::lock_guard is RAII for mutexes. In LexiCore, every TrieNode and BKNode is owned by a unique_ptr, so the entire tree is freed automatically when the root unique_ptr goes out of scope — zero manual delete calls."*

---

**Q26: What is `std::unique_ptr`? How does it differ from a raw pointer?**

> *"unique_ptr is a smart pointer that uniquely owns a heap-allocated object. Key properties: it cannot be copied, only moved — enforcing sole ownership. When it goes out of scope or is destroyed, it automatically calls delete on the managed object. make_unique<T>(args) is the factory function — it allocates memory and constructs the object atomically, which is exception-safe.*
>
> *Contrast with raw pointer: raw pointers have no ownership semantics — you must remember to call delete, you might accidentally call it twice (double free), or forget to call it (leak), or use the pointer after deletion (use-after-free). unique_ptr eliminates all of these by construction. If you need shared ownership — multiple owners — use shared_ptr instead, which uses reference counting."*

---

**Q27: What is `std::move` and when do you use it?**

> *"std::move is a cast that converts an lvalue into an rvalue reference, signaling 'I'm done with this, you can steal its internals.' It doesn't actually move anything — it just enables the move constructor or move assignment operator to be called instead of copy. For std::string, move is O(1) — it just transfers the internal char buffer pointer. For a vector, move transfers the internal array. This is critical for performance when passing large objects into functions or containers.*
>
> *In LexiCore's BKNode constructor:*
> ```cpp
> explicit BKNode(std::string w) : word(std::move(w)) {}
> ```
> *The string is taken by value (already copied or moved in), then moved into the member — zero extra copies."*

---

**Q28: Stack vs Heap — when does each get used?**

> *"Stack memory is managed automatically: it's allocated when a function is called, freed when it returns. It's very fast — just moving a stack pointer. Stack size is limited (typically 1-8 MB). Use stack for local variables, function parameters, small fixed-size arrays.*
>
> *Heap memory is explicitly managed (via new/delete or smart pointers). It persists until explicitly freed, can be arbitrarily large, and can be shared across functions via pointers. It's slower due to the allocator overhead and cache-unfriendly scattered locations.*
>
> *In LexiCore, DP vectors (prev, curr in editDistance) are on the heap because they're allocated via std::vector internally. Trie and BK-tree nodes are on the heap via unique_ptr because they need to outlive the function that creates them and their count is not known at compile time."*

---

**Q29: What is a memory leak? What is use-after-free? What is double-free?**

> *"Memory leak: you allocate heap memory (new) but never deallocate it (delete). The memory is wasted for the program's lifetime. Common in code with multiple return paths where only some paths delete the pointer.*
>
> *Use-after-free: you delete a pointer, then later dereference it. The memory may have been reallocated for something else — you're reading/writing garbage data. This is undefined behavior and a common security vulnerability.*
>
> *Double-free: you call delete on the same pointer twice. The allocator's internal metadata gets corrupted. Also undefined behavior and exploitable.*
>
> *All three are eliminated by unique_ptr: you cannot copy it (preventing double-free scenarios), it automatically deletes on destruction (preventing leaks), and once it's destroyed, the raw pointer is gone from your code. LexiCore tests with AddressSanitizer which detects all three at runtime."*

---

## 🔷 Section 6: STL Algorithms & C++ Language Features

---

**Q30: What does `std::sort` use internally? What is its complexity?**

> *"std::sort in modern C++ implementations (like libstdc++) uses introsort — a hybrid of quicksort, heapsort, and insertion sort. Quicksort is the primary algorithm giving O(n log n) average. If the recursion depth exceeds 2·log(n), it switches to heapsort to guarantee O(n log n) worst case (avoiding quicksort's O(n²) worst case on adversarial input). For small subarrays (typically ≤ 16 elements), it uses insertion sort which has better constant factors due to cache efficiency and low overhead. Result: O(n log n) worst-case, in-place, not stable. std::stable_sort is O(n log²n) or O(n log n) with extra memory, preserving relative order of equal elements."*

---

**Q31: What are structured bindings in C++17?**

> *"Structured bindings let you unpack a pair, tuple, or struct into named variables in one declaration. For example:*
> ```cpp
> // Instead of:
> for (auto& p : node->children) {
>     char ch = p.first;
>     auto& child = p.second;
> }
>
> // C++17 structured binding:
> for (const auto& [ch, child] : node->children) {
>     // ch and child directly usable
> }
> ```
> *LexiCore uses this extensively when iterating over unordered_map children in both Trie and BKTree. It's purely syntactic sugar — same performance, much more readable."*

---

**Q32: What is a lambda function in C++? What is a capture?**

> *"A lambda is an anonymous function object defined inline. Syntax: [capture](params) -> return_type { body }. The capture list specifies which outer-scope variables the lambda can access: [=] captures all by value, [&] captures all by reference, [x, &y] captures x by value and y by reference.*
>
> *In LexiCore's benchmark.cpp:*
> ```cpp
> auto linearResult = timeQueries(exactQueries, [&](const std::string& q) {
>     volatile bool found = std::find(words.begin(), words.end(), q) != words.end();
> });
> ```
> *The [&] captures 'words' by reference from the outer scope. The lambda is passed as a template parameter Func — this is zero-overhead abstraction, the compiler inlines the lambda body at the call site. No virtual dispatch, no function pointer overhead."*

---

**Q33: What is `volatile` and why is it used in benchmarks?**

> *"volatile tells the compiler 'this variable may be read/written by something outside your control — do not optimize it away.' Without volatile, the compiler might notice that the result of a computation is never used and eliminate the entire computation as dead code — defeating the benchmark.*
>
> ```cpp
> volatile bool found = hashSet.count(q) > 0;
> (void)found;
> ```
> *The volatile forces the computation to actually execute. The (void)found suppresses the unused-variable warning. This is a standard benchmarking technique — you want to measure the real cost of the operation, not the cost of a no-op that the compiler optimized to nothing."*

---

**Q34: What is a template function? What is template type deduction?**

> *"A template function is a blueprint that generates concrete functions for different types at compile time. Example from benchmark.cpp:*
> ```cpp
> template <typename Func>
> TimingResult timeQueries(const vector<string>& queries, Func&& func) {
>     for (const auto& q : queries) func(q);
> }
> ```
> *When called with a lambda, the compiler deduces Func = the lambda's unique closure type and generates a specific instantiation. The && is a forwarding reference (universal reference) — it accepts both lvalues and rvalues. This pattern is called a generic algorithm with a callback, similar to std::sort's comparator. Zero overhead compared to virtual dispatch because everything is resolved at compile time."*

---

**Q35: What is `std::swap` and what does it do to vectors?**

> *"std::swap is a standard library function that exchanges two objects. For std::vector, the specialization is O(1) — it just swaps the internal pointers (data pointer, size, capacity). It does NOT copy any elements. This is why the rolling-row optimization in editDistance works efficiently:*
> ```cpp
> std::swap(prev, curr);  // O(1) — just pointer swap
> ```
> *Contrast with copying: prev = curr would be O(m) — copying all m+1 elements. The swap trick makes the rolling-row approach truly efficient."*

---

## 🔷 Section 7: Algorithm Complexity & Analysis

---

**Q36: What is the difference between O, Ω, and Θ notation?**

> *"Big-O (O) is an upper bound: f(n) = O(g(n)) means f grows no faster than g asymptotically. It characterizes worst-case or upper-bound scenarios. Big-Omega (Ω) is a lower bound: f grows at least as fast as g. Big-Theta (Θ) is a tight bound: f grows exactly like g — both O and Ω simultaneously.*
>
> *In practice, interviewers usually mean O when they say 'complexity.' But be precise: 'hash table lookup is O(1) average' means on average — worst case is O(n). Trie search is Θ(k) — both upper and lower bounded by the prefix length k. Edit distance DP is Θ(n·m) — you can't skip any cells in the naive algorithm."*

---

**Q37: What is amortized complexity? Give an example.**

> *"Amortized complexity analyzes the average cost per operation over a sequence of operations, even if individual operations vary. The classic example is dynamic array (std::vector) push_back: most insertions are O(1), but occasionally when the array is full, it doubles in size and copies all elements — O(n). However, you double so infrequently that if you do n push_backs total, the total work is O(n), giving O(1) amortized per operation.*
>
> *In LexiCore, the hash table insert during dictionary loading is O(1) amortized — most inserts are O(1), but periodic rehashing is O(n). Over 88K insertions, the total work is O(n), so O(1) amortized."*

---

**Q38: What is the difference between best, average, and worst-case complexity?**

> *"Best case is the complexity on the most favorable input — usually not interesting in practice. Average case is over a random or typical distribution of inputs — often what we care about in practice. Worst case is the maximum complexity over all possible inputs — critical for guarantees.*
>
> *Example: unordered_map lookup: best/average case O(1), worst case O(n) if all keys collide. Quicksort: average O(n log n), worst case O(n²) on sorted input — why std::sort switches to heapsort. BK-tree search: average practical, worst case O(n) when maxDistance is large or tree is degenerate."*

---

**Q39: What is p50, p95, p99 latency? Why does it matter more than average?**

> *"Percentile latency: p50 (median) means 50% of requests finish faster than this value. p95 means 95% of requests are faster. p99 means 99% are faster. These are called tail latency metrics.*
>
> *Why not just use average? Because averages hide outliers. If 99% of queries take 1ms and 1% take 10 seconds, the average might be 0.1 seconds — misleading. In user-facing applications, even rare slow requests harm user experience. For a search engine, p99 = 8.8ms (BK-tree at 88K words from RESULTS.md) means 1 in 100 queries takes 8.8ms — acceptable. But if p99 were 1 second, users would notice. Systems engineers track tail latency to identify and eliminate outliers."*

---

## 🔷 Section 8: File I/O, Strings & Normalization

---

**Q40: How does dictionary loading and normalization work?**

> *"The Dictionary class opens the file with std::ifstream, reads line by line with std::getline. For each line, it calls normalize() which: iterates each character, keeps only alphabetic characters (std::isalpha), converts to lowercase (std::tolower). It uses the result from unordered_set::insert to deduplicate — if .second is false, the word was already seen, so it's skipped. Both a vector (preserving load order) and an unordered_set (for O(1) lookup) are maintained simultaneously. This dual-structure approach is a common pattern: vector for ordered iteration and cache-friendly traversal, set for fast membership test."*

---

**Q41: Why do we cast to `unsigned char` before calling `isalpha` and `tolower`?**

> *"The functions isalpha() and tolower() from <cctype> take an int argument. The C standard says passing a value not representable as unsigned char (other than EOF) is undefined behavior. In C++, char can be signed on many platforms — so if a character has value > 127 (like accented letters in UTF-8), it appears as a negative int when widened. Passing a negative value to isalpha is UB. Casting to unsigned char first ensures the value is in range [0, 255], which is always valid:*
> ```cpp
> std::isalpha(static_cast<unsigned char>(c))
> std::tolower(static_cast<unsigned char>(c))
> ```
> *This is a subtle but important correctness detail that LexiCore gets right — and many production codebases get wrong."*

---

## 🔷 Section 9: Build Systems & Testing

---

**Q42: What is CMake and what does a basic CMakeLists.txt do?**

> *"CMake is a build system generator — it doesn't build code directly but generates build files for your native build system (Makefiles on Linux, Visual Studio projects on Windows, Xcode on Mac). The CMakeLists.txt describes: the project name and C++ standard, library targets (add_library) with their source files, executable targets (add_executable), include directories (target_include_directories), and linking relationships (target_link_libraries).*
>
> *In LexiCore, all core source files compile into lexicore_lib (a static library). Both the main executable and all test executables link against this single library — avoiding recompilation and ensuring tests exercise the exact same binary as the application."*

---

**Q43: What is AddressSanitizer (ASan) and what does it detect?**

> *"AddressSanitizer is a compiler instrumentation tool (enabled with -fsanitize=address) that inserts checks around every memory access at runtime. It detects: heap buffer overflow (reading/writing past allocated memory), stack buffer overflow, use-after-free (accessing freed memory), use-after-return (accessing local variable after function returns), memory leaks (with LeakSanitizer, often bundled). It has roughly 2× runtime overhead and ~3× memory overhead — acceptable for debug/test builds but not production.*
>
> *LexiCore's test command:*
> ```bash
> cmake -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -g"
> ```
> *This enables both ASan (memory safety) and UBSan (undefined behavior: signed overflow, null dereference, type violations). All tests pass clean — this is a meaningful quality signal."*

---

**Q44: What is the difference between unit tests, integration tests, and property-based tests?**

> *"Unit tests: test a single function or class in isolation with known inputs and expected outputs. Example: editDistance('cat', 'bat') == 1. They're fast and pinpoint failures precisely.*
>
> *Integration tests: test multiple components working together. LexiCore's test_correctness.cpp is effectively an integration test — it builds a BK-tree, runs 500 words × 200 queries × 4 thresholds, and compares results against linear scan (the oracle).*
>
> *Property-based tests: instead of specific input-output pairs, you verify mathematical properties hold over many random inputs. LexiCore's test_edit_distance.cpp tests: symmetry (dist(a,b) == dist(b,a)) and triangle inequality (dist(a,c) ≤ dist(a,b) + dist(b,c)) over 1000 random pairs/triples. If either property fails, your edit distance implementation is wrong — regardless of which specific inputs caused it."*

---

## 🔷 Section 10: System Design Fundamentals (Bonus)

---

**Q45: If you had to design a production spell-checker, what would you use?**

> *"I'd think about this in layers. For the dictionary: start with a curated word list plus user-specific words. For exact lookup: a hash set — O(1). For suggestions: a BK-tree for fuzzy matching, but with a few production enhancements: (1) Limit maxDistance to 2 or 3 — beyond that, suggestions aren't useful. (2) Rank by word frequency (unigram model) not just edit distance — 'the' should rank higher than 'thee' for a 1-edit match. (3) Consider phonetic similarity (Soundex or Metaphone) for words that sound like the query. (4) Cache recent queries. For scale — millions of queries per second — you'd shard the BK-tree, serve from memory on each server, and run behind a CDN/cache layer. LexiCore's design is the correct foundation — the ranking and BK-tree modules map directly to a production system."*

---

**Q46: What trade-offs did LexiCore make, and how would you improve them?**

> *"Three main trade-offs I'd discuss: First, memory: unordered_map per Trie node is flexible but has overhead (~50-100 bytes per map). For a production dictionary, a compressed/radix trie would reduce node count from 185K to closer to the word count. Second, BK-tree vs other fuzzy search approaches: BK-tree gives 2-2.5× speedup over linear scan, but modern production systems use techniques like n-gram indexes or SymSpell which can be orders of magnitude faster by precomputing candidates. Third, ranking: LexiCore ranks by edit distance then lexicographic order — clean and deterministic, but ignores word frequency. Real autocomplete systems weight by click-through rates and contextual signals. I'd present these as conscious engineering choices appropriate for the project's goals — learning and benchmarking — not production oversights."*

---

## ✅ Pre-Phase-1 Checklist

Before moving to Phase 1, verify you can answer from memory:

- [ ] Write the full Levenshtein DP recurrence and the rolling-row optimization
- [ ] Explain why the bounded variant returns a sentinel and why you can't use it for BK-tree pruning
- [ ] Implement Trie insert and search in C++ using unique_ptr
- [ ] Explain BK-tree triangle inequality pruning in one clean sentence
- [ ] Explain why unique_ptr prevents leaks, double-frees, and use-after-free
- [ ] Differentiate O, Ω, Θ and give an example of each
- [ ] What does `unordered_set::insert().second` return?
- [ ] What is structured binding syntax and what does it replace?
- [ ] What does `volatile` do in a benchmark context?
- [ ] What is p99 latency and why is it more useful than average latency?

---

> ✅ **Phase 0 complete!**
> Say **"proceed to Phase 1"** to dive into `edit_distance.cpp`, `bktree.cpp`, and `trie.cpp` — the algorithmic core — with full code walkthroughs and intense interview Q&A.
