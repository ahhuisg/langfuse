# Langfuse Architecture Analysis - Part 1: System Architecture

> **Navigation:** [📋 Index](./analysis-index.md) | **Part 1: Architecture** | [Part 2: Data Flows →](./analysis-02-dataflows.md)

---

## 1. System Architecture Overview

### 1.1 High-Level Component Diagram

```mermaid
graph TB
    subgraph "Client Layer"
        UI[Web UI - React/Next.js]
        SDK[SDKs - Python/TypeScript]
    end

    subgraph "Web Application - Next.js 14"
        Pages[Pages Router]
        tRPC[tRPC API Layer]
        PublicAPI[Public REST API]
        Auth[NextAuth.js]
        Components[UI Components]
    end

    subgraph "Worker Application - Express.js"
        Queues[BullMQ Job Queues]
        Processors[Queue Processors]
        IngestionService[Ingestion Service]
        EvalService[Evaluation Service]
    end

    subgraph "Data Layer"
        PostgreSQL[(PostgreSQL<br/>Metadata & Relations)]
        ClickHouse[(ClickHouse<br/>Analytics & Time-Series)]
        Redis[(Redis<br/>Cache & Queues)]
        S3[S3/MinIO<br/>Object Storage]
    end

    subgraph "Shared Packages"
        Prisma[Prisma Schema]
        Types[Shared Types]
        Utils[Shared Utils]
    end

    UI -->|tRPC| tRPC
    SDK -->|REST| PublicAPI

    Pages --> tRPC
    Pages --> Components
    tRPC --> Auth
    PublicAPI --> Auth

    tRPC --> PostgreSQL
    tRPC --> ClickHouse
    PublicAPI --> S3
    PublicAPI --> Queues

    Queues --> Processors
    Processors --> IngestionService
    Processors --> EvalService

    IngestionService --> PostgreSQL
    IngestionService --> ClickHouse
    IngestionService --> Redis
    IngestionService --> S3

    tRPC -.-> Prisma
    Processors -.-> Prisma
    Processors -.-> Types
    tRPC -.-> Utils
```

### 1.2 Monorepo Structure

```mermaid
graph LR
    Root[langfuse/]

    Root --> Web[web/<br/>Next.js App]
    Root --> Worker[worker/<br/>Express App]
    Root --> Packages[packages/]
    Root --> EE[ee/<br/>Enterprise]

    Packages --> Shared[shared/<br/>Prisma + Types]
    Packages --> ESLint[config-eslint/]
    Packages --> TS[config-typescript/]

    Web -.depends on.-> Shared
    Worker -.depends on.-> Shared
    EE -.depends on.-> Shared

    style Web fill:#e1f5ff
    style Worker fill:#ffe1e1
    style Shared fill:#e1ffe1
    style EE fill:#fff4e1
```

---

## 2. Web Application Architecture

### 2.1 Feature-Based Organization

The web application contains **50+ feature directories** in `/web/src/features/`:

```
features/
├── datasets/          # Dataset management
├── evals/            # Evaluation system
├── prompts/          # Prompt versioning
├── projects/         # Project settings
├── rbac/             # Access control
├── public-api/       # Public API types
├── auth/             # Authentication
├── organizations/    # Org management
├── dashboard/        # Analytics
├── experiments/      # A/B testing
└── ... (40+ more)
```

**Standard Feature Structure:**

```
features/[feature-name]/
├── components/          # React UI components
├── contexts/           # React contexts
├── server/
│   ├── [feature]Router.ts   # tRPC router
│   ├── service.ts          # Business logic
│   └── actions/            # Server actions
├── hooks/             # Custom React hooks
├── types/             # TypeScript types
└── utils/             # Helper functions
```

### 2.2 tRPC Procedure Types

