# Last-Minute Interview Crash Guide

## 1. Data Structures & Algorithms

**Trees**
- *Height of a tree:* max depth from root to leaf. `1 + max(height(left), height(right))`.
- *BST property:* left < node < right; inorder traversal gives sorted order.
- *LCA (Lowest Common Ancestor):* recurse — if node is either target, return it; if both sides return non-null, current node is LCA.
- *Diameter of tree:* max of (left height + right height) across all nodes, tracked via DFS.

**Graphs**
- *BFS vs DFS:* BFS = shortest path in unweighted graph (queue); DFS = full exploration/backtracking (stack/recursion).
- *Dijkstra:* shortest path with non-negative weights, uses min-heap, O((V+E)logV).
- *Union-Find (DSU):* detect cycles/connected components efficiently; use path compression + union by rank.
- *Topological Sort:* ordering of DAG nodes so edges go left→right; via DFS finish-time or Kahn's (in-degree) BFS.

**Recursion & Backtracking**
- *Backtracking pattern:* choose → explore → un-choose (undo state) to try all possibilities, prune early when invalid.
- *N-Queens/Subsets/Permutations:* classic backtracking; prune with constraints (row/col/diagonal used).

**Dynamic Programming**
- *0/1 Knapsack:* `dp[i][w] = max(dp[i-1][w], val[i] + dp[i-1][w-wt[i]])` if wt[i] ≤ w.
- *LCS (Longest Common Subsequence):* `dp[i][j] = dp[i-1][j-1]+1` if chars match, else `max(dp[i-1][j], dp[i][j-1])`.
- *Longest Palindromic Subsequence:* LCS of string with its reverse.
- *DP on Trees:* compute answer bottom-up combining children's DP states (e.g., max independent set in tree).
- *Key tip:* always define state clearly, find recurrence, identify base case, then optimize space if needed.

**Searching/Two Pointers/Sliding Window**
- *Binary Search:* works on sorted/monotonic search space; O(log n). Watch for off-by-one (`mid = lo + (hi-lo)/2`).
- *Two Pointers:* used for pair-sum in sorted array, removing duplicates — O(n).
- *Sliding Window:* subarray problems (max sum, longest substring without repeat) — expand right, shrink left when condition breaks.
- *Greedy:* pick locally optimal choice; works only when problem has greedy-choice property (e.g., activity selection, Huffman).

**Sorting**
- *Merge Sort:* O(n log n), stable, uses extra space — divide & merge.
- *Quick Sort:* O(n log n) avg, O(n²) worst, in-place, pivot-based partition.
- *When to use which:* stability needed → merge sort; memory-constrained → quicksort/heapsort.

---

## 2. OOP

- **4 Pillars:** Encapsulation (bundle data+methods, hide internals), Abstraction (expose only essentials), Inheritance (reuse via parent-child), Polymorphism (same interface, different behavior — compile-time overloading, run-time overriding).
- **SOLID:**
  - **S**ingle Responsibility — a class should have one reason to change.
  - **O**pen/Closed — open for extension, closed for modification.
  - **L**iskov Substitution — subclass should be replaceable for base class without breaking behavior.
  - **I**nterface Segregation — many small interfaces > one fat interface.
  - **D**ependency Inversion — depend on abstractions, not concrete implementations.
- **Singleton Pattern:** ensures only one instance of a class exists globally (private constructor + static instance).
- **Factory Pattern:** creates objects without exposing instantiation logic — a factory method decides which subclass to instantiate.
- **Virtual functions/vtable (C++):** enable runtime polymorphism via a pointer table resolved at run-time.

---

## 3. DBMS

