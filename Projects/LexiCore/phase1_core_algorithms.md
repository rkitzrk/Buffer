# 📚 LexiCore Study Guide — Phase 1
### *Core Algorithms: Edit Distance + Trie + BK-Tree*
### *Line-by-Line Walkthroughs + 52 Intense SDE Interview Q&As*

---

## 📂 Files in This Phase (Read in This Order)

| Order | File | Why This Order |
|---|---|---|
| 1 | [`include/edit_distance.hpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/include/edit_distance.hpp) | Contract first — what the functions promise |
| 2 | [`src/edit_distance.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/src/edit_distance.cpp) | Implementation — the DP engine everything depends on |
| 3 | [`include/trie.hpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/include/trie.hpp) | Trie interface |
| 4 | [`src/trie.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/src/trie.cpp) | Trie implementation |
| 5 | [`include/bktree.hpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/include/bktree.hpp) | BK-tree interface |
| 6 | [`src/bktree.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/src/bktree.cpp) | BK-tree — the most complex piece |
| 7 | [`tests/test_edit_distance.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/tests/test_edit_distance.cpp) | See property-based testing in action |
| 8 | [`tests/test_trie.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/tests/test_trie.cpp) | Edge cases that reveal design intent |
| 9 | [`tests/test_bktree.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/tests/test_bktree.cpp) | BK-tree unit tests |
| 10 | [`tests/test_correctness.cpp`](file:///home/kumaresh/Desktop/Dev/Personal/LexiCore/tests/test_correctness.cpp) | The oracle — BK-tree verified against linear scan |

---

---

# 🔬 Part 1 — `edit_distance.hpp` + `edit_distance.cpp`

## Header — The Contract

```cpp
// editDistance: full O(n·m) Levenshtein — always returns the true distance
int editDistance(const std::string& a, const std::string& b);

// editDistanceBounded: returns true distance if ≤ maxDist, else SENTINEL (maxDist+1)
// ⚠️ NEVER use the sentinel for BK-tree pruning intervals
int editDistanceBounded(const std::string& a, const std::string& b, int maxDist);
```

The comment in the header is a contract: it documents the sentinel behavior and explicitly warns against misuse. This is good API design — the footgun is documented.

---

## Implementation — Line-by-Line Walkthrough

### `editDistance()` — Standard Levenshtein with Rolling Rows

```cpp
int editDistance(const std::string& a, const std::string& b) {
    const size_t n = a.size();
    const size_t m = b.size();
```
> `n` = rows (length of a), `m` = columns (length of b). The DP table is conceptually (n+1) × (m+1).

```cpp
    std::vector<int> prev(m + 1);   // represents dp[i-1][*]
    std::vector<int> curr(m + 1);   // represents dp[i][*]
```
> Instead of an n×m matrix, only TWO rows are kept. This is the **space optimization** — O(m) instead of O(n·m).

```cpp
    for (size_t j = 0; j <= m; ++j) {
        prev[j] = static_cast<int>(j);
    }
```
> Base case: `dp[0][j] = j` — transforming empty string into first j chars of b = j insertions.

```cpp
    for (size_t i = 1; i <= n; ++i) {
        curr[0] = static_cast<int>(i);   // dp[i][0] = i (i deletions)
        for (size_t j = 1; j <= m; ++j) {
            int cost = (a[i - 1] == b[j - 1]) ? 0 : 1;
            curr[j] = std::min({
                prev[j] + 1,         // delete a[i-1]   → look at dp[i-1][j]
                curr[j - 1] + 1,     // insert b[j-1]   → look at dp[i][j-1]
                prev[j - 1] + cost   // substitute      → look at dp[i-1][j-1]
            });
        }
        std::swap(prev, curr);   // O(1) pointer swap — prev becomes current row
    }
    return prev[m];   // After last swap, prev holds the final row
```

> **Key insight**: `std::swap(prev, curr)` is **O(1)** — it just swaps vector internals (pointer, size, capacity). Not a copy. After the last iteration, `prev` holds what was just computed as `curr`, so `prev[m]` is the answer.

### Full DP Table Trace — "cat" → "bat"
```
     ""  b  a  t
""  [ 0, 1, 2, 3]   ← prev (initial)
c   [ 1, 1, 2, 3]   ← compute curr, then swap
a   [ 2, 2, 1, 2]
t   [ 3, 3, 2, 1]   ← final prev[3] = 1  ✓
```

---

### `editDistanceBounded()` — Diagonal Band Optimization

```cpp
    if (std::abs(n - m) > maxDist) {
        return maxDist + 1;   // Early exit: length diff alone exceeds threshold
    }
```
> This is the cheapest possible check — O(1) before any work is done.

```cpp
    std::vector<int> prev(m + 1, maxDist + 1);  // fill with sentinel
    std::vector<int> curr(m + 1, maxDist + 1);
```
> Pre-fill with sentinel so cells outside the band are already "invalid" without explicit checks.

```cpp
        int jMin = std::max(1, i - maxDist);
        int jMax = std::min(m, i + maxDist);
```
> **The diagonal band**: for row `i`, only compute columns within `[i-maxDist, i+maxDist]`. Cells outside this band can never contribute to a solution within `maxDist` — they'd already be too far.

```cpp
        bool anyValid = false;
        for (int j = jMin; j <= jMax; ++j) {
            ...
            if (curr[j] <= maxDist) anyValid = true;
        }
        if (!anyValid) return maxDist + 1;  // Early termination
```
> If no cell in this row is within threshold, future rows can only get worse (costs increase monotonically in a band). Bail out immediately.

```cpp
    return (prev[m] <= maxDist) ? prev[m] : maxDist + 1;
```
> Final answer: return true distance if within threshold, else sentinel.

---

---

# 🔬 Part 2 — `trie.hpp` + `trie.cpp`

## Header — The Interface

```cpp
struct TrieNode {
    std::unordered_map<char, std::unique_ptr<TrieNode>> children;
    bool isEndOfWord = false;
};
```
> Each node owns its children via `unique_ptr`. The entire tree is destroyed recursively when root is destroyed — zero manual `delete`. `unordered_map` for flexible alphabet (not hardcoded to 26).

```cpp
class Trie {
public:
    void insert(const std::string& word);
    bool search(const std::string& word) const;
    bool startsWith(const std::string& prefix) const;
    std::vector<std::string> autocomplete(const std::string& prefix, size_t limit = 20) const;
    size_t getNodeCount() const;

private:
    const TrieNode* findNode(const std::string& prefix) const;
    void collectWords(const TrieNode*, const std::string& prefix,
                      std::vector<std::string>& results, size_t limit) const;
    size_t countNodes(const TrieNode*) const;

    std::unique_ptr<TrieNode> root_;
};
```
> **Design pattern**: The private `findNode()` helper is shared between `search()`, `startsWith()`, and `autocomplete()` — **DRY** (Don't Repeat Yourself). Public functions are thin wrappers.

---

## Implementation — Line-by-Line Walkthrough

### Constructor
```cpp
Trie::Trie() : root_(std::make_unique<TrieNode>()) {}
```
> Root node is always pre-created. This simplifies insert/search — you never need to check if root is null.

---

### `insert()`
```cpp
void Trie::insert(const std::string& word) {
    TrieNode* current = root_.get();   // raw pointer — we don't own, just traverse
    for (char c : word) {
        auto& child = current->children[c];   // operator[] creates entry if missing
        if (!child) {
            child = std::make_unique<TrieNode>();   // create node only if absent
        }
        current = child.get();   // move down
    }
    current->isEndOfWord = true;   // mark the end
}
```
> `current->children[c]` — operator[] on unordered_map **inserts a default value** (null unique_ptr) if key `c` doesn't exist. Then we check `!child` and allocate if needed. This is idiomatic but has a subtle side effect: you always touch the map even for existing keys.

---

### `search()` and `startsWith()`
```cpp
bool Trie::search(const std::string& word) const {
    const TrieNode* node = findNode(word);
    return node != nullptr && node->isEndOfWord;  // MUST check isEndOfWord!
}

bool Trie::startsWith(const std::string& prefix) const {
    return findNode(prefix) != nullptr;            // node existing is enough
}
```
> **Critical distinction**: `search("app")` returns false even if "apple" is in the trie — because the node at "app" has `isEndOfWord = false`. `startsWith("app")` returns true.

---

### `findNode()` — The Private Workhorse
```cpp
const TrieNode* Trie::findNode(const std::string& prefix) const {
    const TrieNode* current = root_.get();
    for (char c : prefix) {
        auto it = current->children.find(c);       // find without inserting (unlike [])
        if (it == current->children.end()) {
            return nullptr;                         // prefix not in trie
        }
        current = it->second.get();               // advance to child node
    }
    return current;
}
```
> Uses `find()` not `operator[]` because `const` member functions cannot mutate. `operator[]` inserts on miss — not allowed on const. `find()` returns an iterator or `end()`.

---

### `autocomplete()` — Prefix Walk + DFS Collection
```cpp
std::vector<std::string> Trie::autocomplete(const std::string& prefix,
                                             size_t limit) const {
    std::vector<std::string> results;
    const TrieNode* node = findNode(prefix);
    if (node) {
        collectWords(node, prefix, results, limit);  // DFS from prefix node
    }
    std::sort(results.begin(), results.end());       // sort for deterministic output
    return results;
}
```

### `collectWords()` — Recursive DFS
```cpp
void Trie::collectWords(const TrieNode* node, const std::string& prefix,
                         std::vector<std::string>& results, size_t limit) const {
    if (results.size() >= limit) return;    // limit guard — prune early

    if (node->isEndOfWord) {
        results.push_back(prefix);          // this path spells a complete word
    }

    for (const auto& [ch, child] : node->children) {
        if (results.size() >= limit) return;
        collectWords(child.get(), prefix + ch, results, limit);  // extend prefix by ch
    }
}
```
> `prefix + ch` creates a new string for each recursive call — O(k) string allocation per word. In production you'd use a `string& current` and push/pop the last char (backtracking), avoiding allocations. LexiCore's approach is simpler and fine at this scale.

**DFS Trace for autocomplete("app") with words "apple", "application", "apply":**
```
collectWords(node_at_"app", "app")
  node->isEndOfWord? No
  children: {'l': node_at_"appl"}
    collectWords(node_at_"appl", "appl")
      node->isEndOfWord? No
      children: {'e': node_at_"apple", 'i': node_at_"appli", 'y': node_at_"apply"}
        collectWords(node_at_"apple", "apple")
          isEndOfWord? YES → push "apple"
        collectWords(node_at_"appli", "appli")
          ... → push "application"
        collectWords(node_at_"apply", "apply")
          isEndOfWord? YES → push "apply"
Sort: ["apple", "application", "apply"] ✓
```

---

### `countNodes()` — Recursive Count
```cpp
size_t Trie::countNodes(const TrieNode* node) const {
    if (!node) return 0;
    size_t count = 1;                            // count this node
    for (const auto& [ch, child] : node->children) {
        count += countNodes(child.get());        // recurse
    }
    return count;
}
```
> Simple post-order traversal. For 88K English words, LexiCore has 185,264 nodes — node sharing at shared prefix nodes means far fewer nodes than total character count.

---

---

# 🔬 Part 3 — `bktree.hpp` + `bktree.cpp`

## Header — The Interface

```cpp
struct BKNode {
    std::string word;
    std::unordered_map<int, std::unique_ptr<BKNode>> children;  // keyed by edit distance

    explicit BKNode(std::string w) : word(std::move(w)) {}      // move — no copy
};
```
> Children keyed by `int` (the edit distance from parent to child). `explicit` prevents accidental implicit conversion. `std::move(w)` transfers string ownership into `word` — no copy.

```cpp
class BKTree {
public:
    void insert(const std::string& word);
    std::vector<std::pair<std::string, int>> search(const std::string& query, int maxDistance) const;
    size_t getNodeCount() const;
    size_t getMaxDepth() const;
    bool empty() const { return root_ == nullptr; }   // inline — trivial check

private:
    void insertImpl(BKNode* node, const std::string& word);
    void searchImpl(const BKNode* node, const std::string& query,
                    int maxDistance,
                    std::vector<std::pair<std::string, int>>& results) const;
    size_t countNodes(const BKNode*) const;
    size_t maxDepthImpl(const BKNode*) const;

    std::unique_ptr<BKNode> root_;
};
```
> Same public/private split as Trie. The `Impl` suffix is a convention for recursive helpers that need a node pointer entry point.

---

## Implementation — Line-by-Line Walkthrough

### `insert()` — Root Handling
```cpp
void BKTree::insert(const std::string& word) {
    if (!root_) {
        root_ = std::make_unique<BKNode>(word);   // first word becomes root
        return;
    }
    insertImpl(root_.get(), word);
}
```
> The root case is special because there's no parent to compute distance from. All subsequent insertions go through `insertImpl`.

---

### `insertImpl()` — Recursive Insertion
```cpp
void BKTree::insertImpl(BKNode* node, const std::string& word) {
    int dist = editDistance(node->word, word);   // distance from current node to new word
    if (dist == 0) {
        return;                                   // duplicate — silently skip
    }
    auto it = node->children.find(dist);
    if (it == node->children.end()) {
        node->children[dist] = std::make_unique<BKNode>(word);  // new edge with label dist
    } else {
        insertImpl(it->second.get(), word);       // recurse: slot occupied, go deeper
    }
}
```

**Insertion Trace — Insert "book", "cook", "back", "hook":**
```
Insert "book"  → root = "book"
Insert "cook"  → dist("book","cook") = 1 → no child@1 → create child@1 = "cook"
Insert "back"  → dist("book","back") = 2 → no child@2 → create child@2 = "back"
Insert "hook"  → dist("book","hook") = 1 → child@1 exists ("cook")
                  → recurse: dist("cook","hook") = 1 → no child@1 → create child@1 = "hook"

Tree structure:
"book" (root)
  ├── [1] "cook"
  │     └── [1] "hook"
  └── [2] "back"
```

---

### `searchImpl()` — The Triangle Inequality Pruning

```cpp
void BKTree::searchImpl(const BKNode* node, const std::string& query,
                         int maxDistance,
                         std::vector<std::pair<std::string, int>>& results) const {
    // MUST use true editDistance(), not bounded. See header comment.
    int d = editDistance(query, node->word);
```
> This computes the **true** distance from query to this node's word. `d` is the exact value needed for the pruning interval.

```cpp
    if (d <= maxDistance) {
        results.emplace_back(node->word, d);   // this word is a match
    }
```
> Match check. `emplace_back` constructs the pair in-place — more efficient than `push_back({word, d})`.

```cpp
    int low  = d - maxDistance;
    int high = d + maxDistance;

    for (const auto& [childDist, childNode] : node->children) {
        if (childDist >= low && childDist <= high) {
            searchImpl(childNode.get(), query, maxDistance, results);
        }
    }
}
```
> **The core magic.** For each child with edge label `childDist`:
> - By triangle inequality: `dist(query, child) ≥ |dist(query, node) - dist(node, child)| = |d - childDist|`
> - If `|d - childDist| > maxDistance`, child is provably too far — skip.
> - Equivalently: only recurse if `childDist ∈ [d - maxDistance, d + maxDistance]`

**Search Trace — search("book", maxDist=1) on above tree:**
```
searchImpl("book" [root])
  d = editDistance("book","book") = 0 → 0 ≤ 1 → MATCH ✓
  low = 0-1 = -1, high = 0+1 = 1
  Children: [1]"cook", [2]"back"
  childDist=1: 1 ∈ [-1,1] → recurse "cook"
  childDist=2: 2 ∉ [-1,1] → PRUNE "back" ✓

searchImpl("cook" node)
  d = editDistance("book","cook") = 1 → 1 ≤ 1 → MATCH ✓
  low = 1-1 = 0, high = 1+1 = 2
  Children: [1]"hook"
  childDist=1: 1 ∈ [0,2] → recurse "hook"

searchImpl("hook" node)
  d = editDistance("book","hook") = 1 → 1 ≤ 1 → MATCH ✓
  No children → done

Results: ["book"(0), "cook"(1), "hook"(1)]   ("back" correctly pruned)
```

---

### `maxDepthImpl()` — Tree Health Check
```cpp
size_t BKTree::maxDepthImpl(const BKNode* node) const {
    if (!node) return 0;
    size_t maxChildDepth = 0;
    for (const auto& [dist, child] : node->children) {
        maxChildDepth = std::max(maxChildDepth, maxDepthImpl(child.get()));
    }
    return 1 + maxChildDepth;   // this node + deepest child subtree
}
```
> Standard recursive max-depth. For LexiCore's 88K shuffled words: depth = 19. Without shuffling (alphabetical order): potentially 1000+. The `getMaxDepth()` function is a diagnostic tool to detect degenerate trees.

---

---

# 🔥 Part 4 — INTENSE SDE Interview Q&A
### *All questions from the Phase 1 files — scripted verbatim answers*

---

## 🔷 Section A: Edit Distance Deep-Dive

---

**Q1: Walk me through the `editDistance()` implementation in this codebase. Why two vectors?**

> *"The implementation uses the rolling-row space optimization. A full n×m DP table needs O(n·m) space, but the recurrence for row i only reads from row i-1 — so we only need two rows at a time. We maintain prev (the previously computed row) and curr (the row being computed). After each outer loop iteration, we call std::swap(prev, curr), which is O(1) because it just swaps vector internals — no element copying. After the final iteration, prev holds the last computed row, so prev[m] is the answer. This reduces space from O(n·m) to O(m)."*

---

**Q2: The code has `std::swap(prev, curr)` at the end of the loop. What exactly does std::swap do to vectors?**

> *"For std::vector, std::swap is specialized to be O(1). A vector internally stores three things: a pointer to heap-allocated data, a size, and a capacity. std::swap exchanges all three in constant time — it does NOT copy any elements. So after swap, what was prev is now curr and vice versa, with zero element movement. This is fundamentally different from prev = curr which would copy all m+1 integers, making the loop O(n·m) just for the swaps. The O(1) swap is what makes the rolling-row technique practical."*

---

**Q3: Why does `editDistanceBounded()` pre-fill vectors with `maxDist + 1` instead of 0?**

> *"The bounded variant only computes cells within a diagonal band [i-maxDist, i+maxDist] for each row i. Cells outside the band are never written. By pre-filling with maxDist+1 (the sentinel), cells outside the band already have an 'invalid' value — they're treated as 'too far' automatically. If we pre-filled with 0, those unwritten cells would contribute incorrect values (appearing as 'distance 0 = perfect match') when read by adjacent cells in subsequent iterations. The sentinel fill ensures the band boundary is handled correctly without explicit range checks in the inner loop."*

---

**Q4: The bounded variant has an `anyValid` flag and early returns. What's the mathematical justification?**

> *"Edit distance values in the DP table are non-decreasing toward the edges. Within the diagonal band, if every cell in row i exceeds maxDist, then every cell in row i+1 will also exceed maxDist — because to reach row i+1 you can only add 1 to existing values (deletion/insertion/substitution all cost ≥ 0). So if no cell in the current row is ≤ maxDist, the final answer dp[n][m] is definitely > maxDist. We can bail out with sentinel immediately. This is a valid mathematical shortcut, not a heuristic."*

---

**Q5: In the test file, `editDistanceBounded("cat", "dog", 1)` returns 2. The true distance is 3. Explain.**

> *"editDistanceBounded returns maxDist+1 as a sentinel when the true distance exceeds maxDist. For 'cat' and 'dog' with maxDist=1, the true edit distance is 3 — three substitutions (c→d, a→o, t→g). Since 3 > 1, the bounded function returns 1+1 = 2. It is NOT returning the true distance of 3. This is documented as sentinel behavior. The test asserts `== 2` which is the sentinel value maxDist+1, confirming correct sentinel behavior — NOT that the distance is 2."*

---

**Q6: What is `std::min({a, b, c})` with the initializer list? How does it differ from `std::min(a, std::min(b, c))`?**

> *"Both compute the minimum of three values, but the initializer list version `std::min({a, b, c})` is more readable and avoids nested calls. It uses the `std::initializer_list<T>` overload of std::min which iterates over the list. Performance-wise they're equivalent — the compiler optimizes both to compare-and-select operations. The initializer list version is preferred in modern C++ for clarity when comparing more than two values."*

---

**Q7: The header file has a comment saying NOT to use `editDistanceBounded` for BK-tree pruning. Can you trace exactly what goes wrong if you do?**

> *"Absolutely. Suppose query Q = 'aple', current node N = 'sample', maxDist = 1.*
>
> *Correct: editDistance('aple','sample') = 4. Pruning interval = [4-1, 4+1] = [3,5]. Only recurse into children with edge labels in [3,5].*
>
> *Wrong (using bounded): editDistanceBounded('aple','sample', 1) = 2 (sentinel, since true dist 4 > 1). Pruning interval = [2-1, 2+1] = [1,3]. We'd recurse into children with edges 1,2,3 — including some that are actually distance 6-7 from the query. We'd also skip children with edges 4,5 that might contain valid matches distance 1 from query.*
>
> *The result: missed results (false negatives) and wasted computation on irrelevant subtrees. The search would be both incorrect AND slower. This is a real correctness bug that's hard to detect without a comprehensive oracle test like test_correctness.cpp."*

---

**Q8: The property-based tests check symmetry and triangle inequality. Why test properties instead of just input-output pairs?**

> *"Input-output pairs test that you got specific cases right. Property-based tests verify mathematical invariants that must hold for ALL inputs — they can catch bugs that no specific test case would find. For edit distance, symmetry means dist(a,b) == dist(b,a). The DP table is not symmetric — dp_ab[i][j] is defined differently from dp_ba[i][j]. Yet the final result must be symmetric. If we accidentally broke symmetry in an optimization (say by not swapping rows correctly), a specific test might not catch it. 1000 random pairs dramatically increases the probability of hitting the failure mode. Similarly, the triangle inequality test would catch a broken pruning implementation in BK-tree if we were testing that here."*

---

## 🔷 Section B: Trie Deep-Dive

---

**Q9: In `insert()`, `current->children[c]` is called on an unordered_map. What are the implications?**

> *"operator[] on unordered_map has a very important side effect: if the key `c` doesn't exist, it inserts a default-constructed value (a null unique_ptr in this case). So the expression `auto& child = current->children[c]` either returns a reference to the existing unique_ptr OR inserts a null unique_ptr and returns a reference to it. We then check `!child` to decide whether to allocate. This means insert() is NOT safe to call on a const Trie — it silently modifies the map. This is why findNode() uses `find()` instead of `operator[]`: findNode() is called from const member functions like search() and startsWith()."*

---

**Q10: Why does `findNode()` use `find()` instead of `operator[]`?**

> *"Two reasons. First, `findNode()` is called from `search()` and `startsWith()` which are marked `const` — meaning they cannot modify the object. `operator[]` would insert missing keys, which is a mutation — not allowed in a const method. The compiler would reject it. Second, even ignoring const, `operator[]` would create spurious empty nodes for every character of a queried prefix that doesn't exist in the trie. This would corrupt the trie's structure by inserting empty paths that don't correspond to any word. `find()` is read-only — it returns end() if the key is absent, without modifying the map."*

---

**Q11: `search("app")` returns false even though "apple" is in the trie. Why? Walk through the code.**

> *"search() calls findNode('app'). findNode traverses: root → 'a' node → 'p' node → 'p' node, and returns the node representing the prefix 'app'. This node is NOT null — it exists as an intermediate node on the path to 'apple'. But search() then checks `node->isEndOfWord`. That flag is only set to true when the last character of an inserted word is processed. 'app' was never inserted as a word — only 'apple' was. So isEndOfWord at the 'app' node is false. search() returns false. startsWith('app') on the other hand only checks if findNode returns non-null, which it does — so startsWith returns true."*

---

**Q12: The autocomplete DFS builds `prefix + ch` at each recursive call. What's the performance implication? How would you optimize it?**

> *"prefix + ch creates a brand-new std::string by copying all characters of prefix plus ch. This is O(k) per call where k is the current depth. With D total words collected, the total string allocation cost is O(total characters across all collected words). For a broad prefix like 'a' collecting thousands of words, this becomes expensive.*
>
> *The optimization: use a `std::string& current` parameter passed by reference. Append the character before recursing, then remove it after — this is backtracking:*
> ```cpp
> void collectWords(const TrieNode* node, std::string& current, ...) {
>     if (node->isEndOfWord) results.push_back(current);
>     for (auto& [ch, child] : node->children) {
>         current.push_back(ch);    // extend
>         collectWords(child.get(), current, ...);
>         current.pop_back();       // restore
>     }
> }
> ```
> *push_back/pop_back on string are O(1) amortized. This reduces total allocation to O(max_depth) instead of O(depth × branching). LexiCore's version is simpler but less efficient for broad queries."*

---

**Q13: The test shows `trie.getNodeCount() == 1` for an empty trie. Why is there already one node?**

> *"The constructor `Trie() : root_(std::make_unique<TrieNode>())` pre-allocates the root node. The root doesn't represent any character — it's the starting point for all traversals. An empty trie still has this root node, so getNodeCount() == 1. This design choice simplifies all other functions: insert/search/findNode can always start from root_.get() without checking for null. The alternative — lazy root initialization — would require null checks in every traversal function."*

---

**Q14: The test checks `trie.startsWith("") == true`. Is this the right behavior? Why?**

> *"Yes, this is correct. An empty prefix means 'does any word in the trie start with nothing?' — and every word starts with an empty prefix. Technically, findNode('') traverses zero characters and returns root_, which is always non-null (for a pre-initialized trie). So startsWith('') is always true as long as the trie exists. This is consistent with the mathematical definition: the empty string is a prefix of every string. In an autocomplete UI, querying with an empty prefix should return all words — this is the correct behavior."*

---

**Q15: Trie nodes use `unordered_map<char, unique_ptr<TrieNode>>`. What happens to memory when the Trie goes out of scope?**

> *"When the Trie object is destroyed, its destructor runs. root_ is a unique_ptr, so its destructor is called, which calls delete on the root TrieNode. The root TrieNode's destructor runs, which destroys its unordered_map member. Destroying the map destroys all key-value pairs, which calls the unique_ptr destructors for each child, which recursively destroys each child's TrieNode — and so on, depth-first. The entire tree is destroyed automatically, top-down, with zero manual delete calls. This is RAII in action: memory lifetime == object lifetime."*

---

**Q16: What is the node count for a trie containing "cat", "car", "card"? Walk through it.**

> *"Let me trace:*
> ```
> Insert "cat":  root → c → a → t(end)       Nodes: root, c, a, t = 4 nodes
> Insert "car":  root → c → a → r(end)       New: r = 5 nodes (c,a shared)
> Insert "card": root → c → a → r → d(end)   New: d = 6 nodes (c,a,r shared)
> ```
> *Total: 6 nodes. The 'c' and 'a' nodes are shared by all three words. 'r' is shared by 'car' and 'card'. 't' and 'd' are unique. This sharing of common prefixes is what makes the trie space-efficient compared to storing each word independently."*

---

## 🔷 Section C: BK-Tree Deep-Dive

---

**Q17: Walk through the BK-tree insertion of words "book", "cook", "back", "hook". Show the tree.**

> *"book becomes the root — no distance to compute.*
>
> *Insert 'cook': dist('book','cook') = 1 (one substitution: b→c). No child at edge 1. Create child 'cook' at edge 1.*
>
> *Insert 'back': dist('book','back') = 2 (b→b kept, o→a, o→c, k→k → two substitutions). No child at edge 2. Create child 'back' at edge 2.*
>
> *Insert 'hook': dist('book','hook') = 1 (b→h). Child at edge 1 already exists ('cook'). Recurse into 'cook': dist('cook','hook') = 1 (c→h). No child at edge 1 under 'cook'. Create child 'hook' at edge 1 under 'cook'.*
>
> *Final tree:*
> ```
> "book"
>   ├─[1]─ "cook"
>   │         └─[1]─ "hook"
>   └─[2]─ "back"
> ```"*

---

**Q18: Given the tree above, trace search("book", maxDist=1). Show which nodes are visited and which are pruned.**

> *"At root 'book': d = editDistance('book','book') = 0. 0 ≤ 1 → add 'book' to results.*
> *low = 0-1 = -1, high = 0+1 = 1.*
> *Children: edge 1 ('cook'), edge 2 ('back').*
> *Edge 1: 1 ∈ [-1, 1] → recurse into 'cook'.*
> *Edge 2: 2 ∉ [-1, 1] → PRUNE 'back'. Not visited at all.*
>
> *At 'cook': d = editDistance('book','cook') = 1. 1 ≤ 1 → add 'cook'.*
> *low = 0, high = 2.*
> *Children: edge 1 ('hook').*
> *Edge 1: 1 ∈ [0, 2] → recurse into 'hook'.*
>
> *At 'hook': d = editDistance('book','hook') = 1. 1 ≤ 1 → add 'hook'.*
> *No children. Done.*
>
> *Results: ['book'(0), 'cook'(1), 'hook'(1)]. 'back' was provably pruned — 3 nodes visited out of 4.*"

---

**Q19: Why does BK-tree insertImpl use `editDistance()` and not `editDistanceBounded()`?**

> *"During insertion, I need the EXACT edit distance from the current node to the new word to determine which edge label to assign the child. If I used the bounded variant, I'd get a sentinel value (maxDist+1) when the true distance exceeds some threshold — and I'd assign the wrong edge label to the new child. This corrupts the tree structure permanently. The bounded variant is only meaningful when you already know a threshold you care about — during insertion, there's no threshold; I need the true distance. The cost of using the full editDistance() during insertion is paid once at build time; search performance is what matters at query time."*

---

**Q20: What does `explicit BKNode(std::string w) : word(std::move(w)) {}` do? Why explicit? Why move?**

> *"`explicit` prevents the compiler from using this constructor for implicit conversions. Without it, `BKNode b = "hello"` would be valid — the compiler would implicitly construct a BKNode from the string literal. With explicit, you must write `BKNode b("hello")`. This prevents subtle bugs where a string is accidentally treated as a BKNode.*
>
> *`std::move(w)` — the constructor takes `w` by value, so the string has already been either copied or moved in when we reach the constructor body. Using `std::move(w)` to initialize the member `word` steals w's internal buffer instead of copying it again. So the total cost is: one copy or move (when caller passes argument) + one move (into member). Without move it would be one copy + one copy. For a long string, this saves a significant allocation."*

---

**Q21: How does duplicate insertion work in the BK-tree?**

> *"When insertImpl computes dist = editDistance(node->word, newWord) and gets 0, that means newWord is identical to node->word. The function simply returns without creating any child. So duplicates are silently discarded. The test verifies: insert 'hello' twice, then getNodeCount() == 1 — confirming only one node exists. This is important because having two nodes with the same word would cause search to return duplicate results, and the tree structure would be invalid — two children with the same edit distance from a parent would break the invariant that each edge label maps to exactly one child."*

---

**Q22: The `getMaxDepth()` function exists — what's its practical use?**

> *"It's a diagnostic for detecting degenerate trees. A balanced BK-tree over 88K words should have depth around log(88000) ≈ 17-20. If you insert words alphabetically, adjacent words have very similar edit distances, creating a chain — depth approaches n, making search O(n). getMaxDepth() lets you check tree health after construction. RESULTS.md shows shuffled insertion gives max depth 19 for 88K words — healthy. If you saw depth 5000, you'd know something went wrong with insertion order. It's like checking tree height after building a BST to detect the sorted-input degeneracy."*

---

**Q23: BK-tree uses recursion for both insert and search. What's the risk? How deep can recursion go?**

> *"The recursion depth for insert is bounded by the tree depth — 19 levels for LexiCore's 88K word tree. Each recursive call uses a stack frame: parameters (a few pointers), local variables (dist, it). At 19 levels deep, this is negligible — typical default stack size is 1-8 MB, and each frame is tens of bytes. For search, recursion depth is also bounded by tree depth (19). Compare this to a degenerate unshuffled tree: recursion depth could be 10,000+, risking a stack overflow. The shuffle is not just a performance optimization — it's also a safety measure against stack overflow in pathological cases."*

---

**Q24: In `searchImpl`, why is the result vector passed by reference instead of returned by value?**

> *"Passing by reference accumulates results across all recursive calls without copying. If we returned a vector by value, each recursive call would: (1) collect local results, (2) return a new vector to the parent, (3) the parent would merge the child's vector into its own. With many recursive calls, this creates O(depth × results_found) copying. Passing a reference means all recursive calls write directly into the same vector — zero copies, O(1) overhead per matched word. This is a common pattern for recursive accumulation: pass the accumulator by reference."*

---

**Q25: The correctness test in `test_correctness.cpp` uses a `linearFuzzySearch` as an oracle. What makes this valid?**

> *"An oracle is a reference implementation that is obviously correct, even if slow. linearFuzzySearch iterates over every word and computes editDistance — O(n·m) per query, O(n²·m) total — but it cannot possibly miss a result or return a wrong result, because it checks every word exhaustively. It's too slow for production but trivially correct. The test then compares BK-tree results against the oracle on 500 words × 200 queries × 4 thresholds = 400,000 comparisons. If BK-tree returns a different set for any case, there's a bug. This oracle testing pattern is standard in competitive programming too — you validate a fast solution against a brute-force for correctness."*

---

**Q26: The `resultSetsMatch()` function sorts both vectors before comparing. Why not just compare them directly?**

> *"BK-tree search visits nodes in tree traversal order, which depends on insertion order and the unordered_map iteration order — non-deterministic across runs. linearFuzzySearch iterates in vector order. Both return the same SET of results but potentially in different orders. Direct vector comparison (==) is order-sensitive — it would report a mismatch even when the results are identical sets. Sorting both by the same comparator (word, then distance) makes the comparison order-independent. Alternatively, you could convert both to unordered sets of words and compare, but sorting also validates that distances match — not just word membership."*

---

## 🔷 Section D: Design Patterns & Code Quality

---

**Q27: All three data structures (Trie, BKTree, Dictionary) use the same private/public split pattern. Explain this design.**

> *"This is the public interface / private implementation pattern. Public methods (insert, search, autocomplete) are the stable contract — they define what the class can DO. Private methods (insertImpl, searchImpl, findNode, collectWords) are the HOW — implementation details that can change without affecting users. The public methods handle edge cases (empty tree, null root) and then delegate to private recursive helpers that assume preconditions are met. This separation makes the code easier to test (you test the public interface), easier to modify (you can refactor private methods without changing the API), and communicates intent (users shouldn't call implementation details)."*

---

**Q28: None of the core classes have copy constructors or copy assignment operators. Is this intentional?**

> *"Yes. All three classes own their tree via a `std::unique_ptr`. unique_ptr is non-copyable by design — it enforces single ownership. When a class contains a unique_ptr member, the compiler implicitly deletes the copy constructor and copy assignment operator. This means you cannot accidentally copy a Trie or BKTree — which would be expensive (O(n) deep copy of the entire tree) and semantically ambiguous (should it be a shallow or deep copy?). The class is move-only: you can transfer ownership with std::move. This is intentional and correct — these are heavyweight data structures that should not be casually copied."*

---

**Q29: The benchmark warm-up pass — why is it necessary?**

> *"The first run of any computation suffers from 'cold' effects: CPU instruction cache misses (the code isn't in i-cache yet), data cache misses (the data isn't in L1/L2/L3 cache), branch predictor misses (the predictor hasn't learned the patterns), and OS page faults (memory pages are mapped but not loaded). A warm-up run exercises all of these, so the actual timed runs measure steady-state performance — the performance a real application sees after initial warm-up. Without warm-up, the first trial would be artificially slow, skewing the average. LexiCore runs 1 full pass before timing begins."*

---

**Q30: What C++20 features does LexiCore use?**

> *"The CMakeLists.txt sets CMAKE_CXX_STANDARD 20. The main C++20 features in LexiCore are: structured bindings `const auto& [k,v]` (actually C++17 but commonly associated with modern C++), range-based for loops, auto type deduction, initializer lists with std::min, and lambda functions with captures. The codebase doesn't use the newest C++20 features like concepts, ranges, or coroutines — it's written in modern-but-conservative style that would also compile as C++17 in most places. Setting C++20 ensures the latest standard library behavior and enables the compiler to use C++20 optimizations."*

---

## 🔷 Section E: Complexity & Performance Analysis

---

**Q31: What is the time complexity of building a Trie with n words of average length k?**

> *"Each word insertion is O(k) — one unordered_map lookup/insert per character. For n words, total build time is O(n·k). For LexiCore's 88K English words with average length ~7, that's roughly O(88000 × 7) = O(616,000) operations. RESULTS.md confirms: Trie build time is 39ms for 88K words. Memory: each node has an unordered_map, which has overhead ~50-100 bytes. 185,264 nodes × ~60 bytes = ~11MB — within acceptable bounds."*

---

**Q32: What is the time complexity of building a BK-tree with n words?**

> *"Each insertion traverses the tree from root to an insertion point. In a balanced tree, depth is O(log n), and each level requires one editDistance computation O(m) where m is word length. So per insertion: O(m · depth). For n insertions: O(n · m · log n). But in worst case (degenerate tree): O(n · m · n) = O(n²·m). This is why shuffling matters: it keeps depth ≈ log(n) in practice. RESULTS.md: BK-tree build for 88K words = 150ms. Compare Trie build = 39ms — BK-tree is slower to build because each insertion needs editDistance computations at each level."*

---

**Q33: The benchmark shows BK-tree is only 2-2.5× faster than linear scan. Why isn't it orders-of-magnitude faster like Trie vs linear?**

> *"The Trie achieves 49× speedup because prefix search is O(k) regardless of dictionary size — it genuinely avoids looking at most of the dictionary. The BK-tree's speedup depends entirely on how much the triangle inequality prunes. For English words with maxDist=2: the average inter-word edit distance is around 5-8, so the pruning interval [d-2, d+2] of width 4 covers roughly half of the possible edge label range. Many subtrees are still visited. Additionally, each visited node still requires an O(m) editDistance computation — you can't avoid that. If you increased maxDist to 5, pruning would weaken further and the BK-tree might actually be SLOWER than linear scan due to overhead."*

---

**Q34: From RESULTS.md, Trie performance is nearly identical at 10K vs 88K words (4.51ms vs 4.39ms). Explain why.**

> *"This is the O(k) property in action. Trie search cost depends only on the prefix length k, not the dictionary size n. Once you've traversed to the prefix node — which takes k steps — the DFS collection also depends on the output size, not n. Between 10K and 88K words, the autocomplete queries use prefixes of length 1-5 (from RESULTS.md methodology). The traversal cost is the same whether the dictionary has 10K or 88K words. The DFS might collect slightly more words at 88K, but the query set was the same 1000 queries. This empirically proves O(k) complexity — the Trie's killer feature."*

---

## 🔷 Section F: Tricky Edge Cases (High-Probability Interview Questions)

---

**Q35: What happens if you call `autocomplete("")` (empty prefix) on a Trie?**

> *"findNode('') traverses zero characters and returns root_. collectWords then does a DFS from root, collecting ALL words in the trie — because every word 'starts with' the empty prefix. With a limit of 20 (default), it returns the first 20 words found via DFS order, then sorts them. In LexiCore's main.cpp, this would return 20 words from the dictionary. It's technically correct behavior. The test file confirms: `trie.startsWith('') == true`."*

---

**Q36: What does BK-tree search with `maxDist=0` do?**

> *"maxDist=0 means: find all words with edit distance exactly 0 from the query — i.e., only the exact word itself. At each node, d = editDistance(query, node.word). We add to results only if d == 0. Pruning: low = d-0 = d, high = d+0 = d. We only recurse into children with edge label exactly d. This turns the BK-tree into an exact match lookup. The test verifies: `tree.search('apple', 0)` returns exactly one result with distance 0. Performance: O(depth) — exactly one path from root to the matching word (if it exists)."*

---

**Q37: What happens if you insert a very long string (say, 1000 characters) into both Trie and BK-tree?**

> *"Trie: creates 1000 new nodes (one per character) in a single chain. Cost O(1000) = O(k). Memory: 1000 nodes × ~60 bytes = ~60KB for this one word. The test verifies long words work correctly.*
>
> *BK-tree: insertImpl computes editDistance from the current node to the new word. For a 1000-character word, editDistance is O(1000 × 1000) = O(1,000,000) for each level of recursion. At depth 10, insert cost is O(10,000,000). Very expensive. Also, the unbounded recursion in insertImpl could stack overflow if the tree depth is large. The test_bktree.cpp 'very long words' test uses strings of length 50 — long enough to be meaningful, short enough to be safe."*

---

**Q38: Can you have two children with the same edge label in a BK-tree node? What does the code do?**

> *"No — and the code enforces this. In insertImpl, if `node->children.find(dist)` finds an existing child at that distance, we RECURSE into that child instead of creating a second one. The unordered_map from int → unique_ptr ensures at most one child per edge label. This is by design: BK-tree requires that each node has at most one child per distance value. If two words had the same distance from a parent, only the first would become a direct child; the second would be placed somewhere in the subtree rooted at the first. This is correct — it maintains the metric tree property."*

---

## 🔷 Section G: System-Level Questions

---

**Q39: How would you make the Trie thread-safe for concurrent reads and writes?**

> *"For concurrent reads only (multiple threads calling search/autocomplete simultaneously), the current implementation is already safe — all read operations are marked const and don't modify any state. For mixed read-write (concurrent insert while searching), we need synchronization. The standard approach: a single std::shared_mutex (C++17). Writers acquire an exclusive lock (std::unique_lock) for insert. Readers acquire a shared lock (std::shared_lock) for search/autocomplete — multiple readers can proceed simultaneously. This is the Reader-Writer pattern. For high-throughput production systems, you'd use a more granular lock (per-subtree) or a lock-free structure."*

---

**Q40: How would you serialize a Trie to disk and reload it?**

> *"Serialization: DFS traversal. For each node, write: the character that leads to this node, the isEndOfWord flag, the number of children, then recursively serialize each child. Alternatively, write all words (strings) to a file — faster to implement and the Trie is rebuilt on load. Loading: read all words and call insert() for each — O(n·k) rebuild time.*
>
> *For large production dictionaries, a more efficient approach: store the trie as a serialized array (like a left-child right-sibling tree) or as a DAWG (Directed Acyclic Word Graph) which compresses common suffixes too. LexiCore doesn't implement serialization — it rebuilds from words.txt on every startup, which takes 39ms — acceptable."*

---

**Q41: You're asked to add a 'delete word' operation to the Trie. How would you implement it?**

> *"Deletion is more complex than insertion. Steps: First, traverse to the word's end node, verify the word exists (isEndOfWord is true). Set isEndOfWord = false. Then, walking back up to root, check if the now-unmarked node has any children. If a node has no children AND is not an end-of-word node, it's a dead leaf — remove it from its parent's children map and free it (setting the unique_ptr to null). Continue up until you reach a node that either has children or is an end-of-word node — those must be kept. This is tricky with unique_ptr and requires careful backtracking, typically implemented with a recursive function that returns whether the current node should be deleted."*

---

**Q42: If memory is critical, how would you reduce the Trie's memory usage?**

> *"Several approaches, roughly in order of effectiveness:*
>
> *1. Compressed/Patricia Trie: merge single-child chains into single edges labeled with substrings. Reduces node count dramatically for sparse tries. LexiCore's 185K nodes could be reduced to near-word-count with this.*
>
> *2. DAWG (Directed Acyclic Word Graph): also share common SUFFIXES, not just prefixes. Even more compact.*
>
> *3. Use array<Node*,26> instead of unordered_map — for dense tries (many words), the array avoids hash table overhead, though it uses 26 pointers per node unconditionally.*
>
> *4. Pool allocation: allocate all TrieNodes from a pre-allocated memory pool. Reduces per-node allocator overhead (~16-32 bytes saved per node from avoiding heap metadata).*
>
> *5. Bitset children: store a 26-bit bitset marking which children exist, plus a separate array of child pointers — compressed sparse representation."*

---

**Q43: The `data/words.txt` has 104,334 raw lines but only 88,344 unique words after normalization. What accounts for the difference?**

> *"The dictionary.cpp normalize() function: (1) strips non-alphabetic characters — words with apostrophes like \"don't\" become \"dont\", hyphens removed from compound words, (2) converts to lowercase, (3) removes blank lines. Additionally, deduplication removes words that appear multiple times or that normalize to the same string (e.g., 'Don't' and 'dont' both normalize to 'dont'). The difference of ~16,000 entries is plausible for a raw /usr/share/dict/words file which includes proper nouns, hyphenated words, possessives, and abbreviations that normalize to duplicates."*

---

## ✅ Phase 1 Mastery Checklist

Check these off before Phase 2:

- [ ] **editDistance**: Trace "kitten" → "sitting" manually. Get 3.
- [ ] **Rolling rows**: Explain std::swap O(1) without looking at notes.
- [ ] **Bounded variant**: Know what sentinel is and WHY it can't be used for BK pruning.
- [ ] **Trie insert**: Why `operator[]` is used in insert but `find()` in findNode.
- [ ] **Trie search vs startsWith**: Know exactly what isEndOfWord does.
- [ ] **collectWords**: Know the DFS approach and the optimization (backtracking).
- [ ] **BK-tree insert**: Trace inserting "book","cook","back","hook" and draw the tree.
- [ ] **BK-tree search**: Trace search("book", 1) step by step showing pruning.
- [ ] **Why true editDistance in searchImpl**: Explain the correctness bug with sentinel.
- [ ] **Property-based testing**: Explain symmetry + triangle inequality and why they're tested.
- [ ] **Oracle testing**: Explain why linear scan = valid oracle for BK-tree.
- [ ] **Complexity**: State Trie O(k) insight — why it doesn't scale with n.

---

> ✅ **Phase 1 complete!**
> Say **"proceed to Phase 2"** for `dictionary.cpp`, `ranking.cpp`, and `main.cpp` — the glue layer that connects all algorithms into a working CLI application.

