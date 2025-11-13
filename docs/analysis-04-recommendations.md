# Langfuse Architecture Analysis - Part 4: Improvement Recommendations

> **Navigation:** [📋 Index](./analysis-index.md) | [← Part 3: Analysis](./analysis-03-analysis.md) | **Part 4: Recommendations** | [Part 5: Reference →](./analysis-05-reference.md)

---

## Strategic Roadmap

### Short-Term (0-3 months)
1. Split large router files
2. Implement query result caching with Redis
3. Add queue health monitoring and metrics
4. Enforce feature boundaries with ESLint rules

### Medium-Term (3-6 months)
1. Extract service layer from tRPC routers
2. Implement read replicas for PostgreSQL
3. Generate OpenAPI docs from tRPC
4. Decouple S3 from critical ingestion path

### Long-Term (6-12+ months)
1. Migrate to Next.js App Router incrementally
2. Extract features to package boundaries
3. Implement contract testing between services
4. Consider GraphQL layer for external developers

---

## 1. Code Organization Improvements

### 1.1 Split Large Router Files

**Problem:** Routers with 2000+ lines are hard to maintain

**Solution:**
```typescript
// Before: single file with 50 procedures
// features/datasets/server/dataset-router.ts (2177 lines)

// After: split into logical sub-routers
features/datasets/server/
├── index.ts                    // Re-exports combined router
├── dataset-crud-router.ts      // Create, read, update, delete
├── dataset-items-router.ts     // Dataset items operations
├── dataset-runs-router.ts      // Run management
└── dataset-export-router.ts    // Export operations

// features/datasets/server/index.ts
export const datasetRouter = createTRPCRouter({
  ...datasetCrudRouter,
  ...datasetItemsRouter,
  ...datasetRunsRouter,
  ...datasetExportRouter,
});
```

**Benefits:**
- Easier to navigate and understand
- Smaller PR diffs
- Reduced merge conflicts
- Clear separation of concerns

**Effort:** Medium

---

### 1.2 Enforce Feature Boundaries

**Problem:** Features can arbitrarily import from other features

**Solution:**
```javascript
// .eslintrc.js
module.exports = {
  rules: {
    "import/no-restricted-paths": ["error", {
      zones: [
        {
          target: "./src/features/datasets",
          from: "./src/features/!(datasets|shared)",
          message: "Datasets feature should not depend on other features directly"
        }
      ]
    }]
  }
};
```

**Alternative:** Create a `/features/shared/` directory for cross-cutting concerns

**Benefits:**
- Prevents circular dependencies
- Clearer feature boundaries
- Easier to extract to microservices later

**Effort:** Low

---

### 1.3 Extract Service Layer

**Problem:** Business logic mixed in tRPC routers

**Solution:**
```typescript
// Before: logic in router
export const datasetRouter = createTRPCRouter({
  create: protectedProjectProcedure
    .input(CreateDatasetSchema)
    .mutation(async ({ ctx, input }) => {
      // 50+ lines of business logic here
      const dataset = await ctx.prisma.dataset.create({ ... });
      await auditLog({ ... });
      return dataset;
    })
});

// After: extract to service
// features/datasets/server/service.ts
export class DatasetService {
  constructor(
    private prisma: PrismaClient,
    private auditLogger: AuditLogger
  ) {}

  async createDataset(input: CreateDatasetInput, userId: string) {
    // Business logic here
    const dataset = await this.prisma.dataset.create({ ... });
    await this.auditLogger.log({ ... });
    return dataset;
  }
}

// features/datasets/server/dataset-router.ts
export const datasetRouter = createTRPCRouter({
  create: protectedProjectProcedure
    .input(CreateDatasetSchema)
    .mutation(async ({ ctx, input }) => {
      const service = new DatasetService(ctx.prisma, ctx.auditLogger);
      return service.createDataset(input, ctx.session.user.id);
    })
});
```

