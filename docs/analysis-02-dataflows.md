# Langfuse Architecture Analysis - Part 2: Data Flows & Integration

> **Navigation:** [📋 Index](./analysis-index.md) | [← Part 1: Architecture](./analysis-01-architecture.md) | **Part 2: Data Flows** | [Part 3: Analysis →](./analysis-03-analysis.md)

---

## 1. Frontend → tRPC → Database

This flow shows how a typical user interaction in the web UI retrieves data from the database.

```mermaid
sequenceDiagram
    participant React as React Component
    participant Hook as tRPC Hook
    participant RQ as React Query
    participant HTTP as HTTP Layer
    participant Handler as Next.js API Handler
    participant Router as tRPC Router
    participant MW as Middleware Stack
    participant Prisma as Prisma Client
    participant PG as PostgreSQL

    React->>Hook: api.datasets.allDatasets.useQuery({ projectId })
    Hook->>RQ: Check cache

    alt Cache hit
        RQ-->>React: Return cached data
    else Cache miss/stale
        RQ->>HTTP: POST /api/trpc/datasets.allDatasets
        HTTP->>Handler: Request + SuperJSON
        Handler->>Router: Route to procedure
        Router->>MW: Execute middleware chain

        MW->>MW: OpenTelemetry instrumentation
        MW->>MW: Tracing (Baselime)
        MW->>MW: Error handling wrapper
        MW->>MW: enforceUserIsAuthedAndProjectMember

        MW->>Router: Context + Input validated
        Router->>Prisma: prisma.dataset.findMany()
        Prisma->>PG: SELECT * FROM datasets WHERE...
        PG-->>Prisma: Result rows
        Prisma-->>Router: Typed Dataset[]

        Router-->>Handler: Response data
        Handler-->>HTTP: SuperJSON serialized
        HTTP-->>RQ: Update cache
        RQ-->>React: Render with data
    end
```

**Key Points:**
- React Query provides automatic caching
- SuperJSON handles complex types (Date, BigInt, etc.)
- Middleware chain enforces auth and observability
- Type safety maintained end-to-end

---

## 2. SDK → S3 → Worker → Databases (Ingestion Pipeline)

This is the complete flow for ingesting trace data from SDKs into Langfuse.

```mermaid
flowchart TB
    Start([SDK Event])

    Start --> API[POST /api/public/ingestion]

    API --> Validate{Validate<br/>Auth & Schema}
    Validate -->|Invalid| Error[Return 400/401]
    Validate -->|Valid| S3Write[Write to S3<br/>projectId/traces/traceId/eventId.json]

    S3Write --> QueueAdd[Add to IngestionQueue<br/>shard by projectId hash]
    QueueAdd --> Return[Return 202 Accepted]

    QueueAdd --> Worker[Worker pulls job]
    Worker --> Download[Download S3 files]
    Download --> Dedupe{Redis<br/>dedup check}

    Dedupe -->|Duplicate| Skip[Skip event]
    Dedupe -->|New| Merge[Merge events for same entity]

    Merge --> Enrich[Enrich: tokens, cost, validation]

    Enrich --> WriteDB{Dual Database Write}

    WriteDB --> PGWrite[PostgreSQL via Prisma<br/>Trace/Observation metadata]
    WriteDB --> CHWrite[ClickHouse batched insert<br/>Full analytics data]

    PGWrite --> SetCache[Set Redis dedup key TTL:24h]
    CHWrite --> SetCache

    SetCache --> Trigger[Trigger downstream queues]
    Trigger --> TraceQueue[TraceUpsertQueue<br/>for evaluations]
    Trigger --> EventQueue[EventPropagationQueue<br/>for staging table]

    TraceQueue --> EvalCheck{Matches<br/>JobConfig?}
    EvalCheck -->|Yes| EvalExec[EvaluationExecution Queue]
    EvalCheck -->|No| Done[Done]

    EvalExec --> LLMCall[Call LLM API]
    LLMCall --> ScoreWrite[Write Score to ClickHouse]
    ScoreWrite --> Done

    style Start fill:#e1f5ff
    style S3Write fill:#ffe1e1
    style WriteDB fill:#e1ffe1
    style EvalExec fill:#fff4e1
```

