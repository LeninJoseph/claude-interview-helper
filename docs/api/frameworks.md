# FastAPI & Django REST Framework

## WSGI vs ASGI

| | WSGI | ASGI |
|---|---|---|
| Model | Synchronous, one request per worker thread | Async, many requests per worker via event loop |
| Protocols | HTTP only | HTTP, WebSocket, HTTP/2, lifespan events |
| Servers | Gunicorn, uWSGI | Uvicorn, Hypercorn, Daphne |
| Frameworks | Flask, Django (classic) | FastAPI, Starlette, Django (async views) |

Production: `gunicorn -k uvicorn.workers.UvicornWorker -w 4 app.main:app` (or `uvicorn --workers 4`). Rule of thumb for workers: `2 × CPU cores + 1` for sync; fewer for async.

## FastAPI: a production-shaped example

```text
app/
├── main.py            # app factory, routers, middleware
├── api/routes/orders.py
├── schemas/order.py   # Pydantic models (request/response)
├── models/order.py    # SQLAlchemy ORM models
├── services/orders.py # business logic
├── repositories/      # data access
├── core/config.py     # settings
└── db/session.py
```

### Schemas with Pydantic v2

```python
from datetime import datetime
from decimal import Decimal
from enum import StrEnum
from pydantic import BaseModel, Field, field_validator, ConfigDict

class OrderStatus(StrEnum):
    PENDING = "pending"
    PAID = "paid"

class OrderCreate(BaseModel):
    product_id: int = Field(gt=0)
    quantity: int = Field(ge=1, le=100)
    coupon: str | None = None

    @field_validator("coupon")
    @classmethod
    def upper(cls, v): return v.upper() if v else v

class OrderOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)   # read from ORM objects
    id: int
    status: OrderStatus
    total: Decimal
    created_at: datetime
```

Separate **input** and **output** schemas: clients must never be able to set `id`, `status` or `total`, and you never leak internal fields like `password_hash`.

### Routes, dependency injection, status codes

```python
from fastapi import FastAPI, APIRouter, Depends, HTTPException, status, Query, Header, BackgroundTasks
from sqlalchemy.ext.asyncio import AsyncSession

router = APIRouter(prefix="/api/v1/orders", tags=["orders"])

async def get_db():                              # dependency with cleanup
    async with SessionLocal() as session:
        yield session

async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    user = await decode_token(token)
    if not user:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "invalid token",
                            headers={"WWW-Authenticate": "Bearer"})
    return user

@router.post("", response_model=OrderOut, status_code=status.HTTP_201_CREATED)
async def create_order(
    body: OrderCreate,
    background: BackgroundTasks,
    db: AsyncSession = Depends(get_db),
    user: User = Depends(get_current_user),
    idempotency_key: str | None = Header(default=None),
):
    order = await order_service.create(db, user, body, idempotency_key)
    background.add_task(send_confirmation_email, user.email, order.id)
    return order

@router.get("", response_model=list[OrderOut])
async def list_orders(
    status_: OrderStatus | None = Query(None, alias="status"),
    limit: int = Query(20, le=100),
    cursor: int | None = None,
    db: AsyncSession = Depends(get_db),
    user: User = Depends(get_current_user),
):
    return await order_service.list(db, user.id, status_, limit, cursor)

@router.get("/{order_id}", response_model=OrderOut)
async def get_order(order_id: int, db: AsyncSession = Depends(get_db),
                    user: User = Depends(get_current_user)):
    order = await db.get(Order, order_id)
    if not order or order.user_id != user.id:      # don't leak existence
        raise HTTPException(404, "order not found")
    return order
```

**Dependency injection** = FastAPI resolves `Depends(...)` per request, caches within the request, runs cleanup after `yield`. Great for DB sessions, auth, settings, pagination params — and you can override them in tests: `app.dependency_overrides[get_db] = fake_db`.

### App setup: lifespan, middleware, error handlers

```python
from contextlib import asynccontextmanager
import time, uuid
from fastapi import Request
from fastapi.responses import JSONResponse
from fastapi.middleware.cors import CORSMiddleware

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.redis = await create_redis()       # startup
    yield
    await app.state.redis.close()                # shutdown

app = FastAPI(title="Orders API", version="1.0", lifespan=lifespan)
app.include_router(router)
app.add_middleware(CORSMiddleware, allow_origins=["https://shop.example.com"],
                   allow_methods=["*"], allow_headers=["*"], allow_credentials=True)

@app.middleware("http")
async def request_context(request: Request, call_next):
    request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
    start = time.perf_counter()
    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    response.headers["X-Process-Time"] = f"{time.perf_counter() - start:.4f}"
    return response

class DomainError(Exception):
    def __init__(self, code: int, title: str, detail: str):
        self.code, self.title, self.detail = code, title, detail

@app.exception_handler(DomainError)
async def domain_error_handler(request: Request, exc: DomainError):
    return JSONResponse(status_code=exc.code,
                        content={"title": exc.title, "detail": exc.detail, "status": exc.code})

@app.get("/health")
async def health(): return {"status": "ok"}
```

