# Complexity

Reach for this to state the Big-O of common operations without second-guessing.

The tables below are usable now; expand the notes over time.

## Big-O growth (fastest → slowest)

`O(1)` < `O(log n)` < `O(n)` < `O(n log n)` < `O(n²)` < `O(2ⁿ)` < `O(n!)`

## Java collections

| Operation | ArrayList | LinkedList | HashMap | TreeMap | HashSet | PriorityQueue |
|---|---|---|---|---|---|---|
| add / put | O(1)* | O(1) | O(1)* | O(log n) | O(1)* | O(log n) |
| get / contains | O(1) | O(n) | O(1)* | O(log n) | O(1)* | - |
| remove | O(n) | O(1)† | O(1)* | O(log n) | O(1)* | O(log n) |
| peek min/max | - | - | - | O(log n)‡ | - | O(1) |

\* amortized / average · † at a known node · ‡ `firstKey`/`lastKey`

## Sorting

| Algorithm | Best | Avg | Worst | Space | Stable |
|---|---|---|---|---|---|
| Quicksort | n log n | n log n | n² | log n | no |
| Mergesort | n log n | n log n | n log n | n | yes |
| Heapsort | n log n | n log n | n log n | 1 | no |
| Insertion | n | n² | n² | 1 | yes |
| `Arrays.sort` (primitives) | dual-pivot quicksort | | | | |
| `Arrays.sort` (objects) / `Collections.sort` | Timsort (n log n) | | | | yes |

## Common patterns → complexity

| Pattern | Time |
|---|---|
| Two pointers / sliding window | O(n) |
| Binary search | O(log n) |
| BFS/DFS on graph | O(V + E) |
| Backtracking (subsets) | O(2ⁿ) |
| Backtracking (permutations) | O(n!) |
| DP (2D grid) | O(rows × cols) |
| Heap of K over n items | O(n log k) |

## Amortized vs worst-case (stub - expand)

## Space complexity - recursion stack counts (stub)

## Gotchas / things I always forget

- `ArrayList.add` is amortized O(1) but O(n) on the resize - matters in tight analysis.
- `HashMap` is O(1) *average*; worst case O(log n) since Java 8 (treeified buckets).
- Recursion adds O(depth) space via the call stack - count it.
- `contains` on a `List` is O(n); if you're checking membership, use a `Set`.

## Quick reference

| Need | Structure |
|---|---|
| Fast membership | `HashSet` O(1) |
| Sorted + range queries | `TreeMap` O(log n) |
| Min/max on the fly | `PriorityQueue` |
| Index access | `ArrayList` O(1) |
