# Object-Oriented Programming

## The four pillars

| Pillar | Meaning | Python mechanism |
|---|---|---|
| **Encapsulation** | Bundle data + behaviour, hide internals | `_protected`, `__private` (name mangling), `@property` |
| **Abstraction** | Expose *what*, hide *how* | `abc.ABC`, `@abstractmethod`, `Protocol` |
| **Inheritance** | Reuse & specialise behaviour | `class Child(Parent)`, `super()` |
| **Polymorphism** | Same interface, different behaviour | Method overriding, duck typing, dunder methods |

## Classes and instances

```python
class Account:
    bank = "Optimum Bank"            # class attribute (shared)

    def __init__(self, owner: str, balance: float = 0):
        self.owner = owner            # instance attributes
        self._balance = balance       # "protected" by convention

    def deposit(self, amount: float) -> None:      # instance method
        if amount <= 0:
            raise ValueError("amount must be positive")
        self._balance += amount

    @property
    def balance(self) -> float:      # read-only computed attribute
        return self._balance

    @classmethod
    def from_csv(cls, row: str) -> "Account":     # alternative constructor
        owner, bal = row.split(",")
        return cls(owner, float(bal))

    @staticmethod
    def is_valid_iban(iban: str) -> bool:          # utility, no self/cls
        return iban.startswith("DE") and len(iban) == 22

    def __repr__(self) -> str:       # for developers
        return f"Account(owner={self.owner!r}, balance={self._balance})"

    def __str__(self) -> str:        # for users
        return f"{self.owner}: ₹{self._balance:,.2f}"

acc = Account.from_csv("Lenin,1000")
acc.deposit(500)
acc.balance           # 1500.0
acc.balance = 0       # AttributeError: no setter
```

| Method type | First arg | Use for |
|---|---|---|
| instance | `self` | behaviour using instance state |
| `@classmethod` | `cls` | factory/alternative constructors; works with subclasses |
| `@staticmethod` | none | utility logically grouped with the class |

## Properties with validation

```python
class Temperature:
    def __init__(self, celsius: float):
        self.celsius = celsius        # goes through the setter

    @property
    def celsius(self):
        return self._c

    @celsius.setter
    def celsius(self, value):
        if value < -273.15:
            raise ValueError("below absolute zero")
        self._c = value

    @property
    def fahrenheit(self):
        return self._c * 9 / 5 + 32
```

## Name mangling

```python
class A:
    def __init__(self):
        self.__secret = 1    # stored as _A__secret
a = A()
a._A__secret                # 1  — not truly private, just avoids subclass clashes
```

## Inheritance, `super()` and MRO

```python
class Animal:
    def __init__(self, name): self.name = name
    def speak(self): raise NotImplementedError

class Dog(Animal):
    def __init__(self, name, breed):
        super().__init__(name)
        self.breed = breed
    def speak(self): return f"{self.name}: Woof"

class Cat(Animal):
    def speak(self): return f"{self.name}: Meow"

for pet in (Dog("Rex", "Lab"), Cat("Tom")):
    print(pet.speak())        # polymorphism
```

### Multiple inheritance & the diamond problem

Python resolves methods with the **C3 linearization MRO**. `super()` calls the *next class in the MRO*, not necessarily the parent.

```python
class Base:
    def hello(self): print("Base")
class Left(Base):
    def hello(self): print("Left"); super().hello()
class Right(Base):
    def hello(self): print("Right"); super().hello()
class Child(Left, Right):
    def hello(self): print("Child"); super().hello()

Child().hello()   # Child, Left, Right, Base  (Base only once!)
Child.__mro__     # (Child, Left, Right, Base, object)
```

### Mixins

Small classes adding one capability, combined via multiple inheritance.

```python
import json
class JsonMixin:
    def to_json(self): return json.dumps(self.__dict__)

class User(JsonMixin):
    def __init__(self, name): self.name = name
User("Lenin").to_json()       # '{"name": "Lenin"}'
```

!!! tip "Composition over inheritance"
    Prefer *has-a* over *is-a* when behaviour varies independently. Inheritance couples you to the parent's implementation.
    ```python
    class Engine:
        def start(self): ...

    class Car:
        def __init__(self, engine: Engine): self.engine = engine   # composition
    ```

## Abstract base classes

```python
from abc import ABC, abstractmethod

class PaymentGateway(ABC):
    @abstractmethod
    def charge(self, amount: int) -> str: ...

    def refund(self, txn_id: str) -> None:    # concrete shared method
        print("refunding", txn_id)

class Stripe(PaymentGateway):
    def charge(self, amount): return "txn_123"

PaymentGateway()   # TypeError: can't instantiate abstract class
Stripe().charge(100)
```