**Benefits:**
- Testable business logic (no tRPC mocking needed)
- Reusable across tRPC + Public API + Workers
- Clearer separation of concerns

**Effort:** High

---

## 2. Database & Performance Optimizations

### 2.1 Implement Read Replicas

**Problem:** Analytics queries compete with transactional writes

**Solution:**
```typescript
// Separate read/write Prisma clients
const prismaWrite = new PrismaClient();
const prismaRead = new PrismaClient({
  datasources: {
    db: { url: env.DATABASE_READ_REPLICA_URL }
  }
});

// Use in context
export const createTRPCContext = (opts) => {
  return {
    prisma: prismaWrite,        // For mutations
    prismaReadOnly: prismaRead, // For queries
    // ...
  };
};

// Apply in routers
export const datasetRouter = createTRPCRouter({
  allDatasets: protectedProjectProcedure
    .query(async ({ ctx }) => {
      // Use read replica for queries
      return ctx.prismaReadOnly.dataset.findMany({ ... });
    }),

  create: protectedProjectProcedure
    .mutation(async ({ ctx }) => {
      // Use write DB for mutations
      return ctx.prisma.dataset.create({ ... });
    })
});
```

**Benefits:**
- Reduced load on primary database
- Faster read queries
- Better scalability

**Effort:** Low

---

### 2.2 Implement Query Result Caching

**Problem:** Same queries repeated frequently

**Solution:**
```typescript
// Add Redis caching layer
import { createRedisCache } from "@/server/cache";

const cache = createRedisCache({
  ttl: 60, // 1 minute
  keyPrefix: "trpc:query:"
});

// Wrap queries with cache
export const cachedQuery = <TInput, TOutput>(
  procedure: ProcedureBuilder,
  cacheKey: (input: TInput) => string,
  ttl: number = 60
) => {
  return procedure.use(async ({ ctx, next, input }) => {
    const key = cacheKey(input);

    // Check cache
    const cached = await cache.get<TOutput>(key);
    if (cached) return { ...ctx, data: cached };

    // Execute query
    const result = await next({ ctx });

    // Store in cache
    await cache.set(key, result.data, ttl);

    return result;
  });
};

// Use in router
export const datasetRouter = createTRPCRouter({
  allDatasets: cachedQuery(
    protectedProjectProcedure,
    (input) => `datasets:all:${input.projectId}`,
    300 // 5 minutes
  )
  .input(z.object({ projectId: z.string() }))
  .query(async ({ ctx, input }) => {
    return ctx.prisma.dataset.findMany({ where: { projectId: input.projectId } });
  })
});
```

**Invalidation strategy:**
```typescript
// Invalidate on mutation
create: protectedProjectProcedure
  .mutation(async ({ ctx, input }) => {
    const dataset = await ctx.prisma.dataset.create({ ... });

    // Invalidate cache
    await cache.del(`datasets:all:${input.projectId}`);

    return dataset;
  })
```

**Benefits:**
- Reduced database load
- Faster response times
- Better UX

**Effort:** Medium

---

### 2.3 Optimize ClickHouse Queries with Projections

**Problem:** Full table scans for common aggregations

**Solution:**
```sql
-- Create projection for common trace aggregations
ALTER TABLE traces_rmt
  ADD PROJECTION traces_by_project_date (
    SELECT
      project_id,
      toDate(timestamp) as date,
      count() as count,
      sum(total_cost) as total_cost,
      quantile(0.5)(latency) as p50_latency
    GROUP BY project_id, date
  );

-- Populate projection
ALTER TABLE traces_rmt MATERIALIZE PROJECTION traces_by_project_date;
```

**Benefits:**
- 10-100x faster aggregation queries
- Reduced query costs
- Pre-computed metrics

**Effort:** Low

---

## 3. Queue & Worker Improvements

### 3.1 Decouple S3 from Ingestion Path

**Problem:** S3 adds latency and costs

