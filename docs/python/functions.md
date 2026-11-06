# Functions & Scope

## Functions are first-class objects

They can be assigned, passed, returned and stored — this is the foundation of decorators and callbacks.

```python
def shout(text): return text.upper()
yell = shout
def apply(fn, value): return fn(value)
apply(yell, "hi")         # 'HI'
ops = {"up": str.upper, "low": str.lower}
ops["up"]("x")
```

## Parameter kinds

```python
def f(pos_only, /, normal, *args, kw_only, **kwargs):
    print(pos_only, normal, args, kw_only, kwargs)

f(1, 2, 3, 4, kw_only=5, extra=6)
# 1 2 (3, 4) 5 {'extra': 6}
```

| Syntax | Meaning |
|---|---|
| `/` | params before it are positional-only (3.8+) |
| `*args` | extra positional args → tuple |
| `*` alone | params after it are keyword-only |
| `**kwargs` | extra keyword args → dict |

```python
def connect(host, *, timeout=5, retries=3): ...
connect("db", timeout=10)        # keyword-only = self-documenting calls

nums = [1, 2, 3]
print(*nums)                     # unpack into args
cfg = {"timeout": 1}
connect("db", **cfg)
```

## The mutable default argument trap

Defaults are evaluated **once, at definition time**.

```python
def add_item(item, bucket=[]):      # BUG
    bucket.append(item)
    return bucket

add_item(1)   # [1]
add_item(2)   # [1, 2]  <- shared list!

def add_item(item, bucket=None):    # FIX
    if bucket is None:
        bucket = []
    bucket.append(item)
    return bucket
```

## Scope: LEGB

Name lookup order: **L**ocal → **E**nclosing → **G**lobal → **B**uilt-in.

```python
x = "global"
def outer():
    x = "enclosing"
    def inner():
        x = "local"
        print(x)
    inner()
outer()            # local

counter = 0
def inc():
    global counter     # rebind module-level name
    counter += 1

def make_counter():
    count = 0
    def inc():
        nonlocal count  # rebind enclosing name
        count += 1
        return count
    return inc
c = make_counter(); c(); c()   # 2
```

!!! warning "UnboundLocalError"
    Assigning to a name anywhere in a function makes it local for the **whole** function:
    ```python
    x = 1
    def f():
        print(x)   # UnboundLocalError
        x = 2
    ```

## Closures

A closure is an inner function that **remembers variables from the enclosing scope** after the outer function returns.

```python
def multiplier(n):
    def times(x):
        return x * n
    return times

double = multiplier(2)
double(5)                     # 10
double.__closure__[0].cell_contents   # 2

# Late-binding trap
funcs = [lambda: i for i in range(3)]
[f() for f in funcs]          # [2, 2, 2]
funcs = [lambda i=i: i for i in range(3)]
[f() for f in funcs]          # [0, 1, 2]
```

## Lambdas

Anonymous single-expression functions. Use for short `key=` callbacks; use `def` for anything named or reused.

```python
users = [{"n": "a", "age": 30}, {"n": "b", "age": 25}]
sorted(users, key=lambda u: u["age"])
from operator import itemgetter, attrgetter
sorted(users, key=itemgetter("age"))   # faster, clearer
```

## Functional tools

```python
from functools import reduce, partial, lru_cache, cache, wraps, singledispatch

reduce(lambda acc, x: acc * x, [1, 2, 3, 4], 1)    # 24

int_base2 = partial(int, base=2)
int_base2("1010")                                  # 10

@lru_cache(maxsize=None)          # memoization
def fib(n): return n if n < 2 else fib(n - 1) + fib(n - 2)
fib(100); fib.cache_info()

@singledispatch                   # function overloading by type
def to_json(obj): raise TypeError
@to_json.register
def _(obj: int): return str(obj)
@to_json.register
def _(obj: list): return "[" + ",".join(map(to_json, obj)) + "]"
to_json([1, 2])                   # '[1,2]'
```

```python
import itertools as it
list(it.chain([1, 2], [3]))               # [1,2,3]
list(it.islice(it.count(10), 3))          # [10,11,12]
list(it.permutations("ab"))               # [('a','b'),('b','a')]
list(it.combinations([1, 2, 3], 2))
list(it.product([0, 1], repeat=2))
for key, grp in it.groupby(sorted(words, key=len), key=len): ...
list(it.accumulate([1, 2, 3]))            # [1,3,6]
list(it.batched(range(7), 3))             # 3.12+: (0,1,2),(3,4,5),(6,)
```

## Recursion

```python
def factorial(n: int) -> int:
    if n <= 1:          # base case
        return 1
    return n * factorial(n - 1)
```

- Default recursion limit ≈ 1000 (`sys.getrecursionlimit()`).
- Python has **no tail-call optimization** — convert deep recursion to iteration with an explicit stack.

## Docstrings & annotations

```python
def area(radius: float) -> float:
    """Return the area of a circle.

    Args:
        radius: circle radius, must be >= 0.
    """
    return 3.14159 * radius ** 2

area.__doc__; area.__annotations__
```

!!! success "Say this in the interview"
    "A closure captures variables, not values — they're looked up when the inner function runs. That's why lambdas in a loop all see the last value, and why `nonlocal` exists for rebinding. Decorators are just closures that wrap functions."
