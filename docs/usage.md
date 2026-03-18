# Usage

## Define a model

```python
from airmodel import AirModel, AirField

class UnicornSighting(AirModel):
    id: int | None = AirField(default=None, primary_key=True)
    location: str
    sparkle_rating: int
    confirmed: bool = AirField(default=False)
```

Table names derive from class names: `UnicornSighting` becomes `unicorn_sighting`.

## Connect to PostgreSQL

### With Air

Set `DATABASE_URL` in the environment. Air connects automatically:

```python
import air

app = air.Air()  # reads DATABASE_URL, connects on startup
```

The pool is available as `app.db`.

### Without Air

```python
from airmodel import AirDB

db = AirDB()
```

With FastAPI, Starlette, or any ASGI framework:

```python
app = FastAPI(lifespan=db.lifespan("postgresql://user:pass@host/dbname"))
```

With plain async Python:

```python
import asyncpg

pool = await asyncpg.create_pool("postgresql://user:pass@host/dbname")
db.connect(pool)
```

## Create tables

```python
await db.create_tables()  # or app.db.create_tables() with Air
```

Creates missing tables but won't alter existing ones. Use `ALTER TABLE` or a migration tool for schema changes.

## CRUD

```python
# Create
sighting = await UnicornSighting.create(location="Rainbow Falls", sparkle_rating=11)

# Get one (returns None if not found, raises MultipleObjectsReturned if ambiguous)
sighting = await UnicornSighting.get(id=1)

# Filter
confirmed = await UnicornSighting.filter(confirmed=True, order_by="-sparkle_rating")
page = await UnicornSighting.filter(confirmed=True, limit=10, offset=20)

# All rows
all_sightings = await UnicornSighting.all(order_by="location", limit=50)

# Count
total = await UnicornSighting.count()
bright = await UnicornSighting.count(sparkle_rating__gte=8)

# Update
sighting.sparkle_rating = 12
await sighting.save()
await sighting.save(update_fields=["sparkle_rating"])  # partial update

# Delete
await sighting.delete()
```

## Django-style lookups

Append `__lookup` to any field name in `filter()`, `get()`, or `count()`:

| Lookup | SQL | Example |
|---|---|---|
| `__gt` | `>` | `sparkle_rating__gt=5` |
| `__gte` | `>=` | `sparkle_rating__gte=5` |
| `__lt` | `<` | `sparkle_rating__lt=10` |
| `__lte` | `<=` | `sparkle_rating__lte=10` |
| `__contains` | `LIKE '%...%'` | `location__contains="Falls"` |
| `__icontains` | `ILIKE '%...%'` | `location__icontains="falls"` |
| `__in` | `= ANY(...)` | `sparkle_rating__in=[8, 9, 10]` |
| `__isnull` | `IS NULL` / `IS NOT NULL` | `confirmed__isnull=True` |

## Bulk operations

Single-query operations that minimize round trips:

```python
created = await UnicornSighting.bulk_create([
    {"location": "Rainbow Falls", "sparkle_rating": 11},
    {"location": "Crystal Cave", "sparkle_rating": 8},
])
updated = await UnicornSighting.bulk_update({"confirmed": True}, sparkle_rating__gte=10)
deleted = await UnicornSighting.bulk_delete(confirmed=False)
```

`bulk_update()` and `bulk_delete()` require at least one filter argument.

## Transactions

```python
async with db.transaction():  # or app.db.transaction() with Air
    await UnicornSighting.create(location="Rainbow Falls", sparkle_rating=11)
    await UnicornSighting.create(location="Crystal Cave", sparkle_rating=8)
```

## Supported types

| Python | PostgreSQL |
|---|---|
| `str` | `TEXT` |
| `int` | `INTEGER` |
| `float` | `DOUBLE PRECISION` |
| `bool` | `BOOLEAN` |
| `datetime` | `TIMESTAMP WITH TIME ZONE` |
| `UUID` | `UUID` |

`AirField(primary_key=True)` becomes `BIGSERIAL PRIMARY KEY`. Optional fields (`str | None`) are nullable. Required fields without defaults get `NOT NULL`.
