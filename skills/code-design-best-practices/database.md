# Database

Conventions for relational schemas, columns, indexes, migrations, and data integrity. ORM-agnostic — applies whether you use Prisma, Drizzle, Knex, raw SQL, etc.

## Aggregate naming alignment

The aggregate name is a single **kebab-case English noun** that is the SAME identifier across three layers:

- **Backend** — folder `src/core/<aggregate>/` and file prefixes `<aggregate>.routes.ts`, `<aggregate>.service.ts`, `<aggregate>.schema.ts`, `<aggregate>.dto.ts`.
- **Frontend** — folder `src/core/<aggregate>/`.
- **Database** — tables of the aggregate are prefixed by `<aggregate>_` (or grouped under a schema named `<aggregate>`).

Example — aggregate `freight`:

| Layer | Path / name |
| --- | --- |
| Backend | `src/core/freight/{freight.routes.ts, freight.service.ts, freight.schema.ts, freight.dto.ts, index.ts}` |
| Frontend | `src/core/freight/{pages/, data/, components/, types/, __tests__/}` |
| Database | tables `freight`, `freight_expense`, `freight_event` |

**Before creating a new aggregate, the agent MUST ask the user for the aggregate name** and confirm it before touching any file. The confirmed name is reused verbatim. Never let the name drift across layers.

## Table & column naming

- Table names in **snake_case**, **singular** (`freight`, not `freights`).
- Aggregate children prefixed by the aggregate name: `freight_expense`, `freight_event`.
- Join tables: `<left>_<right>` in alphabetical order (`role_user`, not `user_role`) — unless the relationship has its own name (then use the name, e.g. `membership`).
- Column names in **snake_case** at the DB; ORM maps to **camelCase** in the application. The application identifier is the source of truth in code (`createdAt`, never `created_at` in DTOs).
- Booleans named affirmatively with `is_`/`has_`/`should_` prefix (`is_active`, `has_paid`, never `not_disabled`).

## Primary keys

- Every table has `id` as primary key — **string UUID** (`uuid v4` or `v7`/ULID for time-ordered).
- Never auto-increment integers — they leak count signals, complicate sharding, and break across environments.
- Surrogate keys only. Even if a natural key exists (e.g. `email`), use `id` and put the natural key under a UNIQUE index.

## Audit fields

Every business table carries:

- `created_at` — `timestamptz NOT NULL DEFAULT now()`.
- `updated_at` — `timestamptz NOT NULL DEFAULT now()`, auto-updates on write (trigger or ORM hook).

Optional, when the product needs them:

- `created_by` — FK to `user.id`, set by the service.
- `updated_by` — FK to `user.id`, set by the service.
- `deleted_at` — `timestamptz NULL` (see Soft delete below).

## Soft delete

- Default: **hard delete**. Soft delete is opt-in per table, justified by a product/audit need.
- Soft-deletable tables add `deleted_at timestamptz NULL` and the query layer filters `deleted_at IS NULL` by default.
- Soft-deleted rows MUST still satisfy uniqueness — design unique indexes as partial: `CREATE UNIQUE INDEX ON <table>(<col>) WHERE deleted_at IS NULL`.

## Foreign keys & relationships

- FK columns named `<entity>_id` (DB) / `<entity>Id` (DTO) — e.g. `freight_id` / `freightId`.
- Every FK is **indexed** — non-negotiable; absence is a guaranteed N+1/slow-join in production.
- Declare `ON DELETE` behavior explicitly per FK:
  - `CASCADE` when the child has no meaning without the parent (e.g. `freight_expense → freight`).
  - `RESTRICT` (default) when deletion should fail loudly.
  - `SET NULL` only when the child legitimately survives the parent.
- No cross-aggregate FK without conscious decision — prefer event tables or denormalized references when the aggregates are independent.

## Nullability

- Nullable columns are **explicit and intentional**. Default to `NOT NULL`.
- DTO/types shape mirrors the column: `<field>: <Type> | null` when nullable, `<field>: <Type>` when not.
- Never use sentinel values (`-1`, `''`, `'unknown'`) to mean "missing". Use `NULL`.

