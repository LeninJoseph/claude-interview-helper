# Advanced Python

## Decorators

A decorator is a callable that **takes a function and returns a new function** — `@deco` is sugar for `func = deco(func)`.

```python
import functools, time

def timer(func):
    @functools.wraps(func)            # keep __name__, __doc__, etc.
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            print(f"{func.__name__} took {time.perf_counter() - start:.4f}s")
    return wrapper

@timer
def slow(n): time.sleep(n)
slow(0.1)
```

### Decorator with arguments (three layers)

```python
def retry(times: int = 3, exceptions=(Exception,), delay: float = 0.1):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == times:
                        raise
                    print(f"attempt {attempt} failed: {e}; retrying")
                    time.sleep(delay * 2 ** (attempt - 1))   # exponential backoff
        return wrapper
    return decorator

@retry(times=3, exceptions=(ConnectionError,))
def call_api(): ...
```

### Class-based decorator & stacking

```python
class CountCalls:
    def __init__(self, func):
        functools.update_wrapper(self, func)
        self.func, self.calls = func, 0
    def __call__(self, *args, **kwargs):
        self.calls += 1
        return self.func(*args, **kwargs)

@timer            # applied second (outer)
@CountCalls       # applied first (inner)
def hello(): return "hi"
```

Common real-world decorators: `@login_required`, `@app.get("/")`, `@pytest.fixture`, `@lru_cache`, `@property`, `@dataclass`, caching, auth, rate limiting, logging, transactions.

## Iterators and the iteration protocol

- **Iterable**: has `__iter__()` returning an iterator (list, dict, str, file).
- **Iterator**: has `__next__()` (raises `StopIteration` when done) and `__iter__()` returning itself. Single-pass.

```python
class Countdown:
    def __init__(self, start): self.current = start
    def __iter__(self): return self
    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

list(Countdown(3))    # [3, 2, 1]

it = iter([1, 2]); next(it); next(it); next(it, "done")   # default avoids StopIteration
```

`for x in obj` → calls `iter(obj)` then `next()` repeatedly until `StopIteration`.

## Generators

Functions using `yield` — they **pause and resume**, keeping local state. Lazy, memory-efficient (O(1) memory for infinite streams).

```python
def read_large_file(path):
    with open(path) as f:
        for line in f:
            yield line.rstrip("\n")

def fibonacci():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

from itertools import islice
list(islice(fibonacci(), 8))      # [0,1,1,2,3,5,8,13]

# Pipelines: each stage lazy
lines  = (l for l in read_large_file("app.log"))
errors = (l for l in lines if "ERROR" in l)
codes  = (l.split()[2] for l in errors)
```

### `yield from`, `send`, return value

```python
def sub():
    yield 1
    yield 2
    return "sub done"

def main():
    result = yield from sub()      # delegates, captures return value
    yield result

list(main())    # [1, 2, 'sub done']

def accumulator():
    total = 0
    while True:
        value = yield total        # receives sent value
        total += value
acc = accumulator(); next(acc)     # prime it
acc.send(10); acc.send(5)          # 15
```

**Generator vs list:** list materialises everything (fast random access, re-iterable); generator computes on demand (low memory, single-pass).

## Context managers

Guarantee setup/teardown with `with` — files, locks, DB transactions, connections.

```python
class DBTransaction:
    def __init__(self, conn): self.conn = conn
    def __enter__(self):
        self.conn.begin()
        return self.conn
    def __exit__(self, exc_type, exc, tb):
        if exc_type is None:
            self.conn.commit()
        else:
            self.conn.rollback()
        return False        # False -> re-raise exception; True -> suppress
```

```python
from contextlib import contextmanager, suppress, ExitStack
import time

@contextmanager
def timed(label):
    start = time.perf_counter()
    try:
        yield
    finally:
        print(f"{label}: {time.perf_counter() - start:.3f}s")

with timed("query"):
    sum(range(10**6))

with suppress(FileNotFoundError):
    open("missing.txt")

with ExitStack() as stack:          # dynamic number of resources
    files = [stack.enter_context(open(p)) for p in ["a.txt", "b.txt"] if False]
```

Async version: `async with` + `__aenter__/__aexit__` or `@asynccontextmanager`.

## Descriptors

An object defining `__get__`, `__set__` or `__delete__`, placed as a **class attribute**, controls attribute access on instances. `property`, `classmethod`, `staticmethod`, functions (bound methods) and ORM fields are all descriptors.

```python
class Positive:
    def __set_name__(self, owner, name):
        self.private = "_" + name
    def __get__(self, obj, objtype=None):
        if obj is None: return self
        return getattr(obj, self.private)
    def __set__(self, obj, value):
        if value <= 0:
            raise ValueError(f"{self.private[1:]} must be > 0")
        setattr(obj, self.private, value)

class Product:
    price = Positive()
    qty = Positive()
    def __init__(self, price, qty):
        self.price, self.qty = price, qty

Product(10, 2)
Product(-1, 2)    # ValueError: price must be > 0
```

- **Data descriptor** (`__set__` or `__delete__`) beats the instance `__dict__`.
- **Non-data descriptor** (only `__get__`, e.g. functions) is overridden by instance `__dict__`.

## Metaclasses

A metaclass is the **class of a class** — `type` by default. It controls class *creation*. "If you wonder whether you need one, you don't" — but know how they work.

```python
class RegistryMeta(type):
    registry = {}
    def __new__(mcs, name, bases, ns):
        cls = super().__new__(mcs, name, bases, ns)
        if bases:                         # skip the base class itself
            mcs.registry[name.lower()] = cls
        return cls

class Plugin(metaclass=RegistryMeta): ...
class CsvExporter(Plugin): ...
class PdfExporter(Plugin): ...
RegistryMeta.registry    # {'csvexporter': ..., 'pdfexporter': ...}

# type() creates classes dynamically
Dog = type("Dog", (object,), {"speak": lambda self: "woof"})
```

**Modern alternative:** `__init_subclass__` handles most registry/validation use cases without a metaclass.

```python
class Plugin:
    registry = {}
    def __init_subclass__(cls, key: str, **kw):
        super().__init_subclass__(**kw)
        Plugin.registry[key] = cls
class Csv(Plugin, key="csv"): ...
```

Real uses: Django models / SQLAlchemy declarative base, ABCMeta, enum.

## Enums

```python
from enum import Enum, IntEnum, StrEnum, auto
class Status(StrEnum):        # 3.11+
    PENDING = auto()
    PAID = auto()
Status.PAID == "paid"         # True
Status("pending")             # Status.PENDING
```

## Other must-knows

```python
# Sorting with multiple keys
sorted(emps, key=lambda e: (-e.salary, e.name))

# collections
from collections import deque, ChainMap, namedtuple
dq = deque(maxlen=3)          # sliding window / bounded buffer
dq.extend([1, 2, 3, 4])       # deque([2, 3, 4])

# heapq: min-heap
import heapq
heapq.nlargest(3, nums)

# bisect: binary search on sorted lists
import bisect
bisect.insort(sorted_list, 5)

# pathlib
from pathlib import Path
for p in Path(".").rglob("*.py"): ...

# Copy semantics, shallow vs deep — see Fundamentals
```

!!! success "Say this in the interview"
    "Generators give me lazy, constant-memory pipelines — I use them for streaming large files or paginated APIs. Context managers guarantee cleanup even on exceptions, which is how I wrap DB transactions. Descriptors are what power `property` and ORM fields; metaclasses control class creation, though `__init_subclass__` covers most needs today."
