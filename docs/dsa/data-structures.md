# Data Structures

## Overview

| Structure | Access | Search | Insert | Delete | Use when |
|---|---|---|---|---|---|
| Array / list | O(1) | O(n) | O(n) (end O(1)) | O(n) | Index access, iteration |
| Hash map / set | — | O(1) | O(1) | O(1) | Lookups, counting, dedupe |
| Linked list | O(n) | O(n) | O(1) at node | O(1) at node | Frequent insert/delete with a reference |
| Stack | top O(1) | O(n) | O(1) | O(1) | LIFO: undo, parsing, DFS |
| Queue / deque | ends O(1) | O(n) | O(1) | O(1) | FIFO: BFS, scheduling |
| Heap | min O(1) | O(n) | O(log n) | O(log n) | Top-k, scheduling, Dijkstra |
| Balanced BST | O(log n) | O(log n) | O(log n) | O(log n) | Sorted data with updates |
| Trie | — | O(L) | O(L) | O(L) | Prefix search, autocomplete |
| Graph | — | O(V+E) | — | — | Networks, dependencies |

## Arrays & strings

Python `list` = dynamic array. Key techniques: two pointers, sliding window, prefix sums, in-place swaps.

```python
# Prefix sums: O(1) range-sum queries after O(n) preprocessing
nums = [3, 1, 4, 1, 5]
prefix = [0]
for x in nums:
    prefix.append(prefix[-1] + x)
range_sum = lambda i, j: prefix[j + 1] - prefix[i]     # sum(nums[i..j])
range_sum(1, 3)   # 6

# 2D matrix
rows, cols = 3, 4
grid = [[0] * cols for _ in range(rows)]
transposed = [list(r) for r in zip(*grid)]
rotated_90 = [list(r) for r in zip(*grid[::-1])]
```

## Hash maps & sets

```python
# Two-sum in O(n)
def two_sum(nums, target):
    seen = {}                        # value -> index
    for i, x in enumerate(nums):
        if target - x in seen:
            return [seen[target - x], i]
        seen[x] = i
    return []

# Group anagrams
from collections import defaultdict
def group_anagrams(words):
    groups = defaultdict(list)
    for w in words:
        groups[tuple(sorted(w))].append(w)
    return list(groups.values())
```

**How it works:** hash function → bucket index; collisions handled by chaining (list per bucket) or open addressing (probe next slots — CPython). Load factor triggers resize (rehash all, O(n), amortized O(1)).

## Linked list

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val, self.next = val, next

def reverse(head):                     # iterative, O(n) time O(1) space
    prev = None
    while head:
        head.next, prev, head = prev, head, head.next
    return prev

def has_cycle(head):                   # Floyd's tortoise & hare
    slow = fast = head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
        if slow is fast:
            return True
    return False

def middle(head):
    slow = fast = head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
    return slow

def merge_sorted(a, b):
    dummy = tail = ListNode()          # dummy node simplifies edge cases
    while a and b:
        if a.val <= b.val: tail.next, a = a, a.next
        else:              tail.next, b = b, b.next
        tail = tail.next
    tail.next = a or b
    return dummy.next
```

Doubly linked list: each node has `prev` and `next` → O(1) removal given the node (used in LRU cache).

## Stack (LIFO)

```python
stack = []
stack.append(1); stack.append(2)
stack.pop()        # 2
stack[-1]          # peek

def is_valid_brackets(s: str) -> bool:
    pairs = {")": "(", "]": "[", "}": "{"}
    st = []
    for ch in s:
        if ch in "([{":
            st.append(ch)
        elif not st or st.pop() != pairs[ch]:
            return False
    return not st

# Monotonic stack: next greater element, O(n)
def next_greater(nums):
    res, st = [-1] * len(nums), []       # stack of indices, values decreasing
    for i, x in enumerate(nums):
        while st and nums[st[-1]] < x:
            res[st.pop()] = x
        st.append(i)
    return res
next_greater([2, 1, 2, 4, 3])   # [4, 2, 4, -1, -1]
```

## Queue & deque (FIFO)

```python
from collections import deque
q = deque([1, 2])
q.append(3)        # enqueue right
q.popleft()        # dequeue left, O(1)   (list.pop(0) is O(n)!)
q.appendleft(0)

# Sliding window maximum with a monotonic deque, O(n)
def max_sliding_window(nums, k):
    dq, out = deque(), []              # indices, values decreasing
    for i, x in enumerate(nums):
        while dq and nums[dq[-1]] <= x:
            dq.pop()
        dq.append(i)
        if dq[0] <= i - k:
            dq.popleft()
        if i >= k - 1:
            out.append(nums[dq[0]])
    return out
