# PostgreSQL Locks — what each migration operation takes

Why this matters: migrations run against a live DB. The lock an operation takes decides
whether it's invisible to traffic, blocks writes, or blocks *everything* (including
`SELECT`). The two dangerous facts:

1. **`ACCESS EXCLUSIVE` blocks every other query on the table**, including reads.
2. **A blocked DDL statement queues behind running queries, and everything else queues
   behind the DDL.** So even a *fast* `ACCESS EXCLUSIVE` operation can stall the table if a
   long-running query holds a conflicting lock first — the DDL waits, and new queries pile
   up behind it. This is why `lock_timeout` matters: it makes the DDL give up quickly
   instead of forming a queue.

## Lock levels (weakest → strongest)

| Lock | Conflicts with | Taken by |
|------|----------------|----------|
| `ACCESS SHARE` | `ACCESS EXCLUSIVE` | `SELECT` |
| `ROW SHARE` | `EXCLUSIVE`, `ACCESS EXCLUSIVE` | `SELECT FOR UPDATE` |
| `ROW EXCLUSIVE` | `SHARE`+ | `INSERT`/`UPDATE`/`DELETE` |
| `SHARE UPDATE EXCLUSIVE` | itself + `SHARE`+ | `CREATE INDEX CONCURRENTLY`, `DROP INDEX CONCURRENTLY`, `VALIDATE CONSTRAINT`, `ANALYZE` |
| `SHARE` | writes | `CREATE INDEX` (non-concurrent) |
| `SHARE ROW EXCLUSIVE` | writes + itself | rare (`CREATE TRIGGER`, some `ALTER`) |
| `EXCLUSIVE` | reads-for-update + writes | rare |
| `ACCESS EXCLUSIVE` | **everything** | most `ALTER TABLE`, non-concurrent `DROP INDEX`, `ADD CONSTRAINT` (validating), `TRUNCATE`, `VACUUM FULL` |

`SHARE UPDATE EXCLUSIVE` is the sweet spot for online DDL: it blocks other DDL/vacuum but
**not** reads or writes.

## Per-operation lock reference

| Operation | Lock | Table rewrite? | Table scan? | Blocks reads? | Blocks writes? |
|-----------|------|----------------|-------------|---------------|----------------|
| `ADD COLUMN`, nullable, no default | `ACCESS EXCLUSIVE` (brief, metadata only) | no | no | briefly | briefly |
| `ADD COLUMN` with **constant** default (PG 11+) | `ACCESS EXCLUSIVE` (brief) | no | no | briefly | briefly |
| `ADD COLUMN` with **volatile/computed** default | `ACCESS EXCLUSIVE` held for rewrite | **yes** | yes | **whole rewrite** | **whole rewrite** |
| `DROP COLUMN` | `ACCESS EXCLUSIVE` (brief; data reclaimed later by vacuum) | no | no | briefly | briefly |
| `ALTER COLUMN TYPE` (most) | `ACCESS EXCLUSIVE` held for rewrite | **usually yes** | yes | **whole rewrite** | **whole rewrite** |
| `ALTER COLUMN SET NOT NULL` (direct) | `ACCESS EXCLUSIVE` | no | **yes (full scan)** | during scan | during scan |
| `ADD CONSTRAINT CHECK ... NOT VALID` | `ACCESS EXCLUSIVE` (brief, no scan) | no | no | briefly | briefly |
| `VALIDATE CONSTRAINT` | `SHARE UPDATE EXCLUSIVE` | no | yes | no | no |
| `SET NOT NULL` backed by a validated `CHECK (col IS NOT NULL)` (PG 12+) | `ACCESS EXCLUSIVE` (brief, **scan skipped**) | no | no | briefly | briefly |
| `ADD FOREIGN KEY` (one shot) | `ACCESS EXCLUSIVE` on **both** tables + validation scan | no | yes | during scan | during scan |
| `ADD FOREIGN KEY ... NOT VALID` then `VALIDATE` | brief `ACCESS EXCLUSIVE`, then `SHARE UPDATE EXCLUSIVE` | no | yes (in validate) | no | no |
| `CREATE INDEX` (non-concurrent) | `SHARE` | no | reads table | no | **yes, whole build** |
| `CREATE INDEX CONCURRENTLY` | `SHARE UPDATE EXCLUSIVE` | no | 2 scans | no | no |
| `DROP INDEX` (non-concurrent) | `ACCESS EXCLUSIVE` (brief) | no | no | briefly | briefly |
| `DROP INDEX CONCURRENTLY` | `SHARE UPDATE EXCLUSIVE` | no | no | no | no |
| `ADD CONSTRAINT UNIQUE` (builds index inline) | `ACCESS EXCLUSIVE` held for the build | no | builds index | **whole build** | **whole build** |
| `ADD CONSTRAINT UNIQUE USING INDEX <prebuilt>` | `ACCESS EXCLUSIVE` (brief) | no | no | briefly | briefly |
| `ADD PRIMARY KEY USING INDEX` | `ACCESS EXCLUSIVE` (brief) + the `NOT NULL` requirement | no | no | briefly | briefly |
| `DROP CONSTRAINT` | `ACCESS EXCLUSIVE` (brief) | no | no | briefly | briefly |
| `TRUNCATE` | `ACCESS EXCLUSIVE` | n/a | no | yes | yes |

## Rules that fall out of this table

- **Prefer operations that only need a brief metadata lock** (nullable ADD COLUMN, NOT
  VALID constraints, attach-prebuilt-index) and always give them a small `lock_timeout` so
  they don't queue behind a long query.
- **Anything that scans or rewrites the whole table** (SET NOT NULL direct, ALTER TYPE,
  inline UNIQUE/FK validation, non-concurrent index) must be converted to a concurrent /
  two-step form, or done as add-new-column + backfill + swap across releases.
- **`CONCURRENTLY` operations cannot run inside a transaction** → `atomic = False`, no
  timeouts, one per migration.
- **PG version note:** the "constant default = no rewrite" (PG 11+) and "validated CHECK
  lets SET NOT NULL skip the scan" (PG 12+) optimizations are why these patterns lean
  on NOT VALID/VALIDATE and nullable-then-enforce.
