# Algorithms

## Sorting

| Algorithm | Best | Average | Worst | Space | Stable | Notes |
|---|---|---|---|---|---|---|
| Bubble | O(n) | O(n²) | O(n²) | O(1) | ✅ | Teaching only |
| Selection | O(n²) | O(n²) | O(n²) | O(1) | ❌ | Fewest swaps |
| Insertion | O(n) | O(n²) | O(n²) | O(1) | ✅ | Great for small/nearly sorted |
| **Merge** | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ | Linked lists, external sort |
| **Quick** | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ | Fast in practice, random pivot |
| **Heap** | O(n log n) | O(n log n) | O(n log n) | O(1) | ❌ | Guaranteed, in-place |
| Counting / Radix | O(n + k) | O(n + k) | O(n + k) | O(k) | ✅ | Small integer ranges |
| **Timsort** (Python) | O(n) | O(n log n) | O(n log n) | O(n) | ✅ | Merge + insertion; exploits runs |

**Stable** = equal elements keep their original order (matters when sorting by multiple keys in passes).

```python
def merge_sort(a):
    if len(a) <= 1:
        return a
    mid = len(a) // 2
    left, right = merge_sort(a[:mid]), merge_sort(a[mid:])
    out, i, j = [], 0, 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            out.append(left[i]); i += 1
        else:
            out.append(right[j]); j += 1
    return out + left[i:] + right[j:]

import random
def quick_sort(a, lo=0, hi=None):              # in place, Lomuto partition
    if hi is None: hi = len(a) - 1
    if lo >= hi: return a
    p = random.randint(lo, hi)
    a[p], a[hi] = a[hi], a[p]
    pivot, i = a[hi], lo
    for j in range(lo, hi):
        if a[j] < pivot:
            a[i], a[j] = a[j], a[i]; i += 1
    a[i], a[hi] = a[hi], a[i]
    quick_sort(a, lo, i - 1)
    quick_sort(a, i + 1, hi)
    return a

def insertion_sort(a):
    for i in range(1, len(a)):
        key, j = a[i], i - 1
        while j >= 0 and a[j] > key:
            a[j + 1] = a[j]; j -= 1
        a[j + 1] = key
    return a

# Quickselect: k-th smallest in average O(n)
def quickselect(a, k):
    pivot = random.choice(a)
    lows = [x for x in a if x < pivot]
    pivots = [x for x in a if x == pivot]
    highs = [x for x in a if x > pivot]
    if k < len(lows): return quickselect(lows, k)
    if k < len(lows) + len(pivots): return pivot
    return quickselect(highs, k - len(lows) - len(pivots))
```

## Binary search

On any **sorted / monotonic** space. O(log n).

```python
def binary_search(a, target):
    lo, hi = 0, len(a) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if a[mid] == target: return mid
        if a[mid] < target:  lo = mid + 1
        else:                hi = mid - 1
    return -1

# Template: first index where condition(x) is True (lower bound)
def first_true(lo, hi, cond):
    while lo < hi:
        mid = (lo + hi) // 2
        if cond(mid): hi = mid
        else:         lo = mid + 1
    return lo

import bisect
bisect.bisect_left([1, 2, 2, 3], 2)     # 1  first position for 2
bisect.bisect_right([1, 2, 2, 3], 2)    # 3

# Search in rotated sorted array
def search_rotated(a, t):
    lo, hi = 0, len(a) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if a[mid] == t: return mid
        if a[lo] <= a[mid]:                       # left half sorted
            if a[lo] <= t < a[mid]: hi = mid - 1
            else: lo = mid + 1
        else:                                     # right half sorted
            if a[mid] < t <= a[hi]: lo = mid + 1
            else: hi = mid - 1
    return -1

# Binary search on the answer: min capacity to ship within D days
def ship_within_days(weights, days):
    def ok(cap):
        need, cur = 1, 0
        for w in weights:
            if cur + w > cap: need, cur = need + 1, 0
            cur += w
        return need <= days
    return first_true(max(weights), sum(weights), ok)
```

## Recursion & backtracking

Backtracking = DFS over a decision tree: **choose → explore → un-choose**.

