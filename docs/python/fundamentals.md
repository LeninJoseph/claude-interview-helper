# Fundamentals & Data Types

## What is Python?

Python is a **dynamically typed, strongly typed, interpreted (compiled-to-bytecode), garbage-collected, multi-paradigm** language.

- **Dynamically typed** – types are checked at runtime; a variable is just a name bound to an object.
- **Strongly typed** – no implicit coercion between unrelated types (`"1" + 1` → `TypeError`).
- **CPython** – the reference implementation compiles `.py` → bytecode (`.pyc`) → executed by a stack-based VM. Other implementations: PyPy (JIT), Jython, MicroPython.

```python
x = 10        # name x -> int object 10
x = "ten"     # same name rebound to a str object (dynamic typing)
"1" + 1       # TypeError (strong typing)
```

## Names, objects and references

Everything in Python is an **object** with an *identity* (`id()`), a *type* (`type()`) and a *value*. Variables are **names (references)** bound to objects — assignment never copies.

```python
a = [1, 2, 3]
b = a           # b references the SAME list
b.append(4)
print(a)        # [1, 2, 3, 4]
print(a is b)   # True  -> same identity
print(a == [1, 2, 3, 4])  # True -> equal value
```

!!! warning "`is` vs `==`"
    `==` compares **values** (calls `__eq__`). `is` compares **identity** (same object in memory).
    Use `is` only for singletons: `None`, `True`, `False`, sentinel objects.

## Mutable vs immutable

| Immutable | Mutable |
|---|---|
| `int`, `float`, `complex`, `bool`, `str`, `bytes`, `tuple`, `frozenset`, `NoneType` | `list`, `dict`, `set`, `bytearray`, most user classes |

```python
s = "hello"
t = s
s += " world"      # creates a NEW str; t unchanged
print(t)           # hello

l = [1]
m = l
l += [2]           # list.__iadd__ mutates in place
print(m)           # [1, 2]

t = ([1, 2], 3)    # tuple is immutable, but holds a mutable list
t[0].append(99)    # allowed!
print(t)           # ([1, 2, 99], 3)
```

**Why it matters:** only *hashable* (usually immutable) objects can be dict keys or set members. Function arguments are passed by **object reference** ("call by sharing") — mutating a mutable argument is visible to the caller; rebinding is not.

```python
def modify(lst, n):
    lst.append(1)   # visible to caller
    n += 1          # rebinds local name only

nums, k = [], 5
modify(nums, k)
print(nums, k)      # [1] 5
```

## Numbers

```python
7 / 2       # 3.5   true division -> float
7 // 2      # 3     floor division
-7 // 2     # -4    floors toward -infinity!
7 % 3       # 1
-7 % 3      # 2     sign follows divisor
2 ** 10     # 1024
divmod(17, 5)  # (3, 2)

0.1 + 0.2 == 0.3          # False (binary floating point)
import math
math.isclose(0.1 + 0.2, 0.3)  # True

from decimal import Decimal
Decimal("0.1") + Decimal("0.2")  # Decimal('0.3')  -> use for money

big = 2 ** 200   # ints have arbitrary precision
```

!!! tip "Money"
    Never use `float` for currency. Use `Decimal` or store integer minor units (cents/paise).

## Strings

Strings are **immutable sequences of Unicode code points**.

```python
s = "Python Interview"
s[0], s[-1]           # 'P', 'w'
s[0:6]                # 'Python'   slicing [start:stop:step]
s[::-1]               # reversed
s.lower(), s.upper(), s.title()
s.split()             # ['Python', 'Interview']
"-".join(["a", "b"])  # 'a-b'
s.replace("Python", "Py")
s.startswith("Py"), "view" in s
"  x  ".strip()
s.find("x")           # -1 if absent; s.index raises ValueError

name, score = "Lenin", 92.456
f"{name} scored {score:.2f}"     # 'Lenin scored 92.46'
f"{score=}"                      # 'score=92.456' (debug, 3.8+)
f"{1234567:,}"                   # '1,234,567'
f"{'left':<10}|{'right':>10}"

# Encoding
"café".encode("utf-8")           # b'caf\xc3\xa9'
b"caf\xc3\xa9".decode("utf-8")   # 'café'
```

!!! warning "String concatenation in loops"
    `s += x` inside a loop can be O(n²). Collect pieces in a list and `"".join(parts)` — O(n).

## Lists

Dynamic arrays of references. Over-allocated so `append` is **amortized O(1)**.

```python
nums = [5, 3, 8]
nums.append(1)          # O(1)
nums.extend([9, 7])     # O(k)
nums.insert(0, 100)     # O(n)  shifts elements
nums.pop()              # O(1)
nums.pop(0)             # O(n)  -> use collections.deque for queues
nums.remove(8)          # O(n)  first occurrence
nums.index(3)           # O(n)
3 in nums               # O(n)
nums.sort()             # in place, Timsort O(n log n), stable
sorted(nums, reverse=True)   # returns new list
nums.sort(key=lambda x: -x)

# Copying
import copy
a = [[1, 2], [3]]
shallow = a.copy()          # or a[:] / list(a)
deep = copy.deepcopy(a)
a[0].append(99)
shallow[0]   # [1, 2, 99]  shares inner lists
deep[0]      # [1, 2]

# Classic trap
grid = [[0] * 3] * 3        # 3 references to the SAME row!
grid[0][0] = 1              # [[1,0,0],[1,0,0],[1,0,0]]
grid = [[0] * 3 for _ in range(3)]   # correct
```

## Tuples

Immutable, hashable (if contents are hashable), slightly smaller and faster than lists. Use for fixed records.

