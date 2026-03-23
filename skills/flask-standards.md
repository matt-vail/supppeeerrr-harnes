---
name: flask-standards
description: >
  Flask-specific standards, patterns, and anti-patterns. Covers application
  factory pattern, blueprint structure, SQLAlchemy session management,
  request context, error handling, and Flask-RESTful or Flask-RESTX patterns.
  Activates when Flask is detected in project dependencies.
---

# Skill: flask-standards

Flask standards for the backend engineer. These apply in addition to the core standards in `agents/backend-engineer.md`, not instead of them.

---

## Application factory pattern

Never create the Flask application at module level. A module-level `app` object is instantiated exactly once, at import time, with a fixed configuration — making it impossible to test with different configs or run multiple instances in the same process.

```python
# Wrong — module-level app, no factory
app = Flask(__name__)
app.config["SQLALCHEMY_DATABASE_URI"] = os.environ["DATABASE_URL"]

@app.route("/users")
def list_users():
    ...
```

```python
# Right — application factory
def create_app(config: dict | None = None) -> Flask:
    app = Flask(__name__)

    app.config.from_object("config.defaults")
    app.config.from_envvar("APP_CONFIG_FILE", silent=True)

    if config:
        app.config.update(config)

    db.init_app(app)
    migrate.init_app(app, db)

    from .users.routes import users_bp
    from .orders.routes import orders_bp
    from .auth.routes import auth_bp

    app.register_blueprint(users_bp)
    app.register_blueprint(orders_bp)
    app.register_blueprint(auth_bp)

    register_error_handlers(app)

    return app
```

**Rules:**
- `create_app()` is the single entry point. Extensions (`db`, `migrate`, `jwt`) are initialised with `init_app()` inside the factory, not at module level.
- Blueprints are registered inside the factory, not at module level where circular imports are likely.
- Config overrides are passed as a dict — this is the primary testing mechanism.
- The factory is importable without side effects. No database connections, no network calls at import time.

---

## Blueprint structure

```python
# Wrong — all routes in one file, no domain separation
@app.route("/users")
def list_users(): ...

@app.route("/orders")
def list_orders(): ...

# Right — one blueprint per domain
# users/routes.py
users_bp = Blueprint("users", __name__, url_prefix="/users")

@users_bp.route("/")
def list_users(): ...

@users_bp.route("/<int:user_id>")
def get_user(user_id: int): ...
```

**Structure:**
```
app/
    __init__.py          # create_app() factory
    extensions.py        # db, migrate, jwt — instantiated here, init_app() called in factory
    users/
        __init__.py
        routes.py        # Blueprint definition and route handlers
        services.py      # Business logic
        models.py        # SQLAlchemy models
        schemas.py       # Marshmallow schemas or dataclasses for I/O
    orders/
        ...
```

**Rules:**
- One blueprint per domain (`users`, `orders`, `auth`, `payments`). Group by domain, not by HTTP method.
- URL prefix is defined on the `Blueprint` constructor, not on individual routes.
- Blueprint names are unique across the application — duplicate names cause silent registration failures.
- Import blueprints inside `create_app()` to avoid circular imports (`app → blueprint → model → app`).

---

## SQLAlchemy session management

```python
# Wrong — manually created session with no lifecycle management
def get_user(user_id):
    session = Session()
    user = session.query(User).get(user_id)
    return user
    # session never closed — connection leak

# Right — db.session from Flask-SQLAlchemy, scoped per request
@users_bp.route("/<int:user_id>")
def get_user(user_id: int):
    user = User.query.get_or_404(user_id)
    return user_schema.dump(user), 200
```

```python
# Commit at the request boundary — in the view or service, never in the model
@users_bp.route("/", methods=["POST"])
def create_user():
    data = request.get_json()
    user = UserService.create(data)
    db.session.commit()              # commit here, not inside UserService.create()
    return user_schema.dump(user), 201
```

**Rules:**
- Use `db.session` from Flask-SQLAlchemy. Never instantiate `Session()` or `sessionmaker()` manually — the lifecycle is not managed and connections will leak.
- Session scope is per request. Flask-SQLAlchemy handles teardown automatically via `@teardown_appcontext`.
- `db.session.commit()` belongs at the request boundary (the view function or a thin service layer called from the view). Model methods must not commit — they work within the session, the caller decides when to commit.
- Always call `db.session.rollback()` in error handlers to prevent a poisoned session from contaminating subsequent requests on the same connection.
- Set `db.session.expire_on_commit = False` (per-session or globally) when returning ORM objects after a commit — otherwise accessing attributes will trigger a lazy load on an already-committed session.

```python
# Error handler — always rollback
@app.errorhandler(Exception)
def handle_unexpected_error(exc):
    db.session.rollback()
    current_app.logger.exception("Unhandled exception")
    return jsonify({"error": {"code": 500, "message": "Internal server error"}}), 500
```

---

## Request context

Flask's request context provides `request`, `g`, and `session` for the duration of a single request. Misusing it is a common source of subtle bugs.

