# Design Patterns in Python

Patterns are **named solutions to recurring design problems** (Gang of Four, 1994). In Python many are simpler because functions are first-class and duck typing replaces interfaces — say so in interviews.

| Creational | Structural | Behavioural |
|---|---|---|
| Singleton, Factory Method, Abstract Factory, Builder, Prototype | Adapter, Decorator, Facade, Proxy, Composite, Bridge, Flyweight | Strategy, Observer, Command, State, Template Method, Iterator, Chain of Responsibility, Mediator, Memento, Visitor |

---

## Creational

### Singleton

*Exactly one instance, global access point.* Use for config, connection pools, loggers — sparingly (global state hurts testing).

```python
# Pythonic: a module is already a singleton
# config.py
settings = Settings()          # import config; config.settings

# Class-based, thread-safe
import threading
class ConnectionPool:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:          # double-checked locking
                    cls._instance = super().__new__(cls)
                    cls._instance.connections = []
        return cls._instance

assert ConnectionPool() is ConnectionPool()

# Or: functools.cache on a factory
from functools import cache
@cache
def get_settings(): return Settings()
```

### Factory Method / Simple Factory

*Create objects without the caller knowing the concrete class.*

```python
from abc import ABC, abstractmethod

class Notification(ABC):
    @abstractmethod
    def send(self, to: str, msg: str) -> None: ...

class Email(Notification):
    def send(self, to, msg): print(f"email {to}: {msg}")
class Sms(Notification):
    def send(self, to, msg): print(f"sms {to}: {msg}")
class Push(Notification):
    def send(self, to, msg): print(f"push {to}: {msg}")

class NotificationFactory:
    _registry: dict[str, type[Notification]] = {"email": Email, "sms": Sms, "push": Push}

    @classmethod
    def register(cls, kind: str, klass: type[Notification]) -> None:
        cls._registry[kind] = klass               # open/closed: extend without editing

    @classmethod
    def create(cls, kind: str) -> Notification:
        try:
            return cls._registry[kind]()
        except KeyError:
            raise ValueError(f"unknown channel {kind}") from None

NotificationFactory.create("sms").send("+91...", "OTP 1234")
```

### Abstract Factory

*Create families of related objects that must be used together.*

```python
class UIFactory(ABC):
    @abstractmethod
    def button(self): ...
    @abstractmethod
    def checkbox(self): ...

class DarkFactory(UIFactory):
    def button(self): return "DarkButton"
    def checkbox(self): return "DarkCheckbox"
class LightFactory(UIFactory):
    def button(self): return "LightButton"
    def checkbox(self): return "LightCheckbox"

def render(factory: UIFactory):
    return factory.button(), factory.checkbox()     # always a consistent family
```

Real example: SQLAlchemy dialects (Postgres vs MySQL families of compilers/types).

### Builder

*Construct complex objects step by step.*

```python
class QueryBuilder:
    def __init__(self, table: str):
        self._table, self._where, self._order, self._limit = table, [], None, None
    def where(self, cond: str):
        self._where.append(cond); return self       # fluent interface
    def order_by(self, col: str):
        self._order = col; return self
    def limit(self, n: int):
        self._limit = n; return self
    def build(self) -> str:
        sql = f"SELECT * FROM {self._table}"
        if self._where: sql += " WHERE " + " AND ".join(self._where)
        if self._order: sql += f" ORDER BY {self._order}"
        if self._limit: sql += f" LIMIT {self._limit}"
        return sql

QueryBuilder("orders").where("status = 'paid'").order_by("created_at DESC").limit(10).build()
```

In Python, keyword arguments with defaults (or dataclasses) often replace builders for simple objects.

### Prototype

*Clone existing objects instead of building from scratch.*

```python
import copy
template = {"theme": "dark", "widgets": ["clock", "weather"]}
user_dash = copy.deepcopy(template)
user_dash["widgets"].append("stocks")
```

---

## Structural

### Adapter

*Make an incompatible interface work with the one your code expects.*

```python
from typing import Protocol

class PaymentGateway(Protocol):
    def charge(self, amount_paise: int) -> str: ...

class LegacyPayPalSdk:                       # third-party, can't change
    def make_payment(self, dollars: float, currency: str) -> dict:
        return {"transaction": "PP-123"}

class PayPalAdapter:
    def __init__(self, sdk: LegacyPayPalSdk, fx_rate: float):
        self.sdk, self.fx = sdk, fx_rate
    def charge(self, amount_paise: int) -> str:
        result = self.sdk.make_payment(amount_paise / 100 / self.fx, "USD")
        return result["transaction"]

def checkout(gateway: PaymentGateway): return gateway.charge(50_000)
checkout(PayPalAdapter(LegacyPayPalSdk(), fx_rate=83.0))
```

### Decorator (structural pattern)

*Add behaviour to an object dynamically by wrapping it with the same interface.* Python's `@decorator` syntax is the function-level version.