**Solution Options:**

**Option A: Direct write to queue with large payload support**
```typescript
await IngestionQueue.add({
  projectId,
  events: largeEventArray, // Store in job data
  metadata: { ... }
});
```

**Option B: Hybrid approach - small events direct, large via S3**
```typescript
if (eventSize < 1MB) {
  await IngestionQueue.add({ projectId, events });
} else {
  await s3.upload(events);
  await IngestionQueue.add({ projectId, s3Key });
}
```

**Option C: Use Redis Streams for durability**
```typescript
await redis.xadd(
  `langfuse:ingestion:${projectId}`,
  '*',
  'events', JSON.stringify(events)
);
```

**Tradeoffs:**
- Option A: Simpler, but loses replay capability
- Option B: Best of both worlds, adds complexity
- Option C: Redis cost, but fast and durable

**Effort:** Medium

---

### 3.2 Implement Queue Health Monitoring

**Problem:** Queue failures hard to detect

**Solution:**
```typescript
// Add health check endpoints
app.get("/health/queues", async (req, res) => {
  const queues = await WorkerManager.getAllQueues();

  const health = await Promise.all(
    queues.map(async (queue) => {
      const [waiting, active, failed, delayed] = await Promise.all([
        queue.getWaitingCount(),
        queue.getActiveCount(),
        queue.getFailedCount(),
        queue.getDelayedCount()
      ]);

      return {
        name: queue.name,
        waiting,
        active,
        failed,
        delayed,
        healthy: failed < 100 && waiting < 10000
      };
    })
  );

  res.json({ queues: health });
});

// Add Prometheus metrics
import { Counter, Histogram } from "prom-client";

const queueJobsProcessed = new Counter({
  name: "langfuse_queue_jobs_processed_total",
  help: "Total number of queue jobs processed",
  labelNames: ["queue", "status"]
});

const queueJobDuration = new Histogram({
  name: "langfuse_queue_job_duration_seconds",
  help: "Queue job processing duration",
  labelNames: ["queue"]
});
```

**Benefits:**
- Proactive alerting on queue issues
- Better observability
- Easier debugging

**Effort:** Low

---

## 4. API & Developer Experience

### 4.1 Generate OpenAPI Docs from tRPC

**Problem:** tRPC endpoints not documented for external use

**Solution:**
```typescript
// Install: pnpm add trpc-openapi

// web/src/server/api/trpc.ts
import { OpenApiMeta } from "trpc-openapi";

export const openApiProcedure = t.procedure.meta<OpenApiMeta>();

// In routers:
export const datasetRouter = createTRPCRouter({
  allDatasets: openApiProcedure
    .meta({
      openapi: {
        method: "GET",
        path: "/api/datasets",
        tags: ["datasets"],
        summary: "List all datasets",
      }
    })
    .input(z.object({ projectId: z.string() }))
    .output(z.array(DatasetSchema))
    .query(async ({ ctx, input }) => { ... })
});

// Generate OpenAPI spec
import { generateOpenApiDocument } from "trpc-openapi";

const openApiDocument = generateOpenApiDocument(appRouter, {
  title: "Langfuse API",
  version: "1.0.0",
  baseUrl: "https://api.langfuse.com"
});
```

**Benefits:**
- Auto-generated docs stay in sync
- Support for OpenAPI tooling (Postman, Swagger UI)
- Better DX for API consumers

**Effort:** Medium

---

## 5. Testing & Quality

### 5.1 Implement Contract Testing

**Problem:** Breaking changes between web and worker not caught

**Solution:**
```typescript
// Use Pact for contract testing
import { Pact } from "@pact-foundation/pact";

describe("Ingestion Queue Contract", () => {
  const provider = new Pact({
    consumer: "web-app",
    provider: "worker",
    port: 8080
  });

  it("should accept ingestion job with correct schema", async () => {
    await provider.addInteraction({
      state: "queue is ready",
      uponReceiving: "an ingestion job",
      withRequest: {
        method: "POST",
        path: "/queue/ingestion",
        body: {
          projectId: "proj_123",
          events: [{ type: "trace", id: "trace_123" }]
        }
      },
      willRespondWith: {
        status: 200,
        body: { jobId: "job_123" }
      }
    });

    await provider.verify();
  });
});
```

