# Backend Conventions

## Folder Layout

Organize the backend by **layer**, not by feature. Each layer gets its own directory. Domain-specific logic lives in subdirectories within each layer.

```
server/src/
├── routes/             # Route definitions — one file per domain
│   └── index.ts        # Aggregator: mounts all routes, applies global middleware order
├── controllers/        # HTTP handlers — one file per domain
├── services/           # Business logic — one subdirectory per domain
├── repositories/       # Data access implementations
│   ├── index.ts        # DI container — all instances live here
│   └── {source}/       # One subdirectory per data source (e.g., prisma/, fetch/, xlsx/)
├── adapters/           # External service integrations (mail, HTTP, etc.)
│   ├── index.ts        # DI: adapter instances
│   └── {provider}/     # One subdirectory per provider implementation
├── types/
│   ├── entities/       # Domain interfaces and enums
│   └── repositories/   # Repository contracts (interfaces)
├── common/             # Cross-cutting middlewares and shared utilities
├── constants/          # Domain constants and enums
├── lib/                # Shared infrastructure (JWT, HTTP client wrappers)
├── app.ts              # Express entry point
├── env.ts              # Environment variable validation (Zod)
├── prisma.ts           # Singleton ORM client
└── logger.ts           # Logging setup (Winston + Morgan)
```

**Key principle:** the folder names reflect the architectural layer, not the business domain. Business domains split into files/subdirectories _within_ each layer (`services/reports-service/`, `services/user-service/`, etc.).

## Logger

The logging setup lives in `server/src/logger.ts` and uses **Winston** as the logging framework with **Morgan** integrated as the HTTP request logger.

### Winston + Morgan Integration

Morgan is added as a middleware that writes every HTTP request to a custom stream, which feeds directly into Winston. This ensures all logs — application-level and HTTP-level — use the same transport, format, and destination.

```typescript
import winston from "winston";
import morgan from "morgan";

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL ?? "info",
  format: winston.format.combine(
    winston.format.timestamp({ format: "YYYY-MM-DDTHH:mm:ss.SSSZ" }),
    winston.format.errors({ stack: true }),
    winston.format.printf(({ timestamp, level, message, ...rest }) => {
      return `${timestamp} [${level.toUpperCase()}] ${message} ${
        Object.keys(rest).length ? JSON.stringify(rest) : ""
      }`.trimEnd();
    }),
  ),
  transports: [new winston.transports.Console()],
});

// Morgan writes to Winston via a custom stream
export const morganMiddleware = morgan(
  (tokens, req, res) => {
    return [
      tokens.method(req, res),
      tokens.url(req, res),
      tokens.status(req, res),
      tokens["response-time"](req, res) + " ms",
      "-",
      tokens.res(req, res, "content-length") ?? "0",
      "bytes",
    ].join(" ");
  },
  {
    stream: { write: (message: string) => logger.info(message.trim()) },
  },
);
```

### Log Date/Time Format

All timestamps follow **ISO 8601 with timezone offset**:

```
YYYY-MM-DDTHH:mm:ss.SSSZ
```

**Examples:**

```
2024-01-15T10:30:00.123-03:00
2024-06-20T14:05:42.987Z
```

- Uses 24-hour format (`HH`).
- Includes milliseconds (`.SSS`).
- Timezone offset (`Z` for UTC or `±HH:mm` for local).
- Machine-parseable and human-readable, consistent across environments.

### Log Pattern

| Component | Pattern | Example |
|---|---|---|
| **Application logs** (Winston) | `{timestamp} [{LEVEL}] {message}` | `2024-01-15T10:30:00.123-03:00 [INFO] GenerateReportsService: starting...` |
| **HTTP logs** (Morgan → Winston) | `{timestamp} [{LEVEL}] {METHOD} {URL} {STATUS} {RESPONSE_TIME} ms - {CONTENT_LENGTH} bytes` | `2024-01-15T10:30:01.456-03:00 [INFO] GET /api/reports 200 15.234 ms - 1024 bytes` |

**Field details:**

| Field | Meaning |
|---|---|
| `timestamp` | ISO 8601 with milliseconds and timezone |
| `LEVEL` | Uppercase log level (INFO, WARN, ERROR) |
| `METHOD` | HTTP method (GET, POST, PUT, DELETE) |
| `URL` | Request path |
| `STATUS` | HTTP response status code |
| `RESPONSE_TIME` | Response time in milliseconds |
| `CONTENT_LENGTH` | Response body size in bytes |

### Log Levels

| Level | When to use |
|---|---|
| `logger.info` | Entry/exit of service methods, lifecycle events |
| `logger.warn` | Guard clauses, degraded states, non-critical issues |
| `logger.error` | Catch blocks, unrecoverable errors |

**Controllers never log directly** — Morgan handles all HTTP request logging.

### Logging Rules

1. **Services** — `info` at entry and exit, `error` in catch, `warn` for guard clauses.
2. **Repositories** — `error` only, return gracefully (empty arrays / null).
3. **Controllers** — **no logging**. Morgan handles HTTP logging globally.
4. **Tests** — mock the logger to suppress output.

