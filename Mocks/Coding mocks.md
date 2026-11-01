# Coding Mocks

## Mock - 1

SESSION STATE
- Mock type: coding + behavioral (Amazon SDE style, projects-based LPs)
- Rounds done: 1 complete (Dive Deep LP + coding). Ownership LP question asked, awaiting answer.
- Questions already asked:
  - LP: Dive Deep (floating-point amount mismatch bug)
  - LP: Ownership (gap nobody asked you to cover), unanswered
  - Coding: users with at least 3 logs in any closed k-second window, sorted output (group by user, sort, sliding window)
  - Follow-ups: streaming version at scale; testing the streaming version
- Score trend: Communication 2.5, Problem solving 4, Code quality 3, Edge cases 3; LP Dive Deep 2.5
- Top 3 recurring weaknesses:
  1. Rambling, unstructured answers; no plan stated upfront
  2. Competitive-programming habits (bits/stdc++.h, weak names like s/ans, unintended copies, non-const refs)
  3. Edge-case lists incomplete (missed duplicates, exact boundary, multiple-user ordering); loose narration in dry-runs
- Handled well: clarifying questions (including stream order), sliding-window approach and complexity, streaming eviction instinct, single-query optimization story
- Facts to fix:
  - `vector<int> v = user.second;` copies; use `auto&` or a reference
  - `std::to_string(double)` gives 6 decimals ("13644.960000"), mismatching numeric(15,2) strings; fix is fixed + setprecision(2)
  - Closed window [t, t+k]: stale when timestamp < T - k
  - Don't ask "is it okay"; state the plan and proceed
- Next focus: structured STAR delivery (Situation, Task, Action, Result with a quantified result); a harder coding problem (medium-hard, new pattern, such as graph, heap, or intervals) with a fuller edge-case list stated upfront

---

