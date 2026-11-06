# Python Q&A Bank

Click a question to reveal the answer. Practise answering out loud first.

## Core

??? question "1. What's the difference between a list and a tuple?"
    Lists are mutable dynamic arrays; tuples are immutable and hashable (if contents are), slightly smaller and faster to create. Use tuples for fixed records and dict keys, lists for collections that change.

??? question "2. Is Python pass-by-value or pass-by-reference?"
    Neither — it's **pass-by-object-reference** (call by sharing). The function receives a reference to the same object; mutating it is visible to the caller, rebinding the parameter is not.

??? question "3. `is` vs `==`?"
    `==` compares values via `__eq__`; `is` compares identity. Use `is` only for `None`/singletons.

??? question "4. What is the mutable default argument problem?"
    Default values are evaluated once at function definition, so a default list/dict is shared between calls. Use `None` and create the object inside.

??? question "5. What is hashable? Why can't a list be a dict key?"
    An object is hashable if it has a hash that never changes during its lifetime and supports `==`. Lists are mutable, so their hash would change → not hashable.

??? question "6. How does a Python dict work internally?"
    A hash table with open addressing. `hash(key)` picks a slot; collisions probe other slots. A separate compact array preserves insertion order. Average O(1), worst O(n). Resizes when ~2/3 full.

??? question "7. Shallow vs deep copy?"
    Shallow copy creates a new container holding references to the same elements (`list.copy()`, `copy.copy`). Deep copy recursively copies everything (`copy.deepcopy`).

??? question "8. What are `*args` and `**kwargs`?"
    `*args` collects extra positional arguments into a tuple; `**kwargs` collects extra keyword arguments into a dict. Also used to unpack when calling.

??? question "9. Explain LEGB."
    Name resolution order: Local, Enclosing, Global, Built-in. `global` and `nonlocal` let you rebind names in outer scopes.

??? question "10. What is a closure?"
    An inner function that retains access to variables from its enclosing scope after the outer function has returned.

??? question "11. What are list comprehensions vs generator expressions?"
    `[...]` builds the whole list in memory; `(...)` creates a lazy generator yielding one item at a time — O(1) memory, single-pass.

??? question "12. What does `if __name__ == '__main__'` do?"
    Runs the block only when the file is executed directly, not when imported.

??? question "13. Difference between `append` and `extend`?"
    `append(x)` adds x as one element; `extend(iterable)` adds each element of the iterable.

??? question "14. How do you reverse a string / list?"
    `s[::-1]`; `lst.reverse()` (in place) or `reversed(lst)` (iterator).

??? question "15. What's the difference between `remove`, `pop` and `del`?"
    `remove(value)` deletes first matching value; `pop(i)` removes by index and returns it; `del lst[i]` / `del lst[a:b]` deletes by index/slice without returning.

## OOP

??? question "16. `@staticmethod` vs `@classmethod`?"
    Classmethod receives `cls` — used for alternative constructors and works polymorphically with subclasses. Staticmethod receives nothing — a plain function namespaced in the class.

??? question "17. What is the MRO?"
    Method Resolution Order — the linearised order Python searches classes for attributes, computed with C3 linearization. See `Class.__mro__`. `super()` follows it.

??? question "18. `__new__` vs `__init__`?"
    `__new__` creates and returns the instance (used for immutables, singletons); `__init__` initialises the already-created instance and returns None.

??? question "19. `__str__` vs `__repr__`?"
    `__repr__` is unambiguous for developers (ideally valid code to recreate the object); `__str__` is readable for end users. `print` uses `__str__`, falling back to `__repr__`.

??? question "20. What is duck typing?"
    Using an object based on the methods/behaviour it supports rather than its type. Formalised for type checkers with `typing.Protocol`.

??? question "21. How do you make an attribute private?"
    Python has no true private. `_name` signals internal; `__name` triggers name mangling to `_Class__name` to avoid subclass collisions. Use `@property` to control access.

??? question "22. What are dataclasses and why use them?"
    A decorator generating `__init__`, `__repr__`, `__eq__` (optionally ordering, hashing, frozen, slots) from type-annotated fields — less boilerplate for data-holding classes.

??? question "23. What is `__slots__`?"
    Declares a fixed set of attributes, removing the per-instance `__dict__` — less memory and faster attribute access, but no dynamic attributes.

??? question "24. Abstract class vs interface in Python?"
    `abc.ABC` with `@abstractmethod` = abstract class (nominal, can include implementation). `Protocol` = interface by structure (no inheritance required).

## Advanced

??? question "25. What is a decorator? Write one."
    A callable taking a function and returning a replacement. Always use `functools.wraps`. See the timer/retry examples on the Advanced page.

??? question "26. Iterator vs iterable vs generator?"
    Iterable has `__iter__`; iterator has `__next__` and `__iter__` returning self; a generator is an iterator created by a function with `yield` (or a generator expression).