**Benefits:**
- Catches contract violations early
- Documents expected interfaces
- Safer refactoring

**Effort:** Medium

---

### 5.2 Add Snapshot Testing for Database Migrations

**Problem:** Migrations can break queries

**Solution:**
```typescript
// packages/shared/prisma/tests/migration-snapshots.test.ts
import { execSync } from "child_process";
import { readFileSync } from "fs";

describe("Migration Snapshots", () => {
  it("should generate expected schema", () => {
    // Run migrations on test DB
    execSync("pnpm run db:migrate", { cwd: __dirname });

    // Generate current schema
    execSync("pnpm prisma db pull --print", { cwd: __dirname });
    const currentSchema = readFileSync("schema.prisma", "utf-8");

    // Compare to snapshot
    expect(currentSchema).toMatchSnapshot();
  });
});
```

**Benefits:**
- Detects unintended schema changes
- Documents schema evolution
- Catches migration errors

**Effort:** Low

---

## 6. Migration & Technical Debt

### 6.1 Migrate to Next.js App Router

**Problem:** Pages Router is legacy, App Router is future

**Solution - Incremental migration strategy:**

```
Phase 1: New features in App Router
  - Keep existing pages in /pages/
  - Add new pages in /app/
  - Next.js supports both simultaneously

Phase 2: Migrate read-only pages
  - Start with viewer pages (no auth complexity)
  - Use Server Components for performance
  - Migrate layouts first

Phase 3: Migrate interactive pages
  - Use Server Actions for mutations
  - Migrate tRPC to Server Components + Actions
  - Update authentication to App Router patterns

Phase 4: Remove Pages Router
  - Delete /pages/ (except API routes)
  - Keep /pages/api/ for tRPC + Public API
```

**Benefits:**
- Better performance (Server Components)
- Improved layouts and nested routing
- Future-proof architecture

**Effort:** Very High (months of work)

---

### 6.2 Extract Features to Package Boundaries

**Problem:** Features tightly coupled to monolith

**Solution:**
```typescript
// Restructure as vertical slices with clear boundaries
packages/
├── feature-datasets/
│   ├── src/
│   │   ├── api/          # Public API (exports tRPC router)
│   │   ├── components/   # UI components
│   │   ├── domain/       # Business logic
│   │   └── db/           # Database queries
│   ├── package.json
│   └── tsconfig.json
├── feature-evals/
│   └── ...
└── feature-prompts/
    └── ...

// Web app imports features
import { datasetRouter } from "@langfuse/feature-datasets";
import { DatasetTable } from "@langfuse/feature-datasets/components";
```

**Benefits:**
- Clearer boundaries
- Easier to extract to microservices later
- Better encapsulation

**Effort:** Very High (major refactor)

---

## Summary of Recommendations

| Recommendation | Impact | Effort | Priority |
|---------------|--------|--------|----------|
| Split large router files | Medium | Medium | High |
| Query result caching | High | Medium | High |
| Queue health monitoring | Medium | Low | High |
| Enforce feature boundaries | Medium | Low | Medium |
| Extract service layer | High | High | Medium |
| Read replicas | High | Low | High |
| OpenAPI docs generation | Medium | Medium | Medium |
| Decouple S3 from ingestion | High | Medium | Low |
| Contract testing | Medium | Medium | Low |
| Migration snapshot tests | Low | Low | Medium |
| Migrate to App Router | High | Very High | Low |
| Extract features to packages | High | Very High | Low |

---

**Next:** [Part 5: Reference Materials →](./analysis-05-reference.md)
