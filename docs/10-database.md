# Chapter 10: Database Layer

**Sources:** `packages/effect-drizzle-sqlite/`, `packages/effect-sqlite-node/`

The database layer consists of two packages that together provide an Effect-native SQLite ORM built on top of Drizzle ORM. The stack has three layers, each building on the one below.

---

## Architecture Overview

```
packages/effect-sqlite-node/   ← Node.js sqlite driver for Effect's SqlClient
          ↓ (SqlClient)
packages/effect-drizzle-sqlite/
  └── src/sqlite-core/effect/  ← Layer 1: abstract Drizzle-Effect adapter
  └── src/effect-sqlite/       ← Layer 2: concrete implementation (wraps SqlClient)
  └── src/up-migrations/       ← Layer 3: schema migration with version tracking
```

---

## Layer 1 — Abstract Drizzle-Effect Adapter

**Location:** `packages/effect-drizzle-sqlite/src/sqlite-core/effect/`

This layer translates Drizzle ORM's query builder into Effect operations. The key types are:

### `SQLiteEffectDatabase<TRelations>`

The Effect-native equivalent of Drizzle's `LibSQLDatabase`. This is the main API surface used in application code.

```ts
// Query builder methods (all return Effects or Streams)
db.select(fields?)          // → SQLiteEffectSelectBuilder
db.selectDistinct(fields?)  // → SQLiteEffectSelectBuilder
db.insert(table)           // → SQLiteEffectInsertBuilder
db.update(table)           // → SQLiteEffectUpdateBuilder
db.delete(table)           // → SQLiteEffectDeleteBase
db.run(query)              // → Effect<SqliteRunResult, SqlError>
db.all(query)              // → Effect<Row[], SqlError>
db.get(query)              // → Effect<Row | undefined, SqlError>
db.values(query)           // → Effect<unknown[][], SqlError>
db.$count(table, filter?)  // → Effect<number, SqlError>
db.query[tableName]        // → relational query builder (SQLiteEffectRelationalQueryBuilder)
db.transaction(fn)         // → Effect<R, E, A>
db.$with(qb)               // → CTE builder
db.with(...)               // → CTEs
```

### `SQLiteEffectSession`

Abstract class. Concrete implementations must provide:
- `prepareQuery(query, mappers)` — returns a `SQLiteEffectPreparedQuery`
- `prepareRelationalQuery(config)` — returns a `SQLiteEffectRelationalQuery`
- `transaction(fn)` — runs the callback in a database transaction

The session provides concrete implementations of `run`, `all`, `get`, `values`, and `count` by delegating to the appropriate prepared query method.

### `SQLiteEffectPreparedQuery`

Handles the full lifecycle of a compiled query:

- **Caching** via `EffectCacheShape` — caches compiled query objects across calls
- **JIT mapper dispatch** — resolves column-to-field mapping at first execution, caches for subsequent calls
- **Logging** via `EffectLogger` — logs query SQL and timing
- **Methods:** `run()`, `all()`, `get()`, `values()`, `execute()` — all return Effects

### Builder Classes

All query builders return Effect-aware builders that mirror Drizzle's fluent API:
- `SQLiteEffectSelectBuilder` / `SQLiteEffectSelectBase`
- `SQLiteEffectInsertBuilder` / `SQLiteEffectInsertBase`
- `SQLiteEffectUpdateBuilder` / `SQLiteEffectUpdateBase`
- `SQLiteEffectDeleteBase`
- `SQLiteEffectCountBuilder`
- `SQLiteEffectRelationalQueryBuilder` / `SQLiteEffectRelationalQuery`
- `SQLiteEffectRaw` — for raw SQL execution

---

## Layer 2 — Concrete Implementation

**Location:** `packages/effect-drizzle-sqlite/src/effect-sqlite/`

### `EffectSQLiteDatabase<TRelations>`

Extends `SQLiteEffectDatabase`. Uses Effect's `SqlClient` interface (from `effect/unstable/sql`) as the underlying connection.

### `EffectSQLiteSession<TRelations>`

Extends `SQLiteEffectSession`. Connects to `SqlClient`:

- `execute(query)` dispatches to the appropriate `SqlClient` method:
  - `values` mode: `client.unsafe(sql, params).values`
  - `get` mode: `client.unsafe(sql, params).withoutTransform` (returns first row)
  - Default: `client.unsafe(sql, params)`
- `transaction(fn)` — nested savepoint support. Depth tracked via `client.transactionService`. At depth 0 uses `client.withTransaction`; deeper levels use `SAVEPOINT/RELEASE SAVEPOINT` SQL.

### `make(config)` — Creating a Database

```ts
const db = yield* EffectSQLiteDatabase.make({
  schema: myDrizzleSchema,   // Drizzle schema tables
  relations: myRelations,    // Drizzle relations (optional)
})
// Requires in context: SqlClient, EffectCache, EffectLogger
```

### `makeWithDefaults(config)` — With Default Services

Provides `DefaultServices = Layer.merge(EffectCache.Default, EffectLogger.Default)` automatically. The application only needs to provide `SqlClient`.

---

## Layer 3 — Migration System

**Location:** `packages/effect-drizzle-sqlite/src/up-migrations/`

The migration system uses Drizzle's `migrate()` function with an additional "up-migration" step to handle schema evolution of the migrations table itself.

### Migration Table Versioning

The `MIGRATIONS_TABLE_VERSIONS` constant defines the expected version for each database type:

