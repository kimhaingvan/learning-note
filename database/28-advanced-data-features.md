# Advanced Data Features

> **Level**: Advanced | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 12 — JSON/JSONB Deep Dive, Full-Text Search, PostGIS, Hstore, Range Patterns, Domain Types, Generated Columns  
> Advanced PostgreSQL data features beyond basic types — deep JSONB querying with SQL/JSON, full-text search, geospatial data, and custom types. For basic type definitions, see [15-data-types.md](15-data-types.md).

---

## Table of Contents

1. [JSON / JSONB — Advanced Usage](#1-json--jsonb--advanced-usage)
   - [SQL/JSON Path Language](#sqljson-path-language)
   - [SQL/JSON Standard Functions (PG 17+)](#sqljson-standard-functions-pg-17)
   - [JSONB Manipulation Functions](#jsonb-manipulation-functions)
   - [JSONB Indexing Strategies](#jsonb-indexing-strategies)
   - [JSONB Aggregation & Construction](#jsonb-aggregation--construction)
2. [Full-Text Search](#2-full-text-search)
   - [Core Concepts](#core-concepts)
   - [Indexing for Full-Text Search](#indexing-for-full-text-search)
   - [Ranking & Highlighting](#ranking--highlighting)
   - [Phrase Search & Advanced Queries](#phrase-search--advanced-queries)
   - [Configurations & Dictionaries](#configurations--dictionaries)
   - [Practical Patterns](#practical-patterns)
3. [Hstore](#3-hstore)
4. [PostGIS (Geospatial)](#4-postgis-geospatial)
5. [Range Types — Advanced Patterns](#5-range-types--advanced-patterns)
6. [Domain Types](#6-domain-types)
7. [Generated Columns](#7-generated-columns)

---

## 1. JSON / JSONB — Advanced Usage

> For basic JSON/JSONB types, operators (`->`, `->>`, `@>`, `?`), and simple GIN indexing, see [15-data-types.md § JSON/JSONB](15-data-types.md).

### SQL/JSON Path Language

The **jsonpath** type (PG 12+) provides a powerful way to query JSONB using a path expression language (SQL/JSON standard).

```sql
-- Basic path expressions
SELECT jsonb_path_query(
    '{"store": {"books": [{"title": "PG Guide", "price": 29.99}, {"title": "SQL Deep", "price": 39.99}]}}',
    '$.store.books[*].title'
);
-- "PG Guide"
-- "SQL Deep"

-- Filtering with predicates
SELECT jsonb_path_query(
    '{"store": {"books": [{"title": "PG Guide", "price": 29.99}, {"title": "SQL Deep", "price": 39.99}]}}',
    '$.store.books[*] ? (@.price > 30)'
);
-- {"title": "SQL Deep", "price": 39.99}

-- Check existence
SELECT jsonb_path_exists(
    '{"tags": ["postgresql", "database", "tutorial"]}',
    '$.tags[*] ? (@ == "postgresql")'
);  -- true

-- Extract first match
SELECT jsonb_path_query_first(
    '{"items": [{"id": 1}, {"id": 2}, {"id": 3}]}',
    '$.items[*].id'
);  -- 1
```

| Path Function | Returns | Description |
|---------------|---------|-------------|
| `jsonb_path_query(doc, path)` | SETOF jsonb | All matches (multiple rows) |
| `jsonb_path_query_first(doc, path)` | jsonb | First match |
| `jsonb_path_query_array(doc, path)` | jsonb array | All matches as a single JSON array |
| `jsonb_path_exists(doc, path)` | boolean | Whether any match exists |
| `jsonb_path_match(doc, path)` | boolean | Whether the path predicate evaluates to true |

#### Path Syntax Reference

| Syntax | Meaning | Example |
|--------|---------|---------|
| `$` | The root JSON value | `$` |
| `.key` | Object field access | `$.name` |
| `[n]` | Array element by index | `$.items[0]` |
| `[*]` | All array elements | `$.items[*]` |
| `.*` | All object fields | `$.*` |
| `? (predicate)` | Filter | `$.items[*] ? (@.price > 10)` |
| `@` | Current item in filter | `? (@.status == "active")` |
| `.size()` | Array length | `$.items.size()` |
| `.type()` | JSON type as string | `$.value.type()` |
| `like_regex` | Regex match | `? (@ like_regex "^pg")` |

### SQL/JSON Standard Functions (PG 17+)

PostgreSQL 17+ implements the SQL/JSON standard functions:

```sql
-- JSON_TABLE: Transform JSON into relational rows
SELECT jt.*
FROM orders,
  JSON_TABLE(
    order_data,
    '$.items[*]'
    COLUMNS (
      item_id    int         PATH '$.id',
      item_name  text        PATH '$.name',
      quantity   int         PATH '$.qty',
      price      numeric     PATH '$.price',
      in_stock   boolean     PATH '$.available' DEFAULT true ON EMPTY
    )
  ) AS jt;

-- JSON_VALUE: Extract a scalar value
SELECT JSON_VALUE(profile, '$.address.city' RETURNING text) AS city
FROM users;

-- JSON_QUERY: Extract a JSON object or array
SELECT JSON_QUERY(profile, '$.tags' WITH WRAPPER) AS tags
FROM users;

-- JSON_EXISTS: Check if a path exists
SELECT *
FROM users
WHERE JSON_EXISTS(profile, '$.premium' TRUE ON ERROR);
```

### JSONB Manipulation Functions

```sql
-- Build JSONB objects
SELECT jsonb_build_object(
    'id', 1,
    'name', 'PostgreSQL',
    'tags', jsonb_build_array('database', 'sql', 'open-source')
);

-- Merge objects (|| operator)
SELECT '{"a": 1, "b": 2}'::jsonb || '{"b": 3, "c": 4}'::jsonb;
-- {"a": 1, "b": 3, "c": 4}   (b overwritten)

-- Deep set/update a nested value
SELECT jsonb_set(
    '{"user": {"name": "Kim", "role": "dev"}}'::jsonb,
    '{user,role}',    -- path
    '"admin"'         -- new value
);
-- {"user": {"name": "Kim", "role": "admin"}}

-- Remove a key
SELECT '{"a": 1, "b": 2, "c": 3}'::jsonb - 'b';
-- {"a": 1, "c": 3}

-- Remove nested key
SELECT '{"a": {"b": 1, "c": 2}}'::jsonb #- '{a,b}';
-- {"a": {"c": 2}}

-- Iterate over key-value pairs
SELECT key, value
FROM jsonb_each('{"name": "Kim", "age": 30}'::jsonb);
-- key  | value
-- name | "Kim"
-- age  | 30

-- Convert JSONB to record type
SELECT *
FROM jsonb_to_record('{"id": 1, "name": "Kim", "active": true}'::jsonb)
    AS x(id int, name text, active boolean);

-- Pretty print
SELECT jsonb_pretty('{"name":"Kim","tags":["a","b"]}'::jsonb);
```

### JSONB Indexing Strategies

| Index Type | Supports | Use When |
|-----------|---------|---------|
| **GIN (default ops)** | `@>`, `?`, `?\|`, `?&` | General-purpose JSONB querying |
| **GIN (jsonb_path_ops)** | `@>` only | Containment queries only (smaller, faster index) |
| **Expression index** | `=`, `<`, `>`, etc. on extracted scalar | Querying a specific known key frequently |

```sql
-- GIN index (supports all JSONB operators)
CREATE INDEX idx_profile_gin ON users USING GIN (profile);

-- GIN with jsonb_path_ops (30-40% smaller, containment only)
CREATE INDEX idx_profile_gin ON users USING GIN (profile jsonb_path_ops);

-- Expression index (extract a specific field for B-tree lookup)
CREATE INDEX idx_profile_city ON users ((profile->>'city'));

-- Query: uses the expression index
SELECT * FROM users WHERE profile->>'city' = 'Tokyo';

-- Query: uses the GIN index
SELECT * FROM users WHERE profile @> '{"premium": true}';
```

### JSONB Aggregation & Construction

```sql
-- Aggregate rows into a JSON array
SELECT jsonb_agg(
    jsonb_build_object('id', id, 'name', name)
) AS users_json
FROM users
WHERE active = true;

-- Aggregate key-value pairs into an object
SELECT jsonb_object_agg(department, employee_count) AS dept_counts
FROM department_stats;

-- Combine with GROUP BY
SELECT
    department,
    jsonb_agg(jsonb_build_object('name', name, 'salary', salary)
        ORDER BY salary DESC) AS employees
FROM employees
GROUP BY department;
```

---

## 2. Full-Text Search

PostgreSQL has built-in full-text search — no need for Elasticsearch or Solr for many use cases.

### Core Concepts

```
Document text → to_tsvector() → tsvector (normalized, stemmed tokens with positions)
Search query → to_tsquery()  → tsquery  (boolean combination of search terms)
Match:         tsvector @@ tsquery → boolean
```

```sql
-- Convert text to tsvector (tokenize, stem, normalize)
SELECT to_tsvector('english', 'PostgreSQL is a powerful open-source database system');
-- 'databas':7 'open':5 'open-sourc':4 'postgresql':1 'power':4 'sourc':6 'system':8

-- Convert search to tsquery
SELECT to_tsquery('english', 'powerful & database');
-- 'power' & 'databas'        (stemmed!)

-- Match
SELECT to_tsvector('english', 'PostgreSQL is a powerful database')
    @@ to_tsquery('english', 'powerful & database');
-- true
```

| tsvector/tsquery Feature | Syntax | Example |
|-------------------------|--------|---------|
| **AND** | `&` | `'cat & dog'` |
| **OR** | `\|` | `'cat \| dog'` |
| **NOT** | `!` | `'cat & !dog'` |
| **Phrase (adjacent)** | `<->` | `'open <-> source'` |
| **Phrase (within N)** | `<N>` | `'postgresql <2> guide'` (within 2 words) |
| **Prefix match** | `:*` | `'post:*'` (matches postgresql, postgres, etc.) |

```sql
-- Practical example: search articles
SELECT id, title
FROM articles
WHERE to_tsvector('english', title || ' ' || body) @@ to_tsquery('english', 'postgresql & replication');
```

### Indexing for Full-Text Search

Without an index, `to_tsvector()` is computed on every row for every query. Two index options:

#### Option 1: GIN Index on Expression

```sql
-- Index on computed tsvector
CREATE INDEX idx_articles_fts ON articles
    USING GIN (to_tsvector('english', title || ' ' || coalesce(body, '')));

-- Query must match the expression exactly
SELECT * FROM articles
WHERE to_tsvector('english', title || ' ' || coalesce(body, '')) @@ to_tsquery('english', 'postgresql');
```

#### Option 2: Stored tsvector Column (Better Performance)

```sql
-- Add a stored tsvector column
ALTER TABLE articles ADD COLUMN search_vector tsvector;

-- Populate it
UPDATE articles SET search_vector =
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(body, '')), 'B');

-- Index the column
CREATE INDEX idx_articles_search ON articles USING GIN (search_vector);

-- Query
SELECT * FROM articles WHERE search_vector @@ to_tsquery('english', 'postgresql');
```

Keep the column updated with a trigger:

```sql
CREATE FUNCTION articles_search_trigger() RETURNS trigger AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(NEW.body, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_articles_search
    BEFORE INSERT OR UPDATE OF title, body ON articles
    FOR EACH ROW EXECUTE FUNCTION articles_search_trigger();
```

#### GIN vs GiST for Full-Text

| Feature | GIN | GiST |
|---------|-----|------|
| **Build time** | Slower | Faster |
| **Index size** | Larger | Smaller |
| **Query speed** | ✅ Faster (exact) | Slower (lossy — needs recheck) |
| **Update speed** | Slower (pending list) | Faster |
| **Best for** | Read-heavy, static data | Write-heavy, small datasets |

> Use **GIN** in almost all cases. GiST only if write speed dominates.

### Ranking & Highlighting

```sql
-- Rank results by relevance
SELECT
    id, title,
    ts_rank(search_vector, query) AS rank
FROM articles, to_tsquery('english', 'postgresql & performance') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;

-- ts_rank_cd: cover density — rewards terms appearing close together
SELECT
    id, title,
    ts_rank_cd(search_vector, query) AS rank
FROM articles, to_tsquery('english', 'postgresql & performance') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;

-- Highlight matching terms in results
SELECT
    id, title,
    ts_headline('english', body, to_tsquery('english', 'postgresql'),
        'StartSel=<b>, StopSel=</b>, MaxWords=50, MinWords=25'
    ) AS snippet
FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgresql');
```

#### Weighting

```sql
-- Weights: A (most important) > B > C > D (least)
-- Weight them during ranking
SELECT ts_rank(
    '{0.1, 0.2, 0.4, 1.0}',   -- weights for {D, C, B, A}
    search_vector,
    query
) AS rank
FROM ...;
```

### Phrase Search & Advanced Queries

```sql
-- Exact phrase: "open source"
SELECT * FROM articles
WHERE search_vector @@ phraseto_tsquery('english', 'open source');
-- Internally becomes: 'open' <-> 'sourc'

-- Proximity: words within 3 positions
SELECT * FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgresql <3> performance');

-- Web-style search (PG 11+) — handles natural language input
SELECT * FROM articles
WHERE search_vector @@ websearch_to_tsquery('english', 'postgresql performance -slow');
-- Automatically parses: 'postgresql' & 'perform' & !'slow'

-- websearch_to_tsquery supports:
--   word1 word2    → AND
--   "exact phrase" → phrase
--   word1 OR word2 → OR
--   -excluded      → NOT
```

### Configurations & Dictionaries

```sql
-- List available configurations
SELECT cfgname FROM pg_ts_config;  -- english, simple, german, french, etc.

-- List dictionaries
SELECT dictname FROM pg_ts_dict;

-- Test tokenization with different configs
SELECT * FROM ts_debug('english', 'Running PostgreSQL databases');
-- Shows: alias, description, token, dictionaries, dictionary, lexemes
```

| Configuration | Stemming | Stop Words | Use When |
|--------------|:--------:|:----------:|---------|
| `simple` | ❌ | ❌ | Exact token matching (names, codes) |
| `english` | ✅ | ✅ | English text (default for English content) |
| `unaccent + english` | ✅ | ✅ | Accent-insensitive search |

```sql
-- Create accent-insensitive config
CREATE EXTENSION unaccent;

CREATE TEXT SEARCH CONFIGURATION myapp_english (COPY = english);
ALTER TEXT SEARCH CONFIGURATION myapp_english
    ALTER MAPPING FOR hword, hword_part, word WITH unaccent, english_stem;

-- Now: to_tsvector('myapp_english', 'café') matches 'cafe'
```

### Practical Patterns

#### Multi-Language Search

```sql
-- Store language per document
ALTER TABLE articles ADD COLUMN lang regconfig DEFAULT 'english';

-- Use the stored language for indexing
CREATE INDEX idx_articles_fts ON articles
    USING GIN (to_tsvector(lang, title || ' ' || body));

-- Query with the document's language
SELECT * FROM articles
WHERE to_tsvector(lang, title || ' ' || body) @@ to_tsquery(lang, 'suche');
```

#### Autocomplete with Prefix Search

```sql
-- Prefix matching for autocomplete
SELECT DISTINCT title
FROM articles
WHERE search_vector @@ to_tsquery('english', 'postg:*')
ORDER BY title
LIMIT 10;

-- Combined with trigram for fuzzy matching
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_articles_title_trgm ON articles USING GIN (title gin_trgm_ops);

SELECT title, similarity(title, 'postgrsql') AS sim
FROM articles
WHERE title % 'postgrsql'   -- trigram similarity
ORDER BY sim DESC
LIMIT 10;
```

---

## 3. Hstore

Key-value store in a single column. Simpler than JSONB for flat key-value data.

```sql
CREATE EXTENSION hstore;

-- Create table with hstore
CREATE TABLE products (
    id serial PRIMARY KEY,
    name text,
    attributes hstore
);

-- Insert
INSERT INTO products (name, attributes) VALUES
    ('Laptop', 'brand => Dell, ram => "16GB", cpu => "i7"'),
    ('Phone', 'brand => Apple, storage => "256GB", color => black');

-- Query by key
SELECT name, attributes->'brand' AS brand
FROM products;

-- Check key existence
SELECT * FROM products WHERE attributes ? 'color';

-- Contains check
SELECT * FROM products WHERE attributes @> 'brand => Dell';

-- Merge
UPDATE products
SET attributes = attributes || 'warranty => "2 years"'
WHERE name = 'Laptop';

-- Delete a key
UPDATE products
SET attributes = delete(attributes, 'color')
WHERE name = 'Phone';

-- Get all keys/values
SELECT skeys(attributes) FROM products WHERE id = 1;
SELECT each(attributes) FROM products WHERE id = 1;
```

```sql
-- Indexing
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);
-- Supports: ?, ?&, ?|, @>
```

> **When to use hstore vs JSONB**: Use hstore for flat key-value pairs with simple lookups. Use JSONB for nested data, arrays, or when you need SQL/JSON standard functions.

---

## 4. PostGIS (Geospatial)

PostGIS adds geospatial data types, functions, and indexing to PostgreSQL. It's the industry standard for geospatial databases.

```sql
CREATE EXTENSION postgis;
```

### Geometry vs Geography

| Type | Coordinate System | Distance Units | Best For |
|------|------------------|:-------------:|---------|
| **geometry** | Planar (Cartesian) | SRID-dependent (meters, degrees, etc.) | Small areas, precise measurements, complex operations |
| **geography** | Spherical (lat/lon) | Meters (always) | Global data, accurate distances on Earth's surface |

```sql
-- Create table with geography (points on Earth's surface)
CREATE TABLE places (
    id serial PRIMARY KEY,
    name text,
    location geography(POINT, 4326)    -- SRID 4326 = WGS 84 (GPS coordinates)
);

-- Insert points (longitude, latitude)
INSERT INTO places (name, location) VALUES
    ('Tokyo Tower',  ST_MakePoint(139.7454, 35.6586)),
    ('Eiffel Tower', ST_MakePoint(2.2945, 48.8584)),
    ('Statue of Liberty', ST_MakePoint(-74.0445, 40.6892));

-- Distance between two points (meters)
SELECT
    a.name, b.name,
    ST_Distance(a.location, b.location) / 1000 AS distance_km
FROM places a, places b
WHERE a.id < b.id;

-- Find places within 10 km of a point
SELECT name, ST_Distance(location, ST_MakePoint(139.7, 35.6)::geography) AS dist_m
FROM places
WHERE ST_DWithin(location, ST_MakePoint(139.7, 35.6)::geography, 10000)  -- 10 km
ORDER BY dist_m;
```

### Spatial Indexing

```sql
-- GiST index for spatial queries (essential for performance)
CREATE INDEX idx_places_location ON places USING GIST (location);

-- The index accelerates: ST_DWithin, ST_Distance, ST_Intersects, ST_Contains, etc.
```

### Common Spatial Functions

| Function | Purpose | Example |
|----------|---------|---------|
| `ST_Distance(a, b)` | Distance between geometries | `ST_Distance(loc_a, loc_b)` |
| `ST_DWithin(a, b, d)` | Within distance? (uses index) | `ST_DWithin(loc, point, 5000)` |
| `ST_Contains(a, b)` | Does A contain B? | `ST_Contains(polygon, point)` |
| `ST_Intersects(a, b)` | Do A and B overlap? | `ST_Intersects(zone, route)` |
| `ST_Buffer(geom, d)` | Buffer zone around geometry | `ST_Buffer(point, 1000)` |
| `ST_Area(geom)` | Area of polygon | `ST_Area(park::geometry)` |
| `ST_MakePoint(lng, lat)` | Create a point | `ST_MakePoint(139.7, 35.6)` |
| `ST_GeomFromText(wkt)` | Parse WKT | `ST_GeomFromText('POLYGON(...)')` |
| `ST_AsGeoJSON(geom)` | Export as GeoJSON | For API responses |

---

## 5. Range Types — Advanced Patterns

> For basic range types and operators, see [15-data-types.md § Range Types](15-data-types.md).

### Multiranges (PG 14+)

```sql
-- A multirange is a set of non-overlapping ranges
SELECT '{[2024-01-01, 2024-01-15), [2024-02-01, 2024-02-15)}'::datemultirange;

-- Useful for complex availability/scheduling
CREATE TABLE employee_availability (
    employee_id int,
    available datemultirange
);

INSERT INTO employee_availability VALUES
    (1, '{[2024-01-01, 2024-01-15), [2024-02-01, 2024-03-01)}');

-- Check if a date falls within any available range
SELECT * FROM employee_availability
WHERE available @> '2024-02-10'::date;
```

### Temporal Patterns with Range Types

```sql
-- Booking system: prevent double-booking with exclusion constraint
CREATE TABLE room_bookings (
    id serial PRIMARY KEY,
    room_id int NOT NULL,
    during tstzrange NOT NULL,
    guest_name text NOT NULL,

    EXCLUDE USING GIST (
        room_id WITH =,
        during WITH &&          -- no overlapping time ranges for the same room
    )
);

-- This INSERT succeeds
INSERT INTO room_bookings (room_id, during, guest_name)
VALUES (101, '[2024-06-01 14:00, 2024-06-03 11:00)', 'Alice');

-- This INSERT fails (overlapping period for room 101)
INSERT INTO room_bookings (room_id, during, guest_name)
VALUES (101, '[2024-06-02 14:00, 2024-06-04 11:00)', 'Bob');
-- ERROR: conflicting key value violates exclusion constraint

-- Requires btree_gist extension for combining equality with range overlap
CREATE EXTENSION btree_gist;
```

### Versioned Records (Temporal Tables)

```sql
-- System-versioned table using ranges
CREATE TABLE products_history (
    id int NOT NULL,
    name text,
    price numeric,
    valid_during tstzrange NOT NULL DEFAULT tstzrange(now(), 'infinity'),

    EXCLUDE USING GIST (id WITH =, valid_during WITH &&)
);

-- Query: what was the price on a specific date?
SELECT * FROM products_history
WHERE id = 42
    AND valid_during @> '2024-03-15 12:00:00+00'::timestamptz;

-- Price change: close current range, open new one
UPDATE products_history
SET valid_during = tstzrange(lower(valid_during), now())
WHERE id = 42 AND upper_inf(valid_during);

INSERT INTO products_history (id, name, price)
VALUES (42, 'Widget', 29.99);
```

---

## 6. Domain Types

Named types built on base types with constraints — enforce business rules at the type level.

```sql
-- Email domain with validation
CREATE DOMAIN email AS text
    CHECK (VALUE ~ '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$');

-- Positive amount
CREATE DOMAIN positive_amount AS numeric(12, 2)
    CHECK (VALUE > 0);

-- Non-empty text
CREATE DOMAIN nonempty_text AS text
    CHECK (VALUE IS NOT NULL AND length(trim(VALUE)) > 0);

-- Use them in tables
CREATE TABLE invoices (
    id serial PRIMARY KEY,
    customer_email email NOT NULL,
    amount positive_amount NOT NULL,
    description nonempty_text
);

-- Invalid data is rejected at the type level
INSERT INTO invoices (customer_email, amount, description)
VALUES ('not-an-email', 100, 'Test');
-- ERROR: value for domain email violates check constraint

INSERT INTO invoices (customer_email, amount, description)
VALUES ('user@example.com', -50, 'Test');
-- ERROR: value for domain positive_amount violates check constraint
```

```sql
-- Modify a domain (adds constraint to all columns using this domain)
ALTER DOMAIN email ADD CONSTRAINT email_not_empty CHECK (VALUE != '');

-- Drop a domain
DROP DOMAIN IF EXISTS email CASCADE;  -- CASCADE drops dependent columns
```

> **Domain vs CHECK constraint**: Domains are reusable across tables. A CHECK constraint is per-column/table. Use domains when the same validation applies to multiple columns in multiple tables.

---

## 7. Generated Columns

Computed columns whose values are derived from other columns automatically.

### STORED Generated Columns (PG 12+)

```sql
CREATE TABLE products (
    id serial PRIMARY KEY,
    price numeric(10, 2),
    quantity int,
    total_value numeric(10, 2) GENERATED ALWAYS AS (price * quantity) STORED,
    name text,
    search_name tsvector GENERATED ALWAYS AS (to_tsvector('english', name)) STORED
);

INSERT INTO products (price, quantity, name)
VALUES (29.99, 100, 'PostgreSQL Guide Book');
-- total_value = 2999.00 (auto-computed)
-- search_name = 'book':3 'guid':2 'postgresql':1 (auto-computed)

-- Cannot INSERT/UPDATE a generated column directly
UPDATE products SET total_value = 5000 WHERE id = 1;
-- ERROR: column "total_value" can only be updated to DEFAULT
```

### VIRTUAL Generated Columns (PG 18+)

```sql
-- Virtual: computed on read, not stored on disk
CREATE TABLE circles (
    id serial PRIMARY KEY,
    radius numeric,
    area numeric GENERATED ALWAYS AS (pi() * radius * radius) VIRTUAL,
    circumference numeric GENERATED ALWAYS AS (2 * pi() * radius) VIRTUAL
);

-- No disk space used for virtual columns
-- Computed on every SELECT
```

| Feature | STORED | VIRTUAL (PG 18+) |
|---------|:------:|:-----------------:|
| Disk space | ✅ Uses storage | ❌ No storage |
| Computed when | INSERT/UPDATE | Every SELECT |
| Can index | ✅ Yes | ❌ No |
| Best for | Frequently read, expensive computation, needs indexing | Simple computation, rarely read, save storage |

### Rules & Limitations

| Rule | Detail |
|------|--------|
| Expression must be **immutable** | Cannot use `now()`, `random()`, or anything non-deterministic |
| Cannot reference other generated columns | Expression can only use regular columns |
| Cannot have a DEFAULT | GENERATED and DEFAULT are mutually exclusive |
| Foreign keys | A generated column can reference another table's column |
| Partitioned tables | Generated columns are supported in partitioned tables |

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Storing structured data as `json` instead of `jsonb` | Can't index, slower queries, no dedup | Always use `jsonb` unless you need exact JSON formatting |
| No GIN index on JSONB columns | Every query scans entire table | Create GIN index: `USING GIN (col)` or `USING GIN (col jsonb_path_ops)` |
| Using `to_tsvector()` without stored column | Recomputes on every row for every query | Add a `tsvector` column with trigger or generated column |
| PostGIS: using `geometry` for global coordinates | Distance calculations wrong (planar math on sphere) | Use `geography` type for lat/lon data |
| PostGIS: ST_Distance without ST_DWithin | Can't use spatial index efficiently | Filter with `ST_DWithin` first, then compute `ST_Distance` |
| Hstore for nested data | Hstore only supports flat key-value | Use JSONB for nested structures |
| Generated column with volatile function | `ERROR: generation expression is not immutable` | Only use immutable functions in generated column expressions |

---

## What to Learn Next

1. **Data Types** — Basic type definitions referenced throughout — see [15-data-types.md](15-data-types.md).
2. **Indexes** — Indexing strategies for JSONB, FTS, spatial — see [05-indexes.md](05-indexes.md).
3. **Extensions** — PostGIS, pg_trgm, hstore are extensions — see Phase 13.
4. **Schema Design** — When to use JSONB vs normalized tables — see [18-schema-design-data-modeling.md](18-schema-design-data-modeling.md).

---

> *Ref: [Docs — JSON Types](https://www.postgresql.org/docs/18/datatype-json.html) · [Docs — Full Text Search](https://www.postgresql.org/docs/18/textsearch.html) · [Docs — hstore](https://www.postgresql.org/docs/18/hstore.html) · [PostGIS Documentation](https://postgis.net/documentation/) · [Docs — Range Types](https://www.postgresql.org/docs/18/rangetypes.html) · [Docs — CREATE DOMAIN](https://www.postgresql.org/docs/18/sql-createdomain.html) · [Docs — Generated Columns](https://www.postgresql.org/docs/18/ddl-generated-columns.html)*
