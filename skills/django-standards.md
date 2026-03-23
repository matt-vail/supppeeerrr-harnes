---
name: django-standards
description: >
  Django-specific standards, patterns, and anti-patterns. Covers ORM usage,
  N+1 query prevention with select_related/prefetch_related, reversible
  migrations, MTV architecture, settings split for environments, class-based
  vs function-based views, and Django REST Framework patterns. Activates when
  Django is detected in project dependencies.
---

# Skill: django-standards

Django standards for the backend engineer. These apply in addition to the core standards in `agents/backend-engineer.md`, not instead of them.

---

## ORM patterns

Every database access goes through the ORM unless there is a documented, reviewed reason to use raw SQL. When raw SQL is necessary, it uses parameterised queries — never string formatting.

### select_related and prefetch_related

```python
# Wrong — N+1: accesses user.profile for each order in the loop
orders = Order.objects.all()
for order in orders:
    print(order.user.profile.name)  # 2 extra queries per row

# Right — select_related for ForeignKey and OneToOneField (SQL JOIN)
orders = Order.objects.select_related("user__profile").all()
for order in orders:
    print(order.user.profile.name)  # no extra queries

# Right — prefetch_related for ManyToManyField and reverse ForeignKey (separate query, Python join)
orders = Order.objects.prefetch_related("tags", "items__product").all()
for order in orders:
    for tag in order.tags.all():  # no extra queries — already prefetched
        print(tag.name)
```

**Rules:**
- `select_related` for ForeignKey and OneToOneField — produces a SQL JOIN. Use when the related object is always needed.
- `prefetch_related` for ManyToManyField and reverse ForeignKey — issues a separate query and joins in Python. Use for multi-valued relationships.
- Chaining both is valid and common: `Order.objects.select_related("user").prefetch_related("items")`.
- Before writing a loop over a queryset, ask: does the loop body access a related object? If yes, fetch it before the loop.

### Column selection

```python
# Wrong — loads every column, including large text/blob fields
users = User.objects.all()

# Right — only() fetches named columns (plus pk)
users = User.objects.only("id", "email", "name")

# Right — defer() fetches all columns except named ones
users = User.objects.defer("bio", "avatar_data")

# Right — values() for read-only projections (returns dicts, not model instances)
emails = User.objects.values("id", "email")

# Right — values_list() for flat projections
email_list = User.objects.values_list("email", flat=True)
```

**Rules:**
- Use `only()` or `defer()` when the model has large fields (text blobs, JSONField, BinaryField) that are not needed in the calling code.
- Use `values()` and `values_list()` for read-only data that will never be mutated — they skip model instantiation and are significantly faster for large result sets.
- Never use `Model.objects.all()` in a view without pagination. An unbounded queryset against a large table is a production incident.

### Pagination

```python
from django.core.paginator import Paginator

def user_list(request):
    queryset = User.objects.select_related("profile").order_by("-created_at")
    paginator = Paginator(queryset, per_page=50)
    page = paginator.get_page(request.GET.get("page", 1))
    return render(request, "users/list.html", {"page": page})
```

Every view that returns a collection paginates it. No exceptions.

---

## Migrations

Migrations are append-only. Once a migration has been applied to any shared environment (staging, production), it is immutable.

### Reversibility

```python
# Wrong — no reverse operation
class Migration(migrations.Migration):
    operations = [
        migrations.RunSQL("ALTER TABLE users ADD COLUMN score INTEGER DEFAULT 0"),
    ]

# Right — always provide reverse_sql or a reverse callable
class Migration(migrations.Migration):
    operations = [
        migrations.RunSQL(
            sql="ALTER TABLE users ADD COLUMN score INTEGER DEFAULT 0",
            reverse_sql="ALTER TABLE users DROP COLUMN score",
        ),
    ]
```

Every migration must be reversible. If a reverse is genuinely impossible (destructive data deletion), document this explicitly in the migration file and get explicit sign-off before merging.

### Data migrations

```python
# Separate data migrations from schema migrations — never combine RunPython with schema changes

def backfill_display_name(apps, schema_editor):
    User = apps.get_model("users", "User")
    for user in User.objects.filter(display_name="").iterator():
        user.display_name = f"{user.first_name} {user.last_name}".strip()
        user.save(update_fields=["display_name"])

def reverse_backfill(apps, schema_editor):
    User = apps.get_model("users", "User")
    User.objects.update(display_name="")

class Migration(migrations.Migration):
    operations = [
        migrations.RunPython(backfill_display_name, reverse_backfill),
    ]
```

