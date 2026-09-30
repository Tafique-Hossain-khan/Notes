# Handlers, Services & Repositories, Middleware, and Request Context — Complete Notes

## 1. The Request Life Cycle Inside the Server

External to the server, a request simply travels client → server → client over HTTP. But *inside* the server, a single request passes through a well-defined internal life cycle:

```
Client → [OS forwards to listening port] → Entry point → Routing →
(optional middlewares) → Handler/Controller → Service → Repository →
back up through Service → back up through Handler → Response → Client
```

- **Entry point**: the moment the OS forwards an incoming HTTP request to the port your server is listening on (e.g. 3000, 8000).
- **Routing**: the request is matched to a specific handler function based on method + path (see routing notes).
- From there, the request is handed to the **Handler/Controller → Service → Repository** chain — a layered design pattern.

**Important:** this three-layer split is *not* mandatory — you could do everything in one giant handler function. It's a design pattern adopted purely for **scalability, maintainability, and easier debugging** as a codebase grows.

---

## 2. The Three Layers

### 2.1 Handler / Controller Layer

Every handler receives (by default, from the language/framework runtime) two objects:
- **Request object** — to read incoming data.
- **Response object** — to write outgoing data / set headers / status codes.

**Responsibilities of the Handler layer, in order:**

1. **Extract data from the request** — query params (GET), request body (POST/PUT/PATCH/DELETE as applicable).
2. **Deserialize / bind** the incoming JSON into the language's native data format (a struct in Go, a dict/class in Python, an object in Node — though Node often does this earlier via body-parsing middleware). This step is sometimes just called **binding**. If deserialization fails, respond immediately with **`400 Bad Request`** and stop — do not proceed further.
3. **Validate and transform** the now-native data:
   - *Validation*: ensure required fields are present, formats are correct, no malicious/garbage input, etc.
   - *Transformation*: convenience modifications after validation — e.g., **setting defaults** for optional query parameters (a good API design habit: make query parameters optional, and fill sensible defaults downstream, e.g. `sort=date` if the client sent nothing).
4. **Call the Service layer**, passing the validated/transformed data plus any context data it needs (e.g., authenticated user ID, role/permissions).
5. **Receive the result back from the Service layer** and decide the appropriate HTTP status code and response body:
   - Success → `2xx` (200, 201, 204, etc., depending on the operation).
   - Client error → `4xx`.
   - Server error → `5xx`.

**Key principle:** the Handler is responsible for everything *HTTP-related* — parsing requests, choosing status codes, shaping responses. It controls the flow of data from client to server and back.

### 2.2 Service Layer

- The service layer should have **no knowledge of HTTP at all**. Looking at a service method in isolation, you should not be able to tell it's being called from an API — it's just a plain function: takes data in, does processing, returns data (or throws/raises a domain-level error) out.
- Responsibilities:
  - Perform the actual business logic/processing.
  - Call one or more **repository methods** to fetch/persist data.
  - **Orchestrate**: combine results from multiple repository calls, merge fields, call external APIs, send emails/notifications, etc.
  - A single service method can do many things and call many repository methods.

### 2.3 Repository / Database Layer

- **Single responsibility**: take some data, construct and run a database query/operation, return the result. Nothing else.
- **Rule of thumb:** a repository method should do exactly *one kind* of database operation. Don't create a single method that conditionally returns "all books" or "one book" based on an optional parameter — write two separate methods (`get_all_books`, `get_book_by_id`).
- The repository doesn't know or care about HTTP, validation, or business rules beyond "fetch/insert/update/delete this data."

### 2.4 Why Separate These Layers?

Same reason you extract logic into functions in general: to avoid duplicating the same logic across the codebase, to keep responsibilities isolated (easier to test, debug, and reason about), and to make it easy to reuse a service or repository method across multiple handlers.

---

## 3. Middleware

### 3.1 What Is Middleware?