## Services

### Rules

1. **Constructor injection only** — never import repository instances directly. Dependencies arrive via constructor parameters.
2. **No `req` / `res`** — services are framework-agnostic.
3. **Single responsibility** — each service handles one use case.
4. **Logging** — `logger.info` at entry and exit, `logger.error` in catch, `logger.warn` for guard clauses.

### Request Validation

Services validate all input before executing business logic. Use Zod schemas defined alongside the service:

```typescript
import { z } from "zod";

const createOrderSchema = z.object({
  customerId: z.string().uuid(),
  items: z.array(z.object({
    productId: z.string().uuid(),
    quantity: z.number().int().positive(),
  })).min(1),
});

export class CreateOrderService {
  async execute(input: unknown): Promise<IOrder> {
    const data = createOrderSchema.parse(input);  // throws ZodError on invalid
    // ... business logic with typed `data` ...
  }
}
```

**Rules:**
1. **Validate in the service**, not the controller — keeps controllers thin and validation logic testable.
2. **Define schemas next to the service** that uses them — not in a global schemas directory.
3. **Use `.parse()` for throwing** or `.safeParse()` for graceful handling — choose based on whether the caller can recover.
4. **Infer types from schemas** — `z.infer<typeof createOrderSchema>` — to avoid duplicate type definitions.

### Pattern

```typescript
import { logger } from "@/logger";

export class GenerateReportsService {
  constructor(
    private readonly customerRepository: ICustomerRepository,
    private readonly productRepository: IProductRepository,
    private readonly cachedReportRepository: ICachedReportRepository,
  ) {}

  async execute(): Promise<void> {
    try {
      logger.info("Generating reports...");
      const orders = await this.customerRepository.load();
      const products = await this.productRepository.load();

      if (!products || products.length === 0) {
        logger.warn("Product data empty — skipping update");
        return;
      }

      // ... business logic ...

      await this.cachedReportRepository.deleteAll();
      await this.cachedReportRepository.create(data);
      logger.info("Reports generated");
    } catch (err) {
      logger.error(`Error generating reports: ${err instanceof Error ? err.message : err}`);
      throw err;  // rethrow so the controller can respond with the appropriate HTTP status
    }
  }
}
```

### Naming

- `Get*Service` — read-only queries.
- `Generate*Service` — cache generation / computation.
- `Create*Service`, `Update*Service`, `Delete*Service` — mutations.

Services live in domain subdirectories: `services/reports-service/`, `services/user-service/`, etc.

## Controllers

### Rules

1. **Parse → call → respond** — nothing else.
2. **No business logic** — all validation and orchestration belongs in the service.
3. **No logging** — Morgan handles HTTP logging globally.
4. **Error format** — catch errors from services, return `res.status(4xx).json(message)`.

### Pattern

```typescript
import { Request, Response } from "express";

export class ReportController {
  async getReport(req: Request, res: Response) {
    try {
      const id = req.user?.userId;
      if (!id) throw new Error("Missing credentials.");
      const data = await getReportsService.execute(id);
      return res.json(data);
    } catch (err) {
      return res.status(404).json(String(err).replace("Error: ", ""));
    }
  }
}
```

### Instantiation

Controllers are instantiated at the top of their file, receiving repository imports from `@/repositories`:

```typescript
const getReportsService = new GetReportsService(
  userRepository,
  cachedReportRepository,
);
```

## Repositories

### Interface

Defined in `types/repositories/` with two base types:

```typescript
// Read-only provider (load pattern)
export type IProviderRepository<T> = {
  load: () => T;
};

// CRUD repository
export type IDataRepository<T> = {
  create: (data: ...) => Promise<T>;
  findOne: (filter: ...) => Promise<T>;
  findMany: (filter?: ...) => Promise<T[]>;
  update: (filter: ..., data: ...) => Promise<void>;
  delete: (filter: ...) => Promise<void>;
  count: (filter?: ...) => Promise<number>;
};
```

### Implementations

| Directory | Source | Use Case |
|---|---|---|
| `repositories/prisma/` | Local PostgreSQL | Entities persisted by the application |
| `repositories/db/` | Staging tables (uploaded files — e.g., XLSX, CSV) | Cached read layer over uploaded data |
| `repositories/fetch/` | External REST API | Live data from external systems |
| `repositories/xlsx/` | Local XLSX files | Source-specific upload parsers |
| `repositories/bcrypt/` | In-memory hashing | Password operations |
| **Note** | | `db/`, `xlsx/`, and `bcrypt/` are examples from a project with XLSX imports and a custom cache layer. Your project may need different implementations (e.g., `redis/`, `s3/`, `grpc/`). |

### Utility Repositories

Some repositories don't map to a database or external API — they wrap a utility or computation. The `bcrypt/` repository is an example: password hashing is treated as a repository implementation so services depend on an `IPasswordHasher` interface, not on `bcrypt` directly.