```python
def subsets(nums):                      # O(2^n)
    res, path = [], []
    def dfs(i):
        if i == len(nums):
            res.append(path[:]); return
        path.append(nums[i]); dfs(i + 1)      # include
        path.pop();           dfs(i + 1)      # exclude
    dfs(0)
    return res

def permutations(nums):                 # O(n!)
    res = []
    def dfs(path, used):
        if len(path) == len(nums):
            res.append(path[:]); return
        for i, x in enumerate(nums):
            if used[i]: continue
            used[i] = True; path.append(x)
            dfs(path, used)
            path.pop(); used[i] = False
    dfs([], [False] * len(nums))
    return res

def combination_sum(cands, target):     # reuse allowed
    res = []
    def dfs(start, remain, path):
        if remain == 0: res.append(path[:]); return
        for i in range(start, len(cands)):
            if cands[i] <= remain:
                path.append(cands[i])
                dfs(i, remain - cands[i], path)
                path.pop()
    dfs(0, target, [])
    return res

def solve_n_queens(n):
    res, cols, d1, d2 = 0, set(), set(), set()
    def place(r):
        nonlocal res
        if r == n: res += 1; return
        for c in range(n):
            if c in cols or r - c in d1 or r + c in d2: continue
            cols.add(c); d1.add(r - c); d2.add(r + c)
            place(r + 1)
            cols.remove(c); d1.remove(r - c); d2.remove(r + c)
    place(0)
    return res
```

## Graph algorithms

### BFS — shortest path in unweighted graphs

```python
from collections import deque
def bfs_shortest(graph, start, goal):
    q, dist = deque([start]), {start: 0}
    while q:
        node = q.popleft()
        if node == goal: return dist[node]
        for nb in graph[node]:
            if nb not in dist:
                dist[nb] = dist[node] + 1
                q.append(nb)
    return -1

# Grid BFS: number of islands
def num_islands(grid):
    rows, cols, count = len(grid), len(grid[0]), 0
    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == "1":
                count += 1
                q = deque([(r, c)]); grid[r][c] = "0"
                while q:
                    x, y = q.popleft()
                    for dx, dy in ((1, 0), (-1, 0), (0, 1), (0, -1)):
                        nx, ny = x + dx, y + dy
                        if 0 <= nx < rows and 0 <= ny < cols and grid[nx][ny] == "1":
                            grid[nx][ny] = "0"; q.append((nx, ny))
    return count
```

### DFS — connectivity, cycles, paths

```python
from collections import defaultdict

def dfs(graph, start):
    seen, order, stack = set(), [], [start]
    while stack:
        node = stack.pop()
        if node in seen: continue
        seen.add(node); order.append(node)
        stack.extend(reversed(graph[node]))
    return order

# Cycle detection in a directed graph (white/gray/black colouring)
def has_cycle_directed(graph):
    WHITE, GRAY, BLACK = 0, 1, 2
    color = defaultdict(int)
    def visit(u):
        color[u] = GRAY
        for v in graph[u]:
            if color[v] == GRAY: return True           # back edge
            if color[v] == WHITE and visit(v): return True
        color[u] = BLACK
        return False
    return any(color[u] == WHITE and visit(u) for u in list(graph))
```

### Topological sort (Kahn's algorithm) — dependency ordering

```python
from collections import defaultdict, deque
def topo_sort(n, prerequisites):              # course schedule, build order, migrations
    graph, indeg = defaultdict(list), [0] * n
    for course, pre in prerequisites:
        graph[pre].append(course); indeg[course] += 1
    q = deque(i for i in range(n) if indeg[i] == 0)
    order = []
    while q:
        u = q.popleft(); order.append(u)
        for v in graph[u]:
            indeg[v] -= 1
            if indeg[v] == 0: q.append(v)
    return order if len(order) == n else []    # [] -> cycle exists
```

### Dijkstra — shortest path with non-negative weights

```python
import heapq
def dijkstra(graph, src):          # graph: {u: [(v, w), ...]}
    dist = {src: 0}
    pq = [(0, src)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist.get(u, float("inf")): continue     # stale entry
        for v, w in graph.get(u, []):
            nd = d + w
            if nd < dist.get(v, float("inf")):
                dist[v] = nd
                heapq.heappush(pq, (nd, v))
    return dist
# O((V + E) log V)
```

| Algorithm | Use | Complexity |
|---|---|---|
| BFS | Unweighted shortest path | O(V + E) |
| Dijkstra | Non-negative weights | O((V+E) log V) |
| Bellman-Ford | Negative weights, detects negative cycles | O(V·E) |
| Floyd-Warshall | All pairs | O(V³) |
| A* | Heuristic-guided shortest path (maps, games) | depends on heuristic |
| Kruskal / Prim | Minimum spanning tree | O(E log E) |
| Topological sort | DAG ordering | O(V + E) |

## Greedy

Make the locally optimal choice; works when the problem has the **greedy-choice property** (prove it or find a counter-example).

```python
def max_meetings(intervals):           # activity selection: sort by end time
    count, end = 0, float("-inf")
    for s, e in sorted(intervals, key=lambda x: x[1]):
        if s >= end:
            count, end = count + 1, e
    return count

def merge_intervals(intervals):
    out = []
    for s, e in sorted(intervals):
        if out and s <= out[-1][1]:
            out[-1][1] = max(out[-1][1], e)
        else:
            out.append([s, e])
    return out
merge_intervals([[1, 3], [2, 6], [8, 10]])    # [[1, 6], [8, 10]]

def can_jump(nums):
    reach = 0
    for i, x in enumerate(nums):
        if i > reach: return False
        reach = max(reach, i + x)
    return True
```

