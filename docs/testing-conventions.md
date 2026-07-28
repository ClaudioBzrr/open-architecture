# Testing Conventions

## Framework

Jest + ts-jest. Configuration in `server/jest.config.js`.

## Location

```
server/__tests__/services/
├── reports-service/
├── user-service/
├── pricing-service/
├── order-service/
├── ... (one directory per domain)
```

## Approach

**Unit tests only.** Repositories are mocked via constructor injection. No global mocks. No database integration.

### Test Structure

```typescript
import { GetReportsService } from "@/services/reports-service/get-reports-service";
import { ICachedReportRepository } from "@/types/repositories/cached-report-repository";
import { IUserRepository } from "@/types/repositories/users-repository";

jest.mock("@/logger", () => ({
  logger: { info: jest.fn(), warn: jest.fn(), error: jest.fn() },
}));

describe("GetReportsService", () => {
  let service: GetReportsService;
  let mockUserRepo: jest.Mocked<IUserRepository>;
  let mockCachedReportRepo: jest.Mocked<ICachedReportRepository>;

  beforeEach(() => {
    mockUserRepo = { findUnique: jest.fn(), findMany: jest.fn() } as any;
    mockCachedReportRepo = { findMany: jest.fn(), deleteAll: jest.fn(), create: jest.fn() } as any;
    service = new GetReportsService(mockUserRepo, mockCachedReportRepo);
  });

  it("should return analysis data for valid user", async () => {
    mockUserRepo.findUnique.mockResolvedValue({ id: "1", type: "admin" } as any);
    mockCachedReportRepo.findMany.mockResolvedValue([] as any);

    const result = await service.execute("1");
    expect(result).toHaveLength(5);
  });
});
```

### Key Rules

1. **Mock via constructor** — pass mock objects to the service constructor. No `jest.mock` for repositories.
2. **Mock the logger** — always suppress logger output in tests:
   ```typescript
   jest.mock("@/logger", () => ({
     logger: { info: jest.fn(), warn: jest.fn(), error: jest.fn() },
   }));
   ```
3. **One `describe` per service** — `beforeEach` sets up fresh mocks and a new service instance.
4. **Test the `execute` method** — that's the public entry point.

## TypeScript Config for Tests

`tsconfig.test.json` extends the main config to include `__tests__/`:

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "types": ["node", "jest"],
    "rootDir": "."
  },
  "include": ["src/**/*", "__tests__/**/*"]
}
```

`jest.config.js` uses `pathsToModuleNameMapper` to resolve `@/` just like tsconfig.

## Running Tests

```sh
cd server && npm test          # Jest with coverage
```

Coverage output: `server/coverage/` (LCOV, Clover, JSON).
