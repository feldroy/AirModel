# Using AirModel

Async ORM for Pydantic models and PostgreSQL. Define models with type annotations, get async CRUD with Django-style lookups.

## With Air

Zero config. `uv add AirModel` and `app = air.Air()`. If `DATABASE_URL` is set in the environment, Air connects automatically and the pool is available as `app.db`. If `DATABASE_URL` is not set, `app.db` is `None` and no database is configured.

```python
await app.db.create_tables()
```

## Without Air

```bash
uv add AirModel
```

```python
from airmodel import AirDB, AirField, AirModel

db = AirDB()
```

With FastAPI, Starlette, or any ASGI framework that accepts a lifespan:

```python
app = FastAPI(lifespan=db.lifespan("postgresql://user:pass@host/dbname"))
```

With plain async Python:

```python
import asyncpg

pool = await asyncpg.create_pool("postgresql://user:pass@host/dbname")
db.connect(pool)
# ... on shutdown:
await pool.close()
db.disconnect()
```

### Create tables

```python
await db.create_tables()  # CREATE TABLE IF NOT EXISTS for every AirModel subclass
```

Auto-migrates existing tables: if you add a field to a model, `create_tables()` runs `ALTER TABLE ADD COLUMN` for any columns not yet in the database. Non-destructive: never drops columns, never changes types. New columns are added without `NOT NULL` (existing rows have no value for them); Pydantic still enforces requirements at the app layer.

## Define models

```python
class UnicornSighting(AirModel):
    id: int | None = AirField(default=None, primary_key=True)
    location: str
    sparkle_rating: int
    confirmed: bool = AirField(default=False)
```

- `AirField(primary_key=True)` becomes `BIGSERIAL PRIMARY KEY`
- Table name derives from class name: `UnicornSighting` becomes `unicorn_sighting`
- Required fields without defaults get `NOT NULL`
- `str | None` is nullable

### Supported types

`str` (TEXT), `int` (INTEGER), `float` (DOUBLE PRECISION), `bool` (BOOLEAN), `datetime` (TIMESTAMP WITH TIME ZONE), `UUID` (UUID).

## CRUD

Every method is async.

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

## Lookups

Append `__lookup` to any field name in `filter()`, `get()`, or `count()`:

- `__gt`, `__gte`, `__lt`, `__lte` — comparisons
- `__contains` — `LIKE '%...%'`
- `__icontains` — `ILIKE '%...%'` (case-insensitive)
- `__in` — `= ANY(...)`, pass a list
- `__isnull` — `IS NULL` (True) or `IS NOT NULL` (False)

```python
await UnicornSighting.filter(sparkle_rating__gte=8, location__icontains="falls")
```

## Bulk operations

Single-query, require at least one filter for update/delete:

```python
created = await UnicornSighting.bulk_create([
    {"location": "Rainbow Falls", "sparkle_rating": 11},
    {"location": "Crystal Cave", "sparkle_rating": 8},
])
updated = await UnicornSighting.bulk_update({"confirmed": True}, sparkle_rating__gte=10)
deleted = await UnicornSighting.bulk_delete(confirmed=False)
```

## Transactions

With Air use `app.db`, otherwise use your `db` instance:

```python
async with app.db.transaction():
    await UnicornSighting.create(location="Rainbow Falls", sparkle_rating=11)
    await UnicornSighting.create(location="Crystal Cave", sparkle_rating=8)
```