## Dynamic programming

DP = recursion + **overlapping subproblems** + **optimal substructure**. Steps: define state → recurrence → base cases → order (top-down memo or bottom-up table) → optimise space.

```python
from functools import cache

# 1. Fibonacci / climbing stairs
@cache
def climb(n): return n if n <= 2 else climb(n - 1) + climb(n - 2)

def climb_bottom_up(n):
    a, b = 1, 1
    for _ in range(n): a, b = b, a + b
    return a

# 2. House robber: dp[i] = max(dp[i-1], dp[i-2] + nums[i])
def rob(nums):
    prev, cur = 0, 0
    for x in nums: prev, cur = cur, max(cur, prev + x)
    return cur

# 3. Coin change (min coins) — unbounded knapsack
def coin_change(coins, amount):
    dp = [0] + [float("inf")] * amount
    for a in range(1, amount + 1):
        for c in coins:
            if c <= a: dp[a] = min(dp[a], dp[a - c] + 1)
    return dp[amount] if dp[amount] != float("inf") else -1

# 4. 0/1 knapsack
def knapsack(weights, values, cap):
    dp = [0] * (cap + 1)
    for w, v in zip(weights, values):
        for c in range(cap, w - 1, -1):          # iterate backwards for 0/1
            dp[c] = max(dp[c], dp[c - w] + v)
    return dp[cap]

# 5. Longest increasing subsequence — O(n log n) patience sorting
import bisect
def lis(nums):
    tails = []
    for x in nums:
        i = bisect.bisect_left(tails, x)
        if i == len(tails): tails.append(x)
        else: tails[i] = x
    return len(tails)

# 6. Longest common subsequence — 2D DP
def lcs(a, b):
    dp = [[0] * (len(b) + 1) for _ in range(len(a) + 1)]
    for i in range(1, len(a) + 1):
        for j in range(1, len(b) + 1):
            dp[i][j] = dp[i-1][j-1] + 1 if a[i-1] == b[j-1] else max(dp[i-1][j], dp[i][j-1])
    return dp[-1][-1]

# 7. Edit distance (Levenshtein)
def edit_distance(a, b):
    dp = list(range(len(b) + 1))
    for i in range(1, len(a) + 1):
        prev, dp[0] = dp[0], i
        for j in range(1, len(b) + 1):
            cur = dp[j]
            dp[j] = prev if a[i-1] == b[j-1] else 1 + min(prev, dp[j], dp[j-1])
            prev = cur
    return dp[-1]

# 8. Unique paths in a grid
def unique_paths(m, n):
    row = [1] * n
    for _ in range(1, m):
        for j in range(1, n): row[j] += row[j - 1]
    return row[-1]

# 9. Word break
def word_break(s, words):
    words, dp = set(words), [True] + [False] * len(s)
    for i in range(1, len(s) + 1):
        dp[i] = any(dp[j] and s[j:i] in words for j in range(i))
    return dp[-1]

# 10. Maximum subarray (Kadane)
def max_subarray(nums):
    best = cur = nums[0]
    for x in nums[1:]:
        cur = max(x, cur + x)
        best = max(best, cur)
    return best
```

**DP categories:** 1D (stairs, robber), 2D grid (paths), two-sequence (LCS, edit distance), knapsack (subset sum, coin change), intervals (burst balloons, matrix chain), trees (tree DP), bitmask (TSP), state machines (stock buy/sell with cooldown).

## Bit manipulation

```python
x & 1            # odd?
x & (x - 1)      # clear lowest set bit; == 0 -> power of two
x & -x           # lowest set bit
bin(x).count("1")  # popcount (x.bit_count() in 3.10+)
a ^ a == 0; a ^ 0 == a       # XOR tricks

def single_number(nums):     # every element twice except one
    res = 0
    for x in nums: res ^= x
    return res
```

## Math essentials

```python
import math
math.gcd(12, 18); math.lcm(4, 6)          # 6, 12
def is_prime(n):
    if n < 2: return False
    return all(n % i for i in range(2, math.isqrt(n) + 1))
def sieve(n):                              # primes up to n, O(n log log n)
    is_p = [True] * (n + 1); is_p[0:2] = [False, False]
    for i in range(2, math.isqrt(n) + 1):
        if is_p[i]: is_p[i*i::i] = [False] * len(is_p[i*i::i])
    return [i for i, p in enumerate(is_p) if p]
pow(2, 100, 10**9 + 7)                     # fast modular exponentiation
```
