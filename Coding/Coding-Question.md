**Question 1: Maximize profit of buying one stock with different parity**

**Observations:**

* This problem is a variation of the classic "Best Time to Buy and Sell Stock" problem.
* You are allowed to make a single transaction (one buy, one sell) such that the buy price and the sell price have different parities (one must be even, and the other must be odd).
* To maximize the profit `(sell_price - buy_price)`, you need to keep track of the lowest possible buying prices seen so far. Since the parity must be different, you must track the minimum even price and the minimum odd price independently as you traverse the array.

**Approach:**

1. Initialize two variables, `min_even` and `min_odd`, to a very large value (infinity).
2. Initialize `max_profit` to 0.
3. Traverse the array of prices from left to right.
4. For each `price`:
* If `price` is even: The buy price must have been odd. Calculate the profit as `price - min_odd` and update `max_profit` if this is greater than the current `max_profit`. Then, update `min_even = min(min_even, price)`.
* If `price` is odd: The buy price must have been even. Calculate the profit as `price - min_even` and update `max_profit` if this is greater. Then, update `min_odd = min(min_odd, price)`.


5. Return the `max_profit`.

**Code (in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <climits>

using namespace std;

int maxProfitDifferentParity(vector<int>& prices) {
    int min_even = INT_MAX;
    int min_odd = INT_MAX;
    int max_profit = 0;

    for (int i = 0; i < prices.size(); i++) {
        if (prices[i] % 2 == 0) {
            if (min_odd != INT_MAX) {
                max_profit = max(max_profit, prices[i] - min_odd);
            }
            // Update min_even seen so far
            min_even = min(min_even, prices[i]);
        } else {
            if (min_even != INT_MAX) {
                max_profit = max(max_profit, prices[i] - min_even);
            }
            // Update min_odd seen so far
            min_odd = min(min_odd, prices[i]);
        }
    }

    return max_profit;
}

int main() {
    vector<int> prices = {2, 9, 4, 3, 11, 8};
    cout << "Max Profit: " << maxProfitDifferentParity(prices) << endl; 
    return 0;
}

```

**Things to remember for solving same kind of problems (in short):**

* Whenever a problem asks to maximize `A[j] - A[i]` for `i < j` under certain constraints, you can often solve it in O(N) time by keeping a running track of the minimum `A[i]` (or multiple minimums based on states, like parity) seen so far.
* Always handle edge cases where a valid buy price might not have been seen yet (e.g., checking if the minimum is still at infinity before calculating profit).

---
**Question 2: Place at least 2 sensors for every given Stretch range, with constraints no sensor should be placed in given Hazard range.**

**Observations:**

* This problem is a variation of the classic "Interval Covering" or "Minimum Points to Cover Intervals" greedy problem.
* The goal is to minimize the total number of sensors placed while satisfying two conditions:
1. Every `stretch` range `[L, R]` must contain at least 2 sensors.
2. No sensor can be placed inside any `hazard` range.


* Because the ranges can be up to $10^9$, we cannot iterate through coordinates one by one. We must use sorting and binary search.
* To minimize the number of sensors, when a stretch needs more sensors, we should place them **as far right as possible** (closest to the end of the stretch `R`). This maximizes the chance that these newly placed sensors will also fall into the upcoming stretch ranges.

**Approach:**

1. **Merge Hazard Ranges:** First, sort and merge the overlapping hazard intervals so we have disjoint forbidden zones.
2. **Sort Stretches:** Sort the stretch intervals primarily by their end times (ascending order). If end times are equal, sort by start times (descending).
3. **Greedy Placement:** Iterate through each sorted stretch interval.
* Count how many already placed sensors fall within the current stretch bounds `[L, R]`.
* If the count is $\ge 2$, we don't need to do anything.
* If we need to place $k$ more sensors (where $k$ is 1 or 2), start looking from the rightmost coordinate `curr = R` downwards.
* Check if `curr` falls inside a merged hazard zone. If it does, jump `curr` to just before the start of that hazard zone.
* If `curr < L`, it is impossible to place the required sensors (return an error/invalid state).
* Otherwise, place the sensor at `curr`, add it to our list of placed sensors, and decrement `curr` by 1 to look for the next spot if needed.


4. Return the total number of sensors placed.

**Code (in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

// Helper function to merge hazard intervals
vector<pair<int, int>> mergeHazards(vector<pair<int, int>>& hazards) {
    if (hazards.empty()) return {};
    sort(hazards.begin(), hazards.end());
    vector<pair<int, int>> merged;
    merged.push_back(hazards[0]);
    for (int i = 1; i < hazards.size(); i++) {
        if (hazards[i].first <= merged.back().second) {
            merged.back().second = max(merged.back().second,
hazards[i].second);
        } else {
            merged.push_back(hazards[i]);
        }
    }
    return merged;
}

int minSensors(vector<pair<int, int>>& stretches,
vector<pair<int, int>>& hazards) {
    vector<pair<int, int>> mergedHazards = mergeHazards(hazards);
    
// Sort stretches by end time ascending; if same, by start time descending
    sort(stretches.begin(), stretches.end(), 
[](const pair<int, int>& a, const pair<int, int>& b) {
        if (a.second != b.second) return a.second < b.second;
        return a.first > b.first;
    });

    vector<int> placed_sensors;

    for (auto& stretch : stretches) {
        int L = stretch.first;
        int R = stretch.second;
        
        // Count how many sensors are already in this stretch
        int count = 0;
        for (int i = placed_sensors.size() - 1; i >= 0; i--) {
            if (placed_sensors[i] >= L && placed_sensors[i] <= R) {
                count++;
            }
            if (placed_sensors[i] < L || count == 2) break;
        }

       // Place remaining needed sensors greedily as far right as possible
        int needed = 2 - count;
        int curr = R;
        
        while (needed > 0) {
            // Check if curr is in a hazard zone using binary search
            bool in_hazard = false;
            int left = 0, right = mergedHazards.size() - 1;
            while (left <= right) {
                int mid = left + (right - left) / 2;
                if (curr >= mergedHazards[mid].first 
                && curr <= mergedHazards[mid].second) {
                    in_hazard = true;
                    curr = mergedHazards[mid].first - 1;
                                      // Skip the hazard zone
                    break;
                } else if (curr < mergedHazards[mid].first) {
                    right = mid - 1;
                } else {
                    left = mid + 1;
                }
            }
            
            if (!in_hazard) {
                if (curr < L) {
                    return -1; // Impossible to place enough sensors
                }
                placed_sensors.push_back(curr);
                sort(placed_sensors.begin(), placed_sensors.end());
                curr--;
                needed--;
            }
        }
    }

    return placed_sensors.size();
}

int main() {
    vector<pair<int, int>> stretches = {{1, 5}, {4, 8}, {7, 12}};
    vector<pair<int, int>> hazards = {{4, 4}, {9, 10}};
    
    int result = minSensors(stretches, hazards);
    if(result == -1) cout << "Not possible to place sensors." << endl;
    else cout << "Minimum sensors needed: " << result << endl;
    
    return 0;
}

```

**Things to remember for solving same kind of problems (in short):**

* **Greedy strategy on ends:** For interval covering, always sort by the ending coordinate. Placing points as far right as possible is the optimal way to maximize overlap with subsequent intervals.
* **Coordinate compression/Binary Search:** When limits are large (like 10^9), you cannot simulate step-by-step. Merge forbidden zones and use binary search to quickly skip over them in $O(\log N)$ time.

---
**Question 3: Count total possible ways for selecting K non-overlapping subarrays with given score S**

**Observations:**

* The score of a subarray `[i, j]` uniquely depends **only on its boundary elements** `A[i]` and `A[j]`, and its length. The elements strictly between `i` and `j` do not affect the score.
* The score formula is clearly defined:
* Length 1 (i == j): Score is `A[i]`
* Length > 1 (i < j): Score is `A[i] + A[j] + min(A[i], A[j]) * (j - i - 1)`


* We need to find the number of ways to pick exactly `K` non-overlapping subarrays such that the sum of their scores equals `S`. This screams **Dynamic Programming (DP)**, specifically the "Partition DP" pattern.

**Approach:**

1. **Define the Score Function:** Create a helper function `getScore(A, i, j)` that returns the score of subarray starting at index `i` and ending at index `j` based on the given formulas.
2. **DP State Definition:** Let `dp[c][i][v]` be the number of ways to select `c` non-overlapping subarrays from the prefix of the array ending at index `i` (considering elements `0` to `i-1`), such that their total score is exactly `v`.
3. **Base Case:**
* `dp[0][0][0] = 1`: There is 1 way to select 0 subarrays with a score of 0.


4. **State Transitions:** For each count `c` from 1 to `K`, for each index `i` from 1 to `N`, and for each possible sum `v` up to `S`:
* **Exclude the current element:** We can simply ignore the `i`-th element. The ways will be `dp[c][i-1][v]`.
* **Include the current element in a new subarray:** The `i`-th element can be the end of the `c`-th subarray. We can iterate over all possible starting points `j` (from 1 to `i`) for this subarray.
* If the score of subarray `[j, i]` is `curr_score`, and `v >= curr_score`, we add `dp[c-1][j-1][v - curr_score]` to our current state.


5. Return the result at `dp[K][N][S]`. (Since the values can be large, we should return the answer modulo `1e9+7`).

**Code (in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

const int MOD = 1e9 + 7;

// Function to calculate score of subarray A[i...j] (0-indexed)
int getScore(const vector<int>& A, int i, int j) {
    if (i == j) {
        return A[i];
    }
    return A[i] + A[j] + min(A[i], A[j]) * (j - i - 1);
}

int countWays(vector<int>& A, int K, int S) {
    int n = A.size();
    
// dp[c][i][v] = ways to select 'c' subarrays from 
prefix up to index 'i' with sum 'v'
// Using vector representation: c goes 0 to K, i goes 0 
to n, v goes 0 to S
    vector<vector<vector<int>>> dp(K + 1, 
vector<vector<int>>(n + 1, vector<int>(S + 1, 0)));
    
    // Base case
    for (int i = 0; i <= n; i++) {
        dp[0][i][0] = 1;
    }
    
    for (int c = 1; c <= K; c++) {
        for (int i = 1; i <= n; i++) {
            for (int v = 0; v <= S; v++) {
                
                // Option 1: Do not include A[i-1] in any subarray
                dp[c][i][v] = dp[c][i-1][v];
                
                // Option 2: A[i-1] is the end of the c-th subarray
                // Try all possible starting positions j for
 this subarray (0-indexed: j-1)
                for (int j = 1; j <= i; j++) {
                    int score = getScore(A, j - 1, i - 1);
                    if (v >= score) {
                        dp[c][i][v] = 
(dp[c][i][v] + dp[c-1][j-1][v - score]) % MOD;
                    }
                }
            }
        }
    }
    
    return dp[K][n][S];
}

int main() {
    vector<int> A = {2, 1, 3, 2};
    int K = 2;
    int S = 6; // Example target score
    
    cout << "Total ways: " << countWays(A, K, S) << endl;
    return 0;
}

```

**Things to remember for solving same kind of problems (in short):**

* **Knapsack / Partition DP pattern:** Whenever you are asked to select `K` non-overlapping segments to achieve a target `S`, a 3D DP of `[count][index][sum]` is the standard template.
* **Separating inclusion and exclusion:** The transition `dp[c][i] = dp[c][i-1] + sum(dp[c-1][j-1])` cleanly separates the choice of "skipping" the current element versus "ending a subarray" at the current element.
* **Precomputation/Helper functions:** If the scoring metric is weird but only depends on boundaries, isolate it in a helper function to keep your DP transitions clean and bug-free.

---
**Question 4: Maximize remaining edges weight in a tree such that each node has at most K edges**

**Observations:**

* This problem is defined on a tree and asks for optimal edge selection under a degree constraint, which is a classic signal for **Dynamic Programming on Trees (Tree DP)**.
* We can arbitrarily root the tree (e.g., at node 0).
* For any node `u` and its child `v`, we have a binary choice: either **keep** the edge `(u, v)` or **remove** it.
* The state of a node `u` depends purely on whether the edge to its parent is kept or removed:
* If the edge to the parent is **removed**, node `u` can connect to at most `K` of its children.
* If the edge to the parent is **kept**, node `u` has already used up one connection, so it can connect to at most `K - 1` of its children.



**Approach:**

1. Define a 2D DP array, `dp[N][2]`.
* `dp[u][0]`: Maximum weight in the subtree rooted at `u` if the edge between `u` and its parent is **removed**. (Node `u` has `K` available slots for its children).
* `dp[u][1]`: Maximum weight in the subtree rooted at `u` if the edge between `u` and its parent is **kept**. (Node `u` has `K - 1` available slots).


2. Perform a Depth First Search (DFS) starting from the root.
3. For a node `u`, iterate through all its children `v`:
* If we remove `(u, v)`, the contribution from the subtree `v` is `dp[v][0]`.
* If we keep `(u, v)` with weight `w`, the contribution from `v` is `dp[v][1] + w`.
* The "gain" of keeping the edge `(u, v)` over removing it is: `gain = (dp[v][1] + w) - dp[v][0]`.


4. Calculate the `base_sum` by assuming we remove *all* edges to the children of `u` (i.e., sum of all `dp[v][0]`).
5. Collect all positive gains from the children and sort them in descending order.
6. To compute `dp[u][0]`, add the top `K` positive gains to the `base_sum`.
7. To compute `dp[u][1]`, add the top `K - 1` positive gains to the `base_sum`.
8. The final answer is `dp[root][0]`, as the root has no parent.

**Code (in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

struct Edge {
    int to;
    long long weight;
};

// DFS function to compute DP values
void dfs(int u, int p, int k, const vector<vector<Edge>>& adj,
vector<vector<long long>>& dp) {
    long long base_sum = 0;
    vector<long long> gains;

    for (const auto& edge : adj[u]) {
        int v = edge.to;
        long long w = edge.weight;
        if (v == p) continue;

        // Solve for child
        dfs(v, u, k, adj, dp);

        // Base case: Assume we remove the edge to child v
        base_sum += dp[v][0];

        // Calculate the gain if we keep the edge to child v instead
        long long gain = (dp[v][1] + w) - dp[v][0];
        if (gain > 0) {
            gains.push_back(gain);
        }
    }

    // Sort gains in descending order to pick the most profitable edges
    sort(gains.rbegin(), gains.rend());

    // Calculate dp[u][0]: Node u can keep up to K edges to its children
    long long keep_k = 0;
    for (int i = 0; i < min((int)gains.size(), k); ++i) {
        keep_k += gains[i];
    }
    dp[u][0] = base_sum + keep_k;

    // Calculate dp[u][1]: Node u is connected to parent, can
keep up to K-1 edges to children
    long long keep_k_minus_1 = 0;
    if (k > 0) {
        for (int i = 0; i < min((int)gains.size(), k - 1); ++i) {
            keep_k_minus_1 += gains[i];
        }
    } else {
        // If K=0, we can't keep any edges, making this state invalid
        keep_k_minus_1 = -1e15; 
    }
    dp[u][1] = base_sum + keep_k_minus_1;
}

long long maxRemainingWeights(int n,int k,const
vector<vector<Edge>>& adj){
    if (k == 0) return 0; // Edge case: no edges can be kept
    
    vector<vector<long long>> dp(n, vector<long long>(2, 0));
    dfs(0, -1, k, adj, dp); // Start DFS from node 0
    
    return dp[0][0]; 
// Return the state where root is not connected to a parent
}

int main() {
    int n = 4; // Number of nodes
    int k = 1; // Max degree constraint
    
    // Adjacency list representation: {to, weight}
    vector<vector<Edge>> adj(n);
    adj[0].push_back({1, 10});
    adj[1].push_back({0, 10});
    adj[0].push_back({2, 20});
    adj[2].push_back({0, 20});
    adj[1].push_back({3, 30});
    adj[3].push_back({1, 30});

    cout << "Maximum remaining sum: " << 
maxRemainingWeights(n, k, adj) << endl;
    return 0;
}

```

**Things to remember for solving same kind of problems (in short):**

* **State based on parent connection:** In Tree DP problems involving degree constraints or matching (like max weight matching), the state always branches based on whether the current node uses an edge to its parent.
* **Greedy choice on children's DP values:** When transitioning from children to parent, assume a "default" state (e.g., removing all edges), calculate the delta/gain of switching states for each child, sort the gains, and greedily pick the top available valid options.

---
**Question 5: Minimum operations to get desired array by removing from the end and inserting anywhere**

**Observations:**

* The allowed operation is strictly removing from the **end** of the `current` array and inserting it anywhere.
* Because we can only pull elements from the rightmost side, any element that we **do not** move must have originally been at the front of the `current` array. Therefore, the elements that remain untouched will always form a **prefix** of the `current` array.
* Since these untouched elements are never moved, their relative order remains exactly the same. Thus, they must appear in the exact same relative order in the `desired` array (i.e., they must form a **subsequence** of the `desired` array).
* To minimize the number of operations, we need to maximize the number of elements we *don't* move.
* This reduces the problem to finding the length of the **longest prefix of the `current` array that is a subsequence of the `desired` array**. The answer will simply be the total number of elements minus this length.

**Approach:**

1. Initialize two pointers, `curr_ptr = 0` (for the `current` array) and `des_ptr = 0` (for the `desired` array).
2. Traverse the `desired` array using `des_ptr`.
3. If `current[curr_ptr] == desired[des_ptr]`, it means the current element of our prefix matches the current element in the desired array. We can safely increment `curr_ptr` to look for the next element in the prefix.
4. Always increment `des_ptr` to keep searching through the `desired` array.
5. When the traversal is complete, `curr_ptr` will hold the length of the longest matching prefix.
6. The minimum operations required will be `N - curr_ptr`, where `N` is the length of the array.

**Code (in simple C++):**

```cpp
#include <iostream>
#include <vector>

using namespace std;

int minOperations(vector<int>& current, vector<int>& desired) {
    int n = current.size();
    int curr_ptr = 0;
    
    // Traverse the desired array to find the longest prefix of current 
    // that exists as a subsequence in desired.
    for (int des_ptr = 0; des_ptr < n; des_ptr++) {
        if (curr_ptr < n && current[curr_ptr] == desired[des_ptr]) {
            curr_ptr++;
        }
    }
    
    // Total elements minus the elements we don't need to move
    return n - curr_ptr;
}

int main() {
    // Example: 
    // current = [1, 2, 3, 4]
    // desired = [2, 1, 3, 4]
    // Longest prefix of current in desired is [1]. Length = 1.
    // Ops = 4 - 1 = 3.
    vector<int> current = {1, 2, 3, 4};
    vector<int> desired = {2, 1, 3, 4};
    
    cout << "Minimum operations: " << 
minOperations(current, desired) << endl;
    return 0;
}

```

**Things to remember for solving same kind of problems (in short):**

* **Restricted operations on ends:** Whenever a problem allows moving elements only from one specific end (front or back), identify which elements remain **untouched**. They will always form a contiguous prefix or suffix.
* **Prefix vs Subsequence matching:** Matching an untouched segment to a target array almost always boils down to a greedy two-pointer approach checking for a subsequence.

---
**Question 6: Find total no of perfect subarrays from given array (a subarray is perfect if it doesn't contain any zero-sum subarray)**

**Observations:**

* A subarray is called "bad" if its sum is 0. We are looking for "perfect" subarrays, which are subarrays that do **not** contain any zero-sum sub-segment within them.
* The sum of a subarray from index `i` to `j` can be represented using a prefix sum array `P`, where `Sum(i, j) = P[j] - P[i-1]`.
* A subarray from `i` to `j` has a sum of 0 if `P[j] == P[i-1]`.
* Therefore, for a subarray `A[i...j]` to be perfect (containing no zero-sum subarrays), **all prefix sums in the range `P[i-1]` to `P[j]` must be strictly distinct**. If any two prefix sums in this range are equal, it means there is a zero-sum subarray perfectly trapped inside `A[i...j]`.
* This transforms the problem into finding the number of subarrays in the prefix sum array that contain entirely unique elements. We can efficiently solve this using the **Sliding Window (Two Pointers)** technique.

**Approach:**

1. Compute the prefix sum array `P` of size `N + 1`. Set `P[0] = 0`.
2. Use a sliding window defined by two pointers, `L` and `R`, both initially at 0.
3. Use a Hash Set to keep track of the prefix sums currently in our valid window.
4. Iterate `R` from 1 to `N`:
* While `P[R]` is already present in the Hash Set, it means we found a duplicate prefix sum. We must shrink the window from the left by removing `P[L]` from the set and incrementing `L`, until `P[R]` can be uniquely added.
* Add `P[R]` to the Hash Set.
* For the current right endpoint `R`, the number of valid starting points `i` is exactly `R - L`. We add `(R - L)` to our total answer.


5. Take modulo $10^9 + 7$ at each step of adding to the total count to avoid overflow.

**Code (in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <unordered_set>

using namespace std;

const int MOD = 1e9 + 7;

int countPerfectSubarrays(const vector<int>& A) {
    int n = A.size();
    
    // Calculate prefix sums
    // Using long long to prevent overflow of sums
    vector<long long> P(n + 1, 0);
    for (int i = 0; i < n; i++) {
        P[i + 1] = P[i] + A[i];
    }
    
    unordered_set<long long> window;
    long long total_perfect = 0;
    int L = 0;
    
    // P[0] is part of our initial state for 
subarray sums starting at index 0
    window.insert(P[0]);
    
    for (int R = 1; R <= n; R++) {
        // If P[R] is already in the window, shrink from the left
        while (window.count(P[R])) {
            window.erase(P[L]);
            L++;
        }
        
        // Add the current prefix sum to the window
        window.insert(P[R]);
        
        // The number of valid subarrays ending at R is the size of the 
        // current window minus 1 (which simplifies to R - L)
        total_perfect = (total_perfect + (R - L)) % MOD;
    }
    
    return total_perfect;
}

int main() {
    // Example: A = [1, -1, 1]
    // P = [0, 1, 0, 1]
    // Perfect subarrays: [1], [-1], [1] (Total 3)
    vector<int> A = {1, -1, 1};
    cout << "Total perfect subarrays: " <<
 countPerfectSubarrays(A) << endl;
    return 0;
}

```

**Things to remember for solving same kind of problems (in short):**

* **Zero-sum Subarrays = Duplicate Prefix Sums:** Any problem asking about subarrays summing to 0 is instantly a problem about finding duplicate values in a prefix sum array.
* **Sliding Window for Unique Elements:** To find subarrays with all distinct elements, a hash set combined with two pointers `L` and `R` is the standard $O(N)$ template.
* Always remember that for prefix sum logic, an array of size $N$ requires a prefix sum array of size $N+1$ initialized with $0$ at the start.

---
**Question 7: Calculate total no of possible array P such that sum of element = X and for all i P[i] >= A[i]**

**Observations:**

* We are asked to construct an array `P` of size `N` such that the sum of all elements in `P` equals `X`, and every element `P[i] >= A[i]`.
* We can rewrite the condition by saying that each element in `P` must take the base value of `A` plus some non-negative extra value. Let `P[i] = A[i] + k_i`, where `k_i >= 0`.
* Substituting this into the sum equation: `Sum(A[i] + k_i) = X`.
* This simplifies to: `Sum(k_i) = X - Sum(A[i])`.
* Let `S` be the sum of all elements in `A`, and the remaining sum be `R = X - S`.
* If `R < 0`, it is impossible to satisfy the condition, so the answer is 0.
* If `R >= 0`, the problem reduces to finding the number of ways to distribute a sum of `R` into `N` non-negative integer variables (`k_1, k_2, ..., k_N`).
* This is a standard combinatorics problem that can be solved using the **Stars and Bars** theorem. The number of non-negative integer solutions to sum up to `R` across `N` variables is given by the combinations formula `C(R + N - 1, N - 1)`.

**Approach:**

1. Calculate the sum of all elements in array `A`, let's call it `S`.
2. Find the remaining required sum `R = X - S`.
3. If `R < 0`, return 0 immediately.
4. If `R == 0`, return 1 (there is only one way: `P` exactly equals `A`).
5. We need to compute `C(R + N - 1, N - 1) modulo 1e9 + 7`. Since `R` can be very large (up to 1e9), but `N` is typically smaller, we compute the combination as:
`[ (R + 1) * (R + 2) * ... * (R + N - 1) ] / [ 1 * 2 * ... * (N - 1) ]`
6. Compute the numerator iteratively modulo 1e9 + 7.
7. Compute the denominator iteratively (which is just `(N-1)!`) modulo 1e9 + 7.
8. Use Fermat's Little Theorem to find the modular inverse of the denominator.
9. Multiply the numerator by the modular inverse of the denominator modulo 1e9 + 7 to get the final answer.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>

using namespace std;

const int MOD = 1e9 + 7;

// Function to compute (base^exp) % MOD
long long power(long long base, long long exp) {
    long long res = 1;
    base = base % MOD;
    while (exp > 0) {
        if (exp % 2 == 1) res = (res * base) % MOD;
        base = (base * base) % MOD;
        exp /= 2;
    }
    return res;
}

// Function to compute modular inverse using Fermat's Little Theorem
long long modInverse(long long n) {
    return power(n, MOD - 2);
}

int countArrays(const vector<int>& A, long long X) {
    int n = A.size();
    long long sumA = 0;
    
    for (int num : A) {
        sumA += num;
    }
    
    long long R = X - sumA;
    
    if (R < 0) return 0;
    if (n == 1) return 1; // R >= 0 and N=1 means exactly 1 way.
    
    // We need to calculate C(R + n - 1, n - 1)
    long long numerator = 1;
    long long denominator = 1;
    
    // Calculate (R+1)*(R+2)...*(R+n-1) % MOD and (n-1)! % MOD
    for (int i = 1; i < n; i++) {
        numerator = (numerator * ((R + i) % MOD)) % MOD;
        denominator = (denominator * i) % MOD;
    }
    
    // Result = (Numerator * modInverse(Denominator)) % MOD
    long long ans = (numerator * modInverse(denominator)) % MOD;
    
    return ans;
}

int main() {
    // Example: A = [1, 2], X = 5
    // Required R = 5 - (1+2) = 2. N = 2.
    // Ways to distribute 2 into 2 variables: 
C(2 + 2 - 1, 2 - 1) = C(3, 1) = 3.
    // The arrays P can be: [3, 2], [2, 3], [1, 4]
    vector<int> A = {1, 2};
    long long X = 5;
    
    cout << "Total possible arrays: " << countArrays(A, X) << endl;
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **Stars and Bars:** Any problem asking for the "number of ways to sum to X" or "distribute identical items into distinct bins" relies on the combinations formula `C(N + K - 1, K - 1)`.
* **Minimum threshold constraints:** Transform `P[i] >= A[i]` by shifting the target `X`. Subtract the minimum requirements upfront and solve for the non-negative remaining sum.
* **Modular Arithmetic with Combinations:** When calculating `nCr modulo M`, always compute the numerator and denominator separately, then use modular inverse for the division step: `(num * modInverse(den)) % MOD`.

---
**Question 8: Construct array placing multiples of K at left or right**

**Observations:**

* The array is built from the inside out. The first element placed becomes the core, and subsequent elements wrap around it on either the left or the right.
* Looking at the final constructed array of size N, the very last element placed (at step N) must be sitting at either the extreme left (index 0) or extreme right (index N-1).
* Furthermore, this last placed element must be a multiple of N.
* If we strip away this valid last element, the remaining subarray of length N-1 must have been validly constructed in N-1 steps.
* This "shrinking from the boundaries" behavior strongly indicates the use of Interval Dynamic Programming (DP on ranges).

**Approach:**

1. Define a 2D array `dp[i][j]` which stores the number of valid ways to construct the subarray from index `i` to index `j`.
2. The length of the subarray `A[i...j]` is `L = j - i + 1`. This length corresponds exactly to the `L`-th step of the construction process.
3. **Base Case:** Subarrays of length 1 (`i == j`). Since any integer is a multiple of 1, placing `A[i]` at step 1 is always valid. Set `dp[i][i] = 1` for all `i`.
4. **Transitions:** Iterate through all possible subarray lengths `L` from 2 up to N. For each length, iterate through all possible starting indices `i`, and calculate the ending index `j = i + L - 1`.
5. For a given range `[i, j]` of length `L`:
* Check if `A[i]` (leftmost element) is a multiple of `L`. If so, it could have been placed at step `L`. Add `dp[i+1][j]` to `dp[i][j]`.
* Check if `A[j]` (rightmost element) is a multiple of `L`. If so, it could have been placed at step `L`. Add `dp[i][j-1]` to `dp[i][j]`.


6. Take results modulo 1e9+7 at each addition to prevent overflow.
7. The total number of valid ways to construct the entire array is stored in `dp[0][N-1]`.

**Code (in simple C++):**

```cpp
#include <iostream>
#include <vector>

using namespace std;

const int MOD = 1e9 + 7;

int countConstructionWays(const vector<int>& A) {
    int n = A.size();
    if (n == 0) return 0;
    
    // dp[i][j] stores ways to construct subarray A[i...j]
    vector<vector<int>> dp(n, vector<int>(n, 0));
    
    // Base case: length 1. Any number is a multiple of 1.
    for (int i = 0; i < n; i++) {
        dp[i][i] = 1;
    }
    
    // Iterate over all subarray lengths from 2 to n
    for (int len = 2; len <= n; len++) {
        for (int i = 0; i <= n - len; i++) {
            int j = i + len - 1;
            
            // Option 1: A[i] was added on the left at step 'len'
            if (A[i] % len == 0) {
                dp[i][j] = (dp[i][j] + dp[i+1][j]) % MOD;
            }
            
            // Option 2: A[j] was added on the right at step 'len'
            if (A[j] % len == 0) {
                dp[i][j] = (dp[i][j] + dp[i][j-1]) % MOD;
            }
        }
    }
    
    // Return ways to form the complete array from 0 to n-1
    return dp[0][n - 1];
}

int main() {
    // Example: A = [2, 4]
    // Step 1: place 4 (len 1). Step 2: place 2 on left (len 2).
    // OR Step 1: place 2 (len 1). Step 2: place 4 on right(len 2).
    // Both are valid, so answer should be 2.
    vector<int> A = {2, 4};
    
    cout << "Total ways: " << countConstructionWays(A) << endl;
    
    return 0;
}

```

**Things to remember for solving same kind of problems (in short):**

* **Interval DP Pattern:** When a problem involves building a sequence by adding elements strictly to the ends (left or right), or breaking it down by removing from the ends, use `dp[i][j]` representing the range `[i, j]`.
* **Reverse Thinking:** Instead of simulating the building process from empty to full (which is hard to track), simulate the final array breaking down into smaller valid subarrays.
* **Length as Step Count:** In such range DP problems, the length of the current subarray `j - i + 1` almost always directly translates to the current "step number" or "turn number" in the game/process.

---
**Question 9: Find Distinct Second Highest Salary per Department in SQL**

**Observations:**

* We need the "distinct second highest" salary for each department. This means if two employees share the top salary, the next lower salary is considered the second highest.
* The `DENSE_RANK()` window function is ideal for this. Unlike `RANK()`, it assigns consecutive ranks even when there are ties (e.g., 1, 1, 2).
* We need to join the `employee` table with the `dept` table to retrieve the department name.
* Departments with fewer than two distinct salaries will naturally be excluded when we filter for rank 2.

**Approach:**

1. Create a Common Table Expression (CTE) or an inner subquery.
2. Inside the CTE, join `employee` and `dept` on `dept_id`.
3. Use `DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC)` to rank salaries within each department from highest to lowest.
4. In the outer query, select the required columns (`emp_name`, `emp_dept_name`, `salary`) from the CTE.
5. Apply a `WHERE` clause to filter only the rows where the calculated rank is exactly 2.
6. Finally, sort the output using `ORDER BY salary DESC`.

**Code (SQL):**

```sql
WITH RankedSalaries AS (
    SELECT 
        e.name AS emp_name,
        d.dept_name AS emp_dept_name,
        e.salary,
        DENSE_RANK() OVER (
            PARTITION BY e.dept_id 
            ORDER BY e.salary DESC
        ) as salary_rank
    FROM 
        employee e
    JOIN 
        dept d ON e.dept_id = d.dept_id
)
SELECT 
    emp_name, 
    emp_dept_name, 
    salary
FROM 
    RankedSalaries
WHERE 
    salary_rank = 2
ORDER BY 
    salary DESC;

```

**Things to remember for solving same kind of problems (in short):**

* **DENSE_RANK vs RANK:** Always use `DENSE_RANK()` for "N-th highest" problems to handle duplicate values without skipping rank numbers.
* **Window Functions Limitation:** Window functions like `DENSE_RANK()` cannot be used directly inside a `WHERE` clause. You must always wrap them in a CTE or a subquery first.
* **Filtering dynamically:** The `PARTITION BY` clause acts like a `GROUP BY` but preserves individual rows, allowing us to reset the rank counter for every new department.

---
**Question 10: Find the length of the Longest Palindromic Subsequence (LPS)**

*(Note: We have completed your 9 specific questions. We are now moving into the most critical standard questions covering the Infosys guidelines, starting with DP on Palindromic Subsequences.)*

**Observations:**

* A subsequence is a sequence that can be derived from another sequence by deleting some or no elements without changing the order of the remaining elements.
* A palindrome reads the same forwards and backwards.
* A clever trick for this specific problem: The longest palindromic subsequence of a string `S` is exactly the same as the Longest Common Subsequence (LCS) between the string `S` and its reverse string `rev_S`.
* For example, if `S = "bbbab"`, `rev_S = "babbb"`. The LCS of these two is `"bbbb"`, which has a length of 4.

**Approach:**

1. Create a new string `rev_S` which is the exact reverse of the input string `S`.
2. Apply the standard 2D Dynamic Programming approach for Longest Common Subsequence (LCS) on `S` and `rev_S`.
3. Create a 2D array `dp` of size `(N+1) x (N+1)` initialized to 0, where `N` is the length of the string.
4. Iterate `i` from 1 to `N` (for string `S`) and `j` from 1 to `N` (for string `rev_S`):
* If `S[i-1] == rev_S[j-1]`, it means the characters match. We add 1 to the result of the remaining strings: `dp[i][j] = 1 + dp[i-1][j-1]`.
* If they do not match, we take the maximum length by either ignoring the current character of `S` or ignoring the current character of `rev_S`: `dp[i][j] = max(dp[i-1][j], dp[i][j-1])`.


5. The final answer will be stored in `dp[N][N]`.

**Code (in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>

using namespace std;

int longestPalindromeSubseq(string s) {
    string rev_s = s;
    reverse(rev_s.begin(), rev_s.end()); // Reverse the string
    
    int n = s.length();
    
    // dp[i][j] stores the LCS of prefixes
    vector<vector<int>> dp(n + 1, vector<int>(n + 1, 0));
    
    // Fill the DP table
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= n; j++) {
            if (s[i - 1] == rev_s[j - 1]) {
                // Characters match
                dp[i][j] = 1 + dp[i - 1][j - 1];
            } else {
                // No match, take the best of two possibilities
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    
    return dp[n][n]; // Result is at the bottom-right cell
}

int main() {
    // Example: "bbbab" -> optimal palindrome is "bbbb"
    string s = "bbbab";
    cout << "Max LPS length: " << longestPalindromeSubseq(s) << endl;
    
    return 0;
}

```

**Things to remember for solving same kind of problems (in short):**

* **Palindrome DP trick:** Any problem asking for the "Longest Palindromic Subsequence" or minimum insertions/deletions to make a string palindrome can usually be mapped directly to finding the Longest Common Subsequence (LCS) of the string and its reverse.
* **1-based indexing for DP:** Always size your 2D DP array as `[N+1][M+1]` for string matching problems. This allows index 0 to cleanly represent an "empty string" base case without needing extra `if` checks for out-of-bounds indices.

---
**Question 11: 0/1 Knapsack Problem**

**Observations:**

* You are given a set of items, each with a weight and a value, and a knapsack with a maximum weight capacity `W`.
* You need to maximize the total value of items placed in the knapsack without exceeding its weight limit.
* It is called "0/1" because you have a binary choice for each item: you either pick it completely (1) or leave it (0). You cannot take fractions of an item.
* At each step, your decision depends on the remaining capacity of the knapsack and the items left to choose from. This points directly to Dynamic Programming.

**Approach:**

1. Create a 2D DP table `dp[N + 1][W + 1]`, where `N` is the number of items and `W` is the maximum capacity. Initialize it with 0s.
2. The state `dp[i][w]` represents the maximum value that can be achieved using the first `i` items with a knapsack capacity of `w`.
3. Use a nested loop: the outer loop `i` goes from 1 to `N` (iterating through items), and the inner loop `w` goes from 1 to `W` (iterating through all possible capacities).
4. For each cell `dp[i][w]`:
* **If the current item's weight is less than or equal to `w`:** We have a choice. We can either include the item (adding its value and subtracting its weight from the remaining capacity) or exclude it. We take the maximum of these two choices:
`max(val[i-1] + dp[i-1][w - wt[i-1]], dp[i-1][w])`
* **If the current item's weight is greater than `w`:** We cannot include this item. The maximum value is the same as if we didn't have this item:
`dp[i-1][w]`


5. The final answer, representing the max value for all items and full capacity, will be in `dp[N][W]`.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

int knapSack(int W, vector<int>& wt, vector<int>& val, int n) {
    // dp[i][w] array initialized to 0
    vector<vector<int>> dp(n + 1, vector<int>(W + 1, 0));

    // Build table dp[][] in bottom up manner
    for (int i = 1; i <= n; i++) {
        for (int w = 1; w <= W; w++) {
            if (wt[i - 1] <= w) {
                // Max of including or excluding the item
                // Line broken to respect 70 char limit
                dp[i][w] = max(val[i - 1] + dp[i - 1][w - wt[i - 1]],
                               dp[i - 1][w]);
            } else {
                // Item is heavier than current capacity w
                dp[i][w] = dp[i - 1][w];
            }
        }
    }

    return dp[n][W];
}

int main() {
    // Example: 3 items
    vector<int> val = {60, 100, 120};
    vector<int> wt = {10, 20, 30};
    int W = 50; 
    int n = val.size();
    
    cout << "Maximum Value: " << knapSack(W, wt, val, n) << endl;
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **Inclusion/Exclusion Principle:** The core of 0/1 Knapsack is the decision branch `max(include, exclude)`. Many DP problems (like Subset Sum, Target Sum) use this exact same branching logic.
* **Space Optimization:** The 2D DP array only ever looks at the `i-1` row to compute the `i` row. You can optimize the space complexity from O(N*W) to O(W) by using a 1D array and traversing the inner loop backwards (from `W` down to 0).
* **0/1 vs Unbounded:** If you can take multiple instances of the same item (Unbounded Knapsack), you simply change `dp[i-1][w - wt[i-1]]` to `dp[i][w - wt[i-1]]` to allow picking the same item again.

---
**Question 12: Longest Common Subsequence (LCS)**

**Observations:**

* You are given two strings, and you need to find the length of the longest subsequence present in both of them.
* A subsequence maintains the relative order of characters but does not require them to be contiguous (unlike a substring).
* If we look at the last characters of both strings, they either match or they do not. This binary outcome dictates how we build the solution from smaller subproblems, making it a classic Dynamic Programming problem.

**Approach:**

1. Create a 2D DP array `dp` of size `(N + 1) x (M + 1)`, where `N` and `M` are the lengths of the two strings. Initialize it with 0s.
2. `dp[i][j]` will store the length of the LCS for the prefix of the first string up to length `i` and the second string up to length `j`.
3. Use nested loops to iterate through every character combination of both strings (starting from index 1 to `N` and `M`).
4. **State Transition:**
* **If characters match** (`text1[i-1] == text2[j-1]`): The common subsequence grows by 1. We take the result of the strings without these matching characters (which is diagonally back) and add 1:
`dp[i][j] = 1 + dp[i-1][j-1]`
* **If characters do not match:** We cannot increase the sequence length here. The best we can do is take the maximum LCS found by either ignoring the current character of the first string or ignoring the current character of the second string:
`dp[i][j] = max(dp[i-1][j], dp[i][j-1])`


5. The final answer will be located at `dp[N][M]`.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>

using namespace std;

int longestCommonSubsequence(string text1, string text2) {
    int n = text1.length();
    int m = text2.length();
    
    // dp[i][j] array initialized to 0
    vector<vector<int>> dp(n + 1, vector<int>(m + 1, 0));
    
    // Fill the DP table bottom-up
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= m; j++) {
            if (text1[i - 1] == text2[j - 1]) {
                // Characters match, add 1 to the diagonal
                dp[i][j] = 1 + dp[i - 1][j - 1];
            } else {
                // No match, take max of left or top cell
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    
    return dp[n][m];
}

int main() {
    // Example strings
    string s1 = "abcde";
    string s2 = "ace"; 
    // LCS is "ace", length 3
    
    int result = longestCommonSubsequence(s1, s2);
    cout << "Longest Common Subsequence length: " << result << endl;
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **The String DP Template:** Almost every string matching DP problem (Edit Distance, Minimum Insertions/Deletions, Regular Expression Matching) uses this exact same 2D array setup with a check for `if (s1[i-1] == s2[j-1])`.
* **1-Based Indexing:** Notice how the strings are accessed with `i-1` and `j-1`, while the DP table uses `i` and `j`. This allows row 0 and column 0 to cleanly represent "empty strings" without causing array out-of-bounds errors.

---
**Question 13: Aggressive Cows (Maximize the Minimum Distance)**

**Observations:**

* You are given an array representing stall locations and a number `K` representing the number of cows. You need to place all `K` cows such that the minimum distance between any two of them is as large as possible.
* Any time a problem asks to **"maximize the minimum"** or **"minimize the maximum"**, it is a massive hint to use **Binary Search on the Answer**.
* We don't know the exact maximum distance, but we know its boundaries. The absolute minimum distance between two cows is 1. The absolute maximum distance would be placing one cow at the first stall and another at the last stall (after sorting).
* We can pick a hypothetical distance `mid` and greedily check if it is possible to place all `K` cows with at least that much distance between them.

**Approach:**

1. **Sort the array:** We need the stall locations in ascending order so we can easily calculate distances sequentially.
2. Define our search space: `low = 1` and `high = stalls[N-1] - stalls[0]`.
3. Create a helper function `canPlace(stalls, K, dist)`:
* Greedily place the first cow at the very first stall (`stalls[0]`).
* Iterate through the remaining stalls. If the distance between the current stall and the last placed cow is `>= dist`, place a cow here and update the last placed position.
* If we successfully place all `K` cows, return `true`. Otherwise, `false`.


4. Perform Binary Search:
* Calculate `mid = low + (high - low) / 2`.
* If `canPlace(..., mid)` is true, it means this distance works. Since we want to *maximize* the distance, we save this `mid` as our potential answer and move to the right half (`low = mid + 1`) to look for a bigger distance.
* If it is false, the distance is too large, so we move to the left half (`high = mid - 1`).


5. Return the highest valid distance found.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

// Helper to check if K cows can be placed with at least 'dist' gap
bool canPlace(const vector<int>& stalls, int k, int dist) {
    int count = 1; // Place first cow in the first stall
    int last_placed = stalls[0];
    
    for (int i = 1; i < stalls.size(); i++) {
        if (stalls[i] - last_placed >= dist) {
            count++;
            last_placed = stalls[i]; // Update last placed stall
        }
        if (count >= k) {
            return true; // We successfully placed all cows
        }
    }
    return false;
}

int maxMinDistance(vector<int>& stalls, int k) {
    // 1. Sort the array first
    sort(stalls.begin(), stalls.end());
    
    int n = stalls.size();
    int low = 1; 
    int high = stalls[n - 1] - stalls[0];
    int ans = 1;
    
    // 2. Binary Search on the answer space
    while (low <= high) {
        int mid = low + (high - low) / 2;
        
        if (canPlace(stalls, k, mid)) {
            ans = mid;      // Valid distance, save it
            low = mid + 1;  // Try to find a strictly larger distance
        } else {
            high = mid - 1; // Distance too large, decrease it
        }
    }
    
    return ans;
}

int main() {
    // Example: Stalls at 1, 2, 4, 8, 9. Place 3 cows.
    // Optimal placement: stalls 1, 4, 8 (or 1, 4, 9). Max min-dist = 3.
    vector<int> stalls = {1, 2, 8, 4, 9};
    int k = 3;
    
    cout << "Maximum possible minimum distance: " 
         << maxMinDistance(stalls, k) << endl;
    
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **Keywords:** Look for phrases like "Minimize the maximum workload", "Maximize the minimum distance", or "Allocate minimum pages". These are universally solved with the Binary Search on Answer pattern.
* **Greedy Checker:** The `canPlace` or `isValid` helper function almost always uses a straightforward greedy iteration.
* **Always Sort First:** Distance or capability problems on arrays require the array to be sorted before applying greedy logic.

---
**Question 14: Trapping Rain Water (Two Pointers)**

**Observations:**

* You are given an array representing an elevation map where the width of each bar is 1. You need to compute how much water it can trap after raining.
* The amount of water trapped above any specific bar `i` is determined by the tallest bar to its left (`left_max`) and the tallest bar to its right (`right_max`).
* Specifically, the water at index `i` is exactly `min(left_max, right_max) - height[i]`. If this value is negative, it traps 0 water.
* While you can precompute `left_max` and `right_max` arrays in O(N) space, the Two Pointers technique reduces the space complexity to O(1) by updating the maximums dynamically from both ends.

**Approach:**

1. Initialize two pointers: `left = 0` and `right = N - 1`.
2. Initialize two variables to track the maximum heights seen so far from both sides: `left_max = 0` and `right_max = 0`.
3. Initialize `total_water = 0`.
4. Loop while `left <= right`:
* **If `height[left] <= height[right]`:** The water trapped at the `left` pointer is bottlenecked by `left_max` (because we know there is a taller or equal bar at the `right` pointer).
* If `height[left] >= left_max`, update `left_max = height[left]` (no water trapped here, it becomes the new boundary).
* Else, add `left_max - height[left]` to `total_water`.
* Move the `left` pointer forward (`left++`).


* **If `height[left] > height[right]`:** The water trapped at the `right` pointer is bottlenecked by `right_max`.
* If `height[right] >= right_max`, update `right_max = height[right]`.
* Else, add `right_max - height[right]` to `total_water`.
* Move the `right` pointer backward (`right--`).




5. Return `total_water`.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

int trapRainWater(vector<int>& height) {
    int left = 0;
    int right = height.size() - 1;
    
    int left_max = 0;
    int right_max = 0;
    
    int total_water = 0;
    
    while (left <= right) {
        if (height[left] <= height[right]) {
            if (height[left] >= left_max) {
                left_max = height[left]; // Update max on left
            } else {
                total_water += (left_max - height[left]);
            }
            left++;
        } else {
            if (height[right] >= right_max) {
                right_max = height[right]; // Update max on right
            } else {
                total_water += (right_max - height[right]);
            }
            right--;
        }
    }
    
    return total_water;
}

int main() {
    // Example: Elevation map
    vector<int> height = {0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1};
    
    // Result should be 6
    cout << "Total trapped water: " << trapRainWater(height) << endl;
    
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **Min-Max Bottlenecks:** When a result depends on the minimum of two extremes (like left and right boundaries), Two Pointers is the optimal O(1) space strategy.
* **Pointer Movement Logic:** Always move the pointer that points to the smaller value. This guarantees that the smaller boundary is fully responsible for the water trapped at that specific index, allowing you to calculate it safely.

---
**Question 15: Longest Substring Without Repeating Characters (Sliding Window)**

**Observations:**

* You need to find the length of the longest contiguous sequence (substring) of characters where every character is unique.
* "Contiguous sequence" and "condition to maintain" (uniqueness) strongly point to the **Sliding Window** pattern.
* As you expand your window to the right, you might encounter a character you have already seen inside the current window.
* When a duplicate is found, the window becomes invalid. You must shrink it from the left until the duplicate is removed, making the window valid again.
* To optimize shrinking, instead of moving the left pointer step-by-step, you can use a Hash Map (or an array for ASCII characters) to store the exact last index where each character was seen, allowing the left pointer to jump directly.

**Approach:**

1. Initialize an array `last_seen` of size 256 (to cover all ASCII characters) with -1. This acts as our map to store the last index of each character.
2. Initialize two pointers, `left = 0` and `right = 0`.
3. Initialize `max_len = 0` to keep track of the maximum window size.
4. Loop `right` pointer through the string from 0 to length - 1:
* Let `c` be the current character `s[right]`.
* Check if `c` has been seen before AND its last seen index is greater than or equal to `left` (meaning the duplicate is currently inside our valid window).
* If true, move the `left` pointer to `last_seen[c] + 1` to exclude the previous occurrence of `c`.
* Update the last seen position of `c` to the current `right` index: `last_seen[c] = right`.
* Calculate the current window size `(right - left + 1)` and update `max_len` if it is larger.


5. Return `max_len`.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>

using namespace std;

int lengthOfLongestSubstring(string s) {
    // Array to store the last seen index of all 256 ASCII characters
    vector<int> last_seen(256, -1);
    
    int max_len = 0;
    int left = 0;
    
    for (int right = 0; right < s.length(); right++) {
        char c = s[right];
        
        // If character is in current window, jump the left pointer
        if (last_seen[c] >= left) {
            left = last_seen[c] + 1;
        }
        
        // Update the last seen index for the current character
        last_seen[c] = right;
        
        // Update the maximum length found so far
        max_len = max(max_len, right - left + 1);
    }
    
    return max_len;
}

int main() {
    // Example: "abcabcbb"
    // Answer is 3 (for substrings "abc", "bca", or "cab")
    string s = "abcabcbb";
    
    cout << "Longest valid substring length: " 
         << lengthOfLongestSubstring(s) << endl;
         
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **Dynamic Sliding Window:** If a problem asks for the longest/shortest contiguous subarray/substring satisfying a certain constraint, always default to a dynamic sliding window `[left, right]`.
* **Direct Jumping:** When avoiding duplicates, using an array/map to store the most recent index of elements allows you to jump the `left` pointer instantly in O(1) time, making the whole algorithm run in strict O(N) time.
* **Scope Check:** Always check `last_seen[c] >= left`. A character might have been seen before, but if its index is smaller than `left`, it is already outside the current window and safe to ignore.

---
**Question 16: Number of Islands (Graphs / DFS)**

**Observations:**

* You are given a 2D grid of '1's (land) and '0's (water). An island is formed by connecting adjacent lands horizontally or vertically.
* You need to find the total number of distinct islands.
* This is a classic "Connected Components" problem in a graph. Each '1' is a node, and edges exist between adjacent '1's.
* When we find a piece of land ('1'), it belongs to one island. To avoid counting the same island multiple times, we need a way to mark all connected lands as "visited".
* We can use Depth First Search (DFS) or Breadth First Search (BFS) to traverse the entire island and mark the land blocks as '0' (water) once visited.

**Approach:**

1. Initialize a counter `num_islands = 0`.
2. Iterate through every cell `(i, j)` in the given 2D grid using nested loops.
3. If the current cell is '1' (land):
* We have found a new island. Increment `num_islands` by 1.
* Launch a recursive DFS function starting from `(i, j)`.


4. Inside the DFS helper function `dfs(grid, i, j)`:
* **Base Case / Boundary Check:** If `i` or `j` is out of the grid boundaries (less than 0 or greater than grid dimensions), or if `grid[i][j]` is '0', return immediately.
* **Mark as Visited:** Change the current cell `grid[i][j]` to '0' so we don't visit it again.
* **Recursive Calls:** Call the DFS function for all 4 adjacent directions: up `(i-1, j)`, down `(i+1, j)`, left `(i, j-1)`, and right `(i, j+1)`.


5. After the nested loops finish, return `num_islands`.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>

using namespace std;

// DFS to mark the connected land as visited (sink the island)
void dfs(vector<vector<char>>& grid, int i, int j) {
    int rows = grid.size();
    int cols = grid[0].size();
    
    // Boundary checks and water/visited check
    if (i < 0 || i >= rows || j < 0 || j >= cols || 
        grid[i][j] == '0') {
        return;
    }
    
    // Mark current land as visited by making it water
    grid[i][j] = '0';
    
    // Explore all 4 adjacent directions
    dfs(grid, i + 1, j); // Down
    dfs(grid, i - 1, j); // Up
    dfs(grid, i, j + 1); // Right
    dfs(grid, i, j - 1); // Left
}

int numIslands(vector<vector<char>>& grid) {
    if (grid.empty()) return 0;
    
    int num_islands = 0;
    int rows = grid.size();
    int cols = grid[0].size();
    
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            // Found an unvisited land
            if (grid[i][j] == '1') {
                num_islands++;
                dfs(grid, i, j); // Sink the entire island
            }
        }
    }
    
    return num_islands;
}

int main() {
    vector<vector<char>> grid = {
        {'1', '1', '0', '0', '0'},
        {'1', '1', '0', '0', '0'},
        {'0', '0', '1', '0', '0'},
        {'0', '0', '0', '1', '1'}
    };
    
    // Total islands should be 3
    cout << "Number of Islands: " << numIslands(grid) << endl;
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **Grid Traversal Pattern:** Whenever dealing with 2D grids asking for paths, components, or regions, DFS or BFS is the standard approach.
* **In-place Modification:** If allowed, modifying the input grid directly (e.g., changing '1' to '0') is the best way to track visited cells without using O(N*M) extra space for a `visited` array.
* **Boundary Checks First:** In recursive grid functions, always put your bounds checking (indices < 0 or >= size) as the very first line of the function to prevent segmentation faults.

---
**Question 17: Lowest Common Ancestor of a Binary Tree (Trees / Recursion)**

**Observations:**

* You are given a binary tree and two nodes, `p` and `q`. You need to find their Lowest Common Ancestor (LCA).
* The LCA is defined as the lowest node in the tree that has both `p` and `q` as descendants (a node is allowed to be a descendant of itself).
* Trees are naturally recursive structures. We can search for `p` and `q` by traversing down to the leaves and passing information back up to the parents.
* If a node is either `p` or `q`, it could potentially be the LCA (if the other node is below it), so we should immediately return it to signal we found a target.

**Approach:**

1. **Base Case:** If the current `root` is `NULL`, return `NULL`. If the current `root` is equal to `p` or `q`, return `root`.
2. **Recursive Search:** Recursively call the LCA function on the `left` subtree and the `right` subtree. Let the results be `left_lca` and `right_lca`.
3. **Analyze Results:** After the recursive calls return to the current node, we have four possibilities:
* **Both `left_lca` and `right_lca` are NOT NULL:** This means `p` was found in one subtree and `q` was found in the other. Therefore, the current `root` must be their lowest common ancestor! Return `root`.
* **Only `left_lca` is NOT NULL:** Both nodes (or the LCA itself) are located in the left subtree. Return `left_lca` upwards.
* **Only `right_lca` is NOT NULL:** Both nodes are in the right subtree. Return `right_lca` upwards.
* **Both are NULL:** Neither node was found in either subtree. Return `NULL`.


4. The recursion will naturally bubble up the correct LCA node to the very top.

**Code(in simple C++):**

```cpp
#include <iostream>

using namespace std;

// Definition for a binary tree node.
struct TreeNode {
    int val;
    TreeNode *left;
    TreeNode *right;
    TreeNode(int x) : val(x), left(NULL), right(NULL) {}
};

TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, 
                               TreeNode* q) {
    // Base cases
    if (root == NULL) return NULL;
    if (root == p || root == q) return root;

    // Search left and right subtrees
    TreeNode* left_lca = lowestCommonAncestor(root->left, p, q);
    TreeNode* right_lca = lowestCommonAncestor(root->right, p, q);

    // If both return non-null, current node is the LCA
    if (left_lca != NULL && right_lca != NULL) {
        return root;
    }

    // Otherwise, return the non-null child (if any)
    if (left_lca != NULL) {
        return left_lca;
    } else {
        return right_lca;
    }
}

int main() {
    // Build a simple tree:
    //      3
    //     / \
    //    5   1
    TreeNode* root = new TreeNode(3);
    TreeNode* p = root->left = new TreeNode(5);
    TreeNode* q = root->right = new TreeNode(1);
    
    TreeNode* lca = lowestCommonAncestor(root, p, q);
    if (lca) cout << "LCA is: " << lca->val << endl;
    
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **Bottom-Up Recursion:** Tree problems where a parent node needs information from its children usually require a post-order traversal (process children first, then parent).
* **Bubbling Up Results:** When you find a target node, return it. As the returns bubble back up, the first node that receives non-null answers from *both* its left and right children is guaranteed to be the splitting point (the LCA).

---
**Question 18: Combination Sum (Backtracking)**

**Observations:**

* You are given an array of distinct integers and a target sum. You need to find all unique combinations that add up to the target.
* The catch is that the same number can be chosen an unlimited number of times.
* Because we need to generate *all possible valid combinations* rather than just counting them or finding a maximum, this is a textbook **Backtracking** problem.
* To avoid generating duplicate combinations (like [2, 2, 3] and [3, 2, 2]), we must enforce an order. Once we move past an element at a certain index, we should never look back at it.

**Approach:**

1. Create a `result` vector of vectors to store the final valid combinations, and a `current` vector to build the combination step-by-step.
2. Define a recursive `backtrack(candidates, target, index, current, result)` function.
3. **Base Cases:**
* If `target < 0`, the current combination exceeds the sum. Stop exploring this path (return).
* If `target == 0`, we found a valid combination! Add `current` to `result` and return.


4. **Recursive Step:** Loop from the given `index` to the end of the array.
* **Choose:** Add `candidates[i]` to the `current` combination.
* **Explore:** Recursively call `backtrack`. Pass the new target (`target - candidates[i]`). Crucially, pass `i` as the next index (NOT `i + 1`). Passing `i` allows the algorithm to pick the same number again.
* **Un-choose (Backtrack):** Remove the last added element from `current`. This undoes the choice so the loop can move on to try the next number in the array.


5. Initiate the recursion with starting index 0 and return the `result`.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>

using namespace std;

void backtrack(vector<int>& candidates, int target, int index, 
               vector<int>& current, vector<vector<int>>& result) {
    // Base cases
    if (target < 0) {
        return; // Exceeded the target, stop exploring
    }
    if (target == 0) {
        result.push_back(current); // Found a valid combination
        return;
    }

    // Iterate starting from 'index' to prevent duplicate sets
    for (int i = index; i < candidates.size(); i++) {
        // 1. Choose the current element
        current.push_back(candidates[i]);
        
        // 2. Explore further (keep index 'i' to reuse the element)
        backtrack(candidates, target - candidates[i], i, 
                  current, result);
        
        // 3. Un-choose (Backtrack) to explore other paths
        current.pop_back();
    }
}

vector<vector<int>> combinationSum(vector<int>& candidates, 
                                   int target) {
    vector<vector<int>> result;
    vector<int> current;
    
    backtrack(candidates, target, 0, current, result);
    
    return result;
}

int main() {
    vector<int> candidates = {2, 3, 6, 7};
    int target = 7;
    
    vector<vector<int>> result = combinationSum(candidates, target);
    
    cout << "Valid combinations:" << endl;
    for (const auto& combo : result) {
        cout << "[ ";
        for (int num : combo) cout << num << " ";
        cout << "]" << endl;
    }
    
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **The Backtracking Template:** Almost all subset, permutation, and combination problems use the exact same 3-step loop: `push_back()` -> `recursive call()` -> `pop_back()`.
* **Index Passing:** To generate combinations (where order doesn't matter), pass the current index forward. To generate combinations with unlimited reuse, pass `i`. To generate combinations where each item is used once, pass `i + 1`.
* **Target Subtraction:** Instead of passing a running sum and comparing it to the target, it is much cleaner to subtract from the target and check if it hits exactly 0.

---
**Question 19: Search in Rotated Sorted Array (Binary Search)**

**Observations:**

* You are given an array of unique integers that was originally sorted in ascending order but has been rotated at some unknown pivot. You need to search for a target value in `O(log n)` time.
* The `O(log n)` time complexity constraint strongly dictates that we must use **Binary Search**.
* Even though the entire array is not perfectly sorted, a key property remains: if you divide a rotated sorted array into two halves, **at least one half will always be perfectly sorted**.
* We can use this property to decide which half to discard. By checking the sorted half, we can definitively say whether our target lies inside it or not.

**Approach:**

1. Initialize two pointers: `low = 0` and `high = N - 1`.
2. While `low <= high`, calculate `mid = low + (high - low) / 2`.
3. If `nums[mid] == target`, we found the element! Return `mid`.
4. Check if the **left half is sorted** (`nums[low] <= nums[mid]`):
* If it is sorted, check if the `target` falls within this sorted range (i.e., `target >= nums[low]` AND `target < nums[mid]`).
* If the target is inside, we can safely discard the right half: `high = mid - 1`.
* If the target is NOT inside, it must be in the right half: `low = mid + 1`.


5. Otherwise, the **right half must be sorted** (`nums[mid] <= nums[high]`):
* Check if the `target` falls within this sorted range (i.e., `target > nums[mid]` AND `target <= nums[high]`).
* If the target is inside, discard the left half: `low = mid + 1`.
* If it is NOT inside, it must be in the left half: `high = mid - 1`.


6. If the loop ends without finding the target, return `-1`.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>

using namespace std;

int searchRotatedArray(vector<int>& nums, int target) {
    int low = 0;
    int high = nums.size() - 1;
    
    while (low <= high) {
        int mid = low + (high - low) / 2;
        
        if (nums[mid] == target) {
            return mid;
        }
        
        // Check if the left half is perfectly sorted
        if (nums[low] <= nums[mid]) {
            // Check if target is bounded within the sorted left half
            if (target >= nums[low] && target < nums[mid]) {
                high = mid - 1; // Search left
            } else {
                low = mid + 1;  // Search right
            }
        } 
        // Otherwise, the right half must be perfectly sorted
        else {
            // Check if target is bounded within the sorted right half
            if (target > nums[mid] && target <= nums[high]) {
                low = mid + 1;  // Search right
            } else {
                high = mid - 1; // Search left
            }
        }
    }
    
    return -1; // Target not found
}

int main() {
    // Array sorted and rotated at pivot index 3
    vector<int> nums = {4, 5, 6, 7, 0, 1, 2};
    int target = 0;
    
    int index = searchRotatedArray(nums, target);
    cout << "Target found at index: " << index << endl;
    
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **The "One Sorted Half" Rule:** In any rotated sorted array, splitting it at any index guarantees that at least one side of the split is strictly increasing.
* **Determine the Sorted Side First:** Always verify which half is sorted first by comparing `A[low]` with `A[mid]`. Once you know which side is predictable, you can easily check if your target fits inside its boundaries.

---
**Question 20: Merge Overlapping Intervals (Sorting / Greedy)**

**Observations:**

* You are given an array of intervals where each interval has a start and an end time. You need to merge all overlapping intervals into one.
* Two intervals overlap if the start time of the second interval is less than or equal to the end time of the first interval.
* If the intervals are completely unordered, checking for overlaps requires comparing every interval with every other interval (O(N^2)).
* However, if we **sort the intervals based on their start times**, any overlapping intervals will naturally end up right next to each other in the array. This allows us to merge them in a single linear pass (O(N)).

**Approach:**

1. If the input array is empty, return an empty array.
2. Sort the array of intervals in ascending order based on their starting values.
3. Create a `merged` vector to store the final combined intervals.
4. Push the first interval from the sorted array into the `merged` vector. This will act as our current working interval.
5. Iterate through the remaining intervals in the sorted array (from index 1 to N-1):
* Compare the `start` of the current interval with the `end` of the last interval stored in our `merged` vector.
* **If they overlap** (current `start` <= last `end`): Update the `end` of the last interval in `merged` to be the maximum of both ending times.
* **If they do not overlap** (current `start` > last `end`): The current interval is disjoint. Push it into the `merged` vector as a new working interval.


6. Return the `merged` vector.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

vector<vector<int>> mergeIntervals(vector<vector<int>>& intervals) {
    if (intervals.empty()) {
        return {};
    }
    
    // Sort intervals primarily by start time
    sort(intervals.begin(), intervals.end());
    
    vector<vector<int>> merged;
    // Push the first interval to start the merging process
    merged.push_back(intervals[0]);
    
    for (int i = 1; i < intervals.size(); i++) {
        // Get reference to the last interval in merged list
        vector<int>& last_interval = merged.back();
        
        // If current interval overlaps with the last one
        if (intervals[i][0] <= last_interval[1]) {
            // Update the end time to the maximum of both
            last_interval[1] = max(last_interval[1], intervals[i][1]);
        } else {
            // No overlap, safely add the new interval
            merged.push_back(intervals[i]);
        }
    }
    
    return merged;
}

int main() {
    // Example: [[1,3],[2,6],[8,10],[15,18]]
    // Output should be [[1,6],[8,10],[15,18]]
    vector<vector<int>> intervals = {{1, 3}, {2, 6}, {8, 10}, {15, 18}};
    
    vector<vector<int>> result = mergeIntervals(intervals);
    
    cout << "Merged Intervals:" << endl;
    for (const auto& interval : result) {
        cout << "[" << interval[0] << ", " << interval[1] << "] ";
    }
    cout << endl;
    
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **Sorting by Start Time:** Any problem involving scheduling, meeting rooms, or overlapping time ranges is almost always solved by sorting by the start time first.
* **The "Working Interval":** Instead of modifying the original array, keep pushing to a new list and always compare the next element against the `back()` (last element) of your new list.

---
**Question 21: Longest Increasing Subsequence (LIS)**

**Observations:**

* You are given an integer array, and you need to find the length of the strictly increasing subsequence.
* Unlike a subarray, a subsequence does not have to be contiguous, but the relative order of elements must be maintained.
* Every single element is trivially an increasing subsequence of length 1. This gives us our base case.
* To build a longer increasing subsequence ending at index `i`, we can look at all previous elements at index `j` (where `j < i`). If the current element `nums[i]` is strictly greater than `nums[j]`, we can append `nums[i]` to the subsequence ending at `j`.
* This overlapping subproblem structure is perfectly solved using 1D Dynamic Programming.

**Approach:**

1. Create a `dp` array of the same size as the input array `nums`.
2. Initialize all elements in `dp` to 1, because the minimum length of an LIS ending at any index is just the element itself.
3. Use an outer loop `i` from 1 to `N-1` to calculate the LIS ending at each specific index.
4. Use an inner loop `j` from 0 up to `i-1` to check all previously processed elements.
5. **State Transition:**
* If `nums[i] > nums[j]`, it means `nums[i]` can be appended to the increasing sequence ending at `j`.
* Update `dp[i]` to be the maximum of its current value and `dp[j] + 1`.


6. Keep track of the maximum value found in the entire `dp` array, as the longest increasing subsequence could end at any index, not necessarily the last one.
7. Return the maximum value.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

int lengthOfLIS(vector<int>& nums) {
    if (nums.empty()) return 0;
    
    int n = nums.size();
    // dp[i] stores the length of LIS ending at index i
    vector<int> dp(n, 1); 
    
    int max_lis = 1; // Minimum possible answer for non-empty array
    
    // Calculate DP values
    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            // If current element is greater, we can extend the sequence
            if (nums[i] > nums[j]) {
                dp[i] = max(dp[i], dp[j] + 1);
            }
        }
        // Track the global maximum length found so far
        max_lis = max(max_lis, dp[i]);
    }
    
    return max_lis;
}

int main() {
    // Example: LIS is [2, 3, 7, 101] -> length 4
    vector<int> nums = {10, 9, 2, 5, 3, 7, 101, 18};
    
    cout << "Length of Longest Increasing Subsequence: " 
         << lengthOfLIS(nums) << endl;
         
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **1D DP Pattern:** Problems asking for properties of a subsequence ending at a specific position often use a 1D DP array where `dp[i]` depends on a loop checking all `j < i`.
* **Global Maximum Tracking:** In this DP template, the final answer is rarely `dp[N-1]`. Always use a separate variable to track the maximum value written to the `dp` array.
* **Optimization Note:** While this O(N^2) DP approach is the standard, LIS can also be solved in O(N log N) using Binary Search with a greedy array, which is highly recommended to learn as a follow-up.

---
**Question 22: Course Schedule (Cycle Detection / Topological Sort)**

**Observations:**

* You have a total of `N` courses, labeled from 0 to `N-1`. You are given an array of `prerequisites` where `prerequisites[i] = [a, b]` indicates that you must take course `b` first if you want to take course `a`.
* This problem can be perfectly modeled as a **Directed Graph**. Each course is a node, and a prerequisite requirement represents a directed edge from `b` to `a`.
* It is impossible to finish all courses if and only if there is a **cycle** in the graph (e.g., to take course 1 you need course 2, and to take course 2 you need course 1).
* To detect cycles in a directed graph and find a valid ordering, Kahn's Algorithm (Breadth-First Search based Topological Sort) using "In-Degrees" is the most intuitive approach.

**Approach:**

1. **Graph Representation:** Create an adjacency list `adj` to represent the graph.
2. **In-Degree Array:** Create an `inDegree` array of size `N`, initialized to 0. The in-degree of a node is the number of prerequisites it has (incoming edges).
3. Populate both `adj` and `inDegree` by iterating through the `prerequisites` array.
4. **Initialize Queue:** Create a queue and push all nodes (courses) that have an `inDegree` of exactly 0. These are courses with no prerequisites, meaning we can take them immediately.
5. **Process the Queue:** Maintain a `completed_count` variable. While the queue is not empty:
* Pop the front node `curr`.
* Increment `completed_count`.
* Iterate through all neighbors (`nextCourse`) of `curr` in the adjacency list.
* Since we just "completed" course `curr`, decrement the `inDegree` of each neighbor by 1.
* If a neighbor's `inDegree` becomes 0, it means all its prerequisites are now met. Push it into the queue.


6. Finally, if `completed_count` equals `N`, we were able to finish all courses. Otherwise, a cycle exists, so return false.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <queue>

using namespace std;

bool canFinish(int numCourses, vector<vector<int>>& prerequisites) {
    vector<vector<int>> adj(numCourses);
    vector<int> inDegree(numCourses, 0);
    
    // Build adjacency list and in-degree array
    for (auto& pre : prerequisites) {
        adj[pre[1]].push_back(pre[0]);
        inDegree[pre[0]]++;
    }
    
    queue<int> q;
    // Push all courses with 0 prerequisites to the queue
    for (int i = 0; i < numCourses; i++) {
        if (inDegree[i] == 0) {
            q.push(i);
        }
    }
    
    int completed_count = 0;
    while (!q.empty()) {
        int curr = q.front();
        q.pop();
        completed_count++;
        
        // Reduce in-degree for all dependent courses
        for (int nextCourse : adj[curr]) {
            inDegree[nextCourse]--;
            // If all prerequisites are met, add to queue
            if (inDegree[nextCourse] == 0) {
                q.push(nextCourse);
            }
        }
    }
    
    // If we processed all courses, no cycle was found
    return completed_count == numCourses;
}

int main() {
    // Example: 2 courses. To take 1, you must take 0.
    int numCourses = 2;
    vector<vector<int>> prerequisites = {{1, 0}};
    
    if (canFinish(numCourses, prerequisites)) {
        cout << "It is possible to finish all courses." << endl;
    } else {
        cout << "It is NOT possible to finish all courses." << endl;
    }
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **Dependency Mapping:** Whenever a problem describes dependencies, tasks, or prerequisites (like "Task B must happen after Task A"), instantly think of Topological Sorting on a Directed Acyclic Graph (DAG).
* **Kahn's Algorithm Structure:** The three pillars of Kahn's Algorithm are: build the in-degree array, push all nodes with 0 in-degree to a queue, and dynamically reduce neighbors' in-degrees as you process the queue.
* **Cycle Detection:** If the graph has a cycle, some nodes will never reach an in-degree of 0, so the final processed count will be strictly less than the total number of nodes.

---
**Question 23: Diameter of a Binary Tree (Trees / Recursion)**

**Observations:**

* The diameter of a binary tree is the length of the longest path between any two nodes in a tree. This path may or may not pass through the root.
* The length of a path between two nodes is represented by the number of edges between them.
* For any given node, the longest path that passes *through* it (with it acting as the highest point or "curve" of the path) is exactly the depth (or height) of its left subtree plus the depth of its right subtree.
* Since the maximum diameter could be located entirely inside a left or right subtree, we must calculate this sum for *every* node in the tree and keep track of the maximum value found.

**Approach:**

1. Initialize a variable `max_diameter` to 0. We will pass this by reference to our recursive function to keep track of the global maximum.
2. Create a recursive helper function `height(node, max_diameter)` that returns the height of the tree rooted at `node`.
3. **Base Case:** If the `node` is NULL, its height is 0.
4. **Recursive Step:**
* Recursively call `height` for the `left` child to get `left_h`.
* Recursively call `height` for the `right` child to get `right_h`.


5. **Update Diameter:** The longest path passing through the current `node` is `left_h + right_h`. Update `max_diameter = max(max_diameter, left_h + right_h)`.
6. **Return Height:** The height of the current `node` (to be used by its parent) is `1 + max(left_h, right_h)`.
7. Start the recursion from the root and finally return `max_diameter`.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <algorithm>

using namespace std;

// Definition for a binary tree node.
struct TreeNode {
    int val;
    TreeNode *left;
    TreeNode *right;
    TreeNode(int x) : val(x), left(NULL), right(NULL) {}
};

// Helper function to calculate height and update max_diameter
int calculateHeight(TreeNode* node, int& max_diameter) {
    if (node == NULL) {
        return 0; // Base case: height of empty tree is 0
    }
    
    // Find height of left and right subtrees
    int left_h = calculateHeight(node->left, max_diameter);
    int right_h = calculateHeight(node->right, max_diameter);
    
    // The longest path through current node is left_h + right_h
    // Update global maximum if this path is longer
    max_diameter = max(max_diameter, left_h + right_h);
    
    // Return the height of current node to its parent
    return 1 + max(left_h, right_h);
}

int diameterOfBinaryTree(TreeNode* root) {
    int max_diameter = 0;
    calculateHeight(root, max_diameter);
    return max_diameter;
}

int main() {
    // Build a simple tree:
    //      1
    //     / \
    //    2   3
    //   / \
    //  4   5
    TreeNode* root = new TreeNode(1);
    root->left = new TreeNode(2);
    root->right = new TreeNode(3);
    root->left->left = new TreeNode(4);
    root->left->right = new TreeNode(5);
    
    // Diameter is between 4 and 5 (path: 4->2->5) or 4 and 3 
    // (path: 4->2->1->3). Length is 3 edges.
    cout << "Diameter of the tree: " 
         << diameterOfBinaryTree(root) << endl;
    
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **Global vs Return values:** When you need to compute a property that depends on every node but only return the maximum (like max path sum, diameter, longest univalue path), use a reference variable to track the max while returning the standard height/depth to the parent.
* **Post-order Traversal:** Calculate properties of children first, use them to calculate the parent's property, and then update the global state. This is textbook bottom-up recursion.

---
**Question 24: Coin Change (Dynamic Programming)**

**Observations:**

* You are given an integer array of coins of different denominations and an integer `amount` representing a total amount of money. You need to return the fewest number of coins that make up that amount.
* You can assume you have an infinite number of each kind of coin.
* This is a classic "Unbounded Knapsack" DP problem. The minimum coins needed to make `amount` depends on the minimum coins needed to make `amount - coin_value` for each available coin.
* We want to minimize the result, so we should initialize our DP table with a very large value (infinity) representing an impossible state.

**Approach:**

1. Create a 1D DP array of size `amount + 1`.
2. `dp[i]` will store the minimum number of coins needed to make the amount `i`.
3. Initialize all elements in the `dp` array to `amount + 1`. This value acts as our "infinity" because the maximum possible number of coins we could ever use is `amount` (if we only had 1-value coins).
4. **Base Case:** `dp[0] = 0`. It takes 0 coins to make an amount of 0.
5. Loop through each amount `i` from 1 up to `amount`.
6. Inside, loop through each `coin` in the given coins array.
* If the `coin` value is less than or equal to the current amount `i`, we can use this coin.
* State Transition: `dp[i] = min(dp[i], dp[i - coin] + 1)`. We take the minimum between our current best for `dp[i]` and the result of using one more of this `coin`.


7. After filling the DP table, check `dp[amount]`. If it is still `amount + 1`, it means no combination of coins can make the target amount, so return `-1`. Otherwise, return `dp[amount]`.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

int coinChange(vector<int>& coins, int amount) {
    // amount + 1 acts as infinity
    vector<int> dp(amount + 1, amount + 1);
    
    // Base case: 0 coins needed to make amount 0
    dp[0] = 0; 
    
    // Build the DP table bottom-up
    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            // Check if we can use this coin
            if (coin <= i) {
                // Update minimum coins needed
                dp[i] = min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    
    // If it's still > amount, it's impossible to form the amount
    if (dp[amount] > amount) {
        return -1;
    }
    
    return dp[amount];
}

int main() {
    // Example: coins = [1, 2, 5], amount = 11
    // Optimal choice is 5 + 5 + 1 = 3 coins
    vector<int> coins = {1, 2, 5};
    int amount = 11;
    
    cout << "Minimum coins needed: " 
         << coinChange(coins, amount) << endl;
         
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **Initialization to Infinity:** For minimization DP problems, always initialize the DP array to a safe maximum (like `amount + 1` or `INT_MAX - 1`) instead of 0.
* **Unbounded Knapsack Structure:** Because we can use a coin multiple times, we look forward sequentially (`1` to `amount`), reading from previously updated values in the same row. In 0/1 knapsack, we read backwards to avoid using the same item twice.

---
**Question 25: Split Array Largest Sum (Minimize the Maximum)**

**Observations:**

* You are given an integer array `nums` and an integer `k`. You need to split `nums` into `k` non-empty contiguous subarrays such that the largest sum among these `k` subarrays is minimized.
* The phrase **"minimize the largest sum"** is the ultimate signal for using **Binary Search on the Answer**.
* We don't know the exact partition, but we know the boundaries of our answer space:
* **Lower Bound (Best Case):** If `k == nums.size()`, every element is its own subarray. The largest sum is simply the maximum element in the array.
* **Upper Bound (Worst Case):** If `k == 1`, the entire array is one subarray. The largest sum is the sum of all elements.


* If we pick a hypothetical maximum sum `mid`, we can greedily check if it's possible to split the array into `k` or fewer parts without any part exceeding `mid`.

**Approach:**

1. Find the maximum element in the array (set as `low`) and the total sum of the array (set as `high`).
2. Create a helper function `canSplit(nums, k, target_sum)`:
* Keep a running `current_sum` and a `splits` counter starting at 1.
* Iterate through the array. If adding the current element to `current_sum` exceeds `target_sum`, we must end the current subarray and start a new one.
* Increment `splits`, and reset `current_sum` to the current element.
* If `splits` exceeds `k` at any point, return `false`. Otherwise, return `true`.


3. Perform Binary Search while `low <= high`:
* Calculate `mid = low + (high - low) / 2`.
* If `canSplit(nums, k, mid)` is true, it means `mid` is a valid maximum sum. Since we want to *minimize* it, we record `mid` as a potential answer and search the left half (`high = mid - 1`).
* If it is false, `mid` is too small to group the elements into just `k` parts. We must allow larger sums, so search the right half (`low = mid + 1`).


4. Return the lowest valid `mid` found.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <numeric>
#include <algorithm>

using namespace std;

// Greedily check if we can split into <= k parts 
// such that no part exceeds maxSum
bool canSplit(const vector<int>& nums, int k, long long maxSum) {
    long long currentSum = 0;
    int splits = 1; // At least one split exists (the array itself)
    
    for (int num : nums) {
        if (currentSum + num > maxSum) {
            splits++; // Need a new subarray
            currentSum = num; // Start new subarray with current element
            
            if (splits > k) {
                return false; // Too many splits required
            }
        } else {
            currentSum += num; // Add to current subarray
        }
    }
    return true;
}

long long splitArray(vector<int>& nums, int k) {
    long long low = 0;
    long long high = 0;
    
    // Low is max element, High is sum of all elements
    for (int num : nums) {
        low = max(low, (long long)num);
        high += num;
    }
    
    long long ans = high;
    
    // Binary search the answer space
    while (low <= high) {
        long long mid = low + (high - low) / 2;
        
        if (canSplit(nums, k, mid)) {
            ans = mid;       // Valid, but try to find a smaller max
            high = mid - 1;
        } else {
            low = mid + 1;   // Invalid, need a larger max capacity
        }
    }
    
    return ans;
}

int main() {
    // Example: Split into 2 parts. 
    // Best split: [7,2,5] (sum=14) and [10,8] (sum=18). Max is 18.
    vector<int> nums = {7, 2, 5, 10, 8};
    int k = 2;
    
    cout << "Minimized largest sum: " << splitArray(nums, k) << endl;
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **The "Capacity" Pattern:** This exact binary search logic solves a huge family of hard problems: "Capacity to Ship Packages within D Days", "Koko Eating Bananas", and "Minimum Number of Days to Make m Bouquets".
* **State of the Helper:** The greedy helper function always simulates reading left-to-right, maintaining a running total, and forcing a "reset" (incrementing a counter) whenever the hypothetical boundary `mid` is breached.
* **Long Long precision:** When the answer space involves the sum of an entire array, always use 64-bit integers (`long long` in C++) for `low`, `high`, and `mid` to prevent integer overflow.

---
**Question 26: Maximum Sum BST in Binary Tree (Advanced Tree DP)**

**Observations:**

* You are given a binary tree. You need to find the maximum sum of all keys of any sub-tree which is also a valid Binary Search Tree (BST).
* A subtree is a valid BST if and only if:
1. Its left subtree is a valid BST.
2. Its right subtree is a valid BST.
3. The maximum node value in the left subtree is strictly less than the root's value.
4. The minimum node value in the right subtree is strictly greater than the root's value.


* Because a parent node's validity and sum depend entirely on the properties of its left and right children, this requires a **bottom-up (post-order) Tree DP** approach.
* We must pass up multiple pieces of information from children to parents: whether it is a BST, the minimum value, the maximum value, and the sum.

**Approach:**

1. Create a custom structure `SubtreeInfo` to hold four variables: `isBST` (boolean), `minNode` (integer), `maxNode` (integer), and `sum` (integer).
2. Create a recursive function `postOrder(node, maxSum)` that returns this `SubtreeInfo`.
3. **Base Case:** If the node is NULL, it is technically a valid BST with a sum of 0. To make the math work for leaf nodes, return `isBST = true`, `minNode = INT_MAX`, and `maxNode = INT_MIN`.
4. **Recursive Step:** Call the function on the left and right children to get their `SubtreeInfo`.
5. **Check BST Condition:**
* If `left.isBST` is true AND `right.isBST` is true AND the current node's value is strictly greater than `left.maxNode` AND strictly less than `right.minNode`, then the tree rooted at the current node is a valid BST.


6. **If it is a valid BST:**
* Calculate `currentSum = root->val + left.sum + right.sum`.
* Update the global `maxSum` with `currentSum` (if it's larger).
* Return a new `SubtreeInfo` where `isBST = true`, `minNode = min(root->val, left.minNode)`, `maxNode = max(root->val, right.maxNode)`, and `sum = currentSum`.


7. **If it is NOT a valid BST:**
* Return `SubtreeInfo` with `isBST = false`. The other values don't matter anymore because this invalidity will propagate upwards.



**Code(in simple C++):**

```cpp
#include <iostream>
#include <algorithm>
#include <climits>

using namespace std;

// Definition for a binary tree node.
struct TreeNode {
    int val;
    TreeNode *left;
    TreeNode *right;
    TreeNode(int x) : val(x), left(NULL), right(NULL) {}
};

// Structure to pass multiple states up the recursive tree
struct SubtreeInfo {
    bool isBST;
    int minNode;
    int maxNode;
    int sum;
};

SubtreeInfo postOrder(TreeNode* root, int& maxSum) {
    if (root == NULL) {
        // Base case: null nodes are valid BSTs
        return {true, INT_MAX, INT_MIN, 0};
    }

    // Process left and right children first (Bottom-up)
    SubtreeInfo left = postOrder(root->left, maxSum);
    SubtreeInfo right = postOrder(root->right, maxSum);

    // Check if the current subtree is a valid BST
    if (left.isBST && right.isBST && 
        root->val > left.maxNode && root->val < right.minNode) {
        
        int currentSum = root->val + left.sum + right.sum;
        maxSum = max(maxSum, currentSum);
        
        // Compute min and max for the current subtree
        int currMin = min(root->val, left.minNode);
        int currMax = max(root->val, right.maxNode);
        
        return {true, currMin, currMax, currentSum};
    }

    // If it's not a BST, just pass false. Other values don't matter.
    return {false, 0, 0, 0};
}

int maxSumBST(TreeNode* root) {
    int maxSum = 0; // If all negative, return 0 (empty BST)
    postOrder(root, maxSum);
    return maxSum;
}

int main() {
    // Tree: 
    //       1
    //      / \
    //     4   3
    //    / \   \
    //   2   4   2
    //    \       \
    //     3       6
    // (Note: left subtree of 1 is not BST, right subtree of 1 is not BST. 
    // The max sum BST is the leaf node 3 or 6).
    TreeNode* root = new TreeNode(1);
    root->left = new TreeNode(4);
    root->left->left = new TreeNode(2);
    root->left->right = new TreeNode(4);
    root->right = new TreeNode(3);
    
    cout << "Maximum Sum of a valid BST: " << maxSumBST(root) << endl;
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **Custom Structs for Tree DP:** When a parent node in a tree needs more than just a single integer from its children to make a decision (e.g., it needs limits, boolean flags, and sums), encapsulate them in a `struct` or a `vector`.
* **Infinity Bounds Trick:** When checking BST boundaries, setting a NULL node's min to `INT_MAX` and max to `INT_MIN` guarantees that the boundary condition (`root->val > left.maxNode && root->val < right.minNode`) perfectly succeeds for leaf nodes without needing extra if-else checks.

---
**Question 27: Subarrays with K Different Integers (Advanced Sliding Window)**

**Observations:**

* You are given an integer array and an integer `K`. You need to find the total number of contiguous subarrays that contain exactly `K` different integers.
* Finding "exactly K" distinct elements using a standard sliding window is very tricky because when you shrink the window, you might still have `K` elements, making it hard to count all valid sub-segments accurately.
* However, finding "at most K" distinct elements is a classic, straightforward sliding window problem.
* We can use a powerful mathematical trick: The number of subarrays with **exactly K** distinct elements is equal to the number of subarrays with **at most K** distinct elements MINUS the number of subarrays with **at most (K - 1)** distinct elements.
* `Exact(K) = AtMost(K) - AtMost(K-1)`

**Approach:**

1. Create a helper function `atMostK(nums, k)` that returns the count of subarrays with at most `k` distinct integers.
2. Inside `atMostK`, initialize two pointers `left = 0` and `right = 0`, a `result` counter, and a Hash Map (or frequency array) to track the count of each integer in the current window.
3. Iterate the `right` pointer through the array:
* Add the current element `nums[right]` to the frequency map. If its count becomes 1, it means we found a new distinct element, so decrement `k`.
* If `k < 0`, our window has exceeded the allowed distinct elements. We must shrink it from the left.
* While `k < 0`, decrement the frequency of `nums[left]`. If its frequency drops to 0, we have completely removed a distinct element from the window, so increment `k`. Move the `left` pointer forward.
* Once the window is valid (`k >= 0`), the number of valid subarrays ending exactly at the `right` pointer is simply `(right - left + 1)`. Add this to `result`.


4. In the main function, simply return `atMostK(nums, K) - atMostK(nums, K - 1)`.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <unordered_map>

using namespace std;

// Helper to find subarrays with AT MOST k distinct integers
int atMostK(const vector<int>& nums, int k) {
    unordered_map<int, int> freq;
    int left = 0;
    int result = 0;
    
    for (int right = 0; right < nums.size(); right++) {
        // If this is the first time seeing this number, reduce k
        if (freq[nums[right]] == 0) {
            k--;
        }
        freq[nums[right]]++;
        
        // Window invalid (too many distinct elements), shrink left
        while (k < 0) {
            freq[nums[left]]--;
            // If completely removed, we gained back a distinct slot
            if (freq[nums[left]] == 0) {
                k++;
            }
            left++;
        }
        
        // Add all valid subarrays ending at the current right pointer
        result += (right - left + 1);
    }
    
    return result;
}

int subarraysWithKDistinct(vector<int>& nums, int k) {
    // Exact(K) = AtMost(K) - AtMost(K-1)
    return atMostK(nums, k) - atMostK(nums, k - 1);
}

int main() {
    // Example: nums = [1,2,1,2,3], K = 2
    // Valid subarrays: [1,2], [2,1], [1,2], [2,3], 
    // [1,2,1], [2,1,2], [1,2,1,2] -> Total 7
    vector<int> nums = {1, 2, 1, 2, 3};
    int k = 2;
    
    cout << "Subarrays with exactly K distinct integers: " 
         << subarraysWithKDistinct(nums, k) << endl;
         
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **The "At Most" Subtraction Trick:** Whenever a sliding window problem asks for an "Exact" count of constraints (exactly K distinct, exactly K odd numbers, sum exactly S), convert it to `AtMost(K) - AtMost(K-1)`. This makes the window logic trivially simple.
* **Combinatorics in Windows:** The formula `(right - left + 1)` is crucial. It represents the number of valid contiguous sub-segments that end specifically at the `right` index.

---
**Question 28: Burst Balloons (Advanced Interval DP)**

**Observations:**

* You are given an array of balloons, each with a coin value. Bursting balloon `i` gets you `nums[i-1] * nums[i] * nums[i+1]` coins.
* Once a balloon is burst, the left and right balloons become adjacent.
* The changing adjacency makes standard left-to-right DP impossible because future states depend heavily on which specific elements were removed in the past.
* **The Trick:** Think in reverse! Instead of picking which balloon to burst *first*, guess which balloon will be burst *last* within a specific sub-range.
* If balloon `k` is the last one to burst in the range `[left, right]`, it means all other balloons inside this range are already gone. Therefore, balloon `k` will directly touch the balloons immediately outside this range: `left - 1` and `right + 1`.

**Approach:**

1. To handle boundary conditions easily (where a balloon doesn't have a left or right neighbor), create a new array `arr` and pad the original array with a `1` at the beginning and a `1` at the end.
2. Initialize a 2D DP table `dp[i][j]` where the value represents the maximum coins you can collect by bursting all balloons strictly bounded within the interval `[i, j]`.
3. Use three nested loops (standard for Interval DP):
* Outer loop: `len` (length of the current interval being solved) from 1 to `N`.
* Middle loop: `left` (start of the interval) from 1 to `N - len + 1`.
* Inner loop: `k` (the index of the last balloon to burst) from `left` to `right`.


4. **State Transition:** For a chosen last balloon `k`, the coins gained will be `arr[left-1] * arr[k] * arr[right+1]`.
5. We add this to the best possible scores of the left sub-problem `dp[left][k-1]` and right sub-problem `dp[k+1][right]`.
6. Update `dp[left][right]` with the maximum value found across all possible choices of `k`.
7. Return `dp[1][N]`.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

int maxCoins(vector<int>& nums) {
    int n = nums.size();
    
    // Pad the array with 1s at both ends for boundary conditions
    vector<int> arr(n + 2, 1);
    for (int i = 0; i < n; i++) {
        arr[i + 1] = nums[i];
    }
    
    // dp[i][j] stores max coins from bursting balloons in [i, j]
    vector<vector<int>> dp(n + 2, vector<int>(n + 2, 0));
    
    // len is the length of the interval we are evaluating
    for (int len = 1; len <= n; len++) {
        for (int left = 1; left <= n - len + 1; left++) {
            int right = left + len - 1;
            
            // k represents the LAST balloon to burst in this range
            for (int k = left; k <= right; k++) {
                
                // Calculate coins gained by bursting 'k' last
                int coins = arr[left - 1] * arr[k] * arr[right + 1];
                
                // Add solutions from left and right sub-intervals
                int total = dp[left][k - 1] + coins + dp[k + 1][right];
                
                dp[left][right] = max(dp[left][right], total);
            }
        }
    }
    
    return dp[1][n]; // Return max coins for the whole original range
}

int main() {
    // Example: Burst 1(gets 15), then 5(gets 15), then 8(gets 40)
    // Max coins: 3*1*5 + 3*5*8 + 1*3*8 + 1*8*1 (order matters)
    // Optimal order yields 167
    vector<int> nums = {3, 1, 5, 8};
    
    cout << "Maximum coins: " << maxCoins(nums) << endl;
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **Reverse State Definition:** Whenever elements merge, collapse, or depend on dynamically changing neighbors (like Matrix Chain Multiplication, Burst Balloons, or Optimal Binary Search Trees), always ask: "What if I choose the LAST element to process instead of the first?"
* **Padding Arrays:** Adding dummy variables (like 1s) to the boundaries of an array eliminates writing messy out-of-bounds `if` logic inside your core DP loops.

---
**Question 29: Binary Tree Cameras (Advanced Tree DP / Greedy)**

**Observations:**

* You are given a binary tree. You need to place the minimum number of cameras on the nodes such that every node is monitored.
* A camera placed on a node monitors its parent, itself, and its immediate children.
* Placing a camera on a leaf node is highly inefficient because it only monitors the leaf and its parent. Placing it on the parent of a leaf monitors the parent, the leaf, the other child, and the grandparent!
* This tells us we should make decisions from the bottom up. We want to strictly avoid placing cameras on leaves.
* To achieve this, a post-order traversal (Tree DP) is required. Each node can be in one of three states:
* State `0`: The node is **unmonitored** and strictly needs a camera from its parent.
* State `1`: The node **has a camera** placed on it.
* State `2`: The node is **monitored** by one of its children (no camera on itself).



**Approach:**

1. Create a variable `cameras = 0` to keep track of the total cameras placed.
2. Create a recursive `dfs(node)` function that returns the state (0, 1, or 2) of the current node.
3. **Base Case:** If the node is `NULL`, pretend it is already monitored by returning `2`. This ensures we don't accidentally place a camera to monitor a non-existent node.
4. **Recursive Step:** Get the states of the `left` and `right` children using `dfs`.
5. **Decision Logic:**
* **Rule 1:** If either the left OR right child returns `0` (unmonitored), we have no choice. We MUST place a camera at the current node to save that child. Increment `cameras` and return `1`.
* **Rule 2:** If either child returns `1` (has a camera), and the other doesn't need one, the current node is safely monitored by that child's camera. Return `2`.
* **Rule 3:** If both children return `2` (they are monitored but have no cameras), the current node is currently left out and unmonitored. It must ask its parent for help. Return `0`.


6. **Root Edge Case:** In the main function, call `dfs(root)`. If the root itself returns `0`, it means it is unmonitored and has no parent to help it. We must manually place a camera at the root (increment `cameras` by 1).

**Code(in simple C++):**

```cpp
#include <iostream>

using namespace std;

// Definition for a binary tree node.
struct TreeNode {
    int val;
    TreeNode *left;
    TreeNode *right;
    TreeNode(int x) : val(x), left(NULL), right(NULL) {}
};

int cameras = 0; // Global or member variable to track count

// Returns 0: Needs Camera, 1: Has Camera, 2: Monitored
int dfs(TreeNode* node) {
    if (node == NULL) {
        return 2; // Null nodes don't need monitoring
    }
    
    int left_state = dfs(node->left);
    int right_state = dfs(node->right);
    
    // If any child is unmonitored, place a camera here
    if (left_state == 0 || right_state == 0) {
        cameras++;
        return 1; // Node now has a camera
    }
    
    // If any child has a camera, current node is monitored
    if (left_state == 1 || right_state == 1) {
        return 2; // Node is monitored without a camera
    }
    
    // If both children are monitored (state 2), 
    // current node is left unmonitored. 
    return 0; 
}

int minCameraCover(TreeNode* root) {
    cameras = 0; // Reset for safety
    
    // If the root is left unmonitored, it needs its own camera
    if (dfs(root) == 0) {
        cameras++;
    }
    
    return cameras;
}

int main() {
    // Tree:     0
    //          /
    //         0
    //        / \
    //       0   0
    // Optimal: Place 1 camera at the middle node to monitor all.
    TreeNode* root = new TreeNode(0);
    root->left = new TreeNode(0);
    root->left->left = new TreeNode(0);
    root->left->right = new TreeNode(0);
    
    cout << "Minimum cameras needed: " << minCameraCover(root) << endl;
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **State Machine on Trees:** For advanced Tree DP, instead of returning just numbers or booleans, define 3 or 4 strict "States" and use integer codes.
* **Greedy Bottom-Up:** When coverage overlaps (like a camera covering parents and children), the most optimal greedy choice is to force the decision onto the parent of the deepest nodes to maximize the coverage spread.
* **Root Edge Cases:** Always check the final return value of your recursive function in the main wrapper. The root node has no parent to fulfill its "Needs Help" state, so it must be handled manually.

---
**Question 30: Alien Dictionary (Advanced Graph / Topological Sort)**

**Observations:**

* You are given a list of strings representing a dictionary of a new alien language. The words are sorted lexicographically by the rules of this new language.
* You need to derive the alphabet order (character precedence).
* When comparing two sorted words (like "apple" and "apply"), the first character that differs ('e' vs 'y') dictates the alphabetical order. Therefore, 'e' must come before 'y'.
* This is a direct mapping to a **Directed Graph**. Every character is a node, and a precedence rule ('e' comes before 'y') is a directed edge from 'e' to 'y'.
* To find a linear ordering of vertices in a directed acyclic graph, we use **Topological Sort** (Kahn's Algorithm with in-degrees is perfect here).

**Approach:**

1. **Initialize Data Structures:** Create an adjacency list (`adj`) mapping characters to a list of characters, and an `inDegree` map tracking incoming edges for every unique character.
2. **Build the Graph:** Iterate through the `words` list, comparing adjacent pairs of words (`w1` and `w2`).
* **Prefix Edge Case:** If `w1` is longer than `w2` and `w2` is a prefix of `w1` (e.g., `w1 = "abcd"`, `w2 = "abc"`), this is invalid sorting in any dictionary. Return an empty string.
* Compare characters of `w1` and `w2` one by one. The first mismatch (`w1[j] != w2[j]`) means `w1[j]` comes before `w2[j]`.
* Add a directed edge from `w1[j]` to `w2[j]`, increment the in-degree of `w2[j]`, and immediately `break` (subsequent characters give no ordering information).


3. **Topological Sort:** Push all characters with an in-degree of 0 into a queue.
4. Process the queue: pop a character, append it to your `result` string, and decrement the in-degree of all its neighbors. If a neighbor's in-degree hits 0, push it to the queue.
5. **Cycle Check:** If the final `result` string length does not match the total number of unique characters, a cycle exists (e.g., 'a' comes before 'b', but 'b' comes before 'a'). Return `""`. Otherwise, return `result`.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <unordered_map>
#include <queue>
#include <algorithm>

using namespace std;

string alienOrder(vector<string>& words) {
    unordered_map<char, vector<char>> adj;
    unordered_map<char, int> inDegree;

    // 1. Initialize in-degree for all unique characters
    for (const string& w : words) {
        for (char c : w) {
            inDegree[c] = 0; 
        }
    }

    // 2. Build the graph by comparing adjacent words
    for (int i = 0; i < words.size() - 1; i++) {
        string w1 = words[i];
        string w2 = words[i + 1];
        
        // Edge case: "abc" before "ab" is an invalid dictionary
        if (w1.length() > w2.length() && 
            w1.substr(0, w2.length()) == w2) {
            return ""; 
        }

        // Find the first differing character
        int minLen = min(w1.length(), w2.length());
        for (int j = 0; j < minLen; j++) {
            if (w1[j] != w2[j]) {
                adj[w1[j]].push_back(w2[j]);
                inDegree[w2[j]]++;
                break; // Only the first difference matters
            }
        }
    }

    // 3. Kahn's Algorithm for Topological Sort
    queue<char> q;
    for (auto& pair : inDegree) {
        if (pair.second == 0) q.push(pair.first);
    }

    string result = "";
    while (!q.empty()) {
        char curr = q.front();
        q.pop();
        result += curr;

        for (char neighbor : adj[curr]) {
            inDegree[neighbor]--;
            if (inDegree[neighbor] == 0) {
                q.push(neighbor);
            }
        }
    }

    // 4. If result doesn't contain all characters, there's a cycle
    if (result.length() != inDegree.size()) {
        return "";
    }
    
    return result;
}

int main() {
    // Example: alien dictionary sorted words
    vector<string> words = {"wrt", "wrf", "er", "ett", "rftt"};
    
    // Expected output: "wertf"
    cout << "Alien Alphabet Order: " << alienOrder(words) << endl;
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **String Comparisons as Edges:** Whenever a problem implies "A comes before B" based on a set of rules or strings, immediately translate it into a directed edge `A -> B`.
* **Prefix Invalidity:** In dictionary problems, a longer word can never come before a shorter word if the shorter word is a strict prefix (like "apple" coming before "app"). This is a common trap that fails 10% of test cases if missed.
* **First Mismatch Only:** You only extract a dependency from the *first* character that differs between two words. The rest of the characters are completely irrelevant to the sorting order.

---
**Question 31: Median of Two Sorted Arrays (Advanced Binary Search)**

**Observations:**

* You are given two sorted arrays of size `m` and `n`. You need to return the median of the two sorted arrays. The overall run time complexity should be `O(log(min(m, n)))`.
* The `O(log)` constraint means we cannot merge the arrays (which would take `O(m + n)`). We must use Binary Search.
* The median's job is to divide an array into two equal halves. For two arrays, we can imagine a "cut" that splits both arrays such that the total number of elements on the left side of the cuts equals the total number on the right side.
* If we know the cut position in the first array, the cut position in the second array is automatically determined by the formula: `(m + n + 1) / 2 - cut1`.
* Therefore, we only need to perform a Binary Search to find the correct cut position in the *smaller* of the two arrays.

**Approach:**

1. Ensure `nums1` is always the smaller array to minimize the binary search space. If it is not, recursively call the function swapping the arguments.
2. Initialize `low = 0` and `high = nums1.size()`. The cut can happen before the first element or after the last element.
3. Perform Binary Search while `low <= high`:
* Calculate `cut1 = low + (high - low) / 2`.
* Calculate `cut2 = (n1 + n2 + 1) / 2 - cut1`.


4. Extract the four elements bordering the cuts. Let's call them `left1`, `left2`, `right1`, and `right2`.
* If a cut is at the very beginning (0), there is no left element, so use `INT_MIN` as a safe dummy value.
* If a cut is at the very end (size), there is no right element, so use `INT_MAX`.


5. **Check Partition Validity:** A partition is valid if all elements on the left are smaller than or equal to all elements on the right. Since the arrays are already internally sorted, we only need to cross-check: `left1 <= right2` AND `left2 <= right1`.
* **If valid:** We found the correct partition!
* If the total number of elements is odd, the median is `max(left1, left2)`.
* If even, the median is `(max(left1, left2) + min(right1, right2)) / 2.0`.


* **If `left1 > right2`:** Our cut in `nums1` is too far to the right. We need to move it left: `high = cut1 - 1`.
* **If `left2 > right1`:** Our cut in `nums1` is too far to the left. We need to move it right: `low = cut1 + 1`.



**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <climits>

using namespace std;

double findMedianSortedArrays(vector<int>& nums1, 
                              vector<int>& nums2) {
    // Always perform binary search on the smaller array
    if (nums1.size() > nums2.size()) {
        return findMedianSortedArrays(nums2, nums1);
    }
    
    int n1 = nums1.size();
    int n2 = nums2.size();
    int low = 0, high = n1;
    
    while (low <= high) {
        int cut1 = low + (high - low) / 2;
        int cut2 = (n1 + n2 + 1) / 2 - cut1;
        
        // Handle edge cases with INT_MIN and INT_MAX
        int left1 = (cut1 == 0) ? INT_MIN : nums1[cut1 - 1];
        int left2 = (cut2 == 0) ? INT_MIN : nums2[cut2 - 1];
        
        int right1 = (cut1 == n1) ? INT_MAX : nums1[cut1];
        int right2 = (cut2 == n2) ? INT_MAX : nums2[cut2];
        
        // Valid partition found
        if (left1 <= right2 && left2 <= right1) {
            if ((n1 + n2) % 2 == 0) {
                // Line broken to respect 70 char limit
                return (max(left1, left2) + 
                        min(right1, right2)) / 2.0;
            } else {
                return max(left1, left2);
            }
        } 
        else if (left1 > right2) {
            high = cut1 - 1; // Move cut1 to the left
        } 
        else {
            low = cut1 + 1;  // Move cut1 to the right
        }
    }
    return 0.0;
}

int main() {
    // Array 1: 1, 3, 8, 9, 15
    // Array 2: 7, 11, 18, 19, 21, 25
    // Combined: 1, 3, 7, 8, 9, 11, 15, 18, 19, 21, 25 (Median: 11)
    vector<int> nums1 = {1, 3, 8, 9, 15};
    vector<int> nums2 = {7, 11, 18, 19, 21, 25};
    
    cout << "Median is: " 
         << findMedianSortedArrays(nums1, nums2) << endl;
         
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **Binary Search on Partitions:** When asked to find the exact midpoint or exact `k`-th element between multiple sorted arrays, you don't binary search the *values*, you binary search the *indices* of the partition line.
* **Infinity Bounds Trick:** When partitioning arrays, you will inevitably hit the extreme ends (0 or `size`). Using `INT_MIN` for an empty left side and `INT_MAX` for an empty right side safely neutralizes edge case errors without requiring extra `if` blocks.

---
**Question 32: Largest Rectangle in Histogram (Monotonic Stack)**

**Observations:**

* You are given an array of integers representing the heights of bars in a histogram (where width of each bar is 1). You need to find the area of the largest rectangle that can be formed.
* The maximum rectangle must be constrained by the *shortest* bar within its width.
* Therefore, for every bar `i`, if we treat it as the shortest bar, we need to know how far it can extend to the left and to the right before hitting a bar that is strictly shorter than it.
* A naive O(N^2) approach checks left and right bounds for every bar. We can optimize this to O(N) using a **Monotonic Increasing Stack**.
* A monotonic increasing stack stores indices such that the heights of the bars at those indices are strictly increasing. Whenever we encounter a shorter bar, it acts as the "right boundary" for the taller bars currently sitting in the stack, allowing us to calculate their areas immediately.

**Approach:**

1. Initialize an empty stack `s` to store indices, and a `max_area` variable to 0.
2. Iterate through the array using an index `i` from `0` to `N`.
3. **The Dummy Bar Trick:** We iterate exactly `N + 1` times. On the very last iteration (`i == N`), we pretend there is a bar of height `0`. This forces the stack to completely empty out and calculate the areas for all remaining bars.
4. While the stack is not empty AND the current bar's height is strictly less than the height of the bar at the top of the stack:
* The current `i` is the right boundary!
* Pop the top index from the stack. The height of the rectangle will be `heights[popped_index]`.
* Now, look at the *new* top of the stack. This is the left boundary!
* If the stack is empty, it means this popped bar was the shortest one seen so far, so its width extends all the way from index 0 to `i`. Width = `i`.
* If not empty, the width is strictly between the new stack top and `i`. Width = `i - s.top() - 1`.
* Calculate `area = height * width` and update `max_area`.


5. Push the current index `i` onto the stack and continue.
6. Return `max_area`.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <stack>
#include <algorithm>

using namespace std;

int largestRectangleArea(vector<int>& heights) {
    stack<int> s;
    int max_area = 0;
    int n = heights.size();
    
    // Iterate to n (inclusive) to handle the dummy zero-height bar
    for (int i = 0; i <= n; i++) {
        // Treat the out-of-bounds index 'n' as height 0
        int curr_h = (i == n) ? 0 : heights[i];
        
        // While stack is not empty and current bar is shorter
        while (!s.empty() && curr_h < heights[s.top()]) {
            int h = heights[s.top()];
            s.pop();
            
            int w;
            if (s.empty()) {
                w = i; // Extends all the way to the beginning
            } else {
                w = i - s.top() - 1; // Bounded by previous smaller
            }
            
            max_area = max(max_area, h * w);
        }
        
        // Push current index to stack
        s.push(i);
    }
    
    return max_area;
}

int main() {
    // Example: Histogram [2, 1, 5, 6, 2, 3]
    // Largest rectangle is formed by 5 and 6 (height 5, width 2) = 10
    vector<int> heights = {2, 1, 5, 6, 2, 3};
    
    cout << "Largest Rectangle Area: " 
         << largestRectangleArea(heights) << endl;
         
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **The Monotonic Stack Template:** Whenever a problem asks for the "Next Greater Element", "Next Smaller Element", or boundaries based on height/values, a monotonic stack is the O(N) golden key.
* **The "Zero-Height Padding" Trick:** Appending a dummy element (like `0` or `-1`) at the end of the array inside the loop eliminates the need to write a second `while(!s.empty())` loop outside the main iteration to clean up the leftovers.
* **Index vs Value:** In stack problems dealing with widths or distances, always store the *indices* in the stack, not the actual values. You can easily retrieve the value using `array[index]`, but you cannot retrieve the index if you only stored the value!

---
**Question 33: Sliding Window Maximum (Monotonic Deque)**

**Observations:**

* You are given an array of integers and a sliding window of size `k` that moves from the extreme left to the extreme right. You need to record the maximum value in each window.
* A naive approach would scan all `k` elements for every single window, resulting in a time complexity of `O(N * K)`.
* To optimize this to `O(N)`, we need a way to instantly know the maximum element, while also having the ability to remove elements that slide out of our window.
* Furthermore, if we see a massive number, any smaller numbers before it in the same window become completely useless—they can never be the maximum.
* A Double-Ended Queue (Deque) allows us to push and pop from both ends efficiently, making it the perfect data structure to maintain a "Monotonically Decreasing" list of useful candidates.

**Approach:**

1. Create a `deque` to store the **indices** of the elements, and a `result` vector to store the maximums.
2. Iterate through the array using an index `i` from 0 to `N-1`.
3. **Clean the Front (Out of Bounds):** The window size is `k`. The valid index range for the current window ends at `i` and starts at `i - k + 1`. If the index at the front of the deque is strictly less than `i - k + 1` (or simply `front == i - k`), it has slid out of the window and must be popped.
4. **Clean the Back (Maintain Monotonicity):** Before adding the current element `nums[i]`, look at the back of the deque. If `nums[i]` is greater than or equal to the element represented by the back index, pop the back index. Repeat this until the back is larger than `nums[i]` or the deque is empty.
5. Push the current index `i` to the back of the deque.
6. **Record Answer:** Once our loop has processed at least `k` elements (i.e., when `i >= k - 1`), the maximum element for the current window is guaranteed to be sitting right at the front of the deque. Push `nums[deque.front()]` to the `result` vector.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>
#include <deque>

using namespace std;

vector<int> maxSlidingWindow(vector<int>& nums, int k) {
    deque<int> dq; // Stores indices, not values
    vector<int> result;
    
    for (int i = 0; i < nums.size(); i++) {
        // 1. Remove indices that are out of the current window
        if (!dq.empty() && dq.front() == i - k) {
            dq.pop_front();
        }
        
        // 2. Remove smaller elements from the back
        // They are useless because the current larger element 
        // is entering the window and will survive longer
        while (!dq.empty() && nums[dq.back()] <= nums[i]) {
            dq.pop_back();
        }
        
        // 3. Add the current element's index
        dq.push_back(i);
        
        // 4. Add the maximum to the result once the window 
        // is fully formed (i >= k - 1)
        if (i >= k - 1) {
            result.push_back(nums[dq.front()]);
        }
    }
    
    return result;
}

int main() {
    // Example: window size 3
    // Windows: 
    // [1  3 -1] -3  5  3  6  7  -> Max: 3
    //  1 [3 -1 -3]  5  3  6  7  -> Max: 3
    //  1  3 [-1 -3  5] 3  6  7  -> Max: 5
    // Result: [3, 3, 5, 5, 6, 7]
    vector<int> nums = {1, 3, -1, -3, 5, 3, 6, 7};
    int k = 3;
    
    vector<int> ans = maxSlidingWindow(nums, k);
    
    cout << "Sliding window maximums: ";
    for (int num : ans) {
        cout << num << " ";
    }
    cout << endl;
    
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **The Monotonic Deque:** Just like the monotonic stack is used for finding the "Next Greater Element", the monotonic deque is the universal tool for finding maximums or minimums within moving dynamic bounds.
* **Indices rule:** As with stacks, always store indices in the deque. You need the index to determine if an element has fallen out of the sliding window (`i - k`). You cannot calculate bounds if you only store the raw values.

---
**Question 34: Department Top Three Salaries (Advanced SQL Window Functions)**

**Observations:**

* You are given two tables: `Employee` (Id, Name, Salary, DepartmentId) and `Department` (Id, Name). You need to find employees who earn the top three salaries in each of the company's departments.
* A department might have multiple employees earning the exact same salary. If two employees tie for the highest salary, they both share rank 1, and the next highest salary should be rank 2.
* Traditional `GROUP BY` and `LIMIT` clauses fall short here because we need to limit results *per group* (per department) rather than limiting the entire result set.
* This is the perfect scenario for SQL Window Functions, which allow you to perform calculations across a set of table rows that are somehow related to the current row.

**Approach:**

1. **Choose the Right Function:** We need to rank salaries. `ROW_NUMBER()` assigns strictly sequential numbers (1, 2, 3), breaking ties arbitrarily. `RANK()` leaves gaps after ties (1, 1, 3). `DENSE_RANK()` assigns the same rank to ties without skipping numbers (1, 1, 2). `DENSE_RANK()` is exactly what we need for "top three unique salaries".
2. **Define the Window:** Use the `OVER()` clause to define how the ranking is applied. We want to restart the ranking for each department, so we use `PARTITION BY d.Id`. We want the highest salary first, so we use `ORDER BY e.Salary DESC`.
3. **Join Tables:** Join the `Employee` and `Department` tables to get the readable department names alongside the employee data.
4. **Use a CTE / Subquery:** Window functions are evaluated *after* the `WHERE` clause in the SQL order of execution. Therefore, you cannot filter by a window function directly in the same `SELECT` statement. You must wrap the query in a Common Table Expression (CTE) or a subquery.
5. **Filter:** Select from the CTE where the calculated rank is `<= 3`.

**Code(in simple SQL):**

```sql
-- Step 1: Create a CTE to hold the ranked data
WITH RankedSalaries AS (
    SELECT 
        d.Name AS Department,
        e.Name AS Employee,
        e.Salary AS Salary,
        -- Apply DENSE_RANK partitioned by department
        DENSE_RANK() OVER (
            PARTITION BY d.Id 
            ORDER BY e.Salary DESC
        ) AS rank_val
    FROM Employee e
    JOIN Department d ON e.DepartmentId = d.Id
)

-- Step 2: Filter the results using the calculated rank
SELECT 
    Department, 
    Employee, 
    Salary
FROM RankedSalaries
WHERE rank_val <= 3;

```

**Things to remember for solving same kind of problems(in short):**

* **The Ranking Trio:** Always know the difference between `ROW_NUMBER()`, `RANK()`, and `DENSE_RANK()`. This is one of the most frequently tested concepts in advanced SQL interviews.
* **Execution Order Trap:** You can never write `WHERE DENSE_RANK() OVER (...) <= 3`. The database engine will throw an error. Always calculate the window function in a subquery or CTE first, then filter on the resulting column in the outer query.
* **Partitioning vs Grouping:** `PARTITION BY` divides the result set into partitions to apply the window function, but it keeps all the original rows. `GROUP BY` aggregates rows into a single summary row.

---
**Question 35: Count Binary Strings without "000" or "111" (Matrix Exponentiation)**

**Observations:**

* You are asked to find the number of valid binary strings of length $n$ that do not contain "000" or "111".
* **The Constraint is the Biggest Clue:** The problem states $n \le 10^{18}$. A standard Dynamic Programming approach takes $O(N)$ time, which will cause a Time Limit Exceeded (TLE) error for $10^{18}$. Whenever you see $10^{18}$ combined with a counting/sequence problem, the required time complexity is $O(\log n)$. This is the universal signature of **Matrix Exponentiation**.
* **Finding the Pattern:** Let's define valid strings by their suffix. A valid string can end in either one identical character (like `...01` or `...10`) or two identical characters (like `...100` or `...011`).
* Let $A_n$ be the number of strings ending in 1 identical character. To form this, you must append the *opposite* character to any valid string of length $n-1$. So, $A_n = A_{n-1} + B_{n-1}$.
* Let $B_n$ be the number of strings ending in 2 identical characters. You can only form this by appending the *same* character to a string that previously ended in 1 identical character. So, $B_n = A_{n-1}$.


* Substituting $B_{n-1}$ into the first equation gives:

$$A_n = A_{n-1} + A_{n-2}$$



This is the exact formula for the **Fibonacci sequence**!
* By tracing the first few values ($n=1, 2, 3, 4$), the total number of valid strings $T_n$ strictly follows $T_n = 2 \times F_{n+1}$ (where $F_1=1, F_2=1, F_3=2, F_4=3, F_5=5$).
* Because the result grows exponentially, such problems universally expect you to return the answer modulo $10^9 + 7$ to prevent integer overflow.

**Approach:**

1. We need to compute the $(n+1)$-th Fibonacci number in $O(\log n)$ time.
2. We can represent the Fibonacci recurrence relation as a matrix multiplication:

$$\begin{pmatrix} F_{n+1} \\ F_n \end{pmatrix} = \begin{pmatrix} 1 & 1 \\ 1 & 0 \end{pmatrix}^n \begin{pmatrix} F_1 \\ F_0 \end{pmatrix}$$


3. Let the transformation matrix be $M = \begin{pmatrix} 1 & 1 \\ 1 & 0 \end{pmatrix}$.
4. If we calculate $M^n$, the top-left element of the resulting matrix will be exactly $F_{n+1}$.
5. We can calculate $M^n$ using **Binary Exponentiation** (similar to calculating $x^y$ by squaring the base and halving the exponent), which takes $O(\log n)$ time.
6. Multiply the top-left element by 2, apply modulo $10^9 + 7$, and return the result.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>

using namespace std;

const int MOD = 1e9 + 7;
typedef vector<vector<long long>> Matrix;

// Helper function to multiply two 2x2 matrices
Matrix multiply(Matrix A, Matrix B) {
    Matrix C(2, vector<long long>(2, 0));
    for (int i = 0; i < 2; i++) {
        for (int j = 0; j < 2; j++) {
            for (int k = 0; k < 2; k++) {
                C[i][j] = (C[i][j] + A[i][k] * B[k][j]) % MOD;
            }
        }
    }
    return C;
}

// Helper for binary matrix exponentiation in O(log N)
Matrix matrixPower(Matrix base, long long exp) {
    // Identity matrix
    Matrix res = {{1, 0}, {0, 1}}; 
    
    while (exp > 0) {
        // If exponent is odd, multiply result by base
        if (exp % 2 == 1) {
            res = multiply(res, base);
        }
        // Square the base and halve the exponent
        base = multiply(base, base);
        exp /= 2;
    }
    return res;
}

int countValidStrings(long long n) {
    if (n == 0) return 0;
    
    // Transformation matrix M
    Matrix T = {{1, 1}, {1, 0}};
    
    // Calculate T^n
    Matrix Tn = matrixPower(T, n);
    
    // Total strings = 2 * F_{n+1}
    // F_{n+1} is stored at the top-left (Tn[0][0])
    long long ans = (2 * Tn[0][0]) % MOD;
    
    return ans;
}

int main() {
    long long n = 4;
    // For n=4, answer should be 10 (as proven in your example)
    cout << "Valid strings for n=" << n << ": " 
         << countValidStrings(n) << endl;
         
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **Read the Constraints First:** The single biggest takeaway here is $N \le 10^{18}$. If you write a standard $O(N)$ dynamic programming loop for this, you will fail the hidden test cases. Big constraints demand Binary Search or Matrix Exponentiation.
* **Combinatorics hide Fibonacci:** A massive number of "count the binary strings without consecutive X" problems secretly boil down to the Fibonacci sequence or the Tribonacci sequence. Try solving for $n=1, 2, 3, 4$ on paper first to spot the sequence!
* **The Identity Matrix:** When doing binary exponentiation on matrices, always initialize your result matrix as the Identity Matrix `{{1, 0}, {0, 1}}` just like you would initialize a regular integer to `1` when calculating standard powers.

---
**Question 36: Total Count of Zeros or Ones in Valid Binary Strings**

**Observations:**

* You are given the same conditions as the previous problem: find the total number of `0`s (or `1`s) present across all valid binary strings of length `n` that do not contain "000" or "111". The constraint is massive: $n \le 10^{18}$.
* **Symmetry is the key:** The constraints "no 000" and "no 111" are perfectly symmetric. This means for every valid string that exists (e.g., `0100`), its exact bitwise complement (`1011`) is also a valid string.
* Because every valid string has a paired opposite, the total number of `0`s across all valid strings combined must be exactly equal to the total number of `1`s.
* In the previous question, we established that the total number of valid strings is $2 \times F_{n+1}$ (where $F$ is the Fibonacci sequence).
* The total number of characters across *all* valid strings is simply `n` multiplied by the total number of strings: $n \times (2 \times F_{n+1})$.
* Since exactly half of these characters are `1`s (and the other half are `0`s), the total number of `1`s is $(n \times 2 \times F_{n+1}) / 2$, which perfectly simplifies to $n \times F_{n+1}$.

**Approach:**

1. Just like the previous problem, use Matrix Exponentiation to compute the $(n+1)$-th Fibonacci number in $O(\log n)$ time to handle the $10^{18}$ constraint without a Time Limit Exceeded (TLE) error.
2. The transformation matrix $M$ is `{{1, 1}, {1, 0}}`.
3. Calculate $M^n$ using binary exponentiation. The value at the top-left of the resulting matrix (`res[0][0]`) will be $F_{n+1}$.
4. Multiply this $F_{n+1}$ value by $n$.
5. Because the numbers get astronomically large, apply the modulo $10^9 + 7$ operation at every single multiplication step, including the final multiplication by $n$.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>

using namespace std;

const int MOD = 1e9 + 7;
typedef vector<vector<long long>> Matrix;

// Helper function to multiply two 2x2 matrices
Matrix multiply(Matrix A, Matrix B) {
    Matrix C(2, vector<long long>(2, 0));
    for (int i = 0; i < 2; i++) {
        for (int j = 0; j < 2; j++) {
            for (int k = 0; k < 2; k++) {
                C[i][j] = (C[i][j] + A[i][k] * B[k][j]) % MOD;
            }
        }
    }
    return C;
}

// Helper for binary matrix exponentiation in O(log N)
Matrix matrixPower(Matrix base, long long exp) {
    Matrix res = {{1, 0}, {0, 1}}; // Identity matrix
    
    while (exp > 0) {
        if (exp % 2 == 1) {
            res = multiply(res, base);
        }
        base = multiply(base, base);
        exp /= 2;
    }
    return res;
}

int countTotalOnesOrZeros(long long n) {
    if (n == 0) return 0;
    
    Matrix T = {{1, 1}, {1, 0}};
    Matrix Tn = matrixPower(T, n);
    
    // F_{n+1} is stored at the top-left (Tn[0][0])
    long long Fn_plus_1 = Tn[0][0];
    
    // Formula: Total 1s = n * F_{n+1}
    // Apply modulo to n first to prevent overflow during multiply
    long long safe_n = n % MOD;
    long long ans = (safe_n * Fn_plus_1) % MOD;
    
    return ans;
}

int main() {
    long long n = 4;
    // Valid strings for n=4 is 10. 
    // Total 1s (or 0s) across them is 4 * 5 = 20.
    cout << "Total 0s (or 1s) for n=" << n << ": " 
         << countTotalOnesOrZeros(n) << endl;
         
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **The Symmetry Shortcut:** Before writing complex code to count specific characters or properties, always check if the problem's rules are symmetric. If you can swap '0' and '1' without breaking the rules, you can usually divide the total complexity by half.
* **Modulo Arithmetic Safety:** When multiplying two large values (like `n` and `Fn_plus_1`), always take the modulo of `n` *before* the multiplication (`(n % MOD) * Fn_plus_1`). If `n` is $10^{18}$, multiplying it before modulo will instantly overflow a 64-bit integer.

---

**Question 37: M-Coloring Problem (Graph / Backtracking)**

**Observations:**

* You are given an undirected graph with `N` nodes and an integer `M`. You need to determine if it is possible to color the graph using at most `M` different colors such that no two adjacent nodes share the same color.
* This is a constraint satisfaction problem. You must assign a color to one node, check if it violates the rules, and if it is safe, move to the next node.
* Because a color choice made early on might lead to a dead-end later, you need the ability to "undo" your choice and try a different color. This makes it a classic **Backtracking** problem.
* The graph might be disconnected, but for this basic variation, we usually just iterate from node 0 to N-1 linearly, relying on the adjacency list to check for conflicts.

**Approach:**

1. Create a `color` array of size `N`, initialized to 0 (meaning all nodes are initially uncolored).
2. Create a recursive function `solve(node, ...)` that tries to color the graph starting from the current `node`.
3. **Base Case:** If `node == N`, it means we have successfully assigned a valid color to all `N` nodes. Return `true`.
4. **Recursive Step:** For the current `node`, try all possible colors from `1` to `M`.
5. **Safety Check:** For a chosen color `i`, iterate through all adjacent neighbors of the current `node`. If any neighbor already has color `i`, this color is invalid.
6. If the color `i` is safe:
* **Choose:** Assign `color[node] = i`.
* **Explore:** Recursively call `solve(node + 1, ...)`. If this call returns `true`, it means this path leads to a full solution, so immediately return `true`.
* **Un-choose (Backtrack):** If the recursive call returns `false`, it means color `i` was a bad choice. Reset `color[node] = 0` and let the loop try the next color.


7. If the loop finishes and no color from 1 to `M` worked, return `false`.

**Code(in simple C++):**

```cpp
#include <iostream>
#include <vector>

using namespace std;

// Helper to check if it's safe to color 'node' with 'col'
bool isSafe(int node, const vector<vector<int>>& adj, 
            const vector<int>& color, int col) {
    for (int neighbor : adj[node]) {
        // If an adjacent node has the same color, it's unsafe
        if (color[neighbor] == col) {
            return false;
        }
    }
    return true;
}

// Recursive backtracking function
bool solve(int node, int n, int m, 
           const vector<vector<int>>& adj, vector<int>& color) {
    // Base case: All nodes have been colored
    if (node == n) {
        return true; 
    }

    // Try assigning every color from 1 to m
    for (int i = 1; i <= m; i++) {
        if (isSafe(node, adj, color, i)) {
            color[node] = i; // Assign color
            
            // Recursively try to color the next node
            if (solve(node + 1, n, m, adj, color)) {
                return true;
            }
            
            // Backtrack: remove color if it leads to a dead end
            color[node] = 0; 
        }
    }
    
    // No color could be assigned to this node
    return false; 
}

bool graphColoring(int n, int m, const vector<vector<int>>& adj) {
    vector<int> color(n, 0); // 0 means uncolored
    return solve(0, n, m, adj, color);
}

int main() {
    int n = 4; // Number of nodes
    int m = 3; // Number of colors available
    
    // Graph representation (Adjacency List)
    // 0-1, 1-2, 2-3, 3-0, 0-2 (A square with one diagonal)
    vector<vector<int>> adj(n);
    adj[0] = {1, 2, 3};
    adj[1] = {0, 2};
    adj[2] = {0, 1, 3};
    adj[3] = {0, 2};
    
    if (graphColoring(n, m, adj)) {
        cout << "Graph can be colored with " << m << " colors." 
             << endl;
    } else {
        cout << "Graph CANNOT be colored with " << m << " colors." 
             << endl;
    }
    
    return 0;
}

```

**Things to remember for solving same kind of problems(in short):**

* **Bipartite Check Connection:** If `M = 2`, this problem becomes exactly the "Check if Graph is Bipartite" problem. A bipartite graph is simply a graph that can be colored using exactly 2 colors.
* **Exponential Time Complexity:** Because you branch `M` ways for `N` nodes, the worst-case time complexity is $O(M^N)$. This is why graph coloring problems usually have very small constraints on `N` (e.g., $N \le 20$).
* **Linear Node Progression:** Unlike DFS where you jump to neighbors, notice that the recursive call here simply goes to `node + 1`. We don't trace the graph's edges to determine our next step; we just move down the list of nodes sequentially while using the edges only for the `isSafe` check.

**Congratulations!** We have successfully made it through all 34 custom-tailored questions. We've covered everything from arrays, strings, dynamic programming, and binary search to advanced graph theory, trees, monotonic stacks, and advanced SQL window functions.
