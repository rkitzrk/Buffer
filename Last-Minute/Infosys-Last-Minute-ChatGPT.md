# Last-Minute Interview Guide — Specialist Programmer / DSE

This is designed for **rapid revision tonight**. Learn the **bold answer/idea**, then practice explaining it aloud in 20–40 seconds.

Your resume strongly points to **C++, DSA, SQL, backend development, Kafka, PostgreSQL, React/Node, MongoDB, JWT, REST APIs, and competitive programming**, so expect questions around those areas.  

---

# 1. DSA — HIGHEST PRIORITY

## Complexity

**Q. What is Big-O?**
It describes how time/space grows with input size. Example: binary search is **O(log n)**; linear search is **O(n)**.

**Q. Array vs linked list?**
Array: contiguous memory, O(1) indexing. Linked list: non-contiguous nodes, O(n) indexing but easy insertion/deletion when node position is known.

**Q. Stack vs queue?**
Stack = **LIFO**; queue = **FIFO**.

**Q. Hash table average complexity?**
Search/insert/delete are **O(1) average**, O(n) worst case due to collisions.

**Q. Why is binary search O(log n)?**
Each step eliminates roughly half the search space.

---

## Trees

**Q. BST property?**
For every node, values in the left subtree are smaller and values in the right subtree are larger.

**Q. Why is BST search O(log n)?**
For a **balanced** BST, tree height is O(log n). In a skewed BST it becomes O(n).

**Q. Tree traversals?**
Preorder = Root-L-R; Inorder = L-Root-R; Postorder = L-R-Root; Level order = BFS.

**Q. Important BST fact?**
**Inorder traversal of a BST gives sorted order.**

**Q. Height of a binary tree?**
`height = 1 + max(leftHeight, rightHeight)`; recursively calculate until null.

**Q. BFS vs DFS?**
BFS uses a queue and is useful for shortest path in an **unweighted graph**; DFS uses stack/recursion and is useful for traversal, components, backtracking.

---

# 2. GRAPHS

**Q. Adjacency matrix vs adjacency list?**
Matrix uses O(V²) space and gives O(1) edge lookup; adjacency list uses O(V+E) space and is better for sparse graphs.

**Q. BFS complexity?**
**O(V + E)** with adjacency lists.

**Q. DFS complexity?**
**O(V + E)**.

**Q. Detect cycle in undirected graph?**
DFS with parent tracking, or DSU; if visiting an already visited node that isn't the parent, a cycle exists.

**Q. Detect cycle in directed graph?**
DFS with a recursion-stack/3-color method; an edge to a currently visiting node means a cycle.

**Q. Topological sort?**
Ordering of vertices in a DAG such that for every edge `u → v`, `u` appears before `v`. Use DFS or Kahn's BFS.

**Q. Dijkstra?**
Shortest path with **non-negative edge weights**, usually O((V+E) log V) with a priority queue.

**Q. When does Dijkstra fail?**
When negative-weight edges exist.

**Q. MST algorithms?**
**Kruskal** = sort edges + DSU. **Prim** = grow tree using minimum edge/priority queue.

---

# 3. RECURSION & BACKTRACKING

**Q. What are the 3 parts of recursion?**
Base case, recursive call, and progress toward the base case.

**Q. What is backtracking?**
Choose → explore → undo choice. Used for permutations, subsets, N-Queens, Sudoku, etc.

**Q. Why can recursion cause stack overflow?**
Every recursive call consumes stack memory; very deep recursion can exhaust it.

**Q. Subsets of n elements?**
There are **2ⁿ subsets**.

**Q. Permutations of n distinct elements?**
There are **n!**.

---

# 4. DYNAMIC PROGRAMMING

### The most important explanation:

**Q. What is DP?**
DP solves problems with **overlapping subproblems + optimal substructure** by storing previously computed results.

**Q. Memoization vs tabulation?**
Memoization = top-down recursion + cache. Tabulation = bottom-up iterative table.

**Q. How do you identify DP?**
Ask: “Can I define a state, transition, base case, and reuse previous results?”

### Knapsack

**0/1 Knapsack transition:**
`dp[i][w] = max(dp[i-1][w], value[i] + dp[i-1][w-weight[i]])` when item fits.

**Q. Why not reuse the same item?**
Because 0/1 knapsack moves from the previous item row/state.

### LCS

**Q. LCS recurrence?**
If characters match: `1 + dp[i-1][j-1]`; otherwise `max(dp[i-1][j], dp[i][j-1])`.

**Q. LCS vs substring?**
Subsequence needn't be contiguous; substring must be contiguous.

