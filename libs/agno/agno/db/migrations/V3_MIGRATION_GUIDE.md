# Agno v3.0 Storage Migration Guide: Normalized Runs Table

## Overview

Agno v3.0 changes how session runs are stored. Runs are no longer kept as a JSON
blob inside the sessions table — each run is now stored as its own row in a dedicated
runs table (`agno_runs` by default).

### The problem (v2.x)

In v2.x, the `agno_sessions.runs` column held the full list of runs (with all their
messages) as a single JSON value:

- **Write amplification**: every save rewrote the entire runs blob. Saving run N wrote
  runs 1..N again, so the total bytes written over a session's life grew quadratically.
- **Unbounded row size**: long-lived sessions reached tens or hundreds of MB in a single
  row, slowing down every read/write and eventually causing upsert failures.
- **No partial reads**: fetching the last few runs required loading and parsing the
  entire blob.

### The solution (v3.0)

Runs are stored one-row-per-run in the runs table:

```
agno_runs
├── run_id        TEXT PRIMARY KEY
├── session_id    TEXT NOT NULL (indexed)
├── run_type      TEXT NOT NULL  -- "agent" | "team" | "workflow"
├── agent_id      TEXT (indexed)
├── team_id       TEXT (indexed)
├── workflow_id   TEXT (indexed)
├── user_id       TEXT (indexed)
├── parent_run_id TEXT           -- set for team member runs
├── status        TEXT (indexed)
├── run_index     BIGINT         -- position of the run within its session
├── run_data      JSONB / JSON   -- the full run payload (messages, tools, metrics, ...)
└── created_at / updated_at
```

- Each run is written once (plus an update when it changes, e.g. paused → completed).
  Saving a new run no longer touches previous runs.
- Session rows stay small.
- Runs can be queried directly (by session, agent, status, ...) without loading sessions.

The fields you filter on (`status`, `agent_id`, `session_id`, ...) are real columns; the
run payload stays as a single JSON value because a run is always read and written as a
unit.

## Supported databases

v3.0 normalized run storage is implemented for:

- `PostgresDb` and `AsyncPostgresDb`
- `SqliteDb` and `AsyncSqliteDb`
- `MySQLDb` and `AsyncMySQLDb`
- `SingleStoreDb`
- `MongoDb` and `AsyncMongoDb` (uses a separate ``agno_runs`` collection)
- `FirestoreDb` (uses a separate ``agno_runs`` collection)
- `RedisDb` (uses ``<prefix>:runs:<run_id>`` keys plus a per-session sorted-set index)
- `ValkeyDb` (uses ``<prefix>:runs:<run_id>`` keys plus a per-session sorted-set index)
- `DynamoDb` (uses a separate ``agno_runs`` table with a ``session_id-created_at`` GSI)
- `SurrealDb` (uses a separate ``agno_runs`` table)
- `JsonDb` (uses a separate ``agno_runs.json`` file)
- `GcsJsonDb` (uses a separate ``agno_runs.json`` object in the bucket)

`InMemoryDb` and `ClickHouse` continue to store runs inline in the session. The
in-memory adapter does not benefit from the split (no I/O, no row-size limits),
and ClickHouse will be addressed in a follow-up release.

### MongoDB note

For MongoDB the v3.0 storage shape is a dedicated ``agno_runs`` collection with one
document per run, with the same fields as the SQL ``agno_runs`` table. The
``run_data`` field is a nested document (not flattened) to match the SQL design.
The legacy ``runs`` field on session documents is preserved by the migration; call
``db.cleanup_legacy_runs_field()`` (rather than ``cleanup_legacy_runs_column()``)
to unset it when you have verified the migration.

### Firestore note

Firestore's 1 MB document size limit is the *hard* version of MongoDB's 16 MB
limit — long sessions can simply fail to save in v2.x. v3.0 puts each run in its
own document in a dedicated ``agno_runs`` collection. The legacy ``runs`` field on
session documents is preserved by the migration; ``db.cleanup_legacy_runs_field()``
removes it once verified (uses Firestore ``DELETE_FIELD``).

### Redis note

Redis stores each run as a separate key (``<prefix>:runs:<run_id>``) and maintains
a per-session sorted set (``<prefix>:runs:by_session:<session_id>``) scored by
``run_index`` for cheap ordered reads. ``get_runs(session_id=...)`` is a
``ZRANGE`` + ``MGET`` round-trip rather than a full scan. The legacy ``runs``
field on the session record is preserved by the migration; call
``db.cleanup_legacy_runs_field()`` to drop it once verified.

