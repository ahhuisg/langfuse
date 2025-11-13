# Langfuse System Architecture

> **Part 1 of 4** - [Overview](analysis.md) | [System Architecture](analysis-system-architecture.md) | [Data Flows](analysis-data-flows.md) | [Recommendations](analysis-recommendations.md)

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

## 3. Worker Application Architecture

### 3.1 BullMQ Queue System

The worker registers **30+ queue processors**:

```mermaid
graph TB
    subgraph "Queue Registration - app.ts"
        Manager[WorkerManager]
    end

    subgraph "Core Queues"
        Ingestion[IngestionQueue<br/>Sharded by projectId]
        OtelIngestion[OtelIngestionQueue]
        TraceUpsert[TraceUpsertQueue]
        EvalExec[EvaluationExecution]
    end

    subgraph "Data Processing Queues"
        BatchExport[BatchExportQueue]
        DatasetRun[DatasetRunItemUpsert]
        EventProp[EventPropagationQueue]
    end

    subgraph "Integration Queues"
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

### 3.2 Evaluation Pipeline

```mermaid
graph TB
    Start[Trace Upserted]

    Start --> TUQ[TraceUpsertQueue Job]
    TUQ --> Check[Check JobConfigurations]
    Check --> Match{Matches<br/>Eval Rules?}

    Match -->|Yes| CreateJob[CreateEvalQueue Job]
    Match -->|No| End[End]

    CreateJob --> Creator[evalJobCreatorQueueProcessor]
    Creator --> FindConfig[Find EvalTemplate Config]
    Creator --> CreateExec[Create EvaluationExecution Job]

    CreateExec --> ExecQueue[EvaluationExecutionQueue]
    ExecQueue --> Executor[evalJobExecutorQueueProcessor]

    Executor --> FetchData[Fetch Trace/Observation Data]
    Executor --> BuildPrompt[Build Eval Prompt from Template]
    Executor --> CallLLM[Call LLM API<br/>OpenAI, Anthropic, etc.]

    CallLLM --> Parse[Parse Response]
    Parse --> StoreScore[Store Score in ClickHouse]
    StoreScore --> Update[Update Job Status]

    style Start fill:#e1f5ff
    style CreateExec fill:#ffe1e1
    style StoreScore fill:#e1ffe1
```

## 4. Database Architecture

### 4.1 Dual Database Strategy

```mermaid
graph LR
    subgraph "Write Path"
        Event[Event Ingestion]
    end

    subgraph "PostgreSQL - Transactional"
        PG[PostgreSQL via Prisma]
        PGData[User Management<br/>Projects<br/>Datasets<br/>Job Configs<br/>Relationships<br/>Foreign Keys]
    end

    subgraph "ClickHouse - Analytics"
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

## 5. Authentication & Authorization

### 5.1 Multi-Layer Auth Architecture

```mermaid
graph TB
    subgraph "Authentication Methods"
        WebUI[Web UI - Session-based]
        APIKey[Public API - API Keys]
        OAuth[OAuth Providers]
    end

    subgraph "NextAuth.js Core"
        NextAuth[NextAuth Session]
        Providers[Auth Providers<br/>Credentials, Google,<br/>GitHub, Azure AD, SAML]
    end

    subgraph "Authorization Layer"
        RBAC[RBAC System]
        OrgCheck[Organization Membership]
        ProjectCheck[Project Membership]
        ScopeCheck[API Key Scopes]
    end

    subgraph "Enforcement Points"
        tRPCMiddleware[tRPC Middleware]
        APIMiddleware[Public API Middleware]
        PageGuard[Page-level Guards]
    end

    subgraph "Database"
        Users[(User, Account, Session)]
        Memberships[(OrganizationMembership<br/>ProjectMembership)]
        Keys[(ApiKey)]
    end

    WebUI --> NextAuth
    OAuth --> Providers
    Providers --> NextAuth

    NextAuth --> Users
    APIKey --> Keys

    tRPCMiddleware --> RBAC
    APIMiddleware --> ScopeCheck

    RBAC --> OrgCheck
    RBAC --> ProjectCheck

    OrgCheck --> Memberships
    ProjectCheck --> Memberships
    ScopeCheck --> Keys

    PageGuard --> NextAuth

    style NextAuth fill:#e1f5ff
    style RBAC fill:#ffe1e1
    style Users fill:#e1ffe1
```

### 5.2 RBAC Role Hierarchy

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

---

**Continue to:** [Part 2: Data Flows →](analysis-data-flows.md)