### Palindromic subsequence

**Q. Longest Palindromic Subsequence?**
For `s[i]==s[j]`, `2 + dp[i+1][j-1]`; otherwise max of excluding either endpoint.

### DP on trees

**Typical idea:**
For each node calculate states such as “best answer if node is taken/not taken,” then combine child results.

**DP on graphs?**
Usually requires a DAG, suitable state ordering, or special graph structure; arbitrary cyclic graphs need other techniques.

---

# 5. BINARY SEARCH

**Q. When can binary search be used?**
Whenever the search space has a **monotonic property** or the data is sorted.

**Q. What is binary search on answer?**
Guess an answer `x`, check whether it is feasible, then binary-search the minimum/maximum feasible `x`.

**Typical template:**
`lo < hi → mid → if feasible(mid) hi=mid else lo=mid+1`.

**Common mistakes?**
Wrong boundary, infinite loop, overflow in `(l+r)/2`; use `l + (r-l)/2`.

---

# 6. TWO POINTERS / SLIDING WINDOW

**Q. Two pointers?**
Maintain two indices and exploit ordering/structure to avoid O(n²) enumeration.

**Q. Sliding window?**
Maintain a contiguous window `[l,r]`, expand `r`, and move `l` when a condition is violated.

**Classic examples:**
Longest substring without repeating characters, minimum window, subarray with sum/constraint.

**Q. Why is sliding window often O(n)?**
Each pointer generally moves only forward, so total movements are O(n).

---

# 7. GREEDY

**Q. What is greedy?**
Make the locally optimal choice at each step, hoping it leads to a globally optimal solution.

**Q. Greedy vs DP?**
Greedy commits immediately; DP explores/stores multiple possibilities when local choice isn't sufficient.

**Classic greedy examples?**
Activity selection, interval scheduling, Huffman coding, Kruskal, fractional knapsack.

**Q. Does greedy always work?**
No. It requires a proof/property such as the greedy-choice property.

---

# 8. SORTING

**Q. Quick sort complexity?**
Average **O(n log n)**, worst **O(n²)**; in-place partitioning is its advantage.

**Q. Merge sort?**
O(n log n) time, O(n) auxiliary space, stable.

**Q. Heap sort?**
O(n log n), O(1) auxiliary space, generally not stable.

**Q. Stable sorting means?**
Equal-valued elements retain their original relative order.

**Q. Which sorting would you choose?**
Depends on constraints: stability, memory, worst-case guarantee, nearly-sorted data, etc.

---

# 9. MUST-KNOW C++ QUESTIONS

Your resume lists **C++** and significant competitive-programming experience, so be ready for detailed C++ questions. 

**Q. Pointer vs reference?**
Pointer stores an address and can be null/reassigned; reference is an alias and normally cannot be null/reseated.

**Q. `const`?**
Prevents modification of the associated object/value; with member functions, `const` means the function won't modify object state.

**Q. Stack vs heap?**
Stack stores automatic/local variables and call frames; heap stores dynamically allocated objects and has manually/automatically managed lifetime depending on mechanism.

**Q. `vector` vs `list`?**
Vector gives O(1) random access and cache-friendly contiguous storage; list gives O(1) insertion/deletion given an iterator but poor random access/cache locality.

**Q. `map` vs `unordered_map`?**
`map`: ordered tree, O(log n). `unordered_map`: hash table, average O(1), no sorted order.

**Q. What is an iterator?**
An object used to traverse/access elements of a container.

**Q. Pass by value vs reference?**
Value copies the argument; reference avoids copying and can modify the original unless passed as `const`.

**Q. Why use `const vector<int>&`?**
Avoid copying while preventing modification.

**Q. What is a virtual function?**
A function enabling runtime polymorphism through dynamic dispatch.

**Q. What is a pure virtual function?**
`virtual void f() = 0;`; makes the class abstract.

**Q. Why virtual destructor?**
When deleting a derived object through a base pointer, a virtual destructor ensures the derived destructor executes correctly.

**Q. Smart pointers?**
`unique_ptr` = single ownership; `shared_ptr` = shared ownership; `weak_ptr` = non-owning reference to a shared object.

---

# 10. OOP — VERY LIKELY

**Q. Four pillars of OOP?**
**Encapsulation, abstraction, inheritance, polymorphism.**

**Encapsulation?**
Bundle data and methods together and control access using access modifiers.

**Abstraction?**
Expose what an object does while hiding implementation details.

**Inheritance?**
Derived class obtains/reuses properties and behavior of a base class.

