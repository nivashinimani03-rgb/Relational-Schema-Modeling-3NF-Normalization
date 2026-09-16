# RabTech Task 03 — Relational Schema Modeling & 3NF Normalization

An entity-relationship schema for a multi-tenant SaaS application:
8 related tables covering 1:1, 1:N, and N:M relationships, written in
strict 3rd Normal Form, with referential integrity enforced across
tenant boundaries.

## Structure

```
schema.sql                     DDL: 8 tables + PK/FK/CHECK/NOT NULL constraints,
                                plus 3 triggers enforcing cross-tenant integrity
seed_data.sql                  realistic multi-row data across 2 tenants
docs/
  normalization-notes.md       per-table functional-dependency proof of 3NF
  er-diagram.dot                Graphviz source for the ER diagram
  er-diagram.png / .svg         rendered ER diagram image
```

## Domain model

| Table          | Relationship                          |
|----------------|-----------------------------------------|
| `tenants`        | root of the multi-tenant hierarchy    |
| `users`          | 1:N — a tenant has many users         |
| `user_profiles`  | 1:1 — one optional profile per user   |
| `teams`          | 1:N — a tenant has many teams         |
| `team_members`   | N:M junction — users ↔ teams          |
| `projects`       | 1:N tenant → projects, 1:N team → projects (team optional) |
| `tags`           | 1:N — a tenant has many tags          |
| `project_tags`   | N:M junction — projects ↔ tags        |

![ER diagram](docs/er-diagram.png)

## Why triggers, not just constraints

A foreign key guarantees a `team_id` or `tag_id` *exists*, but not that
it belongs to the *same tenant* as the row referencing it — that's a
multi-tenant-specific invariant that spans tables, so it can't be a
plain `CHECK`. Three `BEFORE INSERT/UPDATE` triggers enforce it:

- a project's `team_id` must belong to the project's own `tenant_id`
- a `team_members` row's `team_id` and `user_id` must belong to the same tenant
- a `project_tags` row's `project_id` and `tag_id` must belong to the same tenant

Storing `tenant_id` redundantly on the junction tables was deliberately
avoided — it would be a transitively-derived column and break 3NF (see
`docs/normalization-notes.md`).

## Running it locally (PostgreSQL)

```bash
createdb rabtech_saas
psql -d rabtech_saas -f schema.sql
psql -d rabtech_saas -f seed_data.sql
```

Both files are repeatable (`IF NOT EXISTS`, `CREATE OR REPLACE`,
`DROP TRIGGER IF EXISTS`) — safe to re-run against the same database.

### Regenerating the ER diagram

```bash
cd docs
dot -Tpng er-diagram.dot -o er-diagram.png
dot -Tsvg er-diagram.dot -o er-diagram.svg
```

## Verification

`schema.sql` and `seed_data.sql` were run against a live PostgreSQL 16
instance during development. Confirmed:
- all 8 tables create and seed cleanly with no constraint violations
- a full join across tenants → users → team_members → projects returns
  correct multi-tenant data
- all three cross-tenant triggers correctly **reject** an attempt to
  assign a team, teammate, or tag from one tenant to another tenant's
  project