```

Thread-safe: `queue.Queue`; async: `asyncio.Queue`; priority: `queue.PriorityQueue` / `heapq`.

## Heap / priority queue

A complete binary tree stored in an array where parent ≤ children (min-heap). Children of `i` are `2i+1`, `2i+2`.

```python
import heapq
h = []
heapq.heappush(h, 5); heapq.heappush(h, 1); heapq.heappush(h, 3)
heapq.heappop(h)          # 1
heapq.heapify(nums)       # O(n)
# Max-heap: push negatives
heapq.heappush(h, -x)
# Tuples compare element-wise: (priority, counter, item)

def k_largest(nums, k):                # O(n log k)
    h = []
    for x in nums:
        heapq.heappush(h, x)
        if len(h) > k:
            heapq.heappop(h)
    return sorted(h, reverse=True)

def merge_k_sorted(lists):             # O(N log k)
    h = [(lst[0], i, 0) for i, lst in enumerate(lists) if lst]
    heapq.heapify(h)
    out = []
    while h:
        val, i, j = heapq.heappop(h)
        out.append(val)
        if j + 1 < len(lists[i]):
            heapq.heappush(h, (lists[i][j + 1], i, j + 1))
    return out
```

**Median of a stream:** max-heap for lower half + min-heap for upper half, balance sizes → O(log n) insert, O(1) median.

## Trees

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val, self.left, self.right = val, left, right

#        4
#      /   \
#     2     6
#    / \   / \
#   1   3 5   7
root = TreeNode(4, TreeNode(2, TreeNode(1), TreeNode(3)), TreeNode(6, TreeNode(5), TreeNode(7)))
```

### Traversals

```python
def inorder(n):    return inorder(n.left) + [n.val] + inorder(n.right) if n else []   # L N R -> sorted for BST
def preorder(n):   return [n.val] + preorder(n.left) + preorder(n.right) if n else [] # N L R -> copy/serialize
def postorder(n):  return postorder(n.left) + postorder(n.right) + [n.val] if n else [] # L R N -> delete/evaluate

def level_order(root):                 # BFS
    if not root: return []
    out, q = [], deque([root])
    while q:
        level = []
        for _ in range(len(q)):
            node = q.popleft()
            level.append(node.val)
            if node.left: q.append(node.left)
            if node.right: q.append(node.right)
        out.append(level)
    return out
level_order(root)    # [[4], [2, 6], [1, 3, 5, 7]]

def inorder_iterative(root):
    out, st, cur = [], [], root
    while cur or st:
        while cur:
            st.append(cur); cur = cur.left
        cur = st.pop()
        out.append(cur.val)
        cur = cur.right
    return out
```

### Classic tree problems

```python
def max_depth(n):
    return 1 + max(max_depth(n.left), max_depth(n.right)) if n else 0

def is_balanced(root):
    def h(n):
        if not n: return 0
        l, r = h(n.left), h(n.right)
        if l < 0 or r < 0 or abs(l - r) > 1: return -1
        return 1 + max(l, r)
    return h(root) >= 0

def invert(n):
    if n:
        n.left, n.right = invert(n.right), invert(n.left)
    return n

def lowest_common_ancestor(root, p, q):          # binary tree
    if not root or root is p or root is q:
        return root
    l = lowest_common_ancestor(root.left, p, q)
    r = lowest_common_ancestor(root.right, p, q)
    return root if l and r else l or r

def diameter(root):
    best = 0
    def depth(n):
        nonlocal best
        if not n: return 0
        l, r = depth(n.left), depth(n.right)
        best = max(best, l + r)
        return 1 + max(l, r)
    depth(root)
    return best
```

### Binary Search Tree

Left subtree < node < right subtree. O(h) operations: O(log n) if balanced, O(n) if skewed. Self-balancing: **AVL**, **Red-Black** (used in Java TreeMap, Linux scheduler), **B-trees** (databases).

```python
def bst_insert(root, val):
    if not root: return TreeNode(val)
    if val < root.val: root.left = bst_insert(root.left, val)
    else:              root.right = bst_insert(root.right, val)
    return root

def bst_search(root, val):
    while root and root.val != val:
        root = root.left if val < root.val else root.right
    return root

def is_valid_bst(n, lo=float("-inf"), hi=float("inf")):
    if not n: return True
    if not lo < n.val < hi: return False
    return is_valid_bst(n.left, lo, n.val) and is_valid_bst(n.right, n.val, hi)
```