```mermaid
graph LR
    Base[baseProcedure]

    Base -->|+ error handling| Public[publicProcedure]
    Public -->|+ auth check| Authed[authenticatedProcedure]
    Authed -->|+ project membership| Project[protectedProjectProcedure]
    Authed -->|+ org membership| Org[protectedOrgProcedure]
    Authed -->|+ admin check| Admin[adminProcedure]

    style Base fill:#f0f0f0
    style Public fill:#e1f5ff
    style Authed fill:#ffe1e1
    style Project fill:#e1ffe1
```

**Context Creation** (`/web/src/server/api/trpc.ts`):

```typescript
export const createInnerTRPCContext = (opts: CreateContextOptions) => {
  return {
    session: opts.session,      // User session (NextAuth)
    headers: opts.headers,       // HTTP headers
    prisma,                      // Prisma client
    DB,                          // Kysely query builder
  };
};
```

### 2.3 Public API Architecture

**Location:** `/web/src/pages/api/public/`

**Middleware Chain** (`/features/public-api/server/withMiddlewares.ts`):

1. CORS preflight handling
2. OpenTelemetry context propagation
3. Error handling (BaseError, ClickHouseResourceError, ZodError)
4. Route execution

**API Route Helpers:**

- `createAuthedAPIRoute` - Requires API key, no specific project
- `createAuthedProjectAPIRoute` - Requires project-scoped API key
- `createLegacyAuthedAPIRoute` - Legacy authentication

### 2.4 Pages and Routing

**Structure:**

```
pages/
├── _app.tsx                          # App wrapper with providers
├── api/
│   ├── trpc/[trpc].ts               # tRPC endpoint
│   └── public/                       # Public API endpoints
├── auth/                             # Auth pages (sign-in, etc.)
├── project/[projectId]/              # Project-scoped pages
│   ├── datasets/
│   ├── traces/
│   ├── prompts/
│   └── settings/
└── organization/[organizationId]/    # Org-scoped pages
    ├── settings/
    └── members/
```

---

## 3. Worker Application Architecture

### 3.1 BullMQ Queue System

**Entry Point:** `/worker/src/app.ts` (522 lines)

The worker registers **30+ queue processors**:

```mermaid
graph TB
    subgraph "Queue Registration"
        Manager[WorkerManager]
    end

    subgraph "Core Queues"
        Ingestion[IngestionQueue<br/>Sharded by projectId]
        OtelIngestion[OtelIngestionQueue]
        TraceUpsert[TraceUpsertQueue]
        EvalExec[EvaluationExecution]
    end

    subgraph "Data Processing"
        BatchExport[BatchExportQueue]
        DatasetRun[DatasetRunItemUpsert]
        EventProp[EventPropagationQueue]
    end

    subgraph "Integrations"
        PostHog[PostHogIntegrationQueue]
        Webhook[WebhookQueue]
        CloudUsage[CloudUsageMeteringQueue]
    end

    Manager --> Ingestion
    Manager --> OtelIngestion
    Manager --> TraceUpsert
    Manager --> EvalExec
    Manager --> BatchExport
    Manager --> DatasetRun
    Manager --> EventProp
    Manager --> PostHog
    Manager --> Webhook
    Manager --> CloudUsage

    style Ingestion fill:#e1f5ff
    style EvalExec fill:#ffe1e1
    style BatchExport fill:#e1ffe1
```

**Worker Registration Pattern:**

```typescript
// Check if queue is enabled via env var
if (env.QUEUE_CONSUMER_INGESTION_QUEUE_IS_ENABLED === "true") {
  // Get shard names for parallel processing
  const shardNames = IngestionQueue.getShardNames();

  shardNames.forEach((shardName) => {
    WorkerManager.register(
      shardName as QueueName,
      ingestionQueueProcessorBuilder(true),
      {
        concurrency: env.LANGFUSE_INGESTION_QUEUE_PROCESSING_CONCURRENCY,
        lockDuration: 60000,      // 60s job lock
        stalledInterval: 120000   // 120s stalled check
      }
    );
  });
}
```

### 3.2 Key Components

**IngestionService** (`/worker/src/services/IngestionService/index.ts`):
- Merges multiple events for same trace/observation
- Async tokenization (tiktoken via worker threads)
- Cost calculation from model pricing table
- Validation against prompt versions and models
- Dual database writes