```python
from typing import Protocol

class DataSource(Protocol):
    def read(self) -> str: ...

class FileSource:
    def read(self): return "secret data"

class EncryptedSource:
    def __init__(self, inner: DataSource): self.inner = inner
    def read(self): return self.inner.read()[::-1]        # pretend decryption

class LoggingSource:
    def __init__(self, inner: DataSource): self.inner = inner
    def read(self):
        print("reading..."); return self.inner.read()

LoggingSource(EncryptedSource(FileSource())).read()      # stackable
```

### Facade

*A simple interface over a complex subsystem.*

```python
class CheckoutFacade:
    def __init__(self, inventory, payments, shipping, notifier):
        self.inventory, self.payments = inventory, payments
        self.shipping, self.notifier = shipping, notifier

    def place_order(self, cart, user):
        self.inventory.reserve(cart)
        txn = self.payments.charge(user, cart.total)
        tracking = self.shipping.schedule(cart, user.address)
        self.notifier.confirm(user, tracking)
        return txn
```

### Proxy

*A stand-in that controls access: lazy loading, caching, access control, remote calls, rate limiting.*

```python
from typing import Protocol

class ImageService(Protocol):
    def get(self, key: str) -> bytes: ...

class S3ImageService:
    def get(self, key): print("downloading", key); return b"..."

class CachingProxy:
    def __init__(self, real: ImageService):
        self.real, self.cache = real, {}
    def get(self, key):
        if key not in self.cache:
            self.cache[key] = self.real.get(key)
        return self.cache[key]
```

Decorator vs Proxy: decorator **adds** behaviour (often stacked); proxy **controls access** to the same behaviour.

### Composite

*Treat individual objects and groups uniformly (trees).*

```python
class Node(ABC):
    @abstractmethod
    def size(self) -> int: ...

class File(Node):
    def __init__(self, size): self._size = size
    def size(self): return self._size

class Folder(Node):
    def __init__(self, *children: Node): self.children = list(children)
    def size(self): return sum(c.size() for c in self.children)

Folder(File(10), Folder(File(5), File(5))).size()    # 20
```

### Bridge & Flyweight (know the idea)

- **Bridge** — split abstraction from implementation so both vary independently (`Shape` × `Renderer`).
- **Flyweight** — share immutable intrinsic state among many objects to save memory (string interning, `__slots__`, cached glyphs).

---

## Behavioural

### Strategy

*Swap algorithms at runtime.* In Python a strategy is often **just a function**.

```python
from typing import Callable

def no_discount(total: float) -> float: return total
def festive(total: float) -> float: return total * 0.8
def loyalty(total: float) -> float: return total - 100 if total > 1000 else total

class Checkout:
    def __init__(self, pricing: Callable[[float], float] = no_discount):
        self.pricing = pricing
    def total(self, amount: float) -> float:
        return self.pricing(amount)

Checkout(festive).total(2000)     # 1600.0
STRATEGIES = {"festive": festive, "loyalty": loyalty}   # pick by config
```

### Observer (pub/sub)

*Subscribers are notified when a subject changes.*

```python
from collections import defaultdict
from typing import Callable

class EventBus:
    def __init__(self):
        self._subs: dict[str, list[Callable]] = defaultdict(list)
    def subscribe(self, event: str, handler: Callable) -> None:
        self._subs[event].append(handler)
    def publish(self, event: str, payload: dict) -> None:
        for handler in self._subs[event]:
            handler(payload)

bus = EventBus()
bus.subscribe("order.paid", lambda p: print("email receipt", p["id"]))
bus.subscribe("order.paid", lambda p: print("update analytics", p["id"]))
bus.publish("order.paid", {"id": 42})
```

Distributed version: Kafka/RabbitMQ/SNS topics, Django signals, webhooks.

### Command

*Encapsulate a request as an object → queue, log, undo, retry.*

```python
class Command(ABC):
    @abstractmethod
    def execute(self): ...
    @abstractmethod
    def undo(self): ...

class AddText(Command):
    def __init__(self, doc: list, text: str): self.doc, self.text = doc, text
    def execute(self): self.doc.append(self.text)
    def undo(self): self.doc.pop()

class Editor:
    def __init__(self): self.doc, self.history = [], []
    def run(self, cmd: Command):
        cmd.execute(); self.history.append(cmd)
    def undo(self):
        if self.history: self.history.pop().undo()

ed = Editor(); ed.run(AddText(ed.doc, "Hello")); ed.run(AddText(ed.doc, "World")); ed.undo()
ed.doc   # ['Hello']
```

Celery tasks are commands serialized onto a queue.

### State

*An object changes behaviour when its internal state changes — replaces big if/elif on status.*