### Valkey note

Valkey stores each run as a separate key (``<prefix>:runs:<run_id>``) and maintains
a per-session sorted set (``<prefix>:runs:by_session:<session_id>``) scored by
``run_index`` for cheap ordered reads. ``get_runs(session_id=...)`` is a ``ZRANGE``
plus one batched read of those keys rather than a full scan; the batch is issued
through the GLIDE client, which pipelines the whole page in a single round trip.
The legacy ``runs`` field on the session record is preserved by the migration;
call ``db.cleanup_legacy_runs_field()`` to drop it once verified.

### DynamoDB note

DynamoDB has a hard 400 KB per-item limit. The v2.x design — embedding the full
runs list in the session item — could simply fail to write for long sessions.
v3.0 puts each run in its own item in a dedicated ``agno_runs`` table with a
``session_id-created_at-index`` GSI for ordered reads. The legacy ``runs``
attribute on the session item is preserved by the migration; call
``db.cleanup_legacy_runs_field()`` to remove it once verified.

### SingleStore note

SingleStore requires every unique key to contain the shard-key columns, so the runs
table uses ``PRIMARY KEY (run_id, session_id)`` with ``SHARD KEY (session_id)``; a
single-column ``PRIMARY KEY (run_id)`` is rejected with ``ERROR 1744``, so no
deployment can have held the narrower key and there is nothing to migrate. The
consequence is that ``run_id`` alone is not unique here: the same ``run_id`` under two
``session_id``s leaves both rows in place, ``get_run`` returns one of them unordered,
``upsert_run`` inserts instead of updating, and ``delete_run`` removes every copy
across sessions. In normal use a ``run_id`` belongs to one session; the other SQL
backends keep ``PRIMARY KEY (run_id)``.

### SurrealDB note

SurrealDB stores each run in a dedicated ``agno_runs`` table with indexes on the
identity fields. The legacy ``runs`` field on session records is preserved by
the migration; call ``db.cleanup_legacy_runs_field()`` to drop it once verified.

### JSON / GCS-JSON note

The file-backed adapters keep runs in a sibling JSON file (``agno_runs.json``).
This brings API parity with the SQL/NoSQL adapters — same ``get_run`` /
``get_runs`` / ``delete_run`` surface — but it does **not** lift the underlying
I/O cost: every write still rewrites a single JSON file. These adapters are for
local development; for production workloads use one of the SQL or NoSQL
adapters. The legacy ``runs`` field is preserved by the migration and removed
via ``db.cleanup_legacy_runs_field()``.

## Migrating existing data

The migration is intentionally **non-destructive**. It creates the runs table and
copies every legacy run into it, but **leaves the legacy `runs` column on the sessions
table untouched** as a safety net. Writes never null that column either — it stays as
a frozen backup so that upgrading before running the migration can't lose history.
Once you have verified things, you drop the column manually (with `force=True`).

### Step 1: Run the v3.0.0 migration

```python
import asyncio

from agno.db.migrations.manager import MigrationManager
from agno.db.postgres import PostgresDb

db = PostgresDb(db_url="postgresql+psycopg://...")

# Copies every run from agno_sessions.runs into agno_runs.
# Does NOT touch the legacy column.
asyncio.run(MigrationManager(db).up())
```

The migration is idempotent — re-runs use `ON CONFLICT DO NOTHING` and skip rows
that already exist. The legacy `runs` column is preserved so you can sanity-check
the migrated data against the original.

### Step 2 (optional, recommended): Lazy migration also works

You don't strictly *need* to run the migration before upgrading. v3.0 works against
an unmigrated database:

- **Reads** load runs from the runs table and **merge** them with anything still in
  the legacy `runs` column (by `run_id`). The runs table is the source of truth on
  conflicts; runs that only exist in the blob are still returned. This means you
  never lose history, even in partial-migration states.
- **The first save** of any session moves its remaining legacy runs into the runs
  table and clears the legacy column for that session.

This means active sessions self-migrate. The explicit migration is recommended for
dormant sessions and for reclaiming storage in bulk.

### Step 3: Drop the legacy column when you're ready

Once you have verified the migration and taken a backup, drop the legacy column to
reclaim the storage:

```python
db.cleanup_legacy_runs_column()
```