**ClickhouseWriter** (`/worker/src/services/ClickhouseWriter/`):
- Batched async inserts
- Table-specific insert queues
- Automatic flushing on size/time thresholds
- Handles INSERT failures with retries

---

## 4. Database Architecture

### 4.1 Dual Database Strategy

```mermaid
graph LR
    subgraph "Write Path"
        Event[Event Ingestion]
    end

    subgraph "PostgreSQL"
        PG[PostgreSQL via Prisma]
        PGData[User Management<br/>Projects<br/>Datasets<br/>Job Configs<br/>Relationships]
    end

    subgraph "ClickHouse"
        CH[ClickHouse]
        CHData[Traces<br/>Observations<br/>Scores<br/>Time-series Data<br/>High-volume Inserts]
    end

    subgraph "Read Path"
        UI[Web UI Queries]
        Analytics[Analytics Dashboard]
        Export[Data Export]
    end

    Event -->|Metadata| PG
    Event -->|Analytics| CH

    PG --> PGData
    CH --> CHData

    UI -->|Config/Relations| PG
    UI -->|Trace Data| CH
    Analytics -->|Aggregations| CH
    Export -->|Bulk Data| CH

    style PG fill:#e1f5ff
    style CH fill:#ffe1e1
```

**Division of Responsibility:**

| PostgreSQL | ClickHouse |
|------------|------------|
| Users, Organizations | Trace records |
| Projects, Memberships | Observation records |
| Datasets, Items | Score records |
| Job Configurations | High-cardinality analytics |
| Prompt Versions | Time-series aggregations |
| Model Pricing | Session analytics |
| RBAC & Permissions | Generation metrics |
| Foreign Key Constraints | Event logs |

### 4.2 Database Interaction Patterns

**Three Query Approaches:**

1. **Prisma ORM** - Type-safe, relational
2. **Kysely Query Builder** - Complex SQL, CTEs
3. **Raw SQL** - Maximum flexibility

**ClickHouse Client** (`@clickhouse/client`):
```typescript
const result = await clickhouseClient.query({
  query: `
    SELECT
      project_id,
      count() as trace_count,
      sum(total_cost) as total_cost
    FROM traces
    WHERE timestamp >= {start:DateTime64}
      AND timestamp < {end:DateTime64}
      AND project_id = {projectId:String}
    GROUP BY project_id
  `,
  query_params: { start, end, projectId },
  format: "JSONEachRow"
});
```

---

## 5. Authentication & Authorization

### 5.1 RBAC Role Hierarchy

```mermaid
graph TB
    Owner[OWNER<br/>Full access + billing]
    Admin[ADMIN<br/>Full access except billing]
    Member[MEMBER<br/>Read + limited write]
    Viewer[VIEWER<br/>Read-only access]

    Owner -->|inherits| Admin
    Admin -->|inherits| Member
    Member -->|inherits| Viewer

    style Owner fill:#ff6b6b
    style Admin fill:#ffa500
    style Member fill:#4dabf7
    style Viewer fill:#51cf66
```

**Permission Scopes**:

| Scope | Owner | Admin | Member | Viewer |
|-------|-------|-------|--------|--------|
| Project Settings | ✅ | ✅ | ❌ | ❌ |
| Create Prompts | ✅ | ✅ | ✅ | ❌ |
| Delete Traces | ✅ | ✅ | ❌ | ❌ |
| Manage API Keys | ✅ | ✅ | ❌ | ❌ |
| View Dashboard | ✅ | ✅ | ✅ | ✅ |
| Create Datasets | ✅ | ✅ | ✅ | ❌ |
| Score Traces | ✅ | ✅ | ✅ | ❌ |

### 5.2 API Key Scopes

**API keys can be:**
- **Project-scoped:** Access to single project
- **Organization-scoped:** Access to all projects in org

---

**Next:** [Part 2: Data Flows & Integration →](./analysis-02-dataflows.md)