**Polymorphism?**
Same interface, different implementation; e.g. method overriding through virtual functions.

---

## SOLID

**S — Single Responsibility:** one reason to change.
**O — Open/Closed:** open for extension, closed for modification.
**L — Liskov Substitution:** derived types should work wherever base types are expected.
**I — Interface Segregation:** don't force clients to depend on unused methods.
**D — Dependency Inversion:** depend on abstractions, not concrete implementations.

### Singleton

**Q. What is Singleton?**
Ensures only one instance of a class and provides a global access point.

**Problem with Singleton?**
Global state, harder testing, hidden dependencies; use carefully.

### Factory

**Q. Factory pattern?**
Moves object-creation logic into a factory so client code depends on an interface rather than concrete classes.

---

# 11. DBMS / SQL — HIGH PRIORITY

Your resume explicitly mentions **SQL**, PostgreSQL, MongoDB, and query optimization.   

## SQL execution order

Memorize:

**FROM/JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT**

**Q. WHERE vs HAVING?**
`WHERE` filters rows before grouping; `HAVING` filters groups after `GROUP BY`.

### Second highest salary

```sql
SELECT MAX(salary)
FROM Employee
WHERE salary < (SELECT MAX(salary) FROM Employee);
```

### DENSE_RANK

```sql
SELECT name, salary,
       DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
FROM Employee;
```

**Q. RANK vs DENSE_RANK?**
`RANK`: ties create gaps. `DENSE_RANK`: ties don't create gaps.

Example salaries `100,100,90`:
RANK = `1,1,3`; DENSE_RANK = `1,1,2`.

### Top 2 salaries per department

```sql
SELECT *
FROM (
  SELECT e.*,
         DENSE_RANK() OVER
         (PARTITION BY department ORDER BY salary DESC) r
  FROM Employee e
) x
WHERE r <= 2;
```

### INNER vs LEFT JOIN

**INNER JOIN:** only matching rows.
**LEFT JOIN:** every row from left table + matching right rows; unmatched right side becomes NULL.

---

# 12. DATABASE CONCEPTS

**Q. Primary key?**
Uniquely identifies each row; cannot contain NULL.

**Q. Foreign key?**
References a key in another table and enforces referential integrity.

**Q. Normalization?**
Organize tables to reduce redundancy and update anomalies.

**1NF:** atomic values.
**2NF:** 1NF + no partial dependency on composite key.
**3NF:** 2NF + no transitive dependency.

**Q. What is indexing?**
Data structure that speeds reads/searches at the cost of extra storage and slower writes.

**Q. Why not index every column?**
Indexes consume space and must be updated on INSERT/UPDATE/DELETE.

---

# 13. ACID

**Atomicity:** transaction is all-or-nothing.
**Consistency:** transaction preserves valid database state.
**Isolation:** concurrent transactions shouldn't improperly affect each other.
**Durability:** committed data survives failures.

**Q. What is a transaction?**
A logical unit of database operations that either commits completely or rolls back.

---

# 14. SQL PERFORMANCE

**Q. How would you optimize a slow query?**

1. Check execution plan (`EXPLAIN/EXPLAIN ANALYZE`).
2. Add appropriate indexes.
3. Avoid unnecessary columns/rows.
4. Optimize joins/subqueries.
5. Reduce repeated computation and unnecessary sorting.

**Q. Why can an index fail to help?**
Low selectivity, function applied to indexed column, poor query shape, stale statistics, or optimizer deciding a scan is cheaper.

---

# 15. NOSQL / MONGODB

**Q. SQL vs MongoDB?**
SQL uses relational tables/schema and joins; MongoDB stores document-oriented BSON data and is flexible-schema.

**Q. MongoDB document?**
A JSON-like BSON object containing fields and nested data.

**Q. When prefer MongoDB?**
Flexible/changing schemas, document-centric applications, rapid development, horizontal scaling scenarios.

---

# 16. OPERATING SYSTEMS

**Q. Process vs thread?**
Process has its own address space/resources; threads share the process's memory/resources.

**Q. Why multithreading?**
Concurrency, responsiveness and better CPU/resource utilization.

**Q. Context switch?**
CPU switches from one process/thread to another, saving/restoring execution state.

**Q. Race condition?**
Result depends on timing/order of concurrent operations.

**Q. How prevent race conditions?**
Mutex/locks, atomics, semaphores, proper synchronization.

**Q. Mutex vs semaphore?**
Mutex provides mutual exclusion/ownership; semaphore is a signaling/counting synchronization primitive.

