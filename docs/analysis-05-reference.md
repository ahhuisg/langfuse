# Langfuse Architecture Analysis - Part 5: Reference Materials

> **Navigation:** [📋 Index](./analysis-index.md) | [← Part 4: Recommendations](./analysis-04-recommendations.md) | **Part 5: Reference**

---

## 1. Critical File Paths Reference

### 1.1 Configuration Files

| File | Purpose |
|------|---------|
| `/pnpm-workspace.yaml` | Monorepo package definitions |
| `/turbo.json` | Turbo build caching configuration |
| `/.nvmrc` | Node.js version (v24) |
| `/web/src/env.mjs` | Web environment variables |
| `/worker/src/env.ts` | Worker environment variables |
| `/packages/shared/prisma/schema.prisma` | Database schema |

### 1.2 Core Entry Points

| File | Purpose |
|------|---------|
| `/web/src/pages/_app.tsx` | Next.js app wrapper |
| `/web/src/pages/api/trpc/[trpc].ts` | tRPC API handler |
| `/web/src/server/api/root.ts` | tRPC router aggregation |
| `/web/src/server/api/trpc.ts` | tRPC procedures & middleware |
| `/worker/src/app.ts` | Worker queue registrations (522 lines) |
| `/worker/src/index.ts` | Worker HTTP server |

### 1.3 Key Feature Implementations

| Feature | Router | Lines | Notes |
|---------|--------|-------|-------|
| Datasets | `/features/datasets/server/dataset-router.ts` | 2177 | Large, should split |
| Evals | `/features/evals/server/evals-router.ts` | ~500 | Well-sized |
| Prompts | `/features/prompts/server/prompt-router.ts` | ~600 | Well-sized |
| Traces | `/features/traces/server/traces-router.ts` | ~1000 | Consider splitting |

### 1.4 Infrastructure Services

| Service | Location |
|---------|----------|
| Ingestion | `/worker/src/services/IngestionService/` |
| ClickHouse Writer | `/worker/src/services/ClickhouseWriter/` |
| Queue Definitions | `/packages/shared/src/server/queues/` |
| Error Handling | `/packages/shared/src/errors/` |
| Database Clients | `/packages/shared/src/server/db.ts` |

---

## 2. Useful Commands

### Development Commands

```bash
# Installation
pnpm i               # Install dependencies

# Development
pnpm run dev         # Start all services (web + worker)
pnpm run dev:web     # Web app only (localhost:3000) - MOST COMMON
pnpm run dev:worker  # Worker only
pnpm run dx          # Full initial setup: install deps, reset DBs, seed data
```

### Database Commands

```bash
# Navigate to shared package first
cd packages/shared

# Database operations
pnpm run db:generate       # Build prisma models
pnpm run db:migrate        # Run Prisma migrations
pnpm run db:reset          # Reset and reseed databases
pnpm run db:seed           # Seed with example data
```

### Infrastructure Commands

```bash
# Docker services
pnpm run infra:dev:up      # Start Docker services (PostgreSQL, ClickHouse, Redis, MinIO)
pnpm run infra:dev:down    # Stop Docker services
```

### Building

```bash
# Build specific packages
pnpm --filter=web run build      # Build web app
pnpm --filter=worker run build   # Build worker
pnpm --filter=PACKAGE_NAME run build  # Build any package
```

### Testing

```bash
# Web package (uses JEST)
# Sync tests
pnpm test-sync --testPathPattern="$FILE_LOCATION_PATTERN" --testNamePattern="$TEST_NAME_PATTERN"

# Async tests
pnpm test -- --testPathPattern="$FILE_LOCATION_PATTERN" --testNamePattern="$TEST_NAME_PATTERN"

# Client tests
pnpm test-client --testPathPattern="buildStepData" --testNamePattern="buildStepData"

# Worker package (uses vitest)
pnpm run test --filter=worker -- $TEST_FILE_NAME -t "$TEST_NAME"
```

### Utilities

```bash
# Code quality
pnpm run format            # Format code across entire project
pnpm run lint              # Lint all code

# Nuclear option (use with caution!)
pnpm run nuke              # Remove all node_modules, build files, wipe database, docker containers
```

---

## 3. Local Development Setup

### Environment Variables

Copy `.env.dev.example` to `.env` in the project root.

### Prerequisites

- **Node.js:** Version 24 (specified in `.nvmrc`)
- **Package Manager:** pnpm v9.5.0
- **Docker:** For local PostgreSQL, ClickHouse, Redis, MinIO

### Login Credentials for Development

When running locally with seed data:

- **Username:** `demo@langfuse.com`
- **Password:** `password`
- **Demo project URL:** `http://localhost:3000/project/7a88fb47-b4e2-43b8-a06c-a5ce950dc53a`

---

## 4. Key Dependencies

### Web Application

- `@trpc/server`, `@trpc/client`, `@trpc/react-query` - tRPC
- `next-auth` - Authentication
- `@prisma/client` - Database ORM
- `@clickhouse/client` - Analytics database
- `ioredis` - Redis client
- `zod` - Validation
- `superjson` - Serialization
- `@radix-ui/*` - UI primitives
- `recharts`, `@tremor/react` - Charts
- `ai` - Vercel AI SDK

### Worker Application