```python
# Wrong — mutable state on the app object (shared across all requests)
app.current_user = None

@app.before_request
def load_user():
    app.current_user = get_user_from_token(request.headers.get("Authorization"))

# Right — g is per-request, isolated to this request's context
@app.before_request
def load_user():
    g.current_user = get_user_from_token(request.headers.get("Authorization"))

@users_bp.route("/me")
def get_me():
    if not g.current_user:
        abort(401)
    return user_schema.dump(g.current_user), 200
```

**Rules:**
- `g` is per-request storage. Use it for auth context, request ID, trace data, and any value that must be available across functions within a single request.
- Never store mutable state on the `app` object. `app` is shared across all requests and all threads — writes to it are a race condition.
- `@before_request` for auth and session setup. `@after_request` for adding response headers (CORS, cache-control, request ID echo) and cleanup.
- `@teardown_request` for cleanup that must run even if an exception was raised (close file handles, release locks).

```python
import uuid

@app.before_request
def attach_request_id():
    g.request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))

@app.after_request
def echo_request_id(response):
    response.headers["X-Request-ID"] = g.get("request_id", "")
    return response
```

---

## Error handling

Flask's default error responses are HTML. In a JSON API, every error response must be JSON with a consistent shape.

```python
# Wrong — relying on Flask defaults; HTML error pages in a JSON API
# (no errorhandler registered — Flask returns HTML 404 page)

# Right — register handlers for all relevant HTTP codes
def register_error_handlers(app: Flask) -> None:
    @app.errorhandler(400)
    def bad_request(exc):
        return jsonify({"error": {"code": 400, "message": str(exc.description)}}), 400

    @app.errorhandler(401)
    def unauthorized(exc):
        return jsonify({"error": {"code": 401, "message": "Authentication required"}}), 401

    @app.errorhandler(403)
    def forbidden(exc):
        return jsonify({"error": {"code": 403, "message": "Permission denied"}}), 403

    @app.errorhandler(404)
    def not_found(exc):
        return jsonify({"error": {"code": 404, "message": "Resource not found"}}), 404

    @app.errorhandler(409)
    def conflict(exc):
        return jsonify({"error": {"code": 409, "message": str(exc.description)}}), 409

    @app.errorhandler(422)
    def unprocessable(exc):
        return jsonify({"error": {"code": 422, "message": str(exc.description)}}), 422

    @app.errorhandler(500)
    def server_error(exc):
        db.session.rollback()
        current_app.logger.exception("Unhandled 500")
        return jsonify({"error": {"code": 500, "message": "Internal server error"}}), 500
```

**Rules:**
- Register `@app.errorhandler` for every HTTP error code the API can return. At minimum: 400, 401, 403, 404, 422, 500.
- All error responses share the same JSON shape: `{"error": {"code": N, "message": "..."}}`.
- Log the full exception server-side (`current_app.logger.exception`). Return only a safe, sanitised message to the client — never a stack trace, SQL error, or internal path.
- Call `abort(404)` to raise HTTP errors from route handlers and services — it integrates with registered error handlers.
- Custom domain exceptions get their own handler registered in the factory: `app.register_error_handler(DomainError, domain_error_handler)`.

---

## Testing

```python
# conftest.py
import pytest
from myapp import create_app, db as _db

@pytest.fixture(scope="session")
def app():
    return create_app({
        "TESTING": True,
        "SQLALCHEMY_DATABASE_URI": "sqlite:///:memory:",
        "WTF_CSRF_ENABLED": False,
    })

@pytest.fixture(scope="session")
def _database(app):
    with app.app_context():
        _db.create_all()
        yield _db
        _db.drop_all()

@pytest.fixture(autouse=True)
def db_transaction(_database, app):
    with app.app_context():
        connection = _database.engine.connect()
        transaction = connection.begin()
        _database.session.bind = connection
        yield _database
        transaction.rollback()
        connection.close()

@pytest.fixture
def client(app):
    return app.test_client()
```

**Rules:**
- Use `app.test_client()` for integration tests — it exercises the full request/response cycle including middleware and error handlers.
- Pass config overrides into `create_app()` — this is the factory's primary value.
- Never test against the production database. Use `sqlite:///:memory:` or a dedicated test schema.
- Wrap each test in a transaction that is rolled back at teardown — database state does not leak between tests.
- Test error handlers explicitly — assert that error responses have the correct JSON shape and status code.

---

## Anti-patterns — always flag these

| Anti-pattern | Why | Fix |
|-------------|-----|-----|
| Module-level `app = Flask(__name__)` | Untestable, blocks multiple instances, circular import risk | Application factory pattern |
| Business logic in route handlers | Untestable, violates single-responsibility | Service layer (`services.py`) |
| Direct `Session()` creation | Lifecycle not managed — connection leaks | Use `db.session` from Flask-SQLAlchemy |
| Mutable state on the `app` object | Shared across requests and threads — race conditions | Use `g` for per-request state |
| No error handler registration | Flask returns HTML errors in a JSON API | Register `@app.errorhandler` for all 4xx and 5xx codes |
| `db.session.commit()` in model methods | Wrong layer — models must not control transaction boundaries | Commit at the request boundary in the view or service |
| Blueprints registered at module level | Circular import risk; bypasses factory config | Register blueprints inside `create_app()` |
| No `db.session.rollback()` in error handlers | Poisoned session contaminates subsequent requests on same connection | Always rollback in error handlers before returning a response |
