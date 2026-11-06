# Problem-Solving Patterns

Most interview problems are one of these patterns in disguise. Learn the **signal** → **template**.

| # | Pattern | Signal in the problem |
|---|---|---|
| 1 | Hashing | "find pair/duplicate/count", O(n) wanted |
| 2 | Two pointers | sorted array, pairs, palindromes, in-place |
| 3 | Sliding window | contiguous subarray/substring with a constraint |
| 4 | Prefix sum | range sums, "subarray sum equals k" |
| 5 | Fast & slow pointers | linked list cycle/middle |
| 6 | Binary search | sorted input, or "minimum X such that…" |
| 7 | Monotonic stack | next greater/smaller, histogram |
| 8 | Heap / top-k | "k largest/smallest/most frequent", merge k |
| 9 | Intervals | overlapping ranges, meetings |
| 10 | BFS | shortest path (unweighted), levels, grids |
| 11 | DFS / backtracking | all combinations/permutations, paths |
| 12 | Topological sort | dependencies, ordering |
| 13 | Union-find | connected components, merging groups |
| 14 | Dynamic programming | "number of ways", "min/max", choices with overlap |
| 15 | Trie | prefixes, autocomplete, word search |

---

## 1. Hashing

```python
def contains_duplicate(nums): return len(set(nums)) != len(nums)

def longest_consecutive(nums):              # O(n)
    s, best = set(nums), 0
    for x in s:
        if x - 1 not in s:                  # start of a run
            y = x
            while y + 1 in s: y += 1
            best = max(best, y - x + 1)
    return best

from collections import Counter
import heapq
def top_k_frequent(nums, k):
    return [x for x, _ in Counter(nums).most_common(k)]
```

## 2. Two pointers

```python
def two_sum_sorted(a, target):
    l, r = 0, len(a) - 1
    while l < r:
        s = a[l] + a[r]
        if s == target: return [l, r]
        if s < target: l += 1
        else: r -= 1
    return []

def is_palindrome(s):
    s = [c.lower() for c in s if c.isalnum()]
    return s == s[::-1]

def three_sum(nums):                         # O(n²)
    nums.sort(); res = []
    for i in range(len(nums) - 2):
        if i and nums[i] == nums[i - 1]: continue
        l, r = i + 1, len(nums) - 1
        while l < r:
            s = nums[i] + nums[l] + nums[r]
            if s < 0: l += 1
            elif s > 0: r -= 1
            else:
                res.append([nums[i], nums[l], nums[r]])
                while l < r and nums[l] == nums[l + 1]: l += 1
                l += 1; r -= 1
    return res

def max_water(height):                       # container with most water
    l, r, best = 0, len(height) - 1, 0
    while l < r:
        best = max(best, (r - l) * min(height[l], height[r]))
        if height[l] < height[r]: l += 1
        else: r -= 1
    return best

def move_zeroes(a):                          # in place, keep order
    w = 0
    for x in a:
        if x != 0: a[w] = x; w += 1
    a[w:] = [0] * (len(a) - w)
```

## 3. Sliding window

**Template (variable window):** expand `right`; while the window is invalid, shrink `left`; record the answer.

```python
def longest_unique_substring(s):              # "abcabcbb" -> 3
    last, left, best = {}, 0, 0
    for right, ch in enumerate(s):
        if ch in last and last[ch] >= left:
            left = last[ch] + 1
        last[ch] = right
        best = max(best, right - left + 1)
    return best

def min_subarray_len(target, nums):           # smallest window with sum >= target
    left = total = 0
    best = float("inf")
    for right, x in enumerate(nums):
        total += x
        while total >= target:
            best = min(best, right - left + 1)
            total -= nums[left]; left += 1
    return 0 if best == float("inf") else best

def max_sum_k(nums, k):                       # fixed window
    window = best = sum(nums[:k])
    for i in range(k, len(nums)):
        window += nums[i] - nums[i - k]
        best = max(best, window)
    return best

from collections import Counter
def min_window(s, t):                         # smallest substring containing all of t
    need, missing = Counter(t), len(t)
    left = start = end = 0
    for right, ch in enumerate(s, 1):
        if need[ch] > 0: missing -= 1
        need[ch] -= 1
        if missing == 0:
            while need[s[left]] < 0:
                need[s[left]] += 1; left += 1
            if not end or right - left < end - start:
                start, end = left, right
            need[s[left]] += 1; missing += 1; left += 1
    return s[start:end]
```

## 4. Prefix sum + hash map

```python
def subarray_sum_equals_k(nums, k):           # count subarrays summing to k, O(n)
    count, prefix, seen = 0, 0, {0: 1}
    for x in nums:
        prefix += x
        count += seen.get(prefix - k, 0)
        seen[prefix] = seen.get(prefix, 0) + 1
    return count

def product_except_self(nums):                # no division
    n = len(nums); out = [1] * n
    left = 1
    for i in range(n): out[i] = left; left *= nums[i]
    right = 1
    for i in reversed(range(n)): out[i] *= right; right *= nums[i]
    return out
```

## 5. Fast & slow pointers