This refuses to drop the column if any session still has non-null legacy `runs`
content (a sign that that session was not migrated). If you really want to force
it anyway:

```python
db.cleanup_legacy_runs_column(force=True)
```

Async adapters expose the same helper as `await db.cleanup_legacy_runs_column()`.

### Reverting

To roll back to v2.5.6 (rebuilds the blobs from the runs table and drops the runs
table):

```python
asyncio.run(MigrationManager(db).down(target_version="2.5.6"))
```

## Eval runs: per-user isolation

The same `v3.0.0` migration adds an indexed `user_id` column to the eval runs table
(`agno_eval_runs` by default). It backs per-user isolation in AgentOS: with
`user_isolation` enabled a caller sees only their own eval runs; admins and unscoped
deployments see everything.

```
agno_eval_runs
├── run_id      TEXT PRIMARY KEY
├── ...
└── user_id     TEXT (indexed)   -- NULL for runs created before v3.0
```

Existing rows keep a `NULL` `user_id` — nothing is deleted or reassigned. An unowned
run stays visible to unscoped and admin callers and invisible to a scoped one. New
runs are stamped with their owner on write.

Only the seven SQL adapters (`PostgresDb`, `AsyncPostgresDb`, `SqliteDb`,
`AsyncSqliteDb`, `MySQLDb`, `AsyncMySQLDb`, `SingleStoreDb`) need schema work. The
rest store an eval run as a record and carry `user_id` with no schema change, so
`MigrationManager` logs `No version found for table agno_eval_runs` and moves on.
`ClickhouseDb` is traces-only and stores no eval runs.

It runs with the rest of the version — `MigrationManager(db).up()` — or on its own:

```python
asyncio.run(MigrationManager(db).up(table_type="evals"))
```

The column and index are added only when missing, so re-runs are safe; a table whose
adapter schema does not declare `user_id` is logged and skipped rather than failing
the run.

Reverting drops the index and then the column. Every row survives, but **the `user_id`
values are destroyed** — re-running `up()` restores the column with `NULL` everywhere,
not the previous owners. Back up first if the ownership data matters:

```python
asyncio.run(MigrationManager(db).down(target_version="2.5.6", table_type="evals"))
```

SQLite reverts need SQLite 3.35+ for `ALTER TABLE ... DROP COLUMN`; on older builds
the revert logs and skips, matching `v2.5.6`'s behaviour.

## Metrics: per-user buckets

Metrics get the same `user_id` column, and one thing none of the other tables need:
**its unique key changes**.

```
agno_metrics
├── id                  TEXT PRIMARY KEY
├── date                DATE (indexed)
├── aggregation_period  TEXT             -- "daily"
├── user_id             TEXT NOT NULL    -- new in v3.0; "" = unowned
├── ...                                  -- run / session counts, token and model metrics
└── UNIQUE (user_id, date, aggregation_period)   -- was UNIQUE (date, aggregation_period)
```

v3.0 stores one metrics row per user per day instead of one row per day, so `user_id`
has to be part of the key. Until it is, an existing table is not just missing a column —
it is **broken in two ways**, and neither of them is loud:

1. **The column is missing**, so `is_valid_table` rejects the table. What the caller
   sees depends on the backend. On SQLite and Postgres `GET /metrics` answers HTTP 500
   with `Table agno_metrics has an invalid schema`. On MySQL the adapter logs the same
   error and returns nothing, so the route answers HTTP 200 with
   `{"metrics": [], "updated_at": null}`. On SingleStore the check never fires at all —
   SQLAlchemy cannot reflect the table (its JSON columns come back as a type it does not
   recognise), and an inspection failure is treated as "valid" — so the caller gets the
   driver's `(1054, "Unknown column 'user_id' in 'field list'")` instead.

2. **The key is still the legacy one**, even once the column has been added by hand.
   Postgres and SQLite name `(user_id, date, aggregation_period)` as the upsert's
   conflict target and there is no such key, so every recalculation fails: an explicit
   refresh returns HTTP 500, while `GET /metrics` keeps answering 200 with whatever was
   already stored and only logs the failure. MySQL is quieter still —
   `ON DUPLICATE KEY UPDATE` matches whichever unique key it finds, so the write succeeds
   against the legacy one and the second owner's numbers land on the first owner's row.
   One row survives for the day, under the first owner's name, and nothing errors.

Either way, what an operator notices is that metrics stop moving, not that something
failed. On the SQL adapters — the only ones with a schema to fix, see below — the
migration does both halves:

