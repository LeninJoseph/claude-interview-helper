# Complexity & Big-O

**Big-O** describes how running time or memory **grows** as input size *n* grows — the upper bound, ignoring constants and lower-order terms.

- **O** (upper bound), **Ω** (lower bound), **Θ** (tight bound). Interviews say "Big-O" meaning the worst/tight case.
- Drop constants: O(2n) → O(n). Drop smaller terms: O(n² + n) → O(n²).
- Different inputs get different variables: O(a + b), O(a · b).

## Common classes

| Big-O | Name | Example | n = 10⁶ feasible? |
|---|---|---|---|
| O(1) | constant | dict lookup, array index | ✅ |
| O(log n) | logarithmic | binary search, heap push | ✅ |
| O(n) | linear | single loop, linear search | ✅ |
| O(n log n) | linearithmic | sorting, heap of n items | ✅ |
| O(n²) | quadratic | nested loops, bubble sort | ❌ (10¹²) |
| O(2ⁿ) | exponential | subsets, naive fibonacci | ❌ n ≤ ~25 |
| O(n!) | factorial | permutations | ❌ n ≤ ~10 |

**Rule of thumb from constraints:** ~10⁸ simple operations/second in C, ~10⁷ in Python. n ≤ 10⁵ → aim for O(n log n) or better.

## Analysing code

```python
def f1(nums):                       # O(n)
    total = 0
    for x in nums: total += x
    return total

def f2(nums):                       # O(n²)
    for i in range(len(nums)):
        for j in range(i + 1, len(nums)):
            ...

def f3(n):                          # O(log n): halving each step
    while n > 1:
        n //= 2

def f4(nums):                       # O(n log n)
    for x in nums:                  # n
        bisect.insort(sorted_list, x)   # careful: insort is O(n) insert! -> O(n²) overall

def f5(a, b):                       # O(a + b)
    for x in a: ...
    for y in b: ...
```

**Hidden costs in Python:** `x in list` O(n), `list.pop(0)` O(n), slicing `s[i:j]` O(j−i), `"".join` O(n), string `+=` in a loop O(n²), `sorted()` O(n log n), copying a dict O(n).

## Recursion: the recurrence → master theorem

| Recurrence | Example | Result |
|---|---|---|
| T(n) = T(n/2) + O(1) | binary search | O(log n) |
| T(n) = 2T(n/2) + O(n) | merge sort | O(n log n) |
| T(n) = T(n−1) + O(1) | linear recursion | O(n) |
| T(n) = 2T(n−1) + O(1) | naive fibonacci / subsets | O(2ⁿ) |

Recursion uses **O(depth)** stack space.

## Amortized analysis

An operation can be occasionally expensive but cheap on average. `list.append` is usually O(1); when the array is full it copies everything (O(n)), but doubling capacity means total work over n appends is O(n) → **amortized O(1)**.

## Space complexity

Count extra memory relative to input: auxiliary arrays, hash maps, recursion stack. In-place algorithms use O(1) extra.

```python
def reverse_in_place(a):            # O(1) extra space
    l, r = 0, len(a) - 1
    while l < r:
        a[l], a[r] = a[r], a[l]
        l, r = l + 1, r - 1
```

## Python built-in complexity cheat sheet

| | Average | Worst |
|---|---|---|
| list index/append/pop() | O(1) | O(1)* |
| list insert/pop(i)/remove/`in` | O(n) | O(n) |
| dict/set get/set/del/`in` | O(1) | O(n) |
| deque append/popleft | O(1) | O(1) |
| heapq push/pop | O(log n) | O(log n) |
| heapify | O(n) | O(n) |
| sort | O(n log n) | O(n log n) |
| bisect | O(log n) | O(log n) |
