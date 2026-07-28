# Architecture

## Layer Pattern

Every HTTP request follows a strict five-layer chain. No layer may skip or bypass the next.

```
Route → Controller → Service → Repository (interface) → Implementation
```

### Layer Responsibilities

| Layer | Owns | Must NOT own |
|---|---|---|
| **Route** | HTTP path, middleware stacking, controller instantiation | Business logic, response formatting |
| **Controller** | Parse `req.body` / `req.params` / `req.query`, call service, return status + JSON | Business logic, direct DB access |
| **Service** | All business logic, orchestration, validation, logging | `req` / `res` objects, HTTP concerns |
| **Repository (interface)** | Contract (method signatures, return types) | Implementation details |
| **Implementation** | Actual data access (Prisma, Fetch, XLSX, bcrypt) | Business logic |

### Data Flow Example

```
1. Route:   POST /orders  →  requirePage("orders"), requireAdmin, ctrl.create
2. Controller:  const data = req.body → await createService.execute(data) → res.status(201).json(result)
3. Service:  validates input → calls orderRepository.create(data) → logs → returns result
4. Repository interface:  create(data): Promise<IOrder>
5. Prisma implementation:  prisma.order.create({ data })
```

## Manual Dependency Injection

No DI framework. All wiring happens in two files:

- **`server/src/repositories/index.ts`** — instantiates every repository, typed by interface.
- **`server/src/adapters/index.ts`** — instantiates every adapter (mail, HTTP, etc.).

Services receive their dependencies through the constructor. This makes testing trivial — swap real implementations for mocks at construction time.

```typescript
// repositories/index.ts
export const userRepository: IUserRepository = new PrismaUserRepository();
export const productRepository: IProductRepository = new FetchProductProvider();

// A service receives them via constructor
class GetReportsService {
  constructor(
    private readonly userRepository: IUserRepository,
    private readonly cachedReportRepository: ICachedReportRepository,
  ) {}
}
```

### Adding a New Repository

1. Define the interface in `types/repositories/`.
2. Create the implementation in `repositories/{prisma,fetch,db,xlsx}/`.
3. Instantiate and export it in `repositories/index.ts`, typed by the interface.
4. Inject it into any service that needs it.

## Path Alias

`@/` maps to `src/` on both backend and frontend. Configured via `tsconfig.json` paths and `module-alias` (server) / `vite-tsconfig-paths` (web).

```typescript
import { logger } from "@/logger";
import { IUserRepository } from "@/types/repositories/users-repository";
```