Middleware are functions that execute **in between** the fixed boundaries of the request life cycle — between the entry point and routing, between routing and the handler, and/or between the handler and the final response. They are **optional**: your request can skip straight from routing to the handler with zero middlewares if you want.

Like handlers, middleware functions receive the **request** and **response** objects — but they additionally receive a third argument: **`next`**, a function used to pass execution forward to the next middleware (or to the next stage — routing, handler, etc.) in the chain. If a middleware does *not* call `next`, and instead sends a response directly, the request life cycle terminates right there — it never reaches the handler.

### 3.2 Why Use Middleware?

Exactly the same reason you use functions instead of copy-pasting code: **to eliminate duplication.** A backend application typically has hundreds of endpoints; operations like security checks, logging, authentication, and rate limiting need to run for *every* request. Writing that logic into every single handler (even as a shared function you manually call each time) is still needless repetition and coupling. Middleware delegates these cross-cutting concerns to a dedicated layer that runs automatically for every request that passes through it.

### 3.3 Order Matters

Middlewares execute in a defined sequence, each calling `next` to hand off to the following one. **The order you register them in determines the order they run in** — and this has real consequences:

- **CORS** should run *very early* — if the request's origin isn't allowed, you want to reject/handle it before wasting further server resources.
- **Global error-handling middleware** should run **last** (i.e., wrap everything) — because an error can occur at any point in the chain (a handler, a service call, another middleware), and only a middleware that sits at the outermost boundary can reliably catch whatever bubbles up.

### 3.4 Common Middleware Examples

| Middleware | Purpose |
|---|---|
| **CORS** | Checks the request's `Origin` header against a list of allowed origins; if allowed, adds the appropriate CORS response headers so the browser doesn't block the response. |
| **Security headers** | Sets headers like Content-Security-Policy on every response. |
| **Authentication** | Extracts a token (JWT, session ID, etc.) from the request, verifies it. On failure → immediately respond `401 Unauthorized` and stop. On success → extract user ID/role/permissions and store them in the **request context** (see below), then call `next`. |
| **Rate limiting** | Tracks requests per client (commonly by IP) against a threshold over a time window; if exceeded, respond `429 Too Many Requests` and stop; otherwise call `next`. |
| **Logging / monitoring** | Records request metadata (method, path, query params, body, etc.) for debugging and auditing. |
| **Global error handling** | Catches unstructured errors raised anywhere in the pipeline (handler, service, other middleware), classifies them as client (4xx) or server (5xx) errors, and returns a consistently structured error response. Must be registered **last**. |
| **Compression** | Compresses large JSON responses (e.g., gzip) before sending; clients decompress automatically. |
| **Data (de)serialization** | Can optionally be delegated to middleware instead of being handled inside each handler, depending on your codebase's conventions. |

**Typical practical ordering:** CORS → logging → authentication → (app-specific middleware, e.g. permission checks) → handler chain → global error handling (outermost/last).

---

## 4. Request Context

### 4.1 What It Is

A **request context** is a storage/state object **scoped to a single request**, accessible across every middleware and handler that processes that request — without those components needing to be tightly coupled or pass values around as explicit function parameters everywhere.

Think of it as a key-value store that lives for exactly the duration of one request and is visible to every layer that touches that request.

### 4.2 Why It's Needed

Without a context, if your authentication middleware determines the user's ID and role, you'd have to somehow explicitly thread that information through every subsequent function call down to the handler and service layer — tightly coupling every layer together. A request context lets an earlier middleware set values that a later middleware or handler can simply read, with no direct coupling.

### 4.3 Common Uses

1. **Authentication data**: after successful auth, store `user_id`, `role`, `permissions`, etc. Downstream handlers read the user ID **from the context (i.e., from the verified token/session)** — never from client-supplied payload data — specifically to prevent a malicious client from supplying someone else's user ID to perform unauthorized operations.
2. **Request ID / correlation ID**: generate a UUID early in the pipeline, store it in the context, and use it for logging and for propagating via a header (e.g. `X-Request-ID`) to downstream microservices — this lets you trace a single request's full path across logs and services.
3. **Cancellation / deadline signals**: propagate abort/cancellation signals and timeouts to downstream calls so a request doesn't hang indefinitely waiting on a slow external dependency.

