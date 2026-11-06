# ORM (SQLAlchemy 2.0 & Django ORM)

An **Object-Relational Mapper** maps tables ↔ classes and rows ↔ objects.

| Pros | Cons |
|---|---|
| Less boilerplate, DB-agnostic | Hides SQL → N+1 and inefficient queries |
| Safe parameterization | Complex reports are awkward |
| Migrations, relationships, identity map | Abstraction leaks under load |

Use the ORM for CRUD and drop to raw SQL / Core for heavy reporting.

## SQLAlchemy 2.0 models

```python
from datetime import datetime
from decimal import Decimal
from sqlalchemy import ForeignKey, String, Numeric, func, select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class Customer(Base):
    __tablename__ = "customers"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    orders: Mapped[list["Order"]] = relationship(back_populates="customer",
                                                 cascade="all, delete-orphan")

class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    customer_id: Mapped[int] = mapped_column(ForeignKey("customers.id"), index=True)
    status: Mapped[str] = mapped_column(String(20), default="pending")
    total: Mapped[Decimal] = mapped_column(Numeric(12, 2))
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    version: Mapped[int] = mapped_column(default=1)
    customer: Mapped[Customer] = relationship(back_populates="orders")

    __mapper_args__ = {"version_id_col": version}     # optimistic locking built in
```

### Many-to-many

```python
from sqlalchemy import Table, Column
post_tags = Table("post_tags", Base.metadata,
    Column("post_id", ForeignKey("posts.id"), primary_key=True),
    Column("tag_id", ForeignKey("tags.id"), primary_key=True))

class Post(Base):
    __tablename__ = "posts"
    id: Mapped[int] = mapped_column(primary_key=True)
    tags: Mapped[list["Tag"]] = relationship(secondary=post_tags, back_populates="posts")

class Tag(Base):
    __tablename__ = "tags"
    id: Mapped[int] = mapped_column(primary_key=True)
    posts: Mapped[list[Post]] = relationship(secondary=post_tags, back_populates="tags")
```

## Sessions & the Unit of Work

The **Session** is a Unit of Work + Identity Map: it tracks changed objects and flushes them in one transaction on `commit()`.

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, selectinload, joinedload

engine = create_engine("postgresql+psycopg://user:pw@localhost/shop",
                       pool_size=10, max_overflow=20, pool_pre_ping=True)
SessionLocal = sessionmaker(engine, expire_on_commit=False)

with SessionLocal() as session, session.begin():          # commit or rollback automatically
    c = Customer(name="Lenin")
    c.orders.append(Order(total=Decimal("999.00")))
    session.add(c)

with SessionLocal() as session:
    stmt = (select(Order)
            .where(Order.status == "paid", Order.total > 500)
            .order_by(Order.created_at.desc())
            .limit(20))
    orders = session.scalars(stmt).all()

    # aggregates
    stmt = (select(Customer.name, func.count(Order.id), func.sum(Order.total))
            .join(Order).group_by(Customer.name))
    for name, n, total in session.execute(stmt): ...
```

Async: `create_async_engine("postgresql+asyncpg://...")`, `AsyncSession`, `await session.execute(...)`.

## The N+1 problem

```python
customers = session.scalars(select(Customer)).all()   # 1 query
for c in customers:
    print(len(c.orders))                              # +1 query per customer (lazy load)
```

**Fix with eager loading:**

```python
select(Customer).options(selectinload(Customer.orders))   # 2 queries: customers + WHERE IN (...)
select(Order).options(joinedload(Order.customer))         # 1 query with JOIN (good for many-to-one)
```

| Strategy | SQL | Best for |
|---|---|---|
| `lazy` (default) | Query on access | Rarely accessed relationships |
| `joinedload` | JOIN | many-to-one / one-to-one |
| `selectinload` | Second `IN` query | one-to-many / many-to-many collections |
| `raiseload` | Error on lazy access | Catch N+1 in development |

Detect N+1: `echo=True`, `django-debug-toolbar`, `nplusone`, APM traces.

## Django ORM equivalents

```python
Order.objects.filter(status="paid", total__gt=500).order_by("-created_at")[:20]
Order.objects.select_related("customer")                 # JOIN (FK / one-to-one)
Customer.objects.prefetch_related("orders")              # separate IN query (reverse FK / M2M)
Order.objects.values("status").annotate(n=Count("id"))   # GROUP BY
Customer.objects.annotate(spent=Sum("orders__total")).filter(spent__gt=10000)
Product.objects.filter(id=7).update(stock=F("stock") - 1)   # atomic, no race
Order.objects.filter(Q(status="paid") | Q(total__gt=1000))

from django.db import transaction
with transaction.atomic():
    acct = Account.objects.select_for_update().get(pk=1)
    acct.balance -= 100; acct.save()

Order.objects.only("id", "status")            # fewer columns
Order.objects.bulk_create([...], batch_size=1000)
qs.exists()                                   # cheaper than qs.count() > 0
qs.iterator(chunk_size=2000)                  # stream large querysets
```

**QuerySets are lazy** — no SQL runs until evaluated (iteration, `list()`, slicing with step, `len()`, `bool()`).

## Migrations

- **Alembic** (SQLAlchemy): `alembic revision --autogenerate -m "add status"` → `alembic upgrade head`.
- **Django**: `makemigrations` → `migrate`.
- **Zero-downtime migration rules:** add nullable column → backfill in batches → add constraint; never rename/drop in one step (expand → migrate → contract); create indexes `CONCURRENTLY`; avoid long table locks.

## Repository pattern over the ORM

```python
from typing import Protocol

class OrderRepository(Protocol):
    def get(self, order_id: int) -> Order | None: ...
    def add(self, order: Order) -> None: ...

class SqlOrderRepository:
    def __init__(self, session): self.session = session
    def get(self, order_id): return self.session.get(Order, order_id)
    def add(self, order): self.session.add(order)

class InMemoryOrderRepository:                # fast fake for unit tests
    def __init__(self): self.items = {}
    def get(self, order_id): return self.items.get(order_id)
    def add(self, order): self.items[order.id] = order
```

!!! success "Say this in the interview"
    "I use SQLAlchemy 2.0 with typed models, a session per request as the unit of work, and explicit eager loading — selectinload for collections, joinedload for many-to-one — with raiseload in tests to catch N+1. Business logic depends on a repository interface so unit tests use an in-memory fake, and migrations follow expand-and-contract for zero downtime."
