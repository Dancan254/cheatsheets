# LeetCode Patterns

Reach for this to recognize *which* pattern a problem wants. ~15 patterns cover the
large majority of interview questions. Code is Java (your contest language).

## How to use this

Most problems are a known pattern in disguise. Read the problem → match the **signal** →
apply the template. Recognition beats memorizing solutions.

## 1. Two Pointers

**Signal**: sorted array, pair/triplet summing to a target, reversing, dedup in place.

```java
int l = 0, r = n - 1;
while (l < r) {
    int sum = a[l] + a[r];
    if (sum == target) return new int[]{l, r};
    if (sum < target) l++; else r--;
}
```
Problems: Two Sum II, 3Sum, Container With Most Water, valid palindrome.

## 2. Sliding Window

**Signal**: contiguous subarray/substring, "longest/shortest/max sum window", "at most K".

```java
int left = 0, best = 0;
Map<Character,Integer> count = new HashMap<>();
for (int right = 0; right < s.length(); right++) {
    count.merge(s.charAt(right), 1, Integer::sum);
    while (/* window invalid */) {                 // shrink from left
        char c = s.charAt(left++);
        count.merge(c, -1, Integer::sum);
    }
    best = Math.max(best, right - left + 1);
}
```
Problems: Longest Substring Without Repeating, Min Window Substring, Max Sum Subarray of size K.

## 3. Fast & Slow Pointers (Floyd)

**Signal**: linked list cycle, find middle, "happy number", nth from end.

```java
ListNode slow = head, fast = head;
while (fast != null && fast.next != null) {
    slow = slow.next;
    fast = fast.next.next;
    if (slow == fast) return true;   // cycle
}
```

## 4. Prefix Sum

**Signal**: many range-sum queries, subarray summing to K, count subarrays.

```java
Map<Integer,Integer> seen = new HashMap<>();
seen.put(0, 1);
int sum = 0, count = 0;
for (int x : nums) {
    sum += x;
    count += seen.getOrDefault(sum - k, 0);   // subarrays ending here summing to k
    seen.merge(sum, 1, Integer::sum);
}
```
Problems: Subarray Sum Equals K, Range Sum Query.

## 5. Binary Search

**Signal**: sorted input, OR "minimize/maximize a value that's monotonic" (search on the answer).

```java
int lo = 0, hi = n - 1;
while (lo <= hi) {
    int mid = lo + (hi - lo) / 2;             // avoid overflow
    if (a[mid] == target) return mid;
    if (a[mid] < target) lo = mid + 1; else hi = mid - 1;
}
return -1;
```
Search-on-answer: Koko Eating Bananas, Ship Within Days, Split Array Largest Sum.

## 6. BFS (graphs / trees / grids)

**Signal**: shortest path in unweighted graph, level-order, "min steps".

```java
Queue<Node> q = new ArrayDeque<>();
Set<Node> visited = new HashSet<>();
q.add(start); visited.add(start);
int steps = 0;
while (!q.isEmpty()) {
    for (int sz = q.size(); sz > 0; sz--) {    // process one level
        Node cur = q.poll();
        if (cur == goal) return steps;
        for (Node nb : neighbors(cur))
            if (visited.add(nb)) q.add(nb);
    }
    steps++;
}
```
Grid dirs: `int[][] d = {{1,0},{-1,0},{0,1},{0,-1}};`

## 7. DFS / Backtracking

**Signal**: all permutations/combinations/subsets, "generate every", path in maze, N-Queens.

```java
void backtrack(List<Integer> path, boolean[] used) {
    if (path.size() == n) { result.add(new ArrayList<>(path)); return; }
    for (int i = 0; i < n; i++) {
        if (used[i]) continue;
        used[i] = true; path.add(nums[i]);
        backtrack(path, used);
        used[i] = false; path.remove(path.size() - 1);   // undo (the "back")
    }
}
```
Problems: Subsets, Permutations, Combination Sum, Word Search, N-Queens.

## 8. Dynamic Programming

**Signal**: "count the ways", "min/max cost", "can you reach", optimal substructure + overlapping subproblems.

```java
// 1D bottom-up (e.g. climbing stairs / house robber)
int[] dp = new int[n + 1];
dp[0] = base0; dp[1] = base1;
for (int i = 2; i <= n; i++)
    dp[i] = f(dp[i - 1], dp[i - 2]);
return dp[n];
```
Recognize the sub-types: 0/1 knapsack, unbounded knapsack, LIS, LCS, edit distance, grid paths, interval DP.
Start with the recurrence + memoization, then convert to tabulation if needed.

## 9. Top-K / Heap

**Signal**: "K largest/smallest/most frequent", "median of a stream", merge K lists.

```java
PriorityQueue<Integer> heap = new PriorityQueue<>();   // min-heap
for (int x : nums) {
    heap.offer(x);
    if (heap.size() > k) heap.poll();     // keep only K largest
}
return heap.peek();                        // Kth largest
```
Max-heap: `new PriorityQueue<>(Collections.reverseOrder())`.