**Q. Deadlock?**
Processes wait forever for resources held by each other.

### Four deadlock conditions

**Mutual exclusion, hold and wait, no preemption, circular wait.**

**Q. How prevent deadlock?**
Break at least one of those necessary conditions, e.g. fixed lock ordering.

---

# 17. COMPUTER NETWORKING

**Q. TCP vs UDP?**
TCP is connection-oriented, reliable and ordered; UDP is connectionless, faster/lower overhead but doesn't guarantee delivery/order.

**Q. HTTP vs HTTPS?**
HTTPS = HTTP over TLS, providing encryption, integrity and server authentication.

**Q. What happens when you type a URL?**
DNS resolves domain → connection established → TLS if HTTPS → HTTP request → server response → browser renders.

**Q. DNS?**
Maps domain names to IP addresses.

**Q. IP vs MAC?**
IP identifies a host/interface logically for network routing; MAC is a link-layer hardware/interface address used within local networks.

**Q. What is a port?**
Identifies a network service/process endpoint on a host.

**Q. Common ports?**
HTTP 80, HTTPS 443, SSH 22, PostgreSQL 5432.

---

# 18. REST APIs

Your projects contain substantial REST API work.  

**Q. REST?**
Architectural style using resources and standard HTTP methods.

**GET:** retrieve
**POST:** create
**PUT:** replace/update
**PATCH:** partial update
**DELETE:** remove

**Q. What does stateless mean?**
Each request contains the information necessary for the server to process it; server doesn't depend on stored client session state between requests.

**Common status codes:**
200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 500 Server Error.

---

# 19. JWT / AUTHENTICATION

Your resume says you used JWT authentication, HTTP-only cookies and role-based access control.  

**Q. Authentication vs authorization?**
Authentication = **who are you?** Authorization = **what are you allowed to do?**

**Q. JWT?**
Signed token containing claims; server can verify signature and extract identity/claims.

**Q. Why HTTP-only cookie?**
JavaScript cannot directly read it, reducing token theft through many XSS scenarios.

**Q. What is RBAC?**
Access permissions are assigned based on roles such as admin/user.

**Q. Password storage?**
Never store plaintext; use a slow password hashing algorithm such as bcrypt/Argon2 with salt.

---

# 20. CORS

**Q. What is CORS?**
Browser security mechanism controlling whether a frontend from one origin can access resources from another origin.

**Q. Why did CORS matter in your project?**
Your resume specifically mentions resolving CORS issues during deployment. 

**Answer structure:**
“My frontend and backend were on different origins, so the browser blocked requests. I configured the backend's allowed origins/methods/credentials correctly.”

---

# 21. KAFKA — VERY IMPORTANT FOR YOUR HPE INTERVIEW

Your HPE experience specifically involves an event-driven microservice using Kafka. 

**Q. What is Kafka?**
A distributed event-streaming platform used to publish, store and consume streams of records.

**Q. Producer?**
Publishes messages to Kafka topics.

**Q. Consumer?**
Reads messages from topics.

**Q. Topic?**
Logical stream/category of messages.

**Q. Partition?**
Topic is divided into partitions for parallelism and scalability; ordering is guaranteed within a partition.

**Q. Consumer group?**
Consumers with the same group ID divide partitions among themselves.

**Q. Why Kafka instead of direct synchronous calls?**
Decoupling, asynchronous processing, buffering, scalability and resilience.

**Q. What happens if a consumer crashes?**
Another consumer in the same group can take over its partitions after rebalance.

**Q. Offset?**
Position of a consumer in a partition indicating which records it has processed/read.

---

# 22. YOUR HPE PROJECT — EXPECT DEEP QUESTIONS

Your resume says you built a **C++ authorization service for a payment gateway**, used Kafka, a custom thread pool, PostgreSQL connection pooling and SQL optimization. 

Prepare these **exact answers in your own words**:

### “Explain your HPE project.”

Use:

> “I worked on a C++ authorization service for a payment-gateway proof of concept. We used Kafka for event-driven payment processing, performed logical validations, and routed transaction states. I also worked on concurrency using a thread pool and improved database performance using PostgreSQL connection pooling and SQL optimization.”

Then immediately be ready for:

**Why C++?**
Low-level control, performance, predictable execution and efficient concurrency.

**Why Kafka?**
Decouples payment-event producers from consumers and supports scalable asynchronous processing.

**Why thread pool?**
Avoids repeatedly creating/destroying threads and limits concurrency to a controlled number of workers.

**Why connection pool?**
Creating DB connections repeatedly is expensive; reusing existing connections reduces latency.

