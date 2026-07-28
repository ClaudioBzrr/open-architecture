# Database Conventions

## ORM

Prisma 6 with PostgreSQL 16. Schema file: `server/prisma/schema.prisma`.

## Generator Configuration

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["omitApi"]
  binaryTargets   = ["native", "linux-musl-openssl-3.0.x"]
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

## Client Singleton

One `PrismaClient` instance, exported from `server/src/prisma.ts`. Never instantiate a new one.

```typescript
import { PrismaClient } from "@prisma/client";
export const prisma = new PrismaClient();
```

## Schema Conventions

| Convention | Pattern | Reason |
|---|---|---|
| ID | `String @id @default(uuid())` | UUIDs avoid collisions in distributed environments |
| Timestamps | `createdAt @default(now())` + `updatedAt @updatedAt` | Audit trail on every table |
| Table name | `@@map("snake_case")` | Standard SQL convention |
| Soft delete | `active Boolean @default(true)` | Preserves history |
| Flexible data | `Json?` | Variable structures per record |
| Cascade children | `onDelete: Cascade` | Dependent children (steps, views, likes) |
| Optional FK | `onDelete: SetNull` | Reference to a resource that may be removed |
| Composite unique | `@@unique([field1, field2])` | Natural keys |
| Indexes | `@@index([field])` / `@@index([a, b])` | Frequently filtered fields |

## Canonical Model

```prisma
model Resource {
  id        String   @id @default(uuid())
  name      String
  active    Boolean  @default(true)
  ownerId   String
  owner     User     @relation(fields: [ownerId], references: [id])
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([ownerId])
  @@map("resources")
}
```

## After Any Schema Change

1. `npx prisma migrate dev` — create and apply migration.
2. `npx prisma generate` — regenerate client (mandatory).

Never edit the database directly. Never skip `prisma generate`.

## Permission Model

Two-layer system: **Role > UserPermission > unconfigured (blocks)**:

- **Role** — permission group assigned to users (allowedPages, allowedGroups, allowedRegions, allowedDepartments).
- **UserPermission** — direct per-user override (1:1 with User).
- **Resolution** (`resolve-permissions.ts`): Role takes priority → direct permission fallback → `configured: false` blocks access.

## Cache Pattern

Tables with `cached_` prefix store pre-computed data from a periodic schedule job. They serve as a fast read layer for data involving multiple slow external API calls, large file parsing, or heavy aggregations.

```
External sources + files
  → Cache generation job (periodic or on-demand)
    → cached_* tables
      → API reads from cache (fast, no external calls)
```

### When to Use Cache Tables

- **Slow external API calls** — multiple round trips that can't be served in real time.
- **Large file parsing** — XLSX/CSV uploads that take seconds to process.
- **Heavy aggregations** — joins or computations across many tables.

### In-Memory Snapshot

During cache writes, the API must not serve stale or partial data. The pattern:

```
1. Job starts → takes a snapshot of current cached_* table data into memory
2. API reads from the in-memory snapshot (not the DB) during regeneration
3. Job deletes all rows and recreates them in the DB
4. Job completes → snapshot is discarded, API resumes reading from DB
```

This ensures zero-downtime cache regeneration. The API never sees a half-written table.

**Implementation:**
- The snapshot is a simple in-memory array or Map, held in the schedule process.
- Read services check if a snapshot is active; if so, they serve from memory.
- This is a read-through pattern — the snapshot is a temporary stand-in, not a persistent cache.

> See also: [backend-conventions.md → Schedule / Cron Jobs](./backend-conventions.md#schedule--cron-jobs) for the regeneration side of this pattern.

## Batch Import Pipeline

When the system ingests external data via file uploads (XLSX, CSV, etc.), it follows a three-stage pipeline:

```
1. Upload → staging tables (raw, unprocessed)
2. Schedule job → reads staging, transforms, writes to cached_* tables
3. API → reads from cached_* tables (never from staging)
```

### Staging Tables

Staging tables are temporary storage for raw uploaded data. They are:

- **Truncated on each upload** — a new upload replaces the previous data entirely.
- **Unvalidated** — raw rows as parsed from the file, before any business logic.
- **Write-only from feature code** — feature code writes to staging; only the schedule job reads from it.

**Never write feature code that reads staging tables directly** — use the `Cached*Provider` cache layer.

### Upload Flow

```
1. Client uploads file → IP-whitelisted route (no JWT)
2. Upload route parses file → writes rows to staging table
3. Upload route triggers on-demand cache regeneration (or waits for next cron cycle)
4. Schedule job reads staging → transforms → writes to cached_* table
5. API serves cached data to clients
```

### Why This Separation

- **Decoupling** — upload and read paths are independent; a slow upload doesn't block reads.
- **Validation boundary** — staging is raw; cached is validated and transformed. The schedule job is the boundary.
- **Atomicity** — the cache regeneration is atomic (delete-then-create with snapshot); uploads are not.

> See also: [backend-conventions.md → Schedule / Cron Jobs](./backend-conventions.md#schedule--cron-jobs) for the cache regeneration service.
