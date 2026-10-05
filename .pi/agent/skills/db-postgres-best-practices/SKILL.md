---
name: db-postgres-best-practices
description: PostgreSQL anti-patterns and best practices for schema design, data types, SQL constructs, and authentication. Use when writing or reviewing SQL migrations, DDL, queries, pg_hba.conf, or any code touching a PostgreSQL database.
---

# PostgreSQL Best Practices

Anti-patterns to avoid, with what to use instead. Each entry: the mistake, why it is a problem, the fix, and the rare exception.

## Encoding

### Don't use SQL_ASCII

- Why: SQL_ASCII means "no conversions". Bytes are reinterpreted in whatever encoding is active, so the database ends up storing an unrecoverable mixture of encodings.
- Do instead: Use UTF8. For hopeless mixed-encoding input, prefer `bytea`, or autodetect UTF8 and assume a specific fallback encoding (e.g. WIN1252).
- Exception: Last resort for unlabelled mixed-encoding data (old IRC logs, non-MIME emails) — and even then `bytea` is usually better.

## Tool usage

### Don't use `psql -W` / `--password`

- Why: Forces a password prompt even when the server doesn't need one. With peer auth this misleads you into thinking a password is required; a wrong/absent password at the prompt can still "log in" and mask the real credentials.
- Do instead: Just run `psql`; it prompts only when the server actually requires a password.
- Exception: Never, essentially.

## Schema constructs

### Don't use rules

- Why: Rules rewrite queries rather than executing conditional logic; all non-trivial rules are subtly incorrect.
- Do instead: Use triggers.
- Exception: Never. The rewriter is a VIEW implementation detail; don't touch it directly.

### Don't use table inheritance

- Why: Legacy OO-database coupling that didn't work out.
- Do instead: Foreign keys for modeling; native `PARTITION BY` for partitioning.
- Exception: Barely ever — e.g. the `temporal_tables` extension for row versioning when SQL:2011 support is missing, and you accept the parent-table caveats.

## SQL constructs

### Don't use `NOT IN` (especially `NOT IN (SELECT ...)`)

- Why:
  1. NULL poisoning: `col NOT IN (1, null)` always returns 0 rows, because `col IN (1, null)` never evaluates to FALSE.
  2. The planner cannot turn `NOT IN (SELECT ...)` into an anti-join. It plans a hashed subplan (only for small result sets) or a plain subplan (O(N²)). Performance can be fine in tests, then degrade by orders of magnitude past a size threshold.
- Do instead:
  ```sql
  select * from foo
  where not exists (select from bar where foo.col = bar.x);
  ```
- Exception: `NOT IN` with a constant list of non-NULL values is fine for excluding specific values.

### Don't use mixed-case identifiers (`NamesLikeThis`)

- Why: PostgreSQL folds unquoted identifiers to lowercase. `"Bar"` and `bar` are different objects, and whether other tools quote names varies — leading to confusing "no such table" errors.
- Do instead: Use `a-z`, `0-9`, `_` only. For pretty report output, use aliases: `select character_name as "Character Name" from foo;`
- Exception: Pretty names in report output — via column aliases, not column names.

### Don't use `BETWEEN` (especially with timestamps)

- Why: `BETWEEN` is a closed interval — both endpoints included. `timestampcol BETWEEN '2018-06-01' AND '2018-06-08'` includes exactly `2018-06-08 00:00:00` but nothing later that day, so midnight entries get double-counted across adjacent ranges.
- Do instead:
  ```sql
  select * from blah
  where timestampcol >= '2018-06-01' and timestampcol < '2018-06-08';
  ```
- Exception: Safe for discrete types (integers, dates) if you remember both ends are inclusive — but avoid the habit.

## Date/Time

### Don't use `timestamp` (without time zone) for points in time

- Why: `timestamptz` stores a single moment (microseconds since 2000-01-01 UTC); it handles input in any timezone and correct arithmetic across timezones and DST changes. `timestamp` is just "a picture of a calendar and clock" — arithmetic across locations or DST boundaries gives wrong answers.
- Do instead: `timestamptz`. Display in other zones with `at time zone`.
- Exception: Purely abstract timestamps that are saved/retrieved without arithmetic.