---

## 5. Practical Implementation — Python (FastAPI)

FastAPI has direct, idiomatic equivalents for every concept above: **Dependency Injection (`Depends`)** largely plays the role of "service resolution" and reusable pre-processing, **`request.state`** is the request context, and **`@app.middleware("http")`** (or Starlette `BaseHTTPMiddleware`) implements middleware with an explicit `call_next` — FastAPI's equivalent of `next`.

### 5.1 Project structure (handler / service / repository)

```
app/
├── main.py
├── routers/
│   └── books.py          # Handler / Controller layer
├── services/
│   └── book_service.py   # Service layer
├── repositories/
│   └── book_repository.py# Repository layer
├── schemas.py             # Pydantic models (validation)
└── middleware.py
```

### 5.2 Repository Layer — single-responsibility data access

```python
# repositories/book_repository.py

# In real code this wraps a DB client (SQLAlchemy, asyncpg, etc.)
_fake_books_db = [
    {"id": 1, "title": "Clean Code", "author_id": 10, "created_at": "2024-01-01"},
    {"id": 2, "title": "The Pragmatic Programmer", "author_id": 11, "created_at": "2024-02-01"},
]

class BookRepository:
    def get_all_books(self, sort_by: str) -> list[dict]:
        """Single responsibility: fetch ALL books, sorted. Nothing else."""
        return sorted(_fake_books_db, key=lambda b: b[sort_by if sort_by == "title" else "created_at"])

    def get_book_by_id(self, book_id: int) -> dict | None:
        """Single responsibility: fetch ONE book by id. Never returns a list."""
        return next((b for b in _fake_books_db if b["id"] == book_id), None)

    def insert_book(self, title: str, owner_id: int) -> dict:
        """Single responsibility: insert and return the created row."""
        new_book = {"id": len(_fake_books_db) + 1, "title": title, "author_id": owner_id, "created_at": "now"}
        _fake_books_db.append(new_book)
        return new_book
```

### 5.3 Service Layer — pure business logic, no HTTP knowledge

```python
# services/book_service.py
from repositories.book_repository import BookRepository

class BookService:
    def __init__(self, repository: BookRepository):
        self.repository = repository

    def list_books(self, sort_by: str) -> list[dict]:
        # Pure processing — this function has NO idea it's being called from an API.
        # It doesn't know about status codes, query params vs body, etc.
        books = self.repository.get_all_books(sort_by)
        return books

    def create_book(self, title: str, owner_id: int) -> dict:
        # Business rule example: could validate title uniqueness, send a
        # notification, call another service, etc. — all orchestration lives here.
        book = self.repository.insert_book(title=title, owner_id=owner_id)
        return book
```

### 5.4 Validation & Transformation via Pydantic (the "binding" step)

```python
# schemas.py
from pydantic import BaseModel, Field
from typing import Optional
from enum import Enum

class SortOption(str, Enum):
    title = "title"
    date = "date"

class BookCreateRequest(BaseModel):
    title: str = Field(min_length=1, max_length=200)
```

FastAPI automatically deserializes incoming JSON into these native Python objects, validates them, and returns `422 Unprocessable Entity` (the FastAPI equivalent of the "400 Bad Request on binding/validation failure" rule) if validation fails — you get steps 2 and 3 of the Handler responsibilities essentially "for free."

### 5.5 Handler / Controller Layer — FastAPI router