**What happens without a thread pool?**
Frequent thread creation can add overhead and excessive concurrency can hurt performance.

**How would you make authorization reliable?**
Idempotency, validation, timeouts, retries where safe, proper transaction handling and observability.

**What is idempotency?**
Repeating the same request produces the same intended effect, preventing duplicate processing.

**What happens if Kafka delivers the same event twice?**
Consumer logic should be idempotent/deduplicate using a unique transaction/event ID.

---

# 23. INVEST-TRACKER — VERY LIKELY

Your project uses React, Node/Express, Supabase/PostgreSQL, external APIs, JWT and RLS. 

**“Explain the architecture.”**

> “React handles the frontend, Node/Express provides REST APIs, PostgreSQL stores application data, external APIs provide market/news information, and JWT-based authentication protects routes.”

**Q. Why REST API instead of direct frontend DB access?**
Business logic and authentication remain in a controlled backend layer.

**Q. What is Row Level Security?**
Database policies restrict which rows a user can access based on authorization rules.

**Q. What if an external stock API fails?**
Handle timeout/errors gracefully, return cached/partial data where appropriate, and avoid bringing down the whole application.

**Q. What is CORS?**
See above; emphasize frontend/backend origins.

---

# 24. QUICK-PICK E-COMMERCE — VERY LIKELY

Your project uses React, Node/Express, MongoDB, JWT, HTTP-only cookies, RBAC, Cloudinary and PayPal. 

**Q. Explain checkout flow.**

> Cart → create order → initiate payment → payment approval → verify payment → update order/stock → confirmation.

**Q. Why verify payment on backend?**
Never trust only the frontend; the backend should verify payment status with the payment provider.

**Q. Why HTTP-only cookies?**
Reduce exposure of authentication tokens to client-side JavaScript.

**Q. What if two users buy the last item simultaneously?**
Use atomic DB updates/transactions or conditional stock updates to prevent overselling.

**Q. Why bcrypt?**
Password hashing designed to be computationally expensive, making brute-force attacks harder.

---

# 25. GENERATIVE AI / ML — BASICS YOU MUST KNOW

### ML

**Q. Supervised learning?**
Learns from labeled input-output examples; classification and regression are common examples.

**Q. Unsupervised learning?**
Finds patterns without labeled outputs; clustering is a common example.

**Q. Classification vs regression?**
Classification predicts categories; regression predicts continuous numerical values.

**Q. Overfitting?**
Model performs well on training data but poorly on unseen data.

**Q. Underfitting?**
Model is too simple to capture important patterns.

**Q. How prevent overfitting?**
More data, regularization, cross-validation, simpler models, early stopping, etc.

**Q. Train/validation/test?**
Train learns parameters; validation tunes choices; test estimates final generalization.

---

# 26. GENERATIVE AI

**Q. What is Generative AI?**
AI that generates new content such as text, images, code or audio from learned patterns.

**Q. What is an LLM?**
A large neural language model trained to predict/tokenize language and generate text based on context.

**Q. Transformer?**
Architecture based heavily on attention mechanisms, allowing the model to capture relationships between tokens efficiently.

**Q. What is attention?**
Mechanism that assigns importance to different tokens when representing another token.

**Q. What is a token?**
A unit of text processed by the model; it may be a word, subword or character-like piece.

**Q. What is hallucination?**
A model generates plausible-sounding but incorrect or unsupported information.

**Q. How reduce hallucinations?**
Ground responses using trusted data/RAG, constrain outputs, validate results and provide appropriate context.

**Q. What is RAG?**
Retrieval-Augmented Generation: retrieve relevant external documents/data, then provide them as context to the model before generation.

**Q. Fine-tuning vs RAG?**
Fine-tuning changes model behavior/weights using training data; RAG injects external/current information at inference time.

---

# 27. BEHAVIORAL — PREPARE THESE TONIGHT

## “Tell me about yourself.”

Keep it to **45–60 seconds**:

> “I’m a CSE student at VIT-AP with a strong interest in problem solving and software development. I have solved 1000+ competitive-programming problems and achieved a 1455 Specialist rating on Codeforces. I also worked at HPE on a C++/Kafka payment-authorization system and built full-stack projects involving React, Node.js, PostgreSQL and MongoDB.”  

## “Why should we hire you?”

> “My strengths are problem solving, strong DSA fundamentals, and hands-on development experience. I’m comfortable going from understanding a problem to designing, implementing, debugging and optimizing a solution.”

## “What is your strength?”

