# Flyway

Reach for this for schema migrations. Hibernate never owns the schema - `ddl-auto: validate`.

## Naming & location

```
src/main/resources/db/migration/
  V1__create_orders.sql
  V2__add_status_to_orders.sql
  V3__create_order_lines.sql
  R__order_summary_view.sql
```

- **Versioned**: `V<version>__<description>.sql` - run once, in order, tracked by checksum.
- Double underscore `__` between version and description. Version can be `V1`, `V1_1`, `V2024_01_15`.
- Descriptions become the migration name in history - keep them meaningful.

## Versioned vs repeatable

| Prefix | When it runs | Use for |
|---|---|---|
| `V` | once, in version order | tables, columns, constraints, data changes |
| `R` | whenever its checksum changes (after all `V`) | views, stored procedures, functions |
| `U` | manual undo (usually avoid) | rollback scripts |

```sql
-- V2__add_status_to_orders.sql
ALTER TABLE orders ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'PENDING';

-- R__order_summary_view.sql  (re-applied when edited)
CREATE OR REPLACE VIEW order_summary AS
SELECT customer_id, COUNT(*) AS orders, SUM(amount) AS total
FROM orders GROUP BY customer_id;
```

## `ddl-auto: validate` (Hibernate validates, never migrates)

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate     # fail startup if entities don't match the schema
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
```

Flyway owns the schema; Hibernate only checks its entities line up. `validate` (or `none`) in **every** environment - never `update`/`create`.

## `flyway_schema_history`

Flyway records every applied migration (version, description, checksum, success) in this table.
On startup it compares files to history:
- New `V` files → applied in order.
- Changed checksum on an already-applied `V` → **startup fails** (see gotchas).

## Baseline (adopting Flyway on an existing DB)

```yaml
spring:
  flyway:
    baseline-on-migrate: true
    baseline-version: 1        # treat the current schema as version 1
```

Then new migrations start at `V2`. Only for pre-existing databases without a history table.

## Testcontainers + Flyway

Flyway runs automatically at context startup, so integration tests get the real, migrated
schema against a real DB:

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryIntegrationTest {
    @Container @ServiceConnection
    static PostgreSQLContainer postgres =
        new PostgreSQLContainer(DockerImageName.parse("postgres:17-alpine"));
    // Flyway applies V1..Vn before tests run - no manual schema setup
}
```

See the [Testcontainers sheet](../java/testcontainers.md).

## Useful commands (Maven / CLI)

```bash
./mvnw flyway:info        # show applied + pending migrations
./mvnw flyway:migrate     # apply pending
./mvnw flyway:validate    # verify checksums match
./mvnw flyway:repair      # fix the history table after a failed/edited migration
```

## Gotchas / things I always forget

- **Never edit a `V` migration that's already been applied.** The checksum changes → Flyway fails at startup. Write a **new** migration instead.
- One logical change per file - easier to reason about and to roll forward.
- `ddl-auto` must be `validate` or `none`. `update`/`create` lets Hibernate silently diverge from your migrations.
- Migrations run in a transaction (on Postgres) - but some DDL (e.g. `CREATE INDEX CONCURRENTLY`) can't run in one; split those out.
- Repeatable (`R__`) migrations re-run on any checksum change - great for views/procs, but they run **after** all versioned ones.
- A failed migration leaves a `success = false` row; fix the SQL then `flyway:repair` before retrying.
- Don't put environment-specific data in migrations - keep them deterministic across all envs.
- Flyway applies in **version order**, not filename order - `V10` runs after `V9`, but `V1_1` sorts before `V1_2`; be consistent.

## Quick reference

| Thing | Value |
|---|---|
| Location | `classpath:db/migration/` |
| Versioned | `V1__desc.sql` (run once, ordered) |
| Repeatable | `R__desc.sql` (re-run on change) |
| Hibernate setting | `ddl-auto: validate` |
| History table | `flyway_schema_history` |
| Edited-applied migration | never - write a new one |
| Adopt existing DB | `baseline-on-migrate: true` |
| Fix failed run | `flyway:repair` |