See [Linked list](data-structures.md#linked-list): cycle detection, middle node. Also: "happy number", find duplicate number in O(1) space.

```python
def find_duplicate(nums):                     # values in 1..n, one repeated
    slow = fast = nums[0]
    while True:
        slow, fast = nums[slow], nums[nums[fast]]
        if slow == fast: break
    slow = nums[0]
    while slow != fast:
        slow, fast = nums[slow], nums[fast]
    return slow
```

## 6. Binary search (on index or answer)

See [Algorithms → Binary search](algorithms.md#binary-search). Signal: "minimum capacity / speed / time such that…" → binary-search the answer with a feasibility check.

```python
import math
def min_eating_speed(piles, h):               # Koko eating bananas
    lo, hi = 1, max(piles)
    while lo < hi:
        mid = (lo + hi) // 2
        if sum(math.ceil(p / mid) for p in piles) <= h: hi = mid
        else: lo = mid + 1
    return lo
```

## 7. Monotonic stack

```python
def daily_temperatures(t):                    # days until warmer
    res, st = [0] * len(t), []
    for i, x in enumerate(t):
        while st and t[st[-1]] < x:
            j = st.pop(); res[j] = i - j
        st.append(i)
    return res

def largest_rectangle(heights):
    st, best = [], 0                          # indices with increasing heights
    for i, h in enumerate(heights + [0]):
        while st and heights[st[-1]] >= h:
            height = heights[st.pop()]
            width = i if not st else i - st[-1] - 1
            best = max(best, height * width)
        st.append(i)
    return best
```

## 8. Heap / top-k

See [Heaps](data-structures.md#heap-priority-queue). K closest points:

```python
import heapq
def k_closest(points, k):
    return heapq.nsmallest(k, points, key=lambda p: p[0] ** 2 + p[1] ** 2)
```

## 9. Intervals

```python
def insert_interval(intervals, new):
    res = []
    for i, (s, e) in enumerate(intervals):
        if e < new[0]: res.append([s, e])
        elif s > new[1]: return res + [new] + intervals[i:]
        else: new = [min(s, new[0]), max(e, new[1])]
    return res + [new]

def min_meeting_rooms(intervals):            # max overlap
    import heapq
    ends = []
    for s, e in sorted(intervals):
        if ends and ends[0] <= s: heapq.heappop(ends)
        heapq.heappush(ends, e)
    return len(ends)
```

## 10–13. Graph patterns

BFS, DFS, topological sort, union-find → see [Algorithms](algorithms.md#graph-algorithms) and [Union-Find](data-structures.md#union-find-disjoint-set-union).

```python
from collections import deque
def rotting_oranges(grid):                   # multi-source BFS
    rows, cols = len(grid), len(grid[0])
    q = deque((r, c) for r in range(rows) for c in range(cols) if grid[r][c] == 2)
    fresh = sum(row.count(1) for row in grid)
    minutes = 0
    while q and fresh:
        for _ in range(len(q)):
            r, c = q.popleft()
            for dr, dc in ((1, 0), (-1, 0), (0, 1), (0, -1)):
                nr, nc = r + dr, c + dc
                if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == 1:
                    grid[nr][nc] = 2; fresh -= 1; q.append((nr, nc))
        minutes += 1
    return minutes if fresh == 0 else -1

def clone_graph(node):                       # DFS + hashmap old -> new
    if not node: return None
    copies = {}
    def dfs(n):
        if n in copies: return copies[n]
        copy = type(n)(n.val, [])
        copies[n] = copy
        copy.neighbors = [dfs(nb) for nb in n.neighbors]
        return copy
    return dfs(node)
```

## 14. Dynamic programming

See [Algorithms → DP](algorithms.md#dynamic-programming). Signals: "how many ways", "minimum/maximum cost", "can you reach", choices at each step whose sub-results repeat.

## 15. Trie

See [Trie](data-structures.md#trie-prefix-tree). Signals: prefix queries, autocomplete, "word search II" (trie + DFS on board).

---

## Practice list (by pattern)

| Pattern | Must-do problems |
|---|---|
| Hashing | Two Sum, Group Anagrams, Top K Frequent, Longest Consecutive Sequence |
| Two pointers | Valid Palindrome, 3Sum, Container With Most Water, Trapping Rain Water |
| Sliding window | Longest Substring Without Repeating, Minimum Window Substring, Max Sliding Window |
| Prefix sum | Subarray Sum Equals K, Product Except Self |
| Stack | Valid Parentheses, Daily Temperatures, Largest Rectangle, Min Stack |
| Binary search | Search Rotated Array, Koko Bananas, Median of Two Sorted Arrays |
| Linked list | Reverse List, Merge Two Lists, LRU Cache, Linked List Cycle |
| Trees | Max Depth, Invert, Level Order, Validate BST, LCA, Serialize/Deserialize |
| Heap | Kth Largest, Merge K Lists, Find Median from Data Stream |
| Intervals | Merge Intervals, Insert Interval, Meeting Rooms II |
| Graphs | Number of Islands, Clone Graph, Course Schedule, Rotting Oranges, Network Delay (Dijkstra) |
| Backtracking | Subsets, Permutations, Combination Sum, Word Search, N-Queens |
| DP | Climbing Stairs, House Robber, Coin Change, LIS, LCS, Edit Distance, Word Break |