Choose one genuine strength and prove it with an example:
**problem solving + persistence + learning quickly** works naturally with your competitive-programming background. 

## “What is your weakness?”

Give a real but controllable weakness + improvement:

> “Earlier I sometimes spent too much time optimizing before validating the basic solution. I now first establish correctness, then optimize based on constraints and measurements.”

## “Tell me about a challenge.”

Use **HPE performance work**:
Problem → bottleneck → thread pool/connection pool/query optimization → measurable/improved result.

## “Tell me about a failure.”

Never blame teammates. Explain **what happened → what you learned → what changed afterward**.

## “Why this role?”

Focus on: **DSA + software engineering + technically challenging systems + learning opportunities.**

---

# 28. LIVE CODING — THE INTERVIEWER IS WATCHING YOUR THINKING

When given a problem:

### Step 1 — Clarify

Ask:

* Input constraints?
* Can values be negative?
* Duplicates?
* Expected output?
* Need optimal solution?

### Step 2 — Brute force

Say:

> “The straightforward solution is O(...), but with these constraints it may be too slow.”

### Step 3 — Find pattern

Ask yourself:

**HashMap? Sorting? Two pointers? Binary search? Stack? BFS/DFS? DP? Greedy?**

### Step 4 — Explain before coding

State the invariant/idea in 2–3 sentences.

### Step 5 — Code cleanly

### Step 6 — Test

Always test:

* Empty/single element
* Minimum input
* Duplicate values
* Boundary case
* Normal case

### Step 7 — Complexity

Always finish with:

> “Time complexity is O(...), space complexity is O(...).”

---

# 29. 15 CODING PROBLEMS TO REVISE RIGHT NOW

Know the approach, not merely the code.

| Problem                             | Core idea                |
| ----------------------------------- | ------------------------ |
| Two Sum                             | HashMap                  |
| Valid Parentheses                   | Stack                    |
| Binary Search                       | Divide search space      |
| Merge Intervals                     | Sort + merge             |
| Longest Substring Without Repeating | Sliding window + set/map |
| Maximum Subarray                    | Kadane's algorithm       |
| Best Time to Buy/Sell Stock         | Running minimum          |
| Reverse Linked List                 | Iterative pointers       |
| Detect Cycle Linked List            | Slow/fast pointers       |
| Level Order Traversal               | BFS                      |
| Number of Islands                   | DFS/BFS                  |
| Shortest Path Unweighted            | BFS                      |
| Coin Change                         | DP                       |
| 0/1 Knapsack                        | DP                       |
| Longest Common Subsequence          | 2D DP                    |

---

# 30. IMPORTANT EDGE CASES INTERVIEWERS TEST

Before submitting code, check:

**Array:** empty, one element, duplicates.
**String:** empty, all same characters, spaces/case.
**Numbers:** negative, zero, overflow.
**Tree:** null root, one node, skewed tree.
**Graph:** disconnected, cycle, self-loop, duplicate edges.
**DP:** impossible state, zero capacity, base case.

---

# 31. SYSTEM/DESIGN BASICS FOR A FRESHER

**Q. How would you design a URL shortener?**
`long URL → generate unique short ID → store mapping → redirect short ID to original URL.`

**Q. How make an API scalable?**
Stateless servers + load balancing + caching + database indexing/replication + asynchronous processing where appropriate.

**Q. Cache?**
Stores frequently accessed data closer to the application to reduce latency/database load.

**Q. Horizontal vs vertical scaling?**
Vertical = bigger machine; horizontal = more machines/instances.

**Q. Load balancer?**
Distributes requests across multiple servers.

---

# 32. QUESTIONS THEY MAY ASK DIRECTLY FROM YOUR RESUME

Be able to answer every line of your resume. Your interviewers can point to **any technology and ask “Why did you use this?”**.

From your resume, be ready for:

**C++** — why it, memory, STL, threads, pointers.
**Kafka** — producer, consumer, partition, offset, consumer groups.
**PostgreSQL** — joins, indexing, transactions, connection pooling.
**React** — components, props, state, hooks.
**Node/Express** — middleware, routing, REST APIs.
**JWT** — authentication and authorization.
**MongoDB** — documents, indexes, schema flexibility.
**Cloudinary** — external media storage.
**PayPal API** — payment lifecycle and backend verification.
**Supabase/PostgreSQL RLS** — authorization at database level.

These technologies and implementations are explicitly represented on your resume. 

---

# 33. REACT — QUICK REVISION

**Props vs state?**
Props come from parent and are read-only; state is component-managed mutable data.