## Enums

- Stored as `text` constrained by a `CHECK` or a Postgres `enum` type.
- The **application's union type is the source of truth** (`type ExpenseType = 'toll' | 'food' | 'fuel'`). DB constraint mirrors it.
- Adding/renaming values requires a migration AND a code change shipped together.

## Money & quantities

- Money: **integer cents** (`bigint`) + a `currency` column (ISO 4217 code, `text NOT NULL`).
- Never `float`/`double` for money. Never store money as `numeric` without testing rounding behavior.
- Quantities with decimals: `numeric(precision, scale)` with explicit precision — never `float`.

## Dates & times

- Always `timestamptz` (with timezone). Never `timestamp` (naive) or `date` when a moment-in-time is meant.
- Date-only domain concepts (e.g. `birthDate`) use `date`.
- Application layer uses a date library (`date-fns`/`dayjs`/`Temporal`) — no raw `new Date(...)` math.
- API/DTO format: ISO 8601 strings (`2026-05-17T12:34:56Z`).

## Indexes

- Every FK indexed (already stated).
- Every column appearing in a `WHERE` filter of a common query indexed.
- Compound indexes when queries filter on multiple columns — order matters (most selective first).
- Index `created_at` on append-heavy tables that are queried by recency.
- `EXPLAIN ANALYZE` before adding/removing an index on a busy table.

## Constraints & integrity

- Push invariants into the schema when possible: `CHECK (amount >= 0)`, `UNIQUE (email)`, `NOT NULL`.
- Don't rely on the application to enforce uniqueness — race conditions will bite. Always pair with a DB-level UNIQUE.
- Foreign keys enforce referential integrity — never simulate FKs at the application layer.

## Metrics in dedicated event tables

Metric-shaped values (`cost`, `tokens`, `duration`, `model`, `latency`, `bytes_processed`) NEVER live on a domain entity. Create a parallel event table named `<aggregate>_event` keyed by entity ID + an event kind column. See [backend.md → "Metrics live in dedicated tables"](./backend.md#metrics-live-in-dedicated-tables).

## Transactions

- Multi-step writes that share an invariant run inside a transaction.
- Transactions open in the service layer; never split a single business operation across multiple transactions.
- Don't hold transactions across network calls (HTTP, message queue) — commit first, then call out.

## Migrations

- **Forward-only** — no `DOWN` migrations. Roll forward to fix.
- One concern per migration: schema OR data backfill, not both in the same file.
- Migrations are **immutable once shipped** — never edit a merged migration; write a new one.
- Destructive changes (DROP COLUMN, DROP TABLE, adding NOT NULL to an existing column, narrowing an enum) ship in **two deploys**:
  1. Stop writing/reading the column in code → release.
  2. Apply destructive migration → release.
- Backfills are separate migrations after the schema change; large backfills run in batches with progress logging.
- Adding indexes on busy tables: build CONCURRENTLY (Postgres) to avoid blocking writes.

## Seeding & test data

- Production seeds (lookup tables, default roles/permissions) live in idempotent migrations or a dedicated `seed.ts` — re-runnable without breaking.
- Test fixtures live in test helpers (`makeUser()`, `makeFreight()` factories) — not in migrations.
- Service tests run against an **isolated test database**, never the dev DB. See [tests.md](./tests.md).

## Database anti-patterns

1. Different aggregate name across backend / frontend / database (`freight` in code, `fretes` in DB, `freights` in routes).
2. Creating files before the aggregate name has been confirmed with the user.
3. Auto-increment integer primary keys.
4. Plural or mixed-case table names.
5. Missing audit fields (`created_at` / `updated_at`).
6. Unindexed foreign keys.
7. Soft delete without partial unique indexes (uniqueness breaks on re-insert).
8. Sentinel values instead of `NULL`.
9. Floating-point money.
10. Naive `timestamp` without timezone.
11. Metric-shaped fields on a domain entity.
12. Application-only uniqueness without a DB-level UNIQUE.
13. `DOWN` migrations, or editing a merged migration.
14. Destructive schema changes in a single deploy.
15. Transactions held across network calls.