Python has no built-in balanced BST → use `bisect` on a sorted list or the `sortedcontainers` package.

## Trie (prefix tree)

```python
class Trie:
    def __init__(self):
        self.root = {}

    def insert(self, word: str) -> None:
        node = self.root
        for ch in word:
            node = node.setdefault(ch, {})
        node["$"] = True                       # end-of-word marker

    def search(self, word: str) -> bool:
        node = self._walk(word)
        return node is not None and "$" in node

    def starts_with(self, prefix: str) -> bool:
        return self._walk(prefix) is not None

    def _walk(self, s):
        node = self.root
        for ch in s:
            if ch not in node: return None
            node = node[ch]
        return node

t = Trie(); t.insert("python"); t.insert("pytest")
t.starts_with("pyt"), t.search("py")         # (True, False)
```

## Graphs

Representations: **adjacency list** (`dict[node, list[node]]`, O(V+E) space — default) vs **adjacency matrix** (O(V²), O(1) edge check, dense graphs). Directed/undirected, weighted/unweighted, cyclic/acyclic (DAG).

```python
from collections import defaultdict
edges = [("A", "B"), ("A", "C"), ("B", "D"), ("C", "D"), ("D", "E")]
graph = defaultdict(list)
for u, v in edges:
    graph[u].append(v)
    graph[v].append(u)          # undirected
```

Traversals (BFS, DFS), shortest paths, topological sort → see [Algorithms](algorithms.md#graph-algorithms).

## Union-Find (Disjoint Set Union)

Track connected components with near-O(1) `find`/`union` (path compression + union by rank).

```python
class UnionFind:
    def __init__(self, n):
        self.parent, self.rank, self.count = list(range(n)), [0] * n, n

    def find(self, x):
        while self.parent[x] != x:
            self.parent[x] = self.parent[self.parent[x]]   # path halving
            x = self.parent[x]
        return x

    def union(self, a, b) -> bool:
        ra, rb = self.find(a), self.find(b)
        if ra == rb: return False                         # already connected -> cycle
        if self.rank[ra] < self.rank[rb]: ra, rb = rb, ra
        self.parent[rb] = ra
        self.rank[ra] += self.rank[ra] == self.rank[rb]
        self.count -= 1
        return True

uf = UnionFind(5); uf.union(0, 1); uf.union(3, 4); uf.count    # 3 components
```

Uses: number of islands/components, cycle detection, Kruskal's MST, account merging.

## LRU cache (hash map + doubly linked list)

Asked constantly. O(1) `get` and `put`.

```python
class Node:
    __slots__ = ("key", "val", "prev", "next")
    def __init__(self, key=0, val=0):
        self.key, self.val, self.prev, self.next = key, val, None, None

class LRUCache:
    def __init__(self, capacity: int):
        self.cap, self.map = capacity, {}
        self.head, self.tail = Node(), Node()         # sentinels
        self.head.next, self.tail.prev = self.tail, self.head

    def _remove(self, n):
        n.prev.next, n.next.prev = n.next, n.prev

    def _add_front(self, n):
        n.prev, n.next = self.head, self.head.next
        self.head.next.prev = n
        self.head.next = n

    def get(self, key: int) -> int:
        if key not in self.map: return -1
        n = self.map[key]
        self._remove(n); self._add_front(n)
        return n.val

    def put(self, key: int, val: int) -> None:
        if key in self.map:
            self._remove(self.map[key])
        n = Node(key, val)
        self.map[key] = n
        self._add_front(n)
        if len(self.map) > self.cap:
            lru = self.tail.prev
            self._remove(lru)
            del self.map[lru.key]
```

(In real code: `functools.lru_cache` or `OrderedDict` — see the Python Q&A.)

## Bloom filter (probabilistic)

Bit array + k hash functions. "Definitely not present" or "probably present" — no false negatives. Used to avoid DB/cache lookups for non-existent keys.

```python
import hashlib
class BloomFilter:
    def __init__(self, size=10_000, k=3):
        self.bits, self.size, self.k = bytearray(size), size, k
    def _hashes(self, item: str):
        for i in range(self.k):
            yield int(hashlib.sha256(f"{i}:{item}".encode()).hexdigest(), 16) % self.size
    def add(self, item): 
        for h in self._hashes(item): self.bits[h] = 1
    def might_contain(self, item):
        return all(self.bits[h] for h in self._hashes(item))
```