```python
asyncio.run(MigrationManager(db).up(table_type="metrics"))
```

If `up()` already ran on this deployment before the metrics migration existed, the
metrics table may carry a `3.0.0` stamp from that run (older builds stamped tables the
migration never touched) and the call above will skip it. Pass `force=True` once —
the migration's own existence checks make the re-run safe:

```python
asyncio.run(MigrationManager(db).up(table_type="metrics", force=True))
```

Unlike the other isolated tables, existing metrics rows are stamped with `""`, not
`NULL` — SQL treats every `NULL` as distinct, which would silently break a unique key
containing the column. `""` is the "unowned" bucket: it is what pre-isolation history
is, and the adapter maps it back to `None` on the way out, so API consumers never see it.
The rows are not split retroactively per user — a metrics row does not record which
sessions fed it.

**The boundary day is cleared.** As the column lands, the migration deletes one row:
the *unowned* `completed = false` row with the newest date, and only when that date
sits past every completed day. Such a row holds the whole day's traffic for *every*
user; stamped unowned it would become a bucket the per-user recalculation never
rewrites, and the day would be counted once per user and once again in that leftover
row for good. Nothing is lost — the recalculation is certain to revisit that day and
rebuilds it from its sessions on the next refresh. The one exception is a deployment
that prunes that day's sessions *before* migrating: the recalculation then has nothing
to compute from, produces no record, and the day stays gone. Migrate first, prune
after. Every other unfinished row stays,
owned or not: a deeper unfinished day may have had its sessions pruned since, and its
metric row is then the only record of that day, so a stale-but-present row beats a
deleted one. And the delete is restricted to unowned rows (the `""` sentinel or a
hand-patched `NULL`) and only runs when the column or the key actually changed, so a
re-run against an already-migrated table cannot remove per-user buckets. Completed
days are frozen and are left exactly as they are.

**SQLite rebuilds the table.** SQLite writes a unique constraint into the `CREATE TABLE`
statement and has no `ALTER TABLE ... DROP CONSTRAINT`, so the migration renames the
table aside, creates the v3.0 shape from the adapter's own schema, copies the rows in
and drops the old one. The whole rebuild is one transaction: if it is interrupted the
original table is left exactly as it was, with nothing renamed aside and nothing to
clean up. Columns and indexes you added yourself are carried across.

**SingleStore keeps no unique constraint.** A columnstore table may carry only one
`UNIQUE` index once any of them spans multiple columns, and the `id` primary key already
is one, so declaring the triple fails with error 1706 — which also means SingleStore
never had the legacy key and there is nothing to swap. It gets the column and its index
and nothing else. Uniqueness on the triple lives in `bulk_upsert_metrics` instead, as a
select-then-write that is not atomic, so two refreshes running at once can write the
same bucket twice.

**Which adapters need work.** As with eval runs, schema-level work is only needed on the
seven SQL adapters (`PostgresDb`, `AsyncPostgresDb`, `SqliteDb`, `AsyncSqliteDb`,
`MySQLDb`, `AsyncMySQLDb`, `SingleStoreDb`) — they get the column, the key swap and the
revert refusal. The other nine (`MongoDb`, `FirestoreDb`, `DynamoDb`, `SurrealDb`,
`RedisDb`, `ValkeyDb`, `JsonDb`, `GcsJsonDb`, `InMemoryDb`) store a metrics record as a
document and pick `user_id` up without a schema change — and they cannot run this
migration at all: none of them keeps a per-table schema version, so `MigrationManager`
logs `No version found for table agno_metrics` and moves on before dispatching. That is
not specific to metrics; the sessions migration behaves the same way on those backends.

What repairs them instead is the recalculation. `calculate_metrics` is authoritative for
every `(date, aggregation_period)` pair it writes records for: around the write it clears
any *other* owner's record for those pairs. So the single pre-v3.0 record for the day of
the upgrade is removed on the first refresh afterwards rather than being summed on top of
the new per-user records for good, and an upgraded deployment converges on its own.
Completed days from before the upgrade fall outside the recalculation window and stay as
one unowned bucket per day — which is exactly what the SQL migration leaves behind too.
MongoDB needed one thing more: its pre-v3.0 `date_1_aggregation_period_1` unique index
would reject every per-user document after the first for a date, so the collection's
index setup now drops it.