```python
# routers/books.py
from fastapi import APIRouter, Depends, HTTPException, Request, status
from schemas import BookCreateRequest, SortOption
from services.book_service import BookService
from repositories.book_repository import BookRepository

router = APIRouter(prefix="/api/books", tags=["books"])

def get_book_service() -> BookService:
    # Dependency injection wiring — Handler doesn't construct the service itself
    return BookService(repository=BookRepository())

@router.get("")
def list_books(
    sort: SortOption = SortOption.date,   # transformation: optional query param with a default
    service: BookService = Depends(get_book_service),
):
    # 1. FastAPI already extracted + validated the query param (steps 1-3 done via the type hint)
    # 2. Handler calls the Service layer
    books = service.list_books(sort_by=sort.value)
    # 3. Handler decides the response code (200 OK is the default for GET)
    return {"data": books}

@router.post("", status_code=status.HTTP_201_CREATED)
def create_book(
    payload: BookCreateRequest,          # binding + validation handled by Pydantic
    request: Request,
    service: BookService = Depends(get_book_service),
):
    # Pull the authenticated user's ID from the REQUEST CONTEXT (request.state),
    # never from client-supplied payload — this is the anti-spoofing pattern
    # described in section 4.3, point 1.
    user_id = request.state.user_id

    book = service.create_book(title=payload.title, owner_id=user_id)
    return {"data": book}
```

### 5.6 Middleware — CORS, Logging, Auth, Rate Limiting, Global Error Handling

FastAPI middleware signature: `async def middleware(request: Request, call_next)` — `call_next` is FastAPI's equivalent of `next`.

```python
# middleware.py
import time
import uuid
from fastapi import Request, HTTPException
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware

# ---- 1. Request ID middleware (populates the request context early) ----
class RequestIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        request_id = str(uuid.uuid4())
        request.state.request_id = request_id   # <-- writing to the request context
        response = await call_next(request)     # pass execution onward
        response.headers["X-Request-ID"] = request_id
        return response

# ---- 2. Logging middleware ----
class LoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start = time.monotonic()
        response = await call_next(request)
        duration_ms = (time.monotonic() - start) * 1000
        print(
            f"[{getattr(request.state, 'request_id', '-')}] "
            f"{request.method} {request.url.path} -> {response.status_code} ({duration_ms:.1f}ms)"
        )
        return response

# ---- 3. Authentication middleware ----
class AuthMiddleware(BaseHTTPMiddleware):
    PUBLIC_PATHS = {"/token", "/docs", "/openapi.json"}

    async def dispatch(self, request: Request, call_next):
        if request.url.path in self.PUBLIC_PATHS:
            return await call_next(request)

        token = request.headers.get("authorization", "").removeprefix("Bearer ").strip()
        if not token:
            # Terminate here — never reaches the handler. Equivalent to
            # "sending 401 directly from the middleware."
            return JSONResponse(status_code=401, content={"detail": "Authentication failed"})

        try:
            payload = verify_jwt(token)   # your JWT verification function (see auth notes)
        except Exception:
            return JSONResponse(status_code=401, content={"detail": "Authentication failed"})

        # Success: store identity info in the REQUEST CONTEXT for downstream handlers
        request.state.user_id = payload["sub"]
        request.state.user_role = payload.get("role", "user")

        return await call_next(request)

# ---- 4. Rate limiting middleware ----
_request_counts: dict[str, list[float]] = {}

class RateLimitMiddleware(BaseHTTPMiddleware):
    WINDOW_SECONDS = 2
    MAX_REQUESTS = 30

    async def dispatch(self, request: Request, call_next):
        client_ip = request.client.host
        now = time.monotonic()
        timestamps = _request_counts.setdefault(client_ip, [])
        # drop timestamps outside the window
        timestamps[:] = [t for t in timestamps if now - t < self.WINDOW_SECONDS]

        if len(timestamps) >= self.MAX_REQUESTS:
            return JSONResponse(status_code=429, content={"detail": "Too many requests"})

        timestamps.append(now)
        return await call_next(request)

# ---- 5. Global error-handling middleware (register LAST / outermost) ----
class GlobalErrorHandlingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        try:
            return await call_next(request)
        except HTTPException:
            raise  # let FastAPI's own handler deal with intentional HTTP errors
        except Exception as exc:
            # Anything unexpected anywhere downstream (handler, service, repo, other middleware)
            # ends up here, and gets converted into one consistently structured response.
            return JSONResponse(
                status_code=500,
                content={"error": {"code": "INTERNAL_ERROR", "message": "Something went wrong"}},
            )
```

