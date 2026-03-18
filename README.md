# AirModel

![PyPI version](https://img.shields.io/pypi/v/AirModel.svg)

Async ORM for Pydantic models and PostgreSQL, with a Django-inspired query API.

* Created by [Audrey M. Roy Greenfeld](https://audrey.feldroy.com/) | GitHub [@audreyfeldroy](https://github.com/audreyfeldroy) | PyPI [@audreyr](https://pypi.org/user/audreyr/)
* MIT License
* [GitHub](https://github.com/feldroy/AirModel/) | [PyPI](https://pypi.org/project/AirModel/) | [Documentation](https://feldroy.github.io/AirModel/)

Define your models with standard Pydantic type annotations. AirModel turns them into PostgreSQL tables and gives you async `create`, `get`, `filter`, `all`, `count`, `save`, and `delete`, plus Django-style lookups like `price__gte=10` and `name__icontains="dragon"`.

```python
from airmodel import AirDB, AirModel, AirField

class UnicornSighting(AirModel):
    id: int | None = AirField(default=None, primary_key=True)
    location: str
    sparkle_rating: int
    confirmed: bool = AirField(default=False)

# In your async handlers:
await UnicornSighting.create(location="Rainbow Falls", sparkle_rating=11)
sighting = await UnicornSighting.get(id=1)
bright_ones = await UnicornSighting.filter(sparkle_rating__gte=8, confirmed=True)
count = await UnicornSighting.count()
```

`AirField()` works like Pydantic's `Field()` but adds `primary_key=True` and UI presentation metadata (`label`, `widget`, `placeholder`, etc.).

Built on [asyncpg](https://github.com/MagicStack/asyncpg) and [Pydantic v2](https://docs.pydantic.dev/). Works with [Air](https://airwebframework.org/) or any async Python project.

## Install

```bash
uv add AirModel
```

## Connect to PostgreSQL

### With Air

Zero config. Set `DATABASE_URL` in the environment and Air connects automatically:

```python
import air
from airmodel import AirModel, AirField

app = air.Air()  # reads DATABASE_URL, connects on startup

class Item(AirModel):
    id: int | None = AirField(default=None, primary_key=True)
    name: str
```

If `DATABASE_URL` is not set, `app.db` is `None` and no database is configured. The pool is available as `app.db` for transactions and table creation.

### With any async Python project

```python
import asyncpg
from airmodel import AirDB

db = AirDB()
pool = await asyncpg.create_pool("postgresql://user:pass@host/dbname")
db.connect(pool)

# ... use your models ...

await pool.close()
db.disconnect()
```

### Creating tables

Call `create_tables()` after the pool is ready:

```python
await db.create_tables()
```

This runs `CREATE TABLE IF NOT EXISTS` for every `AirModel` subclass. It creates missing tables but won't add new columns to existing ones. Use `ALTER TABLE` or a migration tool for schema changes.

## Query API

Every method is async. Table names are derived from class names (`UnicornSighting` becomes `unicorn_sighting`).

### CRUD

```python
# Create
sighting = await UnicornSighting.create(location="Rainbow Falls", sparkle_rating=11)

# Read one (returns None if not found, raises MultipleObjectsReturned if ambiguous)
sighting = await UnicornSighting.get(id=1)

# Read many
all_sightings = await UnicornSighting.all()
all_sorted = await UnicornSighting.all(order_by="-sparkle_rating", limit=10)
confirmed = await UnicornSighting.filter(confirmed=True, order_by="-sparkle_rating")
page = await UnicornSighting.filter(confirmed=True, limit=10, offset=20)

# filter() with no filter kwargs is equivalent to all():
everything = await UnicornSighting.filter(order_by="location")

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

### Django-style lookups

Append `__lookup` to any field name in `filter()`, `get()`, or `count()`:

| Lookup | SQL | Example |
|---|---|---|
| `field__gt` | `>` | `sparkle_rating__gt=5` |
| `field__gte` | `>=` | `sparkle_rating__gte=5` |
| `field__lt` | `<` | `sparkle_rating__lt=10` |
| `field__lte` | `<=` | `sparkle_rating__lte=10` |
| `field__contains` | `LIKE '%...%'` | `location__contains="Falls"` |
| `field__icontains` | `ILIKE '%...%'` | `location__icontains="falls"` |
| `field__in` | `= ANY(...)` | `sparkle_rating__in=[8, 9, 10]` |
| `field__isnull` | `IS NULL` / `IS NOT NULL` | `confirmed__isnull=True` |

### Bulk operations

Single-query operations that minimize round trips. Both `bulk_update()` and `bulk_delete()` require at least one filter argument to prevent accidental mass operations.

```python
# Insert many rows in one INSERT ... RETURNING *
sightings = await UnicornSighting.bulk_create([
    {"location": "Rainbow Falls", "sparkle_rating": 11},
    {"location": "Crystal Cave", "sparkle_rating": 8},
])

# UPDATE ... WHERE with row count
updated = await UnicornSighting.bulk_update(
    {"confirmed": True}, sparkle_rating__gte=10
)

# DELETE ... WHERE with row count
deleted = await UnicornSighting.bulk_delete(confirmed=False)
```

### Transactions

```python
# With Air: app.db — without Air: your AirDB() instance
async with app.db.transaction():
    await UnicornSighting.create(location="Rainbow Falls", sparkle_rating=11)
    await UnicornSighting.create(location="Crystal Cave", sparkle_rating=8)
    # Both rows commit together, or neither does.
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

Fields with `primary_key=True` become `BIGSERIAL PRIMARY KEY`. Optional fields (`str | None`) are nullable. Required fields without defaults get `NOT NULL`.