## Duck typing & Protocols (structural typing)

"If it walks like a duck…" — Python cares about **behaviour**, not type.

```python
from typing import Protocol

class Closeable(Protocol):
    def close(self) -> None: ...

def shutdown(resource: Closeable) -> None:
    resource.close()          # any object with .close() type-checks

class File:
    def close(self): print("closed")
shutdown(File())              # no inheritance needed
```

## Dunder (magic) methods — operator overloading

```python
from functools import total_ordering

@total_ordering
class Money:
    def __init__(self, amount: int, currency="INR"):
        self.amount, self.currency = amount, currency
    def __add__(self, other):
        if not isinstance(other, Money): return NotImplemented
        return Money(self.amount + other.amount, self.currency)
    def __eq__(self, other): return (self.amount, self.currency) == (other.amount, other.currency)
    def __lt__(self, other): return self.amount < other.amount
    def __hash__(self): return hash((self.amount, self.currency))
    def __bool__(self): return self.amount != 0
    def __repr__(self): return f"Money({self.amount}, {self.currency!r})"

Money(5) + Money(10)        # Money(15, 'INR')
Money(5) <= Money(10)       # True (total_ordering fills the rest)
```

| Category | Dunders |
|---|---|
| Construction | `__new__`, `__init__`, `__del__` |
| Representation | `__repr__`, `__str__`, `__format__` |
| Comparison | `__eq__`, `__lt__`, `__le__`, `__gt__`, `__ge__`, `__hash__` |
| Arithmetic | `__add__`, `__sub__`, `__mul__`, `__radd__`, `__iadd__` … |
| Containers | `__len__`, `__getitem__`, `__setitem__`, `__contains__`, `__iter__` |
| Callable | `__call__` |
| Context manager | `__enter__`, `__exit__` |
| Attribute access | `__getattr__`, `__getattribute__`, `__setattr__` |

!!! warning "`__eq__` and `__hash__`"
    Defining `__eq__` sets `__hash__ = None` (unhashable) unless you also define `__hash__`. Equal objects must have equal hashes.

### `__new__` vs `__init__`

`__new__` **creates** the instance (static, returns object); `__init__` **initialises** it. Override `__new__` for immutable subclasses or singletons.

```python
class Singleton:
    _instance = None
    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
Singleton() is Singleton()   # True
```

### `__getattr__` vs `__getattribute__`

`__getattribute__` runs on **every** attribute access; `__getattr__` only when normal lookup **fails** — great for proxies.

```python
class Proxy:
    def __init__(self, target): self._t = target
    def __getattr__(self, name):
        print("forwarding", name)
        return getattr(self._t, name)
Proxy([1, 2]).append(3)
```

## Dataclasses

Auto-generate `__init__`, `__repr__`, `__eq__` (and optionally ordering, hashing, immutability).

```python
from dataclasses import dataclass, field, asdict

@dataclass(frozen=True, slots=True, order=True)
class Order:
    order_id: int
    amount: float
    tags: tuple[str, ...] = ()
    status: str = field(default="NEW", compare=False)

    def __post_init__(self):
        if self.amount < 0:
            raise ValueError("negative amount")

o = Order(1, 99.5)
asdict(o)              # {'order_id': 1, 'amount': 99.5, 'tags': (), 'status': 'NEW'}
# For lists: tags: list[str] = field(default_factory=list)
```

**dataclass vs NamedTuple vs Pydantic:**

| | dataclass | NamedTuple | Pydantic `BaseModel` |
|---|---|---|---|
| Mutable | yes (unless `frozen`) | no | yes |
| Runtime validation | no | no | **yes** (coercion + errors) |
| Use for | internal domain objects | tiny immutable records | API boundaries, config |

## `__slots__`

Replaces per-instance `__dict__` with a fixed array → **less memory, faster access**, no dynamic attributes.

```python
class Point:
    __slots__ = ("x", "y")
    def __init__(self, x, y): self.x, self.y = x, y
p = Point(1, 2)
p.z = 3        # AttributeError
```

Use when creating millions of small objects.

!!! success "Say this in the interview"
    "I use inheritance for genuine is-a relationships and ABCs/Protocols to define contracts, but default to composition. In Python, `super()` follows the MRO computed by C3 linearization, which is what makes cooperative multiple inheritance and mixins work."