## 10. Intervals

**Signal**: merge/insert intervals, meeting rooms, overlaps.

```java
Arrays.sort(intervals, (a, b) -> a[0] - b[0]);     // sort by start
List<int[]> merged = new ArrayList<>();
for (int[] iv : intervals) {
    if (merged.isEmpty() || merged.get(merged.size()-1)[1] < iv[0])
        merged.add(iv);
    else
        merged.get(merged.size()-1)[1] = Math.max(merged.get(merged.size()-1)[1], iv[1]);
}
```

## 11. Monotonic Stack

**Signal**: "next greater/smaller element", "days until warmer", largest rectangle in histogram.

```java
Deque<Integer> stack = new ArrayDeque<>();    // holds indices
int[] ans = new int[n];
for (int i = 0; i < n; i++) {
    while (!stack.isEmpty() && a[i] > a[stack.peek()])
        ans[stack.pop()] = i;                 // a[i] is next greater
    stack.push(i);
}
```

## 12. Union-Find (Disjoint Set)

**Signal**: connected components, "number of islands/provinces", cycle detection, Kruskal.

```java
int[] parent;
int find(int x) { return parent[x] == x ? x : (parent[x] = find(parent[x])); }  // path compression
void union(int a, int b) { parent[find(a)] = find(b); }
```

## 13. Topological Sort

**Signal**: ordering with dependencies, course schedule, build order, detect cycle in DAG.

```java
// Kahn's algorithm (BFS on in-degrees)
Queue<Integer> q = new ArrayDeque<>();
for (int i = 0; i < n; i++) if (indegree[i] == 0) q.add(i);
List<Integer> order = new ArrayList<>();
while (!q.isEmpty()) {
    int u = q.poll(); order.add(u);
    for (int v : adj.get(u)) if (--indegree[v] == 0) q.add(v);
}
// order.size() < n  → cycle exists
```

## 14. Trie

**Signal**: prefix search, autocomplete, word dictionary, "starts with".

```java
class Trie {
    Trie[] next = new Trie[26];
    boolean end;
    void insert(String w) {
        Trie node = this;
        for (char c : w.toCharArray()) {
            int i = c - 'a';
            if (node.next[i] == null) node.next[i] = new Trie();
            node = node.next[i];
        }
        node.end = true;
    }
}
```

## 15. Bit Manipulation

**Signal**: "single number", subsets via bitmask, count bits, XOR tricks.

```java
x & (x - 1)          // clears lowest set bit
x & -x               // isolates lowest set bit
Integer.bitCount(x)  // popcount
a ^ a == 0           // XOR self-cancels → find the unpaired element
1 << i               // i-th bit; iterate 0..(1<<n) for all subsets
```

## Signal → pattern lookup

| Problem says... | Pattern |
|---|---|
| sorted array, pair sum | Two pointers |
| longest/shortest contiguous | Sliding window |
| linked list cycle / middle | Fast & slow |
| subarray sums to K | Prefix sum |
| sorted / "minimize max" | Binary search (maybe on answer) |
| shortest path, level order | BFS |
| all combinations/permutations | Backtracking |
| count ways / min cost / optimal | DP |
| K largest / most frequent | Heap |
| overlapping ranges | Intervals |
| next greater element | Monotonic stack |
| connected components | Union-Find |
| dependency ordering | Topological sort |
| prefix / autocomplete | Trie |
| single number / subsets | Bit manipulation |

## Gotchas / things I always forget

- `mid = lo + (hi - lo) / 2` to avoid integer overflow - never `(lo + hi) / 2`.
- Binary search off-by-one: pick `while (lo <= hi)` with `mid±1`, OR `while (lo < hi)` without - don't mix.
- Backtracking: **copy** the path when adding to results (`new ArrayList<>(path)`), or every entry points to the same mutated list.
- BFS marks visited **when enqueuing**, not when dequeuing - otherwise nodes get added multiple times.
- `PriorityQueue` is a **min**-heap by default in Java. For "K largest", keep a size-K min-heap and poll the smallest.
- Use `ArrayDeque` for stacks/queues, never `Stack` (legacy, synchronized) or `LinkedList` (slow).
- DP: define the state and recurrence in words first; the code is mechanical after that.
- Union-Find without path compression + union by rank degrades to O(n) - include both.

## Quick reference (Java contest idioms)

| Need | Use |
|---|---|
| Stack / queue | `ArrayDeque` |
| Min-heap | `PriorityQueue<>()` |
| Max-heap | `PriorityQueue<>(Collections.reverseOrder())` |
| Sorted map/set | `TreeMap` / `TreeSet` (`floorKey`, `ceilingKey`) |
| Count / accumulate | `map.merge(k, 1, Integer::sum)` |
| 2D grid dirs | `{{1,0},{-1,0},{0,1},{0,-1}}` |
| Sort by field | `Arrays.sort(a, (x,y) -> x[0]-y[0])` |
| Popcount | `Integer.bitCount(x)` |