```ts
{ sqlite: 1, pg: 1, mysql: 1, ... }  // all at version 1 currently
```

### V0 → V1 Migration

The migrations table gained `name TEXT` and `applied_at TEXT` columns in V1:

- `GET_VERSION_FOR.sqlite(columns)` — returns 0 if `name` column is absent, 1 if present
- V0→V1 upgrade: adds both columns, backfills `name` by matching existing rows to local migration files by their timestamp/hash

### Three Upgrade Variants

- `upgradeSyncIfNeeded` — synchronous (for startup code)
- `upgradeAsyncIfNeeded` — Promise-based
- `upgradeIfNeeded` — Effect-based (primary variant used in the application)

### Running Migrations

```ts
const result = yield* migrate(db, {
  migrationsFolder: "./drizzle",  // folder of .sql migration files
  migrationsTable: "__drizzle_migrations",
})
// Returns: { applied: string[] }  — names of applied migrations
```

Migrations run inside a single SQLite transaction. If any migration fails, the entire batch rolls back.

---

## Node.js SQLite Driver — `packages/effect-sqlite-node/`

Implements Effect's `SqlClient` interface using Node.js `node:sqlite` (`DatabaseSync`).

### Service Interface

```ts
interface SqliteClient extends Client.SqlClient {
  readonly [TypeId]: TypeId
  readonly config: SqliteClientConfig
  readonly loadExtension: (path: string) => Effect.Effect<void, SqlError>
}
```

### `SqliteClientConfig`

```ts
{
  filename: string          // database file path (":memory:" for in-memory)
  readonly?: boolean
  create?: boolean          // create if not exists (default: true)
  readwrite?: boolean
  disableWAL?: boolean      // WAL mode is enabled by default
  timeout?: number          // busy timeout in ms
  allowExtension?: boolean  // allow loadExtension()
  spanAttributes?: Record<string, unknown>
  transformResultNames?: (name: string) => string  // column name transform
  transformQueryNames?: (name: string) => string   // table name transform
}
```

### Connection Behavior

- `DatabaseSync` is created with `enableForeignKeyConstraints: true`
- WAL (Write-Ahead Log) mode is enabled by default (`PRAGMA journal_mode=WAL`) for better concurrent read performance
- A `Semaphore(1)` serializes all database operations (SQLite does not support true concurrent writes)
- Transactions use `BEGIN/COMMIT/ROLLBACK` for top-level, `SAVEPOINT/RELEASE/ROLLBACK TO SAVEPOINT` for nested

### `layer(config)` — Effect Layer

```ts
const dbLayer = SqliteClientLayer({
  filename: path.join(homeDir, "opencode.db"),
})
// Provides: SqliteClient + Client.SqlClient + Reactivity
```

---

## Application Database Setup — `packages/core/src/db.ts`

The application uses a single SQLite database at `~/.local/share/opencode/db.sqlite`.

```ts
// Layers assembled in packages/core/
DatabaseLayer = SqliteClientLayer({ filename })
  → EffectSQLiteDatabase.makeWithDefaults({ schema, relations })

Database = EffectSQLiteDatabase<Relations>  // the service identifier
```

On startup, `migrate()` applies pending migrations from the `packages/core/drizzle/` folder.

### Database Tables

| Table | Purpose |
|---|---|
| `SessionTable` | Session metadata (id, title, cost, model, tokens, etc.) |
| `SessionMessageTable` | Projected session messages (JSON `data` column) |
| `EventTable` | All durable events (aggregate_id, seq, type, data) |
| `EventSequenceTable` | Current sequence number per aggregate |
| `SessionContextEpochTable` | System context snapshots per session turn |
| `SessionInputTable` | Pending prompt inbox (admitted/promoted state) |
| `PermissionSavedTable` | Persistent permission rules |
| `TodoTable` | Session todos (content, status, priority) |
| `CredentialTable` | Provider credentials (encrypted at rest) |
| `ProjectTable` | Project metadata |
| `WorkspaceTable` | Workspace records |
| `McpTable` | MCP server configurations |
| `LspTable` | LSP server state |

---

## Effect Cache and Logger

### `EffectCache`

Used by `SQLiteEffectPreparedQuery` to cache compiled query objects and JIT mappers. `EffectCache.Default` provides an in-memory LRU-style cache.

### `EffectLogger`

Used by `SQLiteEffectPreparedQuery` to log query SQL, parameters, and execution time. `EffectLogger.Default` writes to Effect's structured logging system.

---

## Using the Database in Application Code

The database is accessed via the `Database` service (which is an `EffectSQLiteDatabase`):

```ts
import { Database } from "@opencode-ai/core/db"

const insertSession = Effect.gen(function* () {
  const db = yield* Database
  yield* db.insert(SessionTable).values({
    id: Session.ID.create(),
    title: "My Session",
    // ...
  })
})

const listSessions = Effect.gen(function* () {
  const db = yield* Database
  return yield* db
    .select()
    .from(SessionTable)
    .where(eq(SessionTable.projectId, projectId))
    .orderBy(desc(SessionTable.id))
    .limit(50)
})
```

Transactions:

```ts
const publishEventWithProjection = Effect.gen(function* () {
  const db = yield* Database
  yield* db.transaction(function* (tx) {
    // All these operations are atomic
    yield* tx.insert(EventTable).values(event)
    yield* tx.update(SessionTable).set({ cost: newCost }).where(eq(SessionTable.id, sessionId))
    // If any fails, the entire transaction rolls back
  })
})
```