### 5.7 Wiring Middleware Order in `main.py`

```python
# main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from routers import books
from middleware import (
    RequestIDMiddleware,
    LoggingMiddleware,
    AuthMiddleware,
    RateLimitMiddleware,
    GlobalErrorHandlingMiddleware,
)

app = FastAPI()

# NOTE on ordering in Starlette/FastAPI: middleware added LAST runs FIRST
# on the way in (and last on the way out) — it wraps everything added before it.
# So to get the logical order:
#   CORS -> RequestID -> Logging -> RateLimit -> Auth -> (routes) -> GlobalErrorHandling (outermost)
# we add GlobalErrorHandling FIRST so it wraps everything else.

app.add_middleware(GlobalErrorHandlingMiddleware)   # outermost: catches everything
app.add_middleware(AuthMiddleware)
app.add_middleware(RateLimitMiddleware)
app.add_middleware(LoggingMiddleware)
app.add_middleware(RequestIDMiddleware)
app.add_middleware(
    CORSMiddleware,                                  # runs first / rejects disallowed origins earliest
    allow_origins=["https://your-frontend.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(books.router)
```

### 5.8 Request Context Beyond `request.state` — `contextvars` for Deeper Call Chains

`request.state` works well when you have direct access to the `Request` object (in handlers and middleware). But sometimes a deeply nested service/repository function needs the request ID or user ID **without** the `Request` object being passed all the way down. Python's `contextvars` module is the standard tool for this — it's what many logging/tracing libraries use under the hood.

```python
# context.py
from contextvars import ContextVar

request_id_ctx: ContextVar[str] = ContextVar("request_id", default="-")
user_id_ctx: ContextVar[str] = ContextVar("user_id", default=None)

# In RequestIDMiddleware.dispatch(), after generating request_id:
#     request_id_ctx.set(request_id)
# In AuthMiddleware.dispatch(), after verifying the token:
#     user_id_ctx.set(payload["sub"])

# Now, deep inside book_service.py or book_repository.py, with NO request
# object in scope at all:
from context import request_id_ctx

def some_deep_repository_call():
    current_request_id = request_id_ctx.get()
    print(f"[{current_request_id}] running query...")
```

This is the practical mechanism behind "a request context accessible across every layer without tight coupling" — `request.state` is scoped to the request/middleware layer, while `contextvars` extends that same idea safely across `async` call chains, all the way into your service and repository layers.

---

## 6. Summary Cheat Sheet

| Layer / Concept | Responsibility | FastAPI Mechanism |
|---|---|---|
| Handler / Controller | Extract request data, bind/validate/transform, call service, choose response + status code | Route function (`@router.get/post/...`) + Pydantic models |
| Service | Pure business logic, orchestration, no HTTP knowledge | Plain Python class/functions, injected via `Depends` |
| Repository | Single-responsibility DB read/write | Plain Python class wrapping your DB client |
| Middleware | Cross-cutting concerns (CORS, auth, logging, rate limiting, errors) that run for every request, in a defined order | `BaseHTTPMiddleware` / `@app.middleware("http")`, added via `app.add_middleware()` |
| `next` | Pass execution to the next stage in the chain | `call_next(request)` |
| Request Context | Per-request shared state accessible across middleware + handlers without tight coupling | `request.state.*` (shallow) or `contextvars.ContextVar` (deep call chains) |

**Golden rules to remember:**
- 400/422 on bad input, decided at the Handler/binding layer.
- Never trust a client-supplied user ID — always read identity from the verified auth context.
- One repository method = one kind of database operation.
- Middleware order matters: security/CORS early, error handling as the outermost wrapper.
- Services should be testable in complete isolation from any HTTP concept.