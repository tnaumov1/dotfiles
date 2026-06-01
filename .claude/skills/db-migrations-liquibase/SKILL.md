---
name: db-migrations-liquibase
description: Author Liquibase SQL changelogs. Use when adding/modifying database schema, creating tables, indexes, or seed data via Liquibase.
---

# Liquibase Migration Conventions

## Format & Location

- Write changelogs in **SQL** (not XML/YAML).
- Place under `src/main/resources/db/changelog/`.
- Version into subdirectories: `v0.0.1/`, `v0.1.0/`, etc.
- Name files like `001-init-schema.sql`, `002-add-bookings.sql` (zero-padded ordinal + kebab description).

## SQL Changelog Structure

Each file uses Liquibase formatted-SQL headers:

```sql
--liquibase formatted sql

--changeset author:001-init-schema
CREATE TABLE master (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name        TEXT        NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

--changeset author:002-add-index
CREATE INDEX idx_master_name ON master (name);
```

- One `--changeset` per logical change. Never edit a deployed changeset — add a new one.
- Use stable, descriptive changeset IDs matching the file ordinal where possible.

## Master Changelog

Reference new versioned directories from the root master changelog (typically `db.changelog-master.yaml`) using `includeAll`:

```yaml
databaseChangeLog:
  - includeAll:
      path: db/changelog/v0.0.1/
```