**Rules:**
- `RunPython` always receives a reverse function as the second argument. `migrations.RunPython.noop` is acceptable only when the reverse operation is genuinely a no-op and that is documented.
- Use `apps.get_model()` inside `RunPython` callables — never import models directly. The historical model state must be used.
- Use `.iterator()` for large table backfills to avoid loading the full queryset into memory.

### Zero-downtime schema changes

For adding a column with a constraint, never do it in a single deploy:

1. **Deploy 1:** Add nullable column (no constraint) — old and new code both work.
2. **Backfill:** Populate data for existing rows (data migration or management command).
3. **Deploy 2:** Add the `NOT NULL` constraint or default — data is already present.

Never run `./manage.py migrate` in the same deploy step that introduces a breaking schema change (dropping a column, renaming a column, changing a type) unless the application code has already stopped using the old schema.

---

## MTV architecture

Django's pattern is Model–Template–View. In a REST API context, Templates are replaced by serializers, but the architectural principle is the same: each layer has one responsibility.

### Fat models, thin views

```python
# Wrong — business logic in the view
def create_order(request):
    data = json.loads(request.body)
    user = request.user
    if user.credit_balance < data["total"]:
        return JsonResponse({"error": "Insufficient credit"}, status=402)
    order = Order.objects.create(user=user, total=data["total"])
    user.credit_balance -= data["total"]
    user.save()
    send_order_confirmation_email.delay(order.id)
    return JsonResponse({"id": str(order.id)}, status=201)

# Right — view delegates to the model or service layer
def create_order(request):
    data = json.loads(request.body)
    order = OrderService.create(user=request.user, total=data["total"])
    return JsonResponse({"id": str(order.id)}, status=201)
```

**Rules:**
- Views validate HTTP input and return HTTP responses — nothing more.
- Business logic, multi-model operations, and side effects (emails, tasks) belong in model methods or a service module (`services.py` or `services/` package).
- A service function is a plain Python function or class — no Django view machinery, no `request` object.
- Model methods for logic that is intrinsic to a single model. Service layer for logic that crosses model boundaries.

### Class-based vs function-based views

- Use function-based views (FBVs) for simple, one-off endpoints — less boilerplate, easier to trace.
- Use class-based views (CBVs) when inheriting shared behaviour (`LoginRequiredMixin`, `PermissionRequiredMixin`) or when Django's generic views (`ListView`, `DetailView`) fit the use case precisely.
- Never use CBVs to add complexity — if the `as_view()` method needs more than two mixins to work, reconsider whether a service layer and a simple FBV is clearer.

---

## Settings management

```
# Wrong — single settings.py with all environments
settings.py  # contains DEBUG=False, SECRET_KEY=..., DATABASE_URL=...

# Right — split by environment
settings/
    __init__.py
    base.py       # shared across all environments
    local.py      # development overrides
    production.py # production config, no DEBUG, no hardcoded secrets
    test.py       # test runner config
```

```python
# settings/base.py
import environ

env = environ.Env()

SECRET_KEY = env("DJANGO_SECRET_KEY")
DATABASES = {
    "default": env.db("DATABASE_URL")
}

# settings/local.py
from .base import *  # noqa: F401,F403

DEBUG = True
ALLOWED_HOSTS = ["localhost", "127.0.0.1"]
```

**Rules:**
- `settings/base.py` contains everything shared. `local.py` and `production.py` import from base and override only what differs.
- Secrets (`SECRET_KEY`, `DATABASE_URL`, API keys) are always environment variables — never hardcoded, never committed.
- Use `django-environ` or `python-decouple` for env var parsing with type coercion and defaults.
- The `DJANGO_SETTINGS_MODULE` env var selects the active settings module. Document this in the project README.
- Never use `DEBUG=True` in production. Never use the production `SECRET_KEY` locally.

---

## Django REST Framework

### Serializers