**Key Components:**

1. **S3 Event Storage** (`/worker/src/queues/ingestionQueue.ts:81-150`)
   - Events stored as JSON files
   - Batched downloads for efficiency
   - Fallback to legacy queue if S3 fails

2. **Redis Deduplication** (`/worker/src/queues/ingestionQueue.ts:81-103`)
   ```typescript
   const key = `langfuse:ingestion:recently-processed:${projectId}:${type}:${eventBodyId}:${fileKey}`;
   const exists = await redis.exists(key);
   if (exists) return; // Skip duplicate
   await redis.set(key, "1", "EX", 86400); // 24h TTL
   ```

3. **IngestionService** (`/worker/src/services/IngestionService/index.ts`)
   - Merges multiple events for same trace/observation
   - Async tokenization (tiktoken via worker threads)
   - Cost calculation from model pricing table
   - Validation against prompt versions and models
   - Dual database writes

4. **ClickhouseWriter** (`/worker/src/services/ClickhouseWriter/`)
   - Batched async inserts
   - Table-specific insert queues
   - Automatic flushing on size/time thresholds
   - Handles INSERT failures with retries

---

## 3. Detailed Ingestion Sequence

This shows the full detail of the ingestion pipeline with all participants:

```mermaid
sequenceDiagram
    participant SDK as SDK/API Client
    participant API as Web API Handler
    participant S3 as S3 Bucket
    participant Queue as IngestionQueue (Redis)
    participant Worker as Worker Process
    participant Redis as Redis Cache
    participant Ingestion as IngestionService
    participant PG as PostgreSQL
    participant CH as ClickHouse
    participant Downstream as Downstream Queues

    SDK->>API: POST /api/public/ingestion<br/>{ trace, observations }
    API->>API: Validate & authenticate
    API->>S3: Write event JSON<br/>s3://bucket/projectId/traces/traceId/eventId.json
    API->>Queue: Add job { projectId, s3Keys }
    Queue-->>API: Job ID
    API-->>SDK: 202 Accepted

    Worker->>Queue: Pull job (consistent hash shard)
    Queue-->>Worker: Job data
    Worker->>S3: Download event files
    S3-->>Worker: Event JSON array

    Worker->>Redis: Check deduplication<br/>key: projectId:type:id:fileKey

    alt Event already processed
        Redis-->>Worker: exists = true
        Worker->>Worker: Skip event
    else New event
        Redis-->>Worker: exists = false
        Worker->>Ingestion: mergeAndWrite(events)

        Ingestion->>Ingestion: Merge multiple events<br/>for same entity
        Ingestion->>Ingestion: Calculate tokens & costs
        Ingestion->>Ingestion: Validate prompts/models

        par Dual Database Write
            Ingestion->>PG: Prisma upsert<br/>(metadata, relationships)
            PG-->>Ingestion: OK
        and
            Ingestion->>CH: ClickhouseWriter.insert<br/>(analytics data, batched)
            CH-->>Ingestion: OK
        end

        Ingestion->>Redis: Set deduplication key<br/>TTL: 24h

        Ingestion->>Downstream: Trigger TraceUpsertQueue
        Ingestion->>Downstream: Trigger EventPropagationQueue

        Ingestion-->>Worker: Success
    end

    Worker->>Queue: Complete job
```

---

## 4. Evaluation Trigger Flow

This flow shows how evaluations are automatically triggered after trace ingestion:

```mermaid
graph TB
    Upsert[Trace/Observation Upserted]

    Upsert --> TUQ[TraceUpsertQueue Job]

    TUQ --> Worker1[Worker: traceUpsertQueueProcessor]

    Worker1 --> FetchConfigs[Fetch JobConfigurations<br/>for project]

    FetchConfigs --> MatchLoop{For each config}

    MatchLoop --> FilterCheck[Check filter match:<br/>trace name, metadata, tags]

    FilterCheck -->|No match| Next[Next config]
    FilterCheck -->|Match| Delay{Eval delay?}

    Delay -->|Yes| Schedule[Schedule EvalJobCreator<br/>with delay]
    Delay -->|No| CreateImmediate[Add to CreateEvalQueue]

    Schedule --> CreateEvalQueue[CreateEvalQueue]
    CreateImmediate --> CreateEvalQueue

    CreateEvalQueue --> Worker2[Worker: evalJobCreatorQueueProcessor]

    Worker2 --> FetchTemplate[Fetch EvalTemplate]
    Worker2 --> CheckExisting{Eval already<br/>exists?}

    CheckExisting -->|Yes| End1[Skip]
    CheckExisting -->|No| CreateExec[Create EvalJobExecution record]

    CreateExec --> ExecQueue[Add to EvaluationExecutionQueue]

    ExecQueue --> Worker3[Worker: evalJobExecutorQueueProcessor]

    Worker3 --> FetchData[Fetch trace/observation data]
    Worker3 --> BuildPrompt[Build eval prompt from template]
    Worker3 --> CallLLM[Call LLM API<br/>OpenAI, Anthropic, etc.]

    CallLLM --> ParseResponse[Parse response]
    ParseResponse --> CreateScore[Create Score record]

    CreateScore --> WriteCH[Write to ClickHouse scores table]
    CreateScore --> UpdateJob[Update JobExecution status]

    WriteCH --> End2[Done]
    UpdateJob --> End2
    Next --> End3[Done]

    style Upsert fill:#e1f5ff
    style CallLLM fill:#ffe1e1
    style WriteCH fill:#e1ffe1
```

**Configuration-Driven Evaluation:**

- `JobConfiguration` table defines eval rules
- Matches traces based on filters (trace name, metadata, etc.)
- Triggers evaluation jobs automatically
- Supports custom LLM-based evaluators

---

## 5. Public API Request Flow

This shows how external API requests are processed:

```mermaid
sequenceDiagram
    participant SDK as SDK Client
    participant API as Public API Endpoint
    participant MW as Middleware Stack
    participant Auth as Auth Layer
    participant RL as Rate Limiter
    participant Logic as Business Logic
    participant DB as Database
    participant S3 as S3 Storage
    participant Queue as BullMQ Queue

    SDK->>API: POST /api/public/dataset-items
    API->>MW: withMiddlewares wrapper
    MW->>MW: CORS handling
    MW->>MW: OpenTelemetry context
    MW->>Auth: Validate API key
    Auth-->>MW: { projectId, orgId, apiKeyId }
    MW->>RL: Check rate limits
    RL-->>MW: Allow/Deny
    MW->>MW: Zod v4 schema validation
    MW->>Logic: Execute route function

    alt Ingestion Endpoint
        Logic->>S3: Store event JSON
        Logic->>Queue: Add ingestion job
        Logic-->>SDK: 202 Accepted
    else Sync Endpoint
        Logic->>DB: Prisma query/mutation
        Logic->>Logic: Audit logging
        DB-->>Logic: Result
        Logic-->>SDK: 200 OK + Data
    end
```

**Middleware Chain:**
1. CORS preflight handling
2. OpenTelemetry context propagation
3. Error handling (BaseError, ClickHouseResourceError, ZodError)
4. API key authentication
5. Rate limiting
6. Zod v4 schema validation
7. Route execution

---

## 6. Data Flow Summary

### Write Paths
1. **User Actions (Web UI)** → tRPC → PostgreSQL
2. **SDK Events** → Public API → S3 → Queue → Worker → PostgreSQL + ClickHouse
3. **Evaluations** → Queue → Worker → ClickHouse

### Read Paths
1. **UI Dashboard** → tRPC → ClickHouse (analytics)
2. **UI Settings** → tRPC → PostgreSQL (config)
3. **Public API** → Direct → PostgreSQL/ClickHouse

### Async Processing
- **Ingestion:** S3 → Queue → Worker
- **Evaluations:** Trace Upsert → Eval Creator → Eval Executor
- **Exports:** Export Request → Queue → Worker → S3
- **Integrations:** Event → Queue → Worker → External API

---

**Next:** [Part 3: Pros, Cons & Scale Analysis →](./analysis-03-analysis.md)
