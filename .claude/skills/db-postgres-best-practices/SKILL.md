---
name: db-postgres-best-practices
description: PostgreSQL schema and query best practices covering types, naming, and query patterns. Use when writing SQL DDL/DML, designing tables, reviewing migrations, or writing native/JPQL queries against PostgreSQL.
---

# PostgreSQL best practices

Apply these rules to all PostgreSQL DDL, DML, and query work.

## 1. Timestamps — always `timestamptz`

- Every point-in-time column MUST be `TIMESTAMPTZ`, never bare `TIMESTAMP`.
- Bare `timestamp` has no zone — arithmetic across zones/DST silently breaks.
- Never use `timetz`. Never use `CURRENT_TIME` (returns `timetz`); use `now()` or `CURRENT_TIMESTAMP`.
- Never use `timestamp(0)` / `timestamptz(0)` — they round, pushing values into the future. For second precision, use `date_trunc('second', value)` in queries.

```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT now()
```

## 2. Text — prefer `TEXT`

- Default to `TEXT`. Never `CHAR(n)` (pads with spaces, breaks indexes).
- Avoid blind `VARCHAR(255)` — use `TEXT` unless there's a real business limit.
- For format constraints (e.g., ISO codes), use `CHECK` on `TEXT`:
  ```sql
  country_code TEXT NOT NULL CHECK (country_code ~ '^[A-Z]{3}$')
  ```

## 3. Primary keys — `IDENTITY`, not `serial`

```sql
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
```

Never use `serial` / `bigserial` — legacy, messy ownership/permissions.

## 4. Money — `NUMERIC`, never `money`

- The `money` type rounds unpredictably and depends on `lc_monetary` locale.
- Use `NUMERIC(19, 4)` for monetary values.

```sql
price NUMERIC(19, 4) NOT NULL
```

## 5. Naming — lowercase `snake_case` only

- `a-z`, `0-9`, `_` for tables, columns, indexes, functions.
- PostgreSQL folds unquoted identifiers to lowercase; mixed case forces double-quoting forever.

## 6. Queries — `NOT EXISTS`, never `NOT IN (SELECT ...)`

- A single NULL in the subquery makes `NOT IN` return zero rows silently.
- Planner can't anti-join `NOT IN`; degrades to O(N²).

```sql
-- WRONG
WHERE customer_id NOT IN (SELECT id FROM customers WHERE active = false)

-- CORRECT
WHERE NOT EXISTS (SELECT 1 FROM customers c WHERE c.id = o.customer_id AND c.active = false)
```

`NOT IN (literal_list)` OK only when no NULL is possible.

## 7. Timestamp ranges — half-open, never `BETWEEN`

```sql
-- WRONG (double-counts midnight)
WHERE created_at BETWEEN '2024-01-01' AND '2024-02-01'

-- CORRECT
WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01'
```

`BETWEEN` is OK for `INTEGER`/`DATE`, but prefer half-open as a universal habit.

## 8. Encoding — always UTF-8

```sql
CREATE DATABASE myapp ENCODING 'UTF8' LC_COLLATE 'en_US.UTF-8' LC_CTYPE 'en_US.UTF-8';
```

Never `SQL_ASCII`. JDBC: `?charSet=UTF-8`.

## 9. No `RULE`s — use `TRIGGER`s

Rules rewrite queries unpredictably. Triggers are debuggable and predictable.

## 10. No table inheritance

PostgreSQL table inheritance breaks unique constraints and FKs across parent/child. Use FKs, join tables, or native declarative partitioning. For JPA, use `@Inheritance(JOINED|SINGLE_TABLE)` — these don't use PG inheritance.