One more thing worth an operator's time. On Postgres and MySQL the migrated `user_id`
carries a server `DEFAULT ''`, because `ADD COLUMN ... NOT NULL` needs one on a populated
table; SQLite's is rebuilt straight from the schema and has none. So an `INSERT` that
omits `user_id` lands in the unowned bucket on Postgres and MySQL and fails with
`NOT NULL constraint failed` on SQLite.

### Reverting metrics

Reverting drops the column and puts the legacy key back — but **only while every row is
still unowned**. Once metrics have been collected per user, dropping `user_id` would
merge two owners' buckets for a date into duplicate rows that the legacy key cannot even
accept, so the revert refuses and logs instead:

```
Skipping revert of agno_metrics: it holds per-user metric rows, and dropping user_id
would merge them into duplicates for the same date. Consolidate or delete the owned
rows first.
```

Deleting the owned rows throws the per-user history away. To consolidate instead — one
row per date and period, carrying the summed numbers — run this first. It is written for
Postgres; the other SQL backends need the same shape with their own JSON functions.

```sql
BEGIN;

CREATE TEMP TABLE agno_metrics_merged AS
WITH tokens AS (
    SELECT date, aggregation_period, jsonb_object_agg(key, total) AS token_metrics
    FROM (
        SELECT m.date, m.aggregation_period, t.key, sum((t.value #>> '{}')::bigint) AS total
        FROM agno_metrics m, jsonb_each(m.token_metrics) t
        GROUP BY 1, 2, 3
    ) s
    GROUP BY 1, 2
), models AS (
    SELECT date, aggregation_period, jsonb_agg(jsonb_build_object(
               'model_id', model_id, 'model_provider', model_provider, 'count', total)) AS model_metrics
    FROM (
        SELECT m.date, m.aggregation_period,
               e ->> 'model_id' AS model_id, e ->> 'model_provider' AS model_provider,
               sum((e ->> 'count')::bigint) AS total
        FROM agno_metrics m, jsonb_array_elements(
                 CASE WHEN jsonb_typeof(m.model_metrics) = 'array' THEN m.model_metrics ELSE '[]'::jsonb END) e
        GROUP BY 1, 2, 3, 4
    ) s
    GROUP BY 1, 2
)
SELECT min(m.id) AS id, m.date, m.aggregation_period,
       sum(m.agent_runs_count)        AS agent_runs_count,
       sum(m.team_runs_count)         AS team_runs_count,
       sum(m.workflow_runs_count)     AS workflow_runs_count,
       sum(m.agent_sessions_count)    AS agent_sessions_count,
       sum(m.team_sessions_count)     AS team_sessions_count,
       sum(m.workflow_sessions_count) AS workflow_sessions_count,
       -- one owned bucket is one user; a pre-v3.0 row already counted the day's users
       greatest(count(*) FILTER (WHERE m.user_id IS DISTINCT FROM ''),
                max(m.users_count) FILTER (WHERE m.user_id = '')) AS users_count,
       coalesce(t.token_metrics, '{}'::jsonb) AS token_metrics,
       coalesce(md.model_metrics, '[]'::jsonb) AS model_metrics,
       min(m.created_at) AS created_at,
       max(m.updated_at) AS updated_at,
       bool_and(m.completed) AS completed
FROM agno_metrics m
LEFT JOIN tokens t ON t.date = m.date AND t.aggregation_period = m.aggregation_period
LEFT JOIN models md ON md.date = m.date AND md.aggregation_period = m.aggregation_period
GROUP BY m.date, m.aggregation_period, t.token_metrics, md.model_metrics;

DELETE FROM agno_metrics WHERE id NOT IN (SELECT id FROM agno_metrics_merged);

UPDATE agno_metrics m SET
    user_id = '',
    agent_runs_count        = g.agent_runs_count,
    team_runs_count         = g.team_runs_count,
    workflow_runs_count     = g.workflow_runs_count,
    agent_sessions_count    = g.agent_sessions_count,
    team_sessions_count     = g.team_sessions_count,
    workflow_sessions_count = g.workflow_sessions_count,
    users_count             = g.users_count,
    token_metrics           = g.token_metrics,
    model_metrics           = g.model_metrics,
    created_at              = g.created_at,
    updated_at              = g.updated_at,
    completed               = g.completed
FROM agno_metrics_merged g WHERE m.id = g.id;

DROP TABLE agno_metrics_merged;

COMMIT;
```

