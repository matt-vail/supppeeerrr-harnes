---
name: python-standards
description: >
  Python language standards, idioms, and anti-patterns for backend development.
  Covers PEP 8 style, type annotations, idiomatic patterns (context managers,
  comprehensions, generators, dataclasses), common pitfalls, and pytest
  conventions. Loaded by backend-engineer for all Python projects — alongside
  any active framework skill (fastapi-standards, django-standards,
  flask-standards), not as a replacement for them.
---

# Skill: python-standards

Python language standards for the backend engineer. These apply in addition to the core standards in `agents/backend-engineer.md`. When a framework-specific skill is also active (`fastapi-standards`, `django-standards`, `flask-standards`), these language standards remain in effect — the framework skill adds to them, does not replace them.

---

## Style and conventions

### Formatter and linter config

Before writing any Python, check for formatter and linter configuration files:

```
pyproject.toml     # [tool.ruff], [tool.black], [tool.isort], [tool.mypy]
setup.cfg          # [flake8], [mypy], [isort]
.flake8
.ruff.toml
mypy.ini
```

If a config exists, your output must pass it without modification. Do not introduce a second style. If no config exists (greenfield), establish one explicitly — document the choice and commit the config file before writing application code.

**Formatter precedence (if present):**
1. `ruff format` — preferred in new projects (fast, opinionated, replaces black + isort)
2. `black` + `isort` — common in existing projects
3. Whatever is already configured — match it

### Docstring style

Detect the docstring convention in use before writing any docstrings:

```python
# Google style
def create_user(email: str, name: str) -> User:
    """Create a new user record.

    Args:
        email: The user's email address. Must be unique.
        name: The user's display name.

    Returns:
        The newly created User instance.

    Raises:
        DuplicateEmailError: If the email is already registered.
    """

# NumPy style
def create_user(email: str, name: str) -> User:
    """Create a new user record.

    Parameters
    ----------
    email : str
        The user's email address. Must be unique.
    name : str
        The user's display name.

    Returns
    -------
    User
        The newly created User instance.
    """
```

Never mix docstring styles within a project. If no style is established, default to Google style in new Python projects and document the decision.

---

## Type annotations

Type annotations in Python are functional in FastAPI (validation), useful with mypy/pyright (correctness), and always serve as documentation. All public functions and methods have complete type annotations.

```python
# Wrong — no annotations
def get_users(active, limit):
    ...

# Right — fully annotated
def get_users(active: bool = True, limit: int = 50) -> list[User]:
    ...
```

### Union syntax

```python
# Python 3.10+ — prefer X | Y syntax
def find_user(user_id: int) -> User | None:
    ...

# Python < 3.10 — use Optional or Union, or from __future__ import annotations
from __future__ import annotations  # enables PEP 563 postponed evaluation

def find_user(user_id: int) -> User | None:  # works even on 3.8/3.9 with the import
    ...

# Without the future import on Python < 3.10
from typing import Optional
def find_user(user_id: int) -> Optional[User]:
    ...
```

Match the syntax the existing codebase uses. Do not introduce `X | Y` syntax in a codebase that targets Python < 3.10 unless `from __future__ import annotations` is already present.

### Any

```python
# Wrong — bare Any with no explanation
def process(data: Any) -> Any:
    ...

# Right — Any with a comment explaining why it cannot be narrowed
def process(data: Any) -> dict[str, Any]:
    # data comes from an external deserialization library with no type stubs;
    # the structure is validated at runtime before any field access
    ...
```

Never use bare `Any` as a shortcut. When `Any` is genuinely the correct type, explain why in a comment.

### Reusable type patterns

```python
from typing import TypeVar, Generic, Protocol

T = TypeVar("T")
ID = TypeVar("ID", int, str)  # constrained TypeVar

# Generic repository
class Repository(Generic[T]):
    def get(self, id: int) -> T | None: ...
    def save(self, entity: T) -> T: ...

# Protocol for structural subtyping (duck typing with type safety)
class Closeable(Protocol):
    def close(self) -> None: ...

def shutdown(resource: Closeable) -> None:
    resource.close()
```

Use `TypeVar` and `Generic` for reusable container types. Use `Protocol` for structural subtyping where you want duck typing with static type checking.

---

## Idiomatic patterns

### Context managers

```python
# Wrong — manual resource management, not exception-safe
f = open("data.csv")
content = f.read()
f.close()  # skipped if read() raises

# Right — context manager guarantees cleanup
with open("data.csv") as f:
    content = f.read()

# Custom context manager
from contextlib import contextmanager

@contextmanager
def timer(label: str):
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        logger.info("%s completed in %.3fs", label, elapsed)

with timer("bulk_import"):
    process_records(records)
```

Use `with` for any resource with a lifecycle: files, database connections, locks, HTTP clients, temporary directories. Never use manual open/close patterns.

