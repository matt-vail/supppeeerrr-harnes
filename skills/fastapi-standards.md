---
name: fastapi-standards
description: >
  FastAPI-specific standards, patterns, and anti-patterns for backend engineers.
  Use when FastAPI is detected in the project dependencies (pyproject.toml,
  requirements.txt, or equivalent). Covers Pydantic request/response model
  design, dependency injection with Depends(), routing structure, async vs sync
  handler selection, N+1 in async context, error handling, and Python typing
  patterns specific to FastAPI. Supplements — does not replace — the core REST
  standards in the backend-engineer agent definition.
---

# Skill: fastapi-standards

FastAPI standards for the backend engineer. These apply in addition to the core standards in `agents/backend-engineer.md`, not instead of them.

---

## Pydantic models

Every request body, response body, and query parameter group is a Pydantic model. Never use raw `dict`.

```python
# Wrong
@router.post("/users")
async def create_user(body: dict) -> dict:
    ...

# Right
class CreateUserRequest(BaseModel):
    email: EmailStr
    name: str = Field(..., min_length=1, max_length=100)

class UserResponse(BaseModel):
    id: UUID
    email: str
    name: str

    model_config = ConfigDict(from_attributes=True)

@router.post("/users", response_model=UserResponse, status_code=201)
async def create_user(body: CreateUserRequest) -> UserResponse:
    ...
```

**Rules:**
- Request models and response models are separate classes — never reuse the same model for both.
- Use `Field()` for validation constraints — min/max length, regex, gt/lt for numbers.
- Use `EmailStr`, `UUID`, `HttpUrl` from `pydantic` over plain `str` where the type carries meaning.
- `model_config = ConfigDict(from_attributes=True)` on response models that are built from ORM objects.
- Never expose internal fields (passwords, internal IDs) in a response model — use explicit field selection.

### Validators
```python
from pydantic import field_validator, model_validator

class CreateUserRequest(BaseModel):
    password: str
    confirm_password: str

    @model_validator(mode="after")
    def passwords_match(self) -> "CreateUserRequest":
        if self.password != self.confirm_password:
            raise ValueError("Passwords do not match")
        return self
```

Use `@field_validator` for single-field validation. Use `@model_validator` for cross-field validation. Do not put business logic in validators — keep them to type and format checks.

---

## Dependency injection

`Depends()` is the correct mechanism for anything shared across endpoints: auth, database sessions, config, rate limiting. Never instantiate these inside handler functions.

```python
# Wrong — creates a new DB session with no lifecycle management
@router.get("/users/{id}")
async def get_user(user_id: UUID):
    db = SessionLocal()
    ...

# Right — session lifecycle managed by the dependency
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session

@router.get("/users/{id}")
async def get_user(
    user_id: UUID,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
) -> UserResponse:
    ...
```

**Common dependencies to always use `Depends()` for:**
- Database session
- Current authenticated user
- Permissions check
- Config / settings
- Rate limiter
- Request ID / trace context

**Dependency scoping:**
- Router-level: `router = APIRouter(dependencies=[Depends(require_auth)])` — applies to every route in the router.
- App-level: `app = FastAPI(dependencies=[Depends(verify_api_key)])` — applies globally.
- Route-level: only when the dependency is specific to that one endpoint.

---

## Routing and structure

```python
# router structure
router = APIRouter(
    prefix="/users",
    tags=["users"],
    responses={404: {"description": "Not found"}},
)
```

**Rules:**
- Every router has a `prefix` and `tags`. Tags group endpoints in the generated docs.
- Route paths use nouns, not verbs. `/users` not `/getUsers`.
- Group routes by domain, not by HTTP method. One file per domain (`users.py`, `orders.py`).
- Keep handler functions thin — validate input, call a service, return a response. Business logic belongs in a service layer, not in the route handler.

```python
# Wrong — business logic in the handler
@router.post("/users")
async def create_user(body: CreateUserRequest, db: AsyncSession = Depends(get_db)):
    existing = await db.execute(select(User).where(User.email == body.email))
    if existing.scalar():
        raise HTTPException(status_code=409, detail="Email already registered")
    user = User(email=body.email, name=body.name)
    db.add(user)
    await db.commit()
    return user

# Right — handler delegates to service
@router.post("/users", response_model=UserResponse, status_code=201)
async def create_user(
    body: CreateUserRequest,
    db: AsyncSession = Depends(get_db),
) -> UserResponse:
    return await user_service.create(db, body)
```