Qualify `agno_metrics` with your schema, or set `search_path`, if it is not on the
default one. The counts are summed, `token_metrics` is merged key by key, and
`model_metrics` — a JSON *array* of `{model_id, model_provider, count}` — is unnested,
grouped and re-aggregated so no model is listed twice. `users_count` becomes the number
of owned buckets for the day, which is what a v2.5.6 row meant by it; a row already
carrying a larger count is a pre-v3.0 one and keeps its own. Take a backup first: the
per-user breakdown is gone afterwards, and so is the `user_id` column once the revert
runs.

```python
asyncio.run(MigrationManager(db).down(target_version="2.5.6", table_type="metrics"))
```

## Breaking changes

1. **Direct SQL against `agno_sessions.runs`** stops being a complete view of session
   history once v3.0 is live — new runs go to the `agno_runs` table, not the legacy
   column. The legacy column is **never nulled by writes**; it stays as a frozen
   backup (holding whatever it held at upgrade time) until you explicitly run
   `cleanup_legacy_runs_column()`. Query the runs table instead:

   ```sql
   SELECT run_data FROM ai.agno_runs WHERE session_id = :sid ORDER BY run_index;
   ```

   Because writes no longer null the legacy column, `cleanup_legacy_runs_column()`
   (and `cleanup_legacy_runs_field()` on NoSQL/file adapters) will refuse to run
   while any session still holds a legacy blob — after you've run and verified the
   migration, pass `force=True` to reclaim the storage.

2. **`Session.to_dict()` accepts `include_runs`.** Defaults to `True` (unchanged
   behavior). Adapters use `include_runs=False` internally to avoid serializing
   runs when writing the session row.

3. **Custom table names**: `PostgresDb`, `AsyncPostgresDb`, `SqliteDb` and
   `AsyncSqliteDb` accept a new `runs_table` argument (defaults to `"agno_runs"`).

Unchanged: `Agent`/`Team`/`Workflow` code, `session.get_messages()`,
`get_chat_history()`, AgentOS session endpoints, and `db.get_session()` all behave
as before — runs are reattached to sessions transparently on read.

## New APIs

The upgraded adapters expose direct run access (sync and async variants):

```python
# Get a single run
run = db.get_run(run_id="...")

# Get runs with filters and pagination
runs = db.get_runs(session_id="...", status=RunStatus.completed, limit=20)

# Get run rows without deserializing (returns (rows, total_count))
rows, total = db.get_runs(agent_id="...", deserialize=False)

# Delete runs
db.delete_run(run_id="...")
db.delete_runs(run_ids=["...", "..."])

# Drop the legacy `runs` column once everything is migrated
db.cleanup_legacy_runs_column()
```

## How writes work now (for the curious)

On `db.upsert_session(session)`:

1. The session row is upserted without any run data.
2. Every run on the in-memory session is upserted into the runs table (`ON CONFLICT
   DO UPDATE` on `run_id`).
3. If the sessions table still has a legacy `runs` column, that column is set to
   `NULL` for the session — so the runs table is the only source of truth going
   forward for that session.

So a session with 500 runs writes 500 run rows when you save (each one is small and
indexed, vs the old approach of one growing blob). For most workloads this is a
clear win over the v2.x O(N²) write amplification; if you have a hot path that
writes many times without changing runs, you can optimize further by skipping
sessions you didn't touch.

## Storage comparison

| Metric | v2.x (blob) | v3.0 (runs table) |
|--------|-------------|-------------------|
| Bytes written to store N runs | O(N²) | O(N) |
| Session row size | grows unbounded | small, constant |
| Fetch last N runs | load + parse all runs | indexed SQL query |
| Save a new run | rewrite all runs | per-session run upsert |

## Compatibility matrix

| Scenario | What you get |
|---|---|
| Fresh v3.0 install | No legacy column, runs in `agno_runs`. Just works. |
| v2.x → v3.0, no migration run yet | Reads merge runs table + legacy blob; new runs go to the table, the legacy blob is preserved so nothing is lost. |
| v2.x → v3.0, migration run, column not cleaned up | Reads go through the runs table, merged with the preserved legacy blob (deduped by `run_id`). Legacy column kept as a backup. |
| v2.x → v3.0, migration + `cleanup_legacy_runs_column()` | Final v3.0 state. Smallest sessions table. |
| Half-finished migration / hand-imported runs | Reads merge by `run_id`. No history is silently lost. |