**Why this pattern:**
- **Testability** — swap bcrypt for a plain-text hasher in tests without mocking a library.
- **Consistency** — all dependencies arrive via constructor injection, including utilities.
- **Flexibility** — switch hashing algorithm (argon2, scrypt) by creating a new implementation, not by editing services.

```typescript
// types/repositories/password-hasher.ts
export type IPasswordHasher = {
  hash: (plain: string) => Promise<string>;
  compare: (plain: string, hashed: string) => Promise<boolean>;
};

// repositories/bcrypt/bcrypt-password-hasher.ts
export class BcryptPasswordHasher implements IPasswordHasher { ... }
```

Not every utility needs this treatment. Reserve it for dependencies that have alternative implementations or that you want to mock in tests.

All implement the same repository interface. Services never know which source they talk to.

### Error Handling in Repositories

Repositories **log errors and return gracefully** (empty arrays, null) instead of rethrowing:

```typescript
catch (err) {
  logger.error(`Error fetching prices: ${err}`);
  return [] as IPrice[];
}
```

## Routes

### Mounting Order

The order is **critical**. `authMiddleware` is applied globally after public routes:

```typescript
export const routes = Router();

// 1. Public — no authentication
routes.use(userRoutes);

// 2. IP-protected — no JWT
routes.use(uploadRoutes);

// 3. Global auth — all routes below require JWT
routes.use(authMiddleware);

// 4. Protected routes
routes.use(notificationRoutes);
routes.use(reportRoutes);
// ...
```

Any route mounted **before** `authMiddleware` is public by default.

### Middleware Stacking

```typescript
// Read: permission check only
router.get("/resource", requirePage("feature"), ctrl.findMany);

// Mutation: permission + admin
router.post("/resource", requirePage("feature"), requireAdmin, ctrl.create);
router.put("/resource/:id", requirePage("feature"), requireAdmin, ctrl.update);
router.delete("/resource/:id", requirePage("feature"), requireAdmin, ctrl.delete);
```

## Middleware

| Middleware | Purpose |
|---|---|
| `authMiddleware` | Verifies JWT, injects `userId` into `req.headers.authorization` |
| `requirePage(...keys)` | Checks user has at least one of the given page permissions |
| `requireAdmin` | Requires `user.type === "admin"` |
| `ipWhitelistMiddleware` | Restricts access by IP (upload routes) |

### Permission Resolution Order

**Role > Direct UserPermission > `configured: false`** (blocks access)

## Adapters

External service integrations follow the same interface pattern:

```typescript
// adapters/mail-adapter.ts — interface
interface IMailAdapter {
  sendMail(data: IMailAdapterProps): Promise<void>;
}

// adapters/nodemailer/nodemailer-mail-adapter.ts — implementation
class NodemailerMailAdapter implements IMailAdapter { ... }

// adapters/index.ts — DI
export const mailAdapter: IMailAdapter = new NodemailerMailAdapter();
```

## Schedule / Cron Jobs

Periodic jobs regenerate cached data, sync external sources, or run maintenance tasks. The scheduler lives in `server/src/schedule.ts`.

### Pattern

```typescript
import cron from "node-cron";
import { logger } from "@/logger";

// This file runs in a forked process (see bin/www.ts below).
// Cron callbacks execute here, never in the HTTP process.
export function startSchedule() {
  cron.schedule("0 */6 * * *", async () => {
    logger.info("Schedule: starting cache regeneration...");
    const service = new GenerateReportsService(
      customerRepository,
      productRepository,
      cachedReportRepository,
    );
    await service.execute();
    logger.info("Schedule: cache regeneration complete");
  });
}
```

The scheduler is forked from the main entry point so heavy jobs never block HTTP requests:

```typescript
// server/src/bin/www.ts — entry point
import { fork } from "child_process";
import { join } from "path";

if (process.env.NODE_ENV === "production") {
  fork(join(__dirname, "schedule.js"));
}
```

### Rules

1. **Fork the scheduler** — run cron jobs in a separate process (`child_process.fork`) so heavy computation doesn't block HTTP requests.
2. **One job per schedule** — each cron expression maps to one service execution. Don't chain multiple services in a single job.
3. **Log entry and exit** — same logging rules as services.
4. **Idempotent jobs** — a job must produce the same result whether it runs once or twice in succession. Use delete-then-create (not update) for cache writes.
5. **In-memory snapshot** — while the job writes to cached tables, the API reads from an in-memory snapshot to avoid serving partial/stale data. See `database-conventions.md` → Cache Pattern.

### On-Demand Trigger

Some cache jobs can be triggered on demand (e.g., after a file upload). Expose an endpoint that calls the same service:

```typescript
// routes/report-routes.ts
router.post("/regenerate", requirePage("reports"), requireAdmin, ctrl.regenerate);
```

The service is the same; only the trigger differs (cron vs. HTTP).

## Environment Variables

Validated with Zod in `env.ts`. Server exits with `process.exit(1)` if any required variable is missing.

```typescript
const envSchema = z.object({
  DATABASE_URL: z.string().min(1),
  JWT_SECRET: z.string().min(32),
  // ...
});
```

Variables are exported individually — never access `process.env` directly in code.