**What is `useEffect`?**
Runs side effects such as API calls, subscriptions or synchronization after rendering.

**What is `useState`?**
Hook for maintaining component state.

**Why keys in lists?**
Help React identify which elements changed, added or removed.

**What is controlled input?**
Input whose value is controlled by React state.

---

# 34. NODE.JS / EXPRESS

**Q. What is Node.js?**
JavaScript runtime using Google's V8 engine; commonly used for event-driven server applications.

**Q. What is middleware?**
Function that runs during the request-response cycle, e.g. authentication, logging or validation.

**Q. Why Node.js for APIs?**
Efficient for I/O-heavy applications because of its event-driven, non-blocking model.

**Q. What is async/await?**
Syntax for writing asynchronous Promise-based code in a readable style.

---

# 35. YOUR COMPETITIVE PROGRAMMING PROFILE

Your resume states **1455 Codeforces Specialist, 1000+ problems solved, and several strong contest ranks**, so this is likely to attract discussion. 

### “How did competitive programming help you?”

> “It strengthened my ability to recognize patterns, analyze constraints, derive efficient algorithms and implement them accurately under time pressure.”

### “What is your strongest DSA topic?”

Choose the one you can actually defend.

### “What's the hardest problem you've solved?”

Pick **one problem you genuinely remember** and explain:
**problem → insight → algorithm → complexity → mistake/lesson.**

Do **not** invent a problem or claim an implementation you cannot explain.

---

# 36. FIVE QUESTIONS WHERE YOU SHOULD NEVER GET STUCK

### Why is binary search O(log n)?

Because the search space roughly halves at every iteration.

### Why use HashMap?

To trade extra memory for approximately O(1) average lookup.

### Why indexes?

They reduce data that must be scanned, accelerating reads at the cost of storage/write overhead.

### Why Kafka?

Asynchronous, decoupled, scalable event processing with durable ordered partitions.

### Why thread pool?

Reuse a bounded set of worker threads instead of repeatedly creating threads.

---

# 37. LAST 30-MINUTE MEMORY SHEET

Memorize these without looking:

```text
BFS             → Queue
DFS             → Stack/Recursion
BST inorder     → Sorted
Binary Search   → O(log n)
HashMap         → O(1) average
Sorting         → Usually O(n log n)
BFS/DFS graph   → O(V+E)
Dijkstra        → Non-negative weights
DP              → State + transition + base case
LCS             → Match: 1+diag; else max(top,left)
0/1 Knapsack    → Take / Don't take
Sliding Window  → Two moving boundaries
Greedy          → Local optimum + proof/property

TCP             → Reliable, ordered
UDP             → Fast, connectionless
HTTP            → Application protocol
HTTPS           → HTTP + TLS
DNS             → Domain → IP

ACID            → Atomicity, Consistency, Isolation, Durability
WHERE           → Before grouping
HAVING          → After grouping
RANK            → Gaps after ties
DENSE_RANK      → No gaps
INNER JOIN      → Matching rows
LEFT JOIN       → All left + matching right

OOP             → Encapsulation, Abstraction,
                  Inheritance, Polymorphism

SOLID           → SRP, OCP, LSP, ISP, DIP

Kafka           → Topic → Partition → Consumer Group → Offset
JWT             → Signed authentication token
RBAC            → Role-based authorization
CORS            → Cross-origin browser policy
```

---

# 38. YOUR INTERVIEWER'S LIKELY PROJECT CROSS-EXAMINATION

Expect a chain like:

> **“You used Kafka. Why?”**
> Asynchronous and decoupled event processing.

> **“Why not REST?”**
> REST is suitable for synchronous request-response; Kafka is useful when event-driven asynchronous processing is preferable.

> **“What is a partition?”**
> A topic subdivision enabling parallelism; ordering is maintained within a partition.

> **“How do you handle duplicate payment events?”**
> Make processing idempotent using a unique transaction/event identifier.

> **“You used a thread pool. Why?”**
> Reuse worker threads and control concurrency.

> **“You used a DB connection pool. Why?”**
> Reuse expensive DB connections and reduce connection-establishment latency.

> **“How did you optimize SQL?”**
> Analyze query plans, reduce unnecessary work, use appropriate indexes and improve query structure.

Those answers map directly to the work described on your resume. 

---

# 39. PHYSICAL INTERVIEW — DON'T LOSE EASY POINTS

Before leaving:

**Original college ID + government photo ID**
**2 printed updated resumes**
**Formal attire**
**Pen + paper**
**Resume in a clean file**

During coding:

