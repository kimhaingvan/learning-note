# PostgreSQL Data Types

> **Level**: Beginner–Intermediate | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 1.3  
> A comprehensive reference to PostgreSQL's type system — the richest of any relational database. Choosing the right type affects storage, performance, indexing, and data integrity.

---

## Table of Contents

1. [Why Data Types Matter](#1-why-data-types-matter)
2. [Numeric Types](#2-numeric-types)
3. [Character Types](#3-character-types)
4. [Boolean Type](#4-boolean-type)
5. [Date/Time Types](#5-datetime-types)
6. [UUID Type](#6-uuid-type)
7. [JSON / JSONB Types](#7-json--jsonb-types)
8. [Array Types](#8-array-types)
9. [Enum Types](#9-enum-types)
10. [Network Address Types](#10-network-address-types)
11. [Range Types](#11-range-types)
12. [Composite Types](#12-composite-types)
13. [Binary Data (bytea)](#13-binary-data-bytea)
14. [Special Types](#14-special-types)
15. [Type Selection Guide](#15-type-selection-guide)
16. [Common Mistakes](#16-common-mistakes)
17. [What to Learn Next](#17-what-to-learn-next)

---

## 1. Why Data Types Matter

Choosing the right data type affects:

| Concern | Impact |
|---------|--------|
| **Storage size** | Smaller types = more rows per page = faster scans |
| **Validation** | The type enforces what values are allowed (e.g., `integer` rejects `'hello'`) |
| **Indexing** | Some types support specialized index types (GIN for arrays/JSONB, GiST for ranges) |
| **Performance** | Operations on native types are faster than casting or parsing at runtime |
| **Correctness** | `timestamptz` handles time zones; `timestamp` does not — wrong choice = bugs |

> **Rule of thumb**: Use the most specific type that fits your data. Don't store numbers as text, dates as strings, or booleans as integers.

---

## 2. Numeric Types

### Integer Types

| Type | Size | Range |
|------|------|-------|
| `smallint` | 2 bytes | -32,768 to 32,767 |
| `integer` (or `int`) | 4 bytes | -2,147,483,648 to 2,147,483,647 |
| `bigint` | 8 bytes | -9.2 × 10¹⁸ to 9.2 × 10¹⁸ |

```sql
CREATE TABLE example (
    id integer,          -- Most common for IDs (up to ~2 billion)
    population bigint,   -- Use when integer range isn't enough
    age smallint         -- Use when values are always small
);
```

### Auto-Incrementing (Serial)

| Type | Underlying Type | Notes |
|------|----------------|-------|
| `serial` | `integer` | Legacy — auto-creates a sequence |
| `bigserial` | `bigint` | Legacy — for large tables |
| `smallserial` | `smallint` | Legacy — rarely used |
| `GENERATED ALWAYS AS IDENTITY` | Any integer | **Preferred** — SQL standard |

```sql
-- Modern approach (preferred)
CREATE TABLE users (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL
);

-- Legacy approach (still works, widely seen in older code)
CREATE TABLE users (
    id serial PRIMARY KEY,
    name text NOT NULL
);
```

> **Best practice**: Use `GENERATED ALWAYS AS IDENTITY` for new tables. It's SQL-standard and gives you more control.

### Exact Numeric (Arbitrary Precision)

| Type | Size | Use Case |
|------|------|----------|
| `numeric(precision, scale)` | Variable | Exact values — money, financial calculations |
| `decimal(precision, scale)` | Variable | Alias for `numeric` |

```sql
-- numeric(10, 2) = up to 10 digits total, 2 after decimal
CREATE TABLE products (
    price numeric(10, 2)   -- e.g., 12345678.99
);

-- No precision limit
SELECT 0.1 + 0.2;           -- 0.3 (exact, unlike floating point)
```

> **Key**: `numeric` is **exact** but slower than floating point. Use it when precision matters (money, accounting).

### Floating Point (Approximate)

| Type | Size | Precision |
|------|------|-----------|
| `real` | 4 bytes | ~6 decimal digits |
| `double precision` | 8 bytes | ~15 decimal digits |

```sql
-- Floating point arithmetic is approximate
SELECT 0.1::real + 0.2::real;  -- 0.30000001192092896 (not exactly 0.3)
```

> **Warning**: Never use `real` or `double precision` for money. Use `numeric` instead.

### Money Type

```sql
-- Built-in money type (locale-dependent formatting)
SELECT '52093.89'::money;  -- $52,093.89

-- Avoid in most cases — it's locale-dependent and less flexible than numeric
-- Prefer: numeric(12, 2) with application-level formatting
```

---

## 3. Character Types

| Type | Description | Max Length |
|------|-------------|------------|
| `text` | Variable-length, unlimited | ~1 GB |
| `varchar(n)` | Variable-length, max n characters | n characters |
| `char(n)` | Fixed-length, right-padded with spaces | n characters |

```sql
CREATE TABLE users (
    name text NOT NULL,            -- Preferred for most cases
    email varchar(255) NOT NULL,   -- When you want a length limit
    code char(3)                   -- Fixed-width codes (e.g., 'USD', 'VND')
);
```

### Performance

All three types use the **same internal storage** (varlena). There is **no performance difference** between `text`, `varchar`, and `varchar(n)`.

| Recommendation | Use |
|----------------|-----|
| General text | `text` |
| Need a length constraint | `text` + `CHECK` or `varchar(n)` |
| Fixed-width code (rare) | `char(n)` |

```sql
-- Idiomatic PostgreSQL: text + CHECK constraint
CREATE TABLE users (
    email text NOT NULL CHECK (length(email) <= 255)
);
```

> **Key insight**: In PostgreSQL, `text` is not "worse" than `varchar` — they're identical internally. The PostgreSQL community recommends `text` as the default.

---

## 4. Boolean Type

```sql
-- Accepted literal values for TRUE
TRUE, 't', 'true', 'yes', 'on', '1'

-- Accepted literal values for FALSE
FALSE, 'f', 'false', 'no', 'off', '0'

-- NULL is also valid (three-valued logic)
```

```sql
CREATE TABLE users (
    active boolean NOT NULL DEFAULT true
);

SELECT * FROM users WHERE active;           -- shorthand for active = true
SELECT * FROM users WHERE NOT active;       -- shorthand for active = false
SELECT * FROM users WHERE active IS NULL;   -- check for NULL
```

> **Best practice**: Always set `NOT NULL DEFAULT` on boolean columns. A nullable boolean has three states — usually not what you want.

---

## 5. Date/Time Types

| Type | Size | Description | Example |
|------|------|-------------|---------|
| `date` | 4 bytes | Date only (no time) | `2026-03-27` |
| `time` | 8 bytes | Time only (no date) | `14:30:00` |
| `timetz` | 12 bytes | Time with time zone | `14:30:00+07` |
| `timestamp` | 8 bytes | Date + time, no time zone | `2026-03-27 14:30:00` |
| `timestamptz` | 8 bytes | Date + time, with time zone | `2026-03-27 14:30:00+07` |
| `interval` | 16 bytes | Duration / time span | `1 year 2 months 3 days` |

### timestamp vs timestamptz

This is the **most important choice** in the date/time category.

```sql
-- timestamptz: stores as UTC, converts on display based on session timezone
SET timezone = 'Asia/Ho_Chi_Minh';
SELECT '2026-03-27 14:30:00+00'::timestamptz;
-- Result: 2026-03-27 21:30:00+07

-- timestamp: stores the literal value, no timezone conversion
SELECT '2026-03-27 14:30:00'::timestamp;
-- Result: 2026-03-27 14:30:00 (always, regardless of session timezone)
```

> **Rule**: **Always use `timestamptz`** for real-world timestamps. Use `timestamp` only for abstract dates that don't represent a moment in time (e.g., "store opens at 9:00 AM" regardless of timezone).

### Interval

```sql
SELECT INTERVAL '1 year 2 months 3 days 4 hours';
SELECT NOW() + INTERVAL '7 days';
SELECT NOW() - '2026-01-01'::timestamptz;  -- returns an interval
SELECT AGE('2026-03-27'::date, '1990-05-15'::date);  -- '35 years 10 mons 12 days'
```

### Common Functions

```sql
SELECT NOW();                           -- current timestamp with tz
SELECT CURRENT_DATE;                    -- current date
SELECT EXTRACT(YEAR FROM NOW());        -- 2026
SELECT DATE_TRUNC('month', NOW());      -- 2026-03-01 00:00:00+07
SELECT TO_CHAR(NOW(), 'YYYY-MM-DD');    -- '2026-03-27'
SELECT DATE_PART('dow', NOW());         -- day of week (0=Sunday)
SELECT MAKE_DATE(2026, 3, 27);          -- 2026-03-27
SELECT AGE(NOW(), '1990-01-01'::date);  -- interval
```

---

## 6. UUID Type

Universally Unique Identifier — 128-bit value.

```sql
-- Generate a random UUID (v4) — built-in since PostgreSQL 13
SELECT gen_random_uuid();
-- Result: a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11

-- Generate a time-ordered UUID (v7) — built-in since PostgreSQL 18
SELECT uuidv7();
-- Result: 019513e8-7c00-7000-8000-000000000001
```

### UUIDv4 vs UUIDv7

| Feature | UUIDv4 | UUIDv7 |
|---------|--------|--------|
| Ordering | Random | Time-ordered (first 48 bits = millisecond timestamp) |
| B-tree locality | Poor (random inserts scatter across pages) | Good (sequential inserts stay together) |
| Index performance | Worse for write-heavy workloads | Better (less page splitting) |
| Guessability | Not guessable | Timestamp is extractable (not secret) |

```sql
CREATE TABLE orders (
    id uuid PRIMARY KEY DEFAULT uuidv7(),  -- PG18: prefer UUIDv7
    total numeric(10, 2)
);
```

> **Best practice for PG18**: Use `uuidv7()` for primary keys. It gives you globally unique IDs with good B-tree performance. For older versions, use `gen_random_uuid()`.

---

## 7. JSON / JSONB Types

PostgreSQL has first-class support for semi-structured JSON data.

| Type | Storage | Indexable | Preserves Order/Duplicates |
|------|---------|-----------|---------------------------|
| `json` | Raw text | No | Yes |
| `jsonb` | Parsed binary | Yes (GIN index) | No (deduplicates keys) |

```sql
-- Always prefer jsonb unless you need exact text preservation
CREATE TABLE events (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    data jsonb NOT NULL
);

INSERT INTO events (data) VALUES ('{"type": "click", "x": 100, "y": 200}');
```

### Key Operators

```sql
-- Access operators
SELECT data->'type'          FROM events;  -- "click" (as jsonb)
SELECT data->>'type'         FROM events;  -- click   (as text)
SELECT data#>'{nested,key}'  FROM events;  -- Navigate nested path (as jsonb)
SELECT data#>>'{nested,key}' FROM events;  -- Navigate nested path (as text)

-- Containment
SELECT * FROM events WHERE data @> '{"type": "click"}';  -- contains

-- Key existence
SELECT * FROM events WHERE data ? 'type';           -- has key 'type'
SELECT * FROM events WHERE data ?| array['x','y'];  -- has any key
SELECT * FROM events WHERE data ?& array['x','y'];  -- has all keys
```

### Indexing JSONB

```sql
-- GIN index for containment (@>) and existence (?) queries
CREATE INDEX idx_events_data ON events USING gin (data);

-- GIN index on a specific path
CREATE INDEX idx_events_type ON events USING gin ((data->'type'));

-- B-tree index on a specific extracted value
CREATE INDEX idx_events_type_btree ON events ((data->>'type'));
```

> **Detailed coverage**: See Phase 12 (Advanced Data Features) for JSON path queries, `jsonb_build_object()`, `jsonb_agg()`, and SQL/JSON standard functions.

---

## 8. Array Types

PostgreSQL supports arrays of any built-in type.

```sql
CREATE TABLE posts (
    id serial PRIMARY KEY,
    title text NOT NULL,
    tags text[] NOT NULL DEFAULT '{}'   -- array of text
);

INSERT INTO posts (title, tags) VALUES ('PG Guide', ARRAY['postgresql', 'database', 'tutorial']);
INSERT INTO posts (title, tags) VALUES ('Go Tips', '{golang, programming}');  -- alternative syntax
```

### Array Operations

```sql
-- Access elements (1-indexed!)
SELECT tags[1] FROM posts;                        -- 'postgresql'

-- Containment
SELECT * FROM posts WHERE tags @> ARRAY['postgresql'];   -- contains
SELECT * FROM posts WHERE tags && ARRAY['golang', 'rust']; -- overlap (any match)

-- Append
UPDATE posts SET tags = tags || ARRAY['new-tag'] WHERE id = 1;

-- Unnest (expand array into rows)
SELECT id, unnest(tags) AS tag FROM posts;

-- Aggregate back into array
SELECT array_agg(DISTINCT tag) FROM (SELECT unnest(tags) AS tag FROM posts) t;
```

### Indexing Arrays

```sql
CREATE INDEX idx_posts_tags ON posts USING gin (tags);
```

> **When to use arrays**: For small, fixed sets of values (tags, labels, permissions). For complex nested data, use `jsonb`. For many-to-many relationships, use a join table.

---

## 9. Enum Types

User-defined types with a fixed set of allowed values.

```sql
CREATE TYPE order_status AS ENUM ('pending', 'processing', 'shipped', 'delivered', 'cancelled');

CREATE TABLE orders (
    id serial PRIMARY KEY,
    status order_status NOT NULL DEFAULT 'pending'
);

INSERT INTO orders (status) VALUES ('shipped');     -- OK
INSERT INTO orders (status) VALUES ('invalid');     -- ERROR: invalid input value
```

### Properties

- **Ordered**: `'pending' < 'processing' < 'shipped'` (based on definition order).
- **Compact storage**: 4 bytes regardless of label length.
- **Type-safe**: Invalid values are rejected at the database level.

### Adding Values

```sql
-- Add a new value (cannot be done inside a transaction before PG 12)
ALTER TYPE order_status ADD VALUE 'refunded' AFTER 'cancelled';

-- Renaming a value (PG 10+)
ALTER TYPE order_status RENAME VALUE 'cancelled' TO 'canceled';
```

### Limitations

- **Cannot remove values** — once added, they stay (you can stop using them).
- **Migration-unfriendly** — altering enums in deployed applications can be tricky.
- **Alternative**: A `status text CHECK (status IN ('pending', 'shipped', ...))` column is easier to modify but less type-safe.

---

## 10. Network Address Types

| Type | Description | Example |
|------|-------------|---------|
| `inet` | IPv4 or IPv6 host address (with optional netmask) | `192.168.1.1/24` |
| `cidr` | IPv4 or IPv6 network (strict network address) | `192.168.1.0/24` |
| `macaddr` | MAC address (6-byte) | `08:00:2b:01:02:03` |
| `macaddr8` | MAC address (8-byte, EUI-64) | `08:00:2b:ff:fe:01:02:03` |

```sql
CREATE TABLE access_log (
    client_ip inet NOT NULL,
    network cidr
);

-- Range queries on network types
SELECT * FROM access_log WHERE client_ip <<= '192.168.1.0/24'::cidr;  -- contained in network
```

> These types support dedicated operators for containment, supernet/subnet checks, and bitwise operations. Much more powerful than storing IPs as text.

---

## 11. Range Types

Represent a range of values in a single column.

| Type | Element Type |
|------|-------------|
| `int4range` | `integer` |
| `int8range` | `bigint` |
| `numrange` | `numeric` |
| `tsrange` | `timestamp` |
| `tstzrange` | `timestamptz` |
| `daterange` | `date` |

```sql
CREATE TABLE meetings (
    id serial PRIMARY KEY,
    room text NOT NULL,
    during tstzrange NOT NULL
);

INSERT INTO meetings (room, during)
VALUES ('Room A', '[2026-03-27 09:00, 2026-03-27 10:30)');
--               ↑ inclusive start                      ↑ exclusive end
```

### Range Operators

```sql
-- Containment
SELECT * FROM meetings WHERE during @> '2026-03-27 09:30'::timestamptz;  -- contains element
SELECT * FROM meetings WHERE during @> '[2026-03-27 09:00, 2026-03-27 09:30)'::tstzrange;  -- contains range

-- Overlap
SELECT * FROM meetings WHERE during && '[2026-03-27 10:00, 2026-03-27 11:00)'::tstzrange;

-- Adjacent
SELECT * FROM meetings WHERE during -|- '[2026-03-27 10:30, 2026-03-27 12:00)'::tstzrange;
```

### Exclusion Constraints (Prevent Overlaps)

```sql
-- Requires btree_gist extension
CREATE EXTENSION btree_gist;

ALTER TABLE meetings ADD CONSTRAINT no_overlap
    EXCLUDE USING gist (room WITH =, during WITH &&);
-- No two meetings can have the same room AND overlapping time ranges
```

> **Use cases**: Scheduling, booking systems, versioning, IP range management, temporal data.

---

## 12. Composite Types

Struct-like types that group multiple fields.

```sql
CREATE TYPE address AS (
    street text,
    city text,
    state text,
    zip text
);

CREATE TABLE customers (
    id serial PRIMARY KEY,
    name text NOT NULL,
    home_address address,
    work_address address
);

INSERT INTO customers (name, home_address)
VALUES ('Alice', ROW('123 Main St', 'Springfield', 'IL', '62701'));

-- Access fields with dot notation (parentheses required)
SELECT (home_address).city FROM customers;
```

> **When to use**: Rarely. Composite types add complexity. Prefer separate columns or `jsonb` for most use cases.

---

## 13. Binary Data (bytea)

Stores raw binary data.

```sql
CREATE TABLE files (
    id serial PRIMARY KEY,
    name text NOT NULL,
    content bytea
);

INSERT INTO files (name, content) VALUES ('test.bin', '\xDEADBEEF');
INSERT INTO files (name, content) VALUES ('greeting', 'hello'::bytea);
```

### Escape Formats

| Format | Example | Notes |
|--------|---------|-------|
| Hex (default) | `'\xDEADBEEF'` | Preferred output format since PG 9.0 |
| Escape | `'\\000\\001\\002'` | Legacy format |

> **When to use**: Small binary data (thumbnails, hashes, encrypted blobs). For large files, consider using external storage (S3) and storing the URL.

---

## 14. Special Types

| Type | Description | Use Case |
|------|-------------|----------|
| `money` | Currency with locale-aware formatting | Rarely recommended — prefer `numeric(12,2)` |
| `pg_lsn` | WAL Log Sequence Number | Replication monitoring |
| `txid_snapshot` | Transaction snapshot | MVCC internals |
| `xml` | XML data | Legacy XML storage and XPath queries |
| `tsvector` | Full-text search document | Pre-parsed text for FTS |
| `tsquery` | Full-text search query | Boolean search expression |
| `bit` / `bit varying` | Bit string | Fixed/variable-length bit sequences |
| `oid` | Object identifier | Internal system catalog references |

---

## 15. Type Selection Guide

A quick decision tree for common scenarios:

| Data | Recommended Type | Avoid |
|------|-----------------|-------|
| Primary key (small table) | `integer GENERATED ALWAYS AS IDENTITY` | `serial` (legacy) |
| Primary key (large/distributed) | `uuid` with `uuidv7()` (PG18) or `gen_random_uuid()` | `text` for IDs |
| Money / financial | `numeric(precision, scale)` | `real`, `double precision`, `money` |
| General text | `text` | `varchar` without a reason |
| Text with max length | `varchar(n)` or `text` + `CHECK` | `char(n)` (space-padded) |
| True/false | `boolean NOT NULL DEFAULT` | `integer` (0/1), `text` ('yes'/'no') |
| Timestamp | `timestamptz` | `timestamp` (loses timezone), `bigint` (epoch) |
| Date only | `date` | `text` ('2026-03-27') |
| Small fixed set of values | `enum` or `text` + `CHECK` | `integer` with magic numbers |
| Semi-structured data | `jsonb` | `json` (unless you need text preservation) |
| Tags / labels | `text[]` with GIN index | `text` with comma separation |
| IP addresses | `inet` / `cidr` | `text` |
| Time ranges / scheduling | Range types + exclusion constraints | Two separate columns |
| Binary data (small) | `bytea` | `text` with base64 encoding |
| Embeddings / vectors | `vector` (pgvector extension) | `real[]` |

---

## 16. Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Using `timestamp` instead of `timestamptz` | Time zone bugs across regions | Always use `timestamptz` |
| Using `real`/`double precision` for money | Rounding errors (0.1 + 0.2 ≠ 0.3) | Use `numeric(12, 2)` |
| Using `serial` in new code | Not SQL-standard, weaker ownership semantics | Use `GENERATED ALWAYS AS IDENTITY` |
| Storing JSON as `text` | Can't index or validate | Use `jsonb` |
| Using `char(n)` for variable-length strings | Wastes space (right-padded with spaces) | Use `text` or `varchar(n)` |
| Storing IPs as `text` | Can't do subnet queries or range checks | Use `inet` or `cidr` |
| Using `integer` for IDs in distributed systems | Conflicts across nodes | Use `uuid` |
| Not setting `NOT NULL` on booleans | Three-state logic is confusing | Always `boolean NOT NULL DEFAULT` |

---

## 17. What to Learn Next

1. **Table Basics & Constraints** — Using these types in table definitions — see [16-table-basics-constraints.md](16-table-basics-constraints.md).
2. **Indexes** — How different types affect index choice (GIN for arrays/jsonb, GiST for ranges) — see [05-indexes.md](05-indexes.md).
3. **Full-Text Search** — `tsvector` / `tsquery` types in depth — Phase 12 (Advanced Data Features).
4. **JSON Deep Dive** — JSONB operators, path queries, SQL/JSON — Phase 12.

---

> *Ref: [Docs — Data Types](https://www.postgresql.org/docs/18/datatype.html) · [Neon Tutorial](https://neon.com/postgresql/tutorial) · [Neon — PG18 New Features](https://neon.com/postgresql/postgresql-18-new-features) (UUIDv7)*