### Don't use `timestamp` to store UTC times

- Why: The database can't know the values are UTC, which turns simple calculations ("last midnight in user's timezone") into nested `AT TIME ZONE` gymnastics.
- Do instead: `timestamptz`.
- Exception: Only when compatibility with databases lacking timezone support trumps everything.

### Don't use `timetz`

- Why: The manual itself says it's implemented only for SQL compliance; its semantics are of questionable usefulness.
- Do instead: `date` / `time` / `timestamp` / `timestamptz` cover everything.
- Exception: Never.

### Don't use `CURRENT_TIME`

- Why: Returns a `timetz` value (see above).
- Do instead: `current_timestamp` / `now()` for timestamptz, `localtimestamp` for timestamp, `current_date` for date, `localtime` for time.
- Exception: Never.

### Don't use `timestamp(0)` / `timestamptz(0)`

- Why: A precision specifier rounds instead of truncating: storing `now()` may persist a value up to half a second in the future.
- Do instead: `date_trunc('second', blah)`.
- Exception: Never.

### Don't use `+/-HH:mm` strings as time zone names

- Why: A string offset is parsed as a POSIX time zone spec, where positive shifts west and negative shifts east — inverted from ISO convention.
- Do instead: IANA names (`'Europe/Berlin'`) or abbreviations. For a fixed offset, use an interval: `AT TIME ZONE INTERVAL '04:00'`.
- Exception: ISO-format `timestamptz` literals may carry a signed offset: `'2024-01-31 17:16:25+04'::timestamptz` — parsed per ISO rules.

## Text

### Don't use `char(n)` — including for fixed-length identifiers

- Why: Values are space-padded to width `n`; trailing spaces are ignored in comparisons but significant in `varchar`/`text` — a source of subtle bugs. Padding wastes space and slows operations. It doesn't reject too-short values, so it gives no validation. It also blocks index use when compared against a text-typed driver parameter.
- Do instead: `text` (or a domain over `text`) with a check constraint, e.g. `CHECK (VALUE ~ '^[[:alpha:]]{3}$')` for a 3-letter code — this also validates format.
- Exception: Porting very old software with true fixed-width fields.

### Don't use `varchar(n)` by default

- Why: Same storage and performance as `text`; the only difference is a hard error past `n` characters. An arbitrary limit (e.g. `varchar(20)` for a surname) risks production failures on legitimate input. Length alone is rarely the real constraint.
- Do instead: `text` (or bare `varchar`), and express real constraints with check constraints (min length, allowed characters, etc.).
- Exception: You genuinely want an error on overflow and don't want a check constraint; or you need maximum SQL-standard portability (`varchar` is standard, `text` is not).

## Other data types

### Don't use `money`

- Why: Fixed-point machine int — fast, but no fractional cents, non-obvious rounding, and no stored currency: values are interpreted via `lc_monetary`, so changing the locale silently reinterprets every `money` column (`'$10.00'` may come back as `'10,00 Lei'`).
- Do instead: `numeric`, with the currency in an adjacent column if needed; rarely `integer` (e.g. cents).
- Exception: Single currency, no fractional cents, only addition/subtraction.

### Don't use `serial`

- Why: `serial` has awkward behaviors around schema, dependency, and permission management.
- Do instead: Identity columns: `id bigint generated always as identity` (or `generated by default as identity`).
- Exception: PostgreSQL < 10; shared sequences across tables (better declared explicitly anyway).

## Authentication

### Don't use `trust` authentication over TCP/IP in production

- Why: `trust` accepts any claimed username, including the PostgreSQL superuser. A line like `host all all 0.0.0.0/0 trust` lets anyone on the network become superuser. Even on local Unix sockets in production, anyone with access to the instance can log in as any user.
- Do instead: `scram-sha-256` password auth (PG 10+) for TCP/IP; `peer` auth for local Unix-socket connections on Unix systems.
- Exception: CI/CD test servers on a trusted network; local dev restricted to localhost TCP — and even there, consider `peer`.