- **ACID:** Atomicity (all-or-nothing), Consistency (valid state to valid state), Isolation (concurrent txns don't interfere), Durability (committed data survives crashes).
- **Normalization:** organizing tables to reduce redundancy. 1NF (atomic values), 2NF (no partial dependency), 3NF (no transitive dependency).
- **Joins:** INNER (matching rows only), LEFT (all left + matched right), RIGHT (all right + matched left), FULL OUTER (all rows both sides, nulls where no match).
- **Window functions:** compute values across a set of rows related to current row without collapsing them, e.g. `RANK() OVER (PARTITION BY dept ORDER BY salary DESC)`.
- **DENSE_RANK vs RANK:** RANK leaves gaps after ties (1,1,3); DENSE_RANK doesn't (1,1,2).
- **Second highest salary (classic query):**
```sql
SELECT MAX(salary) FROM Employee
WHERE salary < (SELECT MAX(salary) FROM Employee);
-- or using DENSE_RANK
SELECT salary FROM (
  SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) as rnk
  FROM Employee) t WHERE rnk = 2;
```
- **Query execution order:** FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT.
- **Indexing:** speeds up reads via B-Tree/hash structures; trade-off is slower writes and extra storage.
- **Performance optimization quick points:** proper indexing, avoid `SELECT *`, use `EXPLAIN`, denormalize for read-heavy systems, connection pooling (you've done this at HPE — mention it!).

---

## 4. OS & Networking Basics

- **Process vs Thread:** process has its own memory space; threads share memory within a process (cheaper context switch).
- **Deadlock (4 conditions):** mutual exclusion, hold & wait, no preemption, circular wait. Prevent by breaking any one.
- **Mutex vs Semaphore:** mutex = binary lock owned by one thread; semaphore = counter allowing N threads, signal/wait based.
- **Thread pool (you built one!):** pre-created worker threads pick tasks from a queue, avoiding overhead of repeated thread creation — improves throughput under load.
- **Paging:** memory divided into fixed-size pages/frames to avoid fragmentation, enables virtual memory.
- **TCP vs UDP:** TCP = connection-oriented, reliable, ordered (handshake); UDP = connectionless, faster, no guarantee — used in streaming/gaming.
- **HTTP vs HTTPS:** HTTPS adds TLS/SSL encryption over HTTP for secure data transfer.
- **DNS:** translates domain names to IP addresses via hierarchical resolver lookup.
- **Load balancing:** distributes incoming requests across multiple servers to avoid overload (round robin, least connections).

---

## 5. Generative AI & ML Basics

- **Supervised vs Unsupervised learning:** supervised uses labeled data (classification/regression); unsupervised finds patterns in unlabeled data (clustering).
- **Overfitting:** model memorizes training data, performs poorly on new data — fix with regularization, more data, dropout, cross-validation.
- **LLM basics:** large transformer-based models trained to predict next token, fine-tuned/aligned via RLHF for chat-style behavior.
- **Transformer/Attention:** self-attention lets model weigh relevance of all tokens to each other in parallel (vs sequential RNNs).
- **RAG (Retrieval-Augmented Generation):** LLM retrieves relevant external documents/context before generating an answer — reduces hallucination.
- **Embeddings:** vector representations of text/data capturing semantic meaning, used for similarity search.
- **Prompt engineering:** crafting inputs (few-shot examples, clear instructions) to guide LLM output without retraining.

---

## 6. Be Ready to Explain (from YOUR resume)

- **HPE Authorization Service (C++):** low-latency payment gateway auth logic — be ready to explain how you reduced latency (thread pool + connection pooling), and why C++ was chosen (performance-critical path).
- **Kafka microservice:** explain event-driven design — producer publishes payment events, your consumer validates and routes transaction state; mention why Kafka (decoupling, throughput, durability).
- **Custom thread pool & PostgreSQL connection pool:** explain problem (thread/connection creation overhead) → solution (reusable pool) → measurable improvement in concurrency/latency.
- **Invest-Tracker:** full-stack app, JWT auth + Postgres Row Level Security for per-user isolation, third-party API integration (Finnhub, MetalPriceAPI), deployed on Vercel — mention CORS issues you resolved.
- **Quick-Pick:** e-commerce app, 8 REST modules, JWT + HTTP-only cookies + bcrypt, RBAC middleware, Cloudinary + PayPal integration, MongoDB connection caching for stability.

**For every project, structure your answer as:** Problem → Tech stack choice (why) → Your specific contribution → Challenge faced → Outcome/impact.

---

## 7. Interview Day Checklist
- Original college ID + government photo ID
- Formal attire
- 2 printed resume copies in a file
- Pen & paper
- Stay calm — **explain your approach out loud**, even if unsure; process matters as much as the final answer.

**Good luck! You've got strong fundamentals — trust your prep.**