??? question "27. What does `yield from` do?"
    Delegates to a sub-generator, forwarding `send`/`throw` and capturing its return value.

??? question "28. How do context managers work?"
    `with obj as x` calls `obj.__enter__()` (result bound to x) and guarantees `obj.__exit__(exc_type, exc, tb)` runs; returning True from `__exit__` suppresses the exception. `@contextmanager` builds one from a generator.

??? question "29. What are descriptors?"
    Objects implementing `__get__`/`__set__`/`__delete__` stored as class attributes, controlling attribute access. `property`, methods, classmethod and ORM fields are descriptors.

??? question "30. What is a metaclass?"
    The class of a class (default `type`); it customises class creation. Used by ORMs and ABCs. `__init_subclass__` is usually a simpler alternative.

??? question "31. What is monkey patching?"
    Changing classes/modules at runtime (e.g. replacing a method). Useful in tests (`monkeypatch`), dangerous in production code.

## Internals & concurrency

??? question "32. What is the GIL? Does it make code thread-safe?"
    A lock in CPython letting one thread execute bytecode at a time; it protects interpreter internals, not your logic. `x += 1` is not atomic — use locks.

??? question "33. How is memory managed in Python?"
    Reference counting frees objects immediately at zero refs; a generational cyclic GC collects reference cycles; pymalloc handles small allocations.

??? question "34. Threads vs processes vs asyncio?"
    Threads: I/O-bound with blocking libraries. Processes: CPU-bound (bypass GIL). asyncio: high-concurrency I/O with async libraries on one thread.

??? question "35. What happens if you call a blocking function inside async code?"
    It blocks the whole event loop — every other coroutine stalls. Use an async library or `await asyncio.to_thread(fn)`.

??? question "36. What's the difference between `asyncio.gather` and `TaskGroup`?"
    Both run concurrently. `TaskGroup` (3.11+) provides structured concurrency: if one task fails, the rest are cancelled and errors raised as an `ExceptionGroup`.

??? question "37. How do you find a memory leak?"
    `tracemalloc` snapshots and compare, `objgraph` for growth of types, `gc.get_referrers`, check unbounded caches/globals/listeners.

??? question "38. How do you profile slow Python code?"
    `cProfile` + `snakeviz` for function-level, `line_profiler` for lines, `py-spy` for production sampling without code changes.

## Coding mini-tasks

??? question "39. Flatten a nested list of arbitrary depth"
    ```python
    def flatten(xs):
        for x in xs:
            if isinstance(x, list):
                yield from flatten(x)
            else:
                yield x
    list(flatten([1, [2, [3, [4]]]]))   # [1, 2, 3, 4]
    ```

??? question "40. Count word frequency and return top 3"
    ```python
    from collections import Counter
    Counter(text.lower().split()).most_common(3)
    ```

??? question "41. Check if two strings are anagrams"
    ```python
    from collections import Counter
    def is_anagram(a, b): return Counter(a) == Counter(b)
    ```

??? question "42. Remove duplicates preserving order"
    ```python
    list(dict.fromkeys(items))
    ```

??? question "43. Merge two dicts, summing values for common keys"
    ```python
    from collections import Counter
    dict(Counter(a) + Counter(b))
    ```

??? question "44. Implement an LRU cache"
    ```python
    from collections import OrderedDict
    class LRU:
        def __init__(self, cap): self.cap, self.d = cap, OrderedDict()
        def get(self, k):
            if k not in self.d: return -1
            self.d.move_to_end(k); return self.d[k]
        def put(self, k, v):
            self.d[k] = v; self.d.move_to_end(k)
            if len(self.d) > self.cap: self.d.popitem(last=False)
    ```

??? question "45. Write a singleton"
    Module-level instance is the most Pythonic. Otherwise override `__new__` (see OOP page) or use a decorator caching the instance.

??? question "46. Chunk a list into size n"
    ```python
    def chunks(xs, n): return [xs[i:i + n] for i in range(0, len(xs), n)]
    # 3.12+: itertools.batched(xs, n)
    ```

??? question "47. Group a list of dicts by key"
    ```python
    from collections import defaultdict
    groups = defaultdict(list)
    for row in rows: groups[row["dept"]].append(row)
    ```

??? question "48. Read a 10GB file and count lines with 'ERROR'"
    ```python
    with open("app.log") as f:
        count = sum(1 for line in f if "ERROR" in line)   # streams, O(1) memory
    ```

??? question "49. What's the output?  `print([i * j for i in range(3) for j in range(2)])`"
    `[0, 0, 0, 1, 0, 2]` — the left `for` is the outer loop.

??? question "50. What's the output?  `a = [1, 2]; b = a; a = a + [3]; print(b)`"
    `[1, 2]` — `a + [3]` creates a new list and rebinds `a`; `b` still references the original. (With `a += [3]` it would print `[1, 2, 3]`.)