**Clarify → Explain → Code → Test → Complexity.**

During project discussion:

**Architecture → Why technology → Your contribution → Challenge → Solution → Result.**

During behavioral questions:

**Situation → Action → Result → Learning.**

---

# 40. THE 10 THINGS I WOULD REVISE FIRST TONIGHT

If time becomes extremely short, prioritize these in this exact study sequence:

1. **Arrays + strings + HashMap + two pointers + sliding window**
2. **Trees + BFS/DFS + graph basics**
3. **Binary search**
4. **DP: Knapsack + LCS + palindrome**
5. **SQL joins + GROUP BY/HAVING + DENSE_RANK + execution order**
6. **OOP + SOLID + Singleton + Factory**
7. **Kafka + thread pool + DB connection pool**
8. **JWT + REST + CORS + authentication/authorization**
9. **OS: process/thread/deadlock/mutex**
10. **Your two projects and HPE work — every technology on the resume**

Your resume shows **1000+ solved problems and a 1455 Codeforces Specialist rating**, so your strongest interview advantage is likely to be your problem-solving background; make sure you complement that with very crisp explanations of your real-world engineering work. 

## One final rule for tomorrow

**Never jump straight into code.** Say:

> **“Let me first clarify the constraints, then I'll explain the approach, and after that I'll implement it.”**

That one habit makes your reasoning much easier for the interviewer to evaluate—and your interview notice explicitly says that **demonstrating your approach is as significant as the solution itself**.


These are very likely to come up. The key is to make the answers **specific to you**, rather than giving generic lines about Infosys.

## 1. “Why Infosys?”

A strong answer for your profile:

“I’m interested in Infosys because it gives me an opportunity to work on large-scale technology projects while continuing to strengthen my problem-solving and software engineering skills. My background is strongly focused on DSA, C++, backend development and real-world project experience, including my work with Kafka and PostgreSQL at HPE. I believe Infosys would give me the right environment to apply these skills, learn from experienced teams, and grow as a software engineer.”

### If they ask: “Why Infosys specifically?”

Don't say:

> “Because Infosys is a very big company and has a good reputation.”

That is too generic.

Instead, connect **your profile → role → company**:

> “My interests are strongly aligned with software engineering and problem solving, and the role gives me an opportunity to work on challenging technical problems while developing industry-level engineering skills.”

---

# 2. “Why DSE/SP role?”

This is actually easier for you because your resume supports it very well.

“I’m particularly interested in the DSE/SP role because it is closely aligned with my strengths in problem solving, algorithms and software development. I have solved 1000+ competitive programming problems and have hands-on experience building backend and full-stack applications. I want a role where I can use these skills on real-world engineering problems while continuing to improve technically.”

---

# 3. “Why should we give you Specialist Programmer rather than a regular software role?”

This may come up because you have a strong competitive-programming profile.

A good answer:

“I believe my problem-solving background makes me a good fit for a technically intensive role. I have a strong foundation in DSA and competitive programming, and I also have practical development experience from projects and my HPE experience. I’m interested in solving complex problems rather than limiting myself to a narrow area, so the Specialist Programmer role particularly interests me.”

---

# 4. “Why DSE/SP when you already have HPE experience?”

This is a possible follow-up because your resume contains HPE experience.

Say:

> “My HPE experience gave me exposure to real-world engineering practices such as C++, Kafka, concurrency and database optimization. It actually strengthened my interest in software engineering and made me want to work on technically challenging problems at a larger scale.”

---

# 5. “What if we offer you DSE instead of SP?”

**Do not sound like you will reject DSE.**

Say:

> “I’m particularly interested in SP because of its technical focus, but I would be happy to take up a DSE opportunity as well. My priority is to work in a role where I can contribute, learn and grow technically.”

This is safer than saying **“I only want SP.”**

---

# 6. Very important: don't memorize the answer word-for-word

Use this structure:

### Why Infosys?

**Company opportunity → your technical interests → learning/growth**

### Why SP/DSE?

**Your DSA strength → development experience → technical challenge**

For **you**, the strongest evidence to mention is:

* **1000+ competitive-programming problems**
* **1455 Specialist on Codeforces**
* **C++/Kafka/payment authorization work at HPE**
* **React/Node/PostgreSQL/MongoDB projects**   

## One thing I would change in your delivery

Don't make it sound like:

> “I want SP because it's a higher package/status.”

Instead make it sound like:

> **“My existing skills align naturally with the technical requirements of the role, and I want an environment where I can apply and deepen those skills.”**

That sounds much more convincing in an interview.