### Settings

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")
    database_url: str
    jwt_secret: str
    debug: bool = False
settings = Settings()       # reads env vars; never hard-code secrets
```

### `async def` vs `def` endpoints

- `async def` → runs on the event loop. **Never call blocking code inside.**
- `def` → FastAPI runs it in a **threadpool** automatically — safe for blocking libraries (e.g. sync SQLAlchemy, `requests`).

### Streaming (LLM-style) responses

```python
from fastapi.responses import StreamingResponse
async def token_stream():
    for tok in ["Hello", " ", "world"]:
        yield f"data: {tok}\n\n"
        await asyncio.sleep(0.1)
@app.get("/chat")
async def chat(): return StreamingResponse(token_stream(), media_type="text/event-stream")
```

### WebSockets

```python
from fastapi import WebSocket, WebSocketDisconnect
@app.websocket("/ws")
async def ws(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            msg = await websocket.receive_text()
            await websocket.send_text(f"echo: {msg}")
    except WebSocketDisconnect:
        pass
```

### Background work: BackgroundTasks vs Celery

| | `BackgroundTasks` | Celery / RQ / Arq |
|---|---|---|
| Runs in | Same process after response | Separate worker processes |
| Durability | Lost if process dies | Persisted in broker (Redis/RabbitMQ) |
| Retries / scheduling | No | Yes |
| Use for | Fire-and-forget email, logs | Payments, reports, anything that must happen |

## Django REST Framework (DRF)

```python
# models.py
class Order(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name="orders")
    product = models.ForeignKey(Product, on_delete=models.PROTECT)
    quantity = models.PositiveIntegerField()
    status = models.CharField(max_length=20, default="pending")
    created_at = models.DateTimeField(auto_now_add=True)

# serializers.py
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ["id", "product", "quantity", "status", "created_at"]
        read_only_fields = ["status", "created_at"]

    def validate_quantity(self, value):
        if value > 100:
            raise serializers.ValidationError("max 100")
        return value

# views.py
class OrderViewSet(viewsets.ModelViewSet):
    serializer_class = OrderSerializer
    permission_classes = [permissions.IsAuthenticated]
    filterset_fields = ["status"]
    pagination_class = CursorPagination

    def get_queryset(self):                          # tenant isolation + avoid N+1
        return Order.objects.filter(user=self.request.user).select_related("product")

    def perform_create(self, serializer):
        serializer.save(user=self.request.user)

    @action(detail=True, methods=["post"])
    def cancel(self, request, pk=None):
        order = self.get_object()
        order.status = "cancelled"; order.save()
        return Response({"status": "cancelled"})

# urls.py
router = DefaultRouter()
router.register("orders", OrderViewSet, basename="order")
urlpatterns = [path("api/v1/", include(router.urls))]
```

**DRF building blocks:** Serializers (validation + (de)serialization) · Views: `APIView` → Generic views → `ViewSet` · Routers · Authentication classes (Session, Token, JWT via simplejwt) · Permission classes · Throttling · Filtering · Pagination.

## FastAPI vs Django vs Flask

| | FastAPI | Django + DRF | Flask |
|---|---|---|---|
| Style | Async-first micro-framework | Batteries-included | Minimal micro-framework |
| Validation | Pydantic (type hints) | Serializers | Extensions (marshmallow) |
| ORM | Bring your own (SQLAlchemy, SQLModel) | Built-in Django ORM | Bring your own |
| Docs | Auto OpenAPI/Swagger | drf-spectacular | Extensions |
| Admin/Auth | DIY | Built-in admin, auth, migrations | DIY |
| Best for | High-performance APIs, ML/LLM services | Full products, admin-heavy apps | Small services |

!!! success "Say this in the interview"
    "In FastAPI I keep routes thin: Pydantic schemas validate the boundary, dependencies provide the DB session and current user, a service layer holds business logic, and repositories handle data access. That makes everything testable via dependency overrides. Anything that must reliably happen after the response goes to a durable queue, not BackgroundTasks."