- `bullmq` - Job queues
- `@aws-sdk/client-s3` - S3 client
- `tiktoken` - Tokenization
- `express` - HTTP server

### Shared

- `prisma` - ORM
- `kysely` - Query builder
- `zod` - Schemas

---

## 5. Architecture Patterns

### Frontend Patterns

- **Feature-based organization:** `/web/src/features/[feature-name]/`
- **tRPC for API calls:** Full type safety
- **React Query for state:** Automatic caching and revalidation
- **shadcn/ui components:** Consistent UI primitives

### Backend Patterns

- **tRPC routers:** Type-safe server procedures
- **Middleware chains:** Auth, tracing, error handling
- **Service pattern:** (Recommended) Extract business logic from routers
- **Dual database writes:** PostgreSQL for metadata, ClickHouse for analytics

### Worker Patterns

- **BullMQ queues:** Async job processing
- **Sharded queues:** Parallel processing by projectId
- **S3 event storage:** Durability and replay capability
- **Batch processing:** Efficient ClickHouse inserts

### Database Patterns

- **Prisma for simple queries:** Type-safe, relational
- **Kysely for complex queries:** SQL builder with CTEs
- **Raw SQL for flexibility:** When needed
- **ClickHouse client:** Analytics queries with parameters

---

## 6. Common Troubleshooting

### Database Issues

```bash
# Reset databases
cd packages/shared
pnpm run db:reset

# Check database connection
pnpm run db:generate

# View database in GUI
pnpm prisma studio
```

### Docker Issues

```bash
# Stop all containers
pnpm run infra:dev:down

# Remove volumes and restart
docker-compose down -v
pnpm run infra:dev:up
```

### Build Issues

```bash
# Clean build artifacts
pnpm run clean

# Remove node_modules and reinstall
rm -rf node_modules
pnpm i

# Run linter first (recommended before building)
pnpm run lint
```

### Queue Issues

```bash
# Check Redis connection
redis-cli ping

# View queue contents (in Redis)
redis-cli
> KEYS langfuse:*
> LLEN langfuse:queue:ingestion
```

---

## 7. Development Best Practices

### Code Conventions

- **Conventional commits:** Follow conventional commit format on main branch
- **TypeScript throughout:** Avoid `any` type when possible
- **Zod v4 for validation:** Always import from `zod/v4`
- **CSS variables for theming:** Supports auto dark/light mode
- **No moving functions unnecessarily:** For easier code reviews

### Testing Guidelines

- **Decouple test blocks:** Each `it`/`test` should run independently
- **Avoid `pruneDatabase` in async tests:** Especially in `__tests__/async` directory
- **Tests must pass before pushing:** For backend/API changes
- **Focus on API and integration tests:** Unit tests where appropriate

### Git Workflow

- **Never update git config**
- **Never skip hooks** (`--no-verify`, `--no-gpg-sign`) unless explicitly requested
- **Never force push to main/master**
- **Check authorship before amending:** `git log -1 --format='%an %ae'`

### Frontend Tips

- **Window location handling:** Ensure proper basePath handling
- **tRPC hooks:** Preferred over direct API calls
- **Feature-based imports:** Import from feature directories, not across features

---

## 8. Monitoring & Observability

### OpenTelemetry

All requests are instrumented with OpenTelemetry:

- **Middleware:** `withOtelInstrumentation`
- **Error tracking:** `traceException()` helper
- **Context propagation:** Across services

### Logging

- **Structured logging:** Use appropriate log levels
- **Error handling:** Custom error classes in `/packages/shared/src/errors/`
- **Audit logging:** For sensitive operations

### Metrics (Recommended to add)

Consider adding:
- Queue depth and processing time
- Database query performance
- API endpoint latency
- Worker job success/failure rates

---

## 9. Security Considerations

### Authentication

- **NextAuth.js:** Web UI session-based auth
- **API keys:** For SDK and public API access
- **OAuth providers:** Google, GitHub, Azure AD, SAML

### Authorization

- **RBAC:** 4 role levels (Owner, Admin, Member, Viewer)
- **Multi-tenancy:** projectId filtering on all queries
- **Middleware enforcement:** tRPC and API middleware

### Input Validation

- **Zod v4 schemas:** Runtime validation
- **Type safety:** TypeScript compile-time checks
- **SQL injection prevention:** Parameterized queries

---

## 10. Additional Resources

### Documentation

- **Main repo:** https://github.com/langfuse/langfuse
- **CLAUDE.md:** Project-specific guidelines
- **RBAC README:** `/web/src/features/rbac/README.md`
- **Entitlements README:** `/web/src/features/entitlements/README.md`

### External Resources

- **tRPC:** https://trpc.io
- **Prisma:** https://www.prisma.io
- **Next.js:** https://nextjs.org
- **BullMQ:** https://docs.bullmq.io
- **ClickHouse:** https://clickhouse.com/docs

---

## Document Metadata

- **Version:** 1.0
- **Last Updated:** 2025-11-13
- **Author:** Architecture Analysis via Claude Code
- **Repository:** https://github.com/langfuse/langfuse
- **Total Analysis Size:** ~2200 lines (original), split into 5 parts for GitHub rendering

---

**End of Architecture Analysis**

[← Back to Index](./analysis-index.md)