### Comprehensions and generators

```python
# Prefer comprehensions over map/filter — more readable, same performance
# Wrong
active_emails = list(map(lambda u: u.email, filter(lambda u: u.is_active, users)))

# Right
active_emails = [u.email for u in users if u.is_active]

# Use generators for memory efficiency when the full list is not needed at once
def process_csv(path: Path) -> Generator[dict, None, None]:
    with open(path) as f:
        reader = csv.DictReader(f)
        for row in reader:
            yield transform(row)

# Caller controls materialisation
for record in process_csv(path):
    db.session.add(Model(**record))
```

Use list comprehensions for small, finite collections. Use generator expressions (`(x for x in ...)`) or generator functions (`yield`) when processing large or unbounded sequences to avoid loading everything into memory.

### Structured data

```python
# For structured data, use dataclasses or pydantic — match whichever is in the project
from dataclasses import dataclass, field

@dataclass
class OrderSummary:
    order_id: int
    total: Decimal
    item_count: int
    tags: list[str] = field(default_factory=list)

# Pydantic (if already in the project)
from pydantic import BaseModel

class OrderSummary(BaseModel):
    order_id: int
    total: Decimal
    item_count: int
    tags: list[str] = []
```

Never use plain `dict` for structured data that has a known, fixed shape. The type system cannot validate dict keys — use a dataclass or model.

### Paths

```python
# Wrong
import os
config_path = os.path.join(os.path.dirname(__file__), "..", "config", "settings.toml")

# Right
from pathlib import Path
config_path = Path(__file__).parent.parent / "config" / "settings.toml"
```

`pathlib.Path` over `os.path` for all filesystem operations. Path objects are composable with `/`, readable, and type-safe.

### Logging

```python
# Wrong — print for operator-facing output
print(f"Processing {len(records)} records")
print("ERROR: database connection failed")

# Right — structured logging with levels
import logging
logger = logging.getLogger(__name__)

logger.info("Processing %d records", len(records))
logger.error("Database connection failed", exc_info=True)
```

Use `logging.getLogger(__name__)` in every module. Never use `print()` for operator-facing output. Log at the appropriate level: `DEBUG` for diagnostic detail, `INFO` for normal operations, `WARNING` for recoverable issues, `ERROR` for failures, `CRITICAL` for application-threatening conditions.

### Enums for fixed values

```python
# Wrong — bare string literals, typos are silent bugs
def set_status(order, status: str):
    if status == "pendng":  # typo — no error at runtime
        ...

# Right — Enum, typos are caught at the call site
from enum import Enum

class OrderStatus(str, Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    CANCELLED = "cancelled"

def set_status(order: Order, status: OrderStatus) -> None:
    order.status = status
```

Use `Enum` (or `StrEnum` in Python 3.11+) for any fixed set of values. Inheriting from `str` makes the enum directly serialisable and comparable with string values from external sources.

### Memoization

```python
from functools import lru_cache, cache

# lru_cache with size limit (Python 3.2+)
@lru_cache(maxsize=256)
def get_country_code(country_name: str) -> str | None:
    return COUNTRY_MAP.get(country_name)

# cache — unbounded, slightly faster (Python 3.9+)
@cache
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

Use `functools.lru_cache` or `functools.cache` for pure, deterministic functions that are called repeatedly with the same arguments. Never cache functions with side effects or mutable arguments.

---

## Common pitfalls

### Mutable default arguments

```python
# Wrong — the list is created once at function definition time, shared across all calls
def append_item(item: str, target: list[str] = []) -> list[str]:
    target.append(item)
    return target

append_item("a")  # ["a"]
append_item("b")  # ["a", "b"] — unexpected

# Right — None sentinel, list created fresh on each call
def append_item(item: str, target: list[str] | None = None) -> list[str]:
    if target is None:
        target = []
    target.append(item)
    return target
```

Never use a mutable object (list, dict, set) as a default argument. The object is created once when the function is defined, not on each call.

### Late-binding closures in loops

```python
# Wrong — all lambdas capture the same variable i (late binding)
funcs = [lambda: i * 2 for i in range(5)]
funcs[0]()  # returns 8, not 0 — i is 4 at evaluation time

# Right — bind the current value as a default argument
funcs = [lambda i=i: i * 2 for i in range(5)]
funcs[0]()  # returns 0
```

When creating closures inside a loop that capture the loop variable, use default argument binding to capture the current value.

### Exception handling

```python
# Wrong — swallows everything, hides real errors
try:
    process_record(record)
except:
    pass

# Wrong — catches too broadly
try:
    user = User.objects.get(id=user_id)
except Exception:
    return None

# Right — specific exception types
try:
    user = User.objects.get(id=user_id)
except User.DoesNotExist:
    return None