---

## Async patterns

**Use async for I/O-bound handlers.** Use sync for CPU-bound handlers (FastAPI runs sync handlers in a thread pool automatically).

```python
# I/O bound — use async
@router.get("/users/{id}")
async def get_user(user_id: UUID, db: AsyncSession = Depends(get_db)):
    ...

# CPU bound — sync is correct, FastAPI handles thread pool
@router.post("/process")
def process_image(file: UploadFile):
    # heavy CPU computation
    ...
```

**Never call blocking code inside an async handler:**
```python
# Wrong — blocks the event loop
@router.get("/data")
async def get_data():
    time.sleep(1)           # blocks entire server
    result = requests.get(url)  # blocking HTTP call

# Right
@router.get("/data")
async def get_data():
    await asyncio.sleep(1)
    async with httpx.AsyncClient() as client:
        result = await client.get(url)
```

**Blocking calls that must run in async context:**
```python
import asyncio
result = await asyncio.to_thread(blocking_function, arg1, arg2)
```

**N+1 in async context:**
```python
# Wrong — N sequential awaits
users = await get_all_users(db)
for user in users:
    user.profile = await get_profile(db, user.id)  # N round trips

# Right — gather for independent concurrent calls
profiles = await asyncio.gather(*[get_profile(db, u.id) for u in users])
# Better — batch query
profiles = await get_profiles_batch(db, [u.id for u in users])
```

---

## Error handling

```python
from fastapi import HTTPException
from fastapi.responses import JSONResponse

# Standard HTTP errors
raise HTTPException(status_code=404, detail="User not found")
raise HTTPException(status_code=409, detail="Email already registered")
raise HTTPException(status_code=422, detail=[{"loc": ["body", "email"], "msg": "invalid"}])

# Custom exception handler
@app.exception_handler(DomainError)
async def domain_error_handler(request: Request, exc: DomainError) -> JSONResponse:
    return JSONResponse(status_code=400, content={"detail": str(exc)})
```

**Rules:**
- Use `HTTPException` for expected errors (not found, conflict, validation).
- Use exception handlers for domain errors — do not let domain exceptions propagate to the client raw.
- Never return `500` for a client error. 4xx for client errors, 5xx for server errors.
- Error detail must not leak internal state, stack traces, or database errors.

---

## Python typing in FastAPI context

FastAPI uses type annotations to drive validation, serialisation, and docs generation. Type annotations are not optional here — they are functional.

```python
from typing import Annotated
from fastapi import Depends, Query

# Annotated combines type with metadata — preferred in FastAPI 0.95+
async def get_users(
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
    offset: Annotated[int, Query(ge=0)] = 0,
    db: Annotated[AsyncSession, Depends(get_db)],
) -> list[UserResponse]:
    ...
```

**Generic response patterns:**
```python
from typing import Generic, TypeVar

T = TypeVar("T")

class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    limit: int
    offset: int

# Usage
@router.get("/users", response_model=PaginatedResponse[UserResponse])
async def list_users(...) -> PaginatedResponse[UserResponse]:
    ...
```

---

## Anti-patterns — always flag these

| Anti-pattern | Why | Fix |
|-------------|-----|-----|
| Raw `dict` as request/response | No validation, no docs, no type safety | Pydantic model |
| Business logic in route handler | Untestable, violates SRP | Service layer |
| `SessionLocal()` inside handler | No lifecycle management, connection leaks | `Depends(get_db)` |
| Blocking call in async handler | Blocks entire event loop | `asyncio.to_thread` or async equivalent |
| Sequential `await` for independent calls | Unnecessary latency — N round trips | `asyncio.gather` |
| Catching all exceptions silently | Hides real errors | Catch specific exceptions |
| Secrets in route parameters or query strings | Logged by default | Header or body |
| `response_model=None` on sensitive endpoints | Leaks internal fields | Explicit response model |