```python
class OrderState(ABC):
    def pay(self, order): raise ValueError(f"cannot pay in {self}")
    def ship(self, order): raise ValueError(f"cannot ship in {self}")
    def cancel(self, order): raise ValueError(f"cannot cancel in {self}")
    def __str__(self): return type(self).__name__

class Pending(OrderState):
    def pay(self, order): order.state = Paid()
    def cancel(self, order): order.state = Cancelled()
class Paid(OrderState):
    def ship(self, order): order.state = Shipped()
    def cancel(self, order): order.state = Cancelled()   # + refund
class Shipped(OrderState): ...
class Cancelled(OrderState): ...

class Order:
    def __init__(self): self.state: OrderState = Pending()
    def pay(self): self.state.pay(self)
    def ship(self): self.state.ship(self)
    def cancel(self): self.state.cancel(self)

o = Order(); o.pay(); o.ship(); print(o.state)   # Shipped
o.cancel()                                       # ValueError: cannot cancel in Shipped
```

### Template Method

*Base class defines the skeleton; subclasses fill in steps.*

```python
class DataExporter(ABC):
    def export(self, rows):                  # template method
        data = self.transform(rows)
        payload = self.serialize(data)
        self.write(payload)

    def transform(self, rows): return rows   # hook with default
    @abstractmethod
    def serialize(self, data) -> str: ...
    def write(self, payload): print(payload)

class CsvExporter(DataExporter):
    def serialize(self, data): return "\n".join(",".join(map(str, r)) for r in data)

class JsonExporter(DataExporter):
    def serialize(self, data):
        import json; return json.dumps(data)
```

### Chain of Responsibility

*Pass a request along a chain of handlers until one handles it (middleware!).*

```python
from typing import Callable
Handler = Callable[[dict], dict]

def auth(next_: Handler) -> Handler:
    def h(req):
        if not req.get("user"): return {"status": 401}
        return next_(req)
    return h

def rate_limit(next_: Handler) -> Handler:
    def h(req):
        if req.get("calls", 0) > 100: return {"status": 429}
        return next_(req)
    return h

def endpoint(req): return {"status": 200, "body": "ok"}

app = auth(rate_limit(endpoint))
app({"user": "lenin", "calls": 3})     # {'status': 200, ...}
```

### Iterator

Built into Python via `__iter__`/`__next__` and generators — see [Advanced Python](../python/advanced.md#iterators-and-the-iteration-protocol).

### Mediator, Memento, Visitor (know the idea)

- **Mediator** — objects talk via a central coordinator instead of each other (chat room, workflow orchestrator).
- **Memento** — capture and restore an object's state (undo snapshots).
- **Visitor** — add operations over an object structure without changing its classes (AST walkers: `ast.NodeVisitor`).

---

## Architectural & enterprise patterns

| Pattern | One-liner |
|---|---|
| **MVC / MVT** | Separate model, view, controller (Django: Model-View-Template) |
| **Repository** | Collection-like interface over persistence (see ORM page) |
| **Unit of Work** | Track changes and commit them atomically (SQLAlchemy Session) |
| **Service layer** | Application use cases orchestrating domain + repositories |
| **Dependency Injection** | Supply dependencies from outside (FastAPI `Depends`) |
| **CQRS** | Separate write model (commands) from read model (queries), often different stores |
| **Event sourcing** | Store events as source of truth; rebuild state by replaying |
| **Saga** | Distributed transaction via local steps + compensations |
| **Outbox** | Reliable event publishing alongside DB writes |
| **Circuit breaker** | Stop calling a failing dependency, fail fast, probe to recover |
| **Retry with backoff + jitter** | Handle transient failures without stampedes |
| **Bulkhead** | Isolate resources so one failure doesn't sink everything |
| **API Gateway / BFF** | Single entry point / backend tailored per frontend |
| **Strangler fig** | Incrementally replace a legacy system behind a facade |
| **Sidecar** | Helper process alongside a service (Envoy, log shipper) |

### Circuit breaker example

```python
import time

class CircuitBreaker:
    def __init__(self, failure_threshold=5, reset_timeout=30):
        self.failures, self.threshold, self.timeout = 0, failure_threshold, reset_timeout
        self.state, self.opened_at = "CLOSED", 0.0

    def call(self, fn, *args, **kwargs):
        if self.state == "OPEN":
            if time.monotonic() - self.opened_at < self.timeout:
                raise RuntimeError("circuit open: failing fast")
            self.state = "HALF_OPEN"                  # allow a trial request
        try:
            result = fn(*args, **kwargs)
        except Exception:
            self.failures += 1
            if self.state == "HALF_OPEN" or self.failures >= self.threshold:
                self.state, self.opened_at = "OPEN", time.monotonic()
            raise
        self.failures, self.state = 0, "CLOSED"
        return result
```

## Anti-patterns to name

God object · spaghetti code · golden hammer · premature optimization · copy-paste programming · magic numbers/strings · singleton abuse (hidden global state) · anemic domain model · distributed monolith.

!!! success "Say this in the interview"
    "I reach for patterns when they remove a real pain: Strategy (often just a function) to replace if/elif chains, Factory with a registry so new types don't touch existing code, Adapter to wrap third-party SDKs behind my own interface, State for order lifecycles, and Observer/events to decouple side effects. In Python I avoid heavyweight class hierarchies where a function or module does the job."