```python
point = (3, 4)
x, y = point                       # unpacking
single = (1,)                      # comma makes the tuple
first, *rest = (1, 2, 3, 4)        # first=1, rest=[2,3,4]
a, b = b, a                        # swap via tuple packing

from typing import NamedTuple
class Point(NamedTuple):
    x: int
    y: int
p = Point(1, 2); p.x               # readable, immutable record
```

## Dictionaries

Hash tables. **Insertion-ordered** since Python 3.7 (language guarantee). Average O(1) get/set/delete.

```python
user = {"name": "Lenin", "role": "Lead"}
user["city"] = "Chennai"
user.get("age", 0)             # default instead of KeyError
user.setdefault("skills", []).append("Python")
user.pop("role")
for k, v in user.items(): ...
{**user, "role": "Staff"}      # merge (copy)
user | {"remote": True}        # merge operator 3.9+

squares = {n: n * n for n in range(5)}   # dict comprehension

from collections import defaultdict, Counter, OrderedDict
graph = defaultdict(list)
graph["a"].append("b")         # no KeyError

Counter("mississippi").most_common(2)   # [('i', 4), ('s', 4)]
```

**How dict works:** key → `hash(key)` → index into a sparse table; collisions resolved by **open addressing** (probing). Keys must be hashable and their hash must not change. If two objects are `==`, their hashes **must** be equal.

## Sets

Unordered collections of unique hashable items — a dict with only keys. O(1) membership.

```python
a, b = {1, 2, 3}, {3, 4}
a | b    # union        {1,2,3,4}
a & b    # intersection {3}
a - b    # difference   {1,2}
a ^ b    # symmetric    {1,2,4}
empty = set()          # {} is an empty dict!
frozenset([1, 2])      # immutable, hashable set

# Dedupe preserving order
list(dict.fromkeys([3, 1, 3, 2, 1]))   # [3, 1, 2]
```

## Complexity of built-ins (memorise)

| Operation | list | dict / set | deque |
|---|---|---|---|
| index / get | O(1) | O(1) avg | O(n) |
| append / add | O(1)* | O(1) avg | O(1) both ends |
| insert/pop at front | O(n) | — | O(1) |
| `x in` | O(n) | O(1) avg | O(n) |
| delete | O(n) | O(1) avg | O(n) middle |
| sort | O(n log n) | — | — |

\*amortized

## Comprehensions & generator expressions

```python
evens = [n for n in range(10) if n % 2 == 0]
pairs = [(x, y) for x in range(3) for y in range(3) if x != y]
flat  = [x for row in [[1, 2], [3]] for x in row]
labels = ["even" if n % 2 == 0 else "odd" for n in range(4)]

total = sum(n * n for n in range(10**6))   # generator: O(1) memory
```

## Control flow essentials

```python
# for/else: else runs if loop did NOT break
for n in [3, 5, 7]:
    if n % 2 == 0:
        break
else:
    print("no even numbers")

# Walrus operator (3.8+)
if (n := len(data := [1, 2, 3])) > 2:
    print(n)

# Structural pattern matching (3.10+)
def handle(cmd):
    match cmd:
        case {"action": "move", "x": x, "y": y}:
            return f"move to {x},{y}"
        case ["quit" | "exit"]:
            return "bye"
        case int(code) if code >= 500:
            return "server error"
        case _:
            return "unknown"

# Truthiness: falsy = None, False, 0, 0.0, "", [], {}, set(), range(0)
```

## Useful built-ins

```python
enumerate(["a", "b"], start=1)        # (1,'a'), (2,'b')
zip([1, 2], ["a", "b"], strict=True)  # 3.10+: error on length mismatch
any(x > 5 for x in nums); all(...)
min(users, key=lambda u: u["age"])
map(str, [1, 2]); filter(None, [0, 1, ""])
reversed(seq); sorted(d.items(), key=lambda kv: kv[1])
isinstance(x, (int, float))
```

## Exceptions

```python
class InsufficientFunds(Exception):
    """Domain-specific error."""

def withdraw(balance, amount):
    if amount > balance:
        raise InsufficientFunds(f"need {amount - balance} more")
    return balance - amount

try:
    withdraw(100, 150)
except InsufficientFunds as e:
    print("Error:", e)
except (ValueError, TypeError):
    ...
else:
    print("runs only if no exception")
finally:
    print("always runs - cleanup")

# Chaining preserves root cause
try:
    int("x")
except ValueError as e:
    raise RuntimeError("bad config") from e

# 3.11+: ExceptionGroup / except*
```

**Hierarchy:** `BaseException` → (`SystemExit`, `KeyboardInterrupt`, `GeneratorExit`, `Exception` → everything else). Catch `Exception`, never bare `except:` (it swallows Ctrl-C).

**EAFP vs LBYL:** Python favours *Easier to Ask Forgiveness than Permission* (`try/except`) over *Look Before You Leap* (`if key in d`) — it avoids race conditions and is idiomatic.

## Modules & packages

- **Module** = a `.py` file. **Package** = directory of modules (with `__init__.py`; namespace packages don't need it).
- `if __name__ == "__main__":` — code runs only when executed directly, not on import.
- Imports are cached in `sys.modules`; a module body runs **once**.
- Search order: `sys.path` (script dir, `PYTHONPATH`, site-packages).
- Circular imports: fix by moving shared code to a third module or importing inside functions.

!!! success "Say this in the interview"
    "Python variables are references to objects. Assignment binds names, it never copies. Mutability decides whether an operation changes the object in place or creates a new one, which is why lists can't be dict keys and why mutable default arguments are a bug."