```python
# Wrong — raw json.dumps, no validation, no type safety
def get_user(request, pk):
    user = get_object_or_404(User, pk=pk)
    return JsonResponse({"id": user.id, "email": user.email})

# Right — serializer handles validation, type coercion, and field selection
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ["id", "email", "name", "created_at"]
        read_only_fields = ["id", "created_at"]

class UserCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ["email", "name", "password"]
        extra_kwargs = {"password": {"write_only": True}}

    def create(self, validated_data):
        return User.objects.create_user(**validated_data)
```

**Rules:**
- Serializers over raw `json.dumps` — always. Serializers provide validation, type coercion, and consistent field selection.
- `ModelSerializer` for CRUD operations against a model. Custom `Serializer` for non-model data (e.g., aggregated responses, external service payloads).
- Request serializers and response serializers are separate classes. Never reuse the same serializer for both input validation and output rendering when the fields differ.
- Use `SerializerMethodField` for computed or derived fields — not model `@property` decorators that bleed presentation logic into the model.
- Never expose sensitive fields (passwords, tokens, internal IDs) in a response serializer. Use `write_only=True` for input-only fields.

```python
class OrderSummarySerializer(serializers.ModelSerializer):
    item_count = serializers.SerializerMethodField()
    total_display = serializers.SerializerMethodField()

    class Meta:
        model = Order
        fields = ["id", "status", "item_count", "total_display", "created_at"]

    def get_item_count(self, obj):
        return obj.items.count()

    def get_total_display(self, obj):
        return f"${obj.total:.2f}"
```

### ViewSets and routers

```python
# Standard CRUD — ViewSet + Router
class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.select_related("profile").order_by("-created_at")
    serializer_class = UserSerializer
    permission_classes = [IsAuthenticated]

    def get_serializer_class(self):
        if self.action in ("create", "update", "partial_update"):
            return UserCreateSerializer
        return UserSerializer

    @action(detail=True, methods=["post"], url_path="deactivate")
    def deactivate(self, request, pk=None):
        user = self.get_object()
        UserService.deactivate(user)
        return Response(status=status.HTTP_204_NO_CONTENT)

router = DefaultRouter()
router.register("users", UserViewSet)
```

```python
# Custom endpoint with no model backing — APIView
class TokenRefreshView(APIView):
    permission_classes = [AllowAny]

    def post(self, request):
        serializer = TokenRefreshSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        return Response(serializer.validated_data)
```

**Rules:**
- ViewSets + Routers for standard CRUD. `APIView` for custom endpoints that do not map to a model.
- `permission_classes` on every view. Never rely on URL-level security (URL patterns are not a security boundary).
- Override `get_serializer_class()` when create/update actions need a different serializer than read actions.
- Use the `@action` decorator for non-standard operations on a ViewSet rather than creating a separate `APIView`.
- Use `get_object()` inside `@action` methods — it applies the object-level permission check automatically.

### Permissions

```python
# Wrong — no permission class, relying on URL-level auth
class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer

# Right — explicit permission class on every viewset/view
class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = [IsAuthenticated, IsOwnerOrAdmin]
```

Set `DEFAULT_PERMISSION_CLASSES` in settings to a safe default (e.g., `[IsAuthenticated]`). Override to `AllowAny` explicitly where public access is intentional — never leave it as an accident.

---

## Anti-patterns — always flag these

| Anti-pattern | Why | Fix |
|-------------|-----|-----|
| ForeignKey access in a loop without select_related | N+1 query per iteration | `select_related` or `prefetch_related` before the loop |
| Business logic in views | Untestable, violates MTV | Move to model methods or service layer |
| Editing an applied migration | Breaks other developers' environments and CI | Create a new migration |
| Single `settings.py` for all environments | Secrets in version control, no environment isolation | Settings split: `base.py`, `local.py`, `production.py` |
| Raw SQL without parameterisation | SQL injection | ORM queryset or `cursor.execute(sql, params)` |
| `Model.objects.all()` without pagination | Full table load — memory and query cost at scale | `Paginator` or `.paginate_queryset()` in DRF |
| `json.dumps` instead of DRF serializers | No validation, no type safety, no field control | `ModelSerializer` or `Serializer` |
| No `permission_classes` on a view | Accidental open endpoint | Explicit `permission_classes` on every view |
| `RunPython` without a reverse function | Migration is irreversible | Always provide a reverse callable or `RunPython.noop` with documentation |
| Importing models directly in `RunPython` | Uses current model state, not historical | `apps.get_model("app", "Model")` inside the callable |