except DatabaseError as exc:
    logger.error("DB error fetching user %d", user_id, exc_info=True)
    raise
```

Always catch specific exception types. Bare `except:` and `except Exception:` (without re-raising) hide bugs and make debugging nearly impossible in production.

### String building

```python
# Wrong — O(n²) memory allocation; each concatenation creates a new string
result = ""
for item in large_list:
    result += str(item) + ", "

# Right — O(n) with join
result = ", ".join(str(item) for item in large_list)
```

Never concatenate strings inside a loop. Use `str.join()` with a generator expression.

### Imports inside functions

```python
# Wrong — import buried inside function with no explanation
def send_email(to: str, subject: str, body: str) -> None:
    import smtplib  # why is this here?
    ...

# Acceptable — only when there is a documented reason
def get_optional_feature() -> object | None:
    try:
        import optional_dependency  # optional; not installed in all environments
        return optional_dependency.Feature()
    except ImportError:
        return None
```

Imports belong at the top of the module. Imports inside functions are sometimes justified (optional dependencies, circular import breaking, lazy loading for performance) but must have a comment explaining the reason.

---

## Testing with pytest

### Fixtures over setUp/tearDown

```python
# Wrong — unittest-style setup
class TestUserService(unittest.TestCase):
    def setUp(self):
        self.db = create_test_db()
        self.service = UserService(self.db)

    def tearDown(self):
        self.db.close()

# Right — pytest fixtures with explicit scope and dependency injection
@pytest.fixture
def db():
    database = create_test_db()
    yield database
    database.close()

@pytest.fixture
def user_service(db):
    return UserService(db)

def test_create_user(user_service):
    user = user_service.create(email="test@example.com", name="Test")
    assert user.id is not None
```

### Parametrize

```python
# Wrong — repeated test functions with minor variations
def test_valid_email_accepted():
    assert validate_email("user@example.com") is True

def test_valid_email_with_subdomain():
    assert validate_email("user@mail.example.com") is True

# Right — parametrize for data-driven tests
@pytest.mark.parametrize("email,expected", [
    ("user@example.com", True),
    ("user@mail.example.com", True),
    ("user+tag@example.com", True),
    ("not-an-email", False),
    ("@missing-local.com", False),
    ("missing-domain@", False),
])
def test_validate_email(email: str, expected: bool):
    assert validate_email(email) is expected
```

### Assertions

```python
# Wrong — unittest-style assertions
self.assertEqual(result.status, "active")
self.assertIsNone(result.deleted_at)

# Right — plain assert, pytest provides detailed diffs on failure
assert result.status == "active"
assert result.deleted_at is None
```

### Async tests

```python
import pytest
import pytest_asyncio

@pytest.mark.asyncio
async def test_fetch_user(async_client, db):
    response = await async_client.get("/users/1")
    assert response.status_code == 200
    assert response.json()["email"] == "user@example.com"
```

Use `pytest-asyncio` for async test functions. Configure `asyncio_mode = "auto"` in `pyproject.toml` to avoid marking every test with `@pytest.mark.asyncio`.

### File system tests

```python
# Wrong — hardcoded tmp path
def test_export(exporter):
    exporter.export("/tmp/test_output.csv")
    assert Path("/tmp/test_output.csv").exists()

# Right — tmp_path fixture provides an isolated, auto-cleaned directory
def test_export(exporter, tmp_path):
    output = tmp_path / "output.csv"
    exporter.export(output)
    assert output.exists()
    assert output.stat().st_size > 0
```

Use pytest's `tmp_path` fixture for filesystem tests. It provides an isolated temporary directory unique to the test invocation, and pytest cleans it up automatically.

---

## Anti-patterns — always flag these

| Anti-pattern | Why | Fix |
|-------------|-----|-----|
| Mutable default argument | Shared state across calls — silent data corruption | `None` sentinel with `if x is None: x = []` |
| Bare `except:` or `except Exception:` without re-raise | Swallows all exceptions silently | Catch specific exception types |
| `os.path` for filesystem operations | Verbose, error-prone concatenation | `pathlib.Path` |
| `print()` for operator-facing output | No levels, no structured output, no filtering | `logging.getLogger(__name__)` |
| String concatenation in a loop | O(n²) memory allocation | `"".join(parts)` |
| Bare string literals for fixed value sets | Typos are silent runtime bugs, no IDE support | `Enum` or `StrEnum` |
| `dict` for structured data with known shape | No type checking, no IDE completion, no validation | `dataclass` or `pydantic.BaseModel` |
| Late-binding closure in a loop | All closures capture final loop value | Default argument binding: `lambda x=x: x` |
| `Any` without explanation | Defeats type checking for everyone downstream | Narrow the type; document why `Any` is unavoidable |
| Imports inside functions without a comment | Obscures intent, surprises readers | Top-level import, or comment explaining the local import |
