# CloudBoard — Architecture

> **Audience**: Students learning Clean Architecture, Azure, and .NET 9.
> This document describes the high-level architecture, project responsibilities, data flow, and key design decisions.

---

## Table of Contents

1. [Solution Overview](#solution-overview)
2. [High-Level System Diagram](#high-level-system-diagram)
3. [Clean Architecture Layers](#clean-architecture-layers)
4. [Project Dependency Graph](#project-dependency-graph)
5. [Data Model (ER Diagram)](#data-model-er-diagram)
6. [API Request Flow](#api-request-flow)
7. [CQRS Pattern — MediatR vs. Hand-Rolled](#cqrs-pattern--mediatr-vs-hand-rolled)
8. [Authentication & Authorization](#authentication--authorization)
9. [Cosmos DB Design](#cosmos-db-design)
10. [Blob Storage Design](#blob-storage-design)
11. [Azure Functions Integration](#azure-functions-integration)
12. [Paging Strategy](#paging-strategy)
13. [Feature Flags & Swappable Infrastructure](#feature-flags--swappable-infrastructure)

---

## Solution Overview

CloudBoard is a "tasks / notes / files" collaboration platform organized around **Workspaces**.
Each workspace can have members, tags, comments, and an activity log.

| Project | SDK / Type | Purpose | Local Port |
|---|---|---|---|
| `CloudBoard.Domain` | Class Library | Entities, value objects, enums — zero NuGet deps | — |
| `CloudBoard.Application` | Class Library | CQRS commands/queries, DTOs, interfaces, validators | — |
| `CloudBoard.Infrastructure` | Class Library | Cosmos repos, Blob storage, Functions client | — |
| `CloudBoard.Api` | ASP.NET Core Web API | REST controllers, auth, DI, Swagger | `https://localhost:7100` |
| `CloudBoard.Client` | Blazor WASM (standalone) | End-user UI, calls API via HttpClient | `https://localhost:7300` |
| `CloudBoard.AdminPortal` | Blazor Server (InteractiveServer) | Admin UI for SystemAdmin + WorkspaceAdmin | `https://localhost:7200` |
| `CloudBoard.Functions` | Azure Functions v4 (isolated worker) | Serverless jobs triggered by the API only | `http://localhost:7071` |

---

## High-Level System Diagram

```mermaid
graph LR
    subgraph "Browser"
        WASM["CloudBoard.Client<br/>(Blazor WASM)"]
        ADMIN["CloudBoard.AdminPortal<br/>(Blazor Server)"]
    end

    subgraph "Azure / Local"
        API["CloudBoard.Api<br/>(ASP.NET Core Web API)"]
        FUNC["CloudBoard.Functions<br/>(Azure Functions v4)"]
        COSMOS[("Azure Cosmos DB<br/>NoSQL")]
        BLOB[("Azure Blob Storage")]
        ENTRA["Microsoft Entra<br/>External ID"]
    end

    WASM -- "REST + JWT" --> API
    ADMIN -- "REST + Cookie/OIDC" --> API
    API -- "HTTP + Function Key" --> FUNC
    API -- "Cosmos SDK" --> COSMOS
    API -- "Blob SDK" --> BLOB
    FUNC -- "Cosmos SDK" --> COSMOS
    WASM -- "MSAL Login" --> ENTRA
    ADMIN -- "OIDC Login" --> ENTRA
    API -- "Validate JWT" --> ENTRA

    style WASM fill:#4FC3F7,stroke:#0288D1,color:#000
    style ADMIN fill:#7986CB,stroke:#303F9F,color:#fff
    style API fill:#81C784,stroke:#388E3C,color:#000
    style FUNC fill:#FFB74D,stroke:#F57C00,color:#000
    style COSMOS fill:#CE93D8,stroke:#7B1FA2,color:#000
    style BLOB fill:#A5D6A7,stroke:#388E3C,color:#000
    style ENTRA fill:#EF9A9A,stroke:#C62828,color:#000
```

### Key rules

- **Client never calls Functions directly** — the API is the single gateway.
- **AdminPortal calls the same API** — it shares the API app registration in Entra.
- **Functions are HTTP-triggered with function keys** — only the API knows the key.

---

## Clean Architecture Layers

```mermaid
graph TB
    subgraph "Outer — Infrastructure & Presentation"
        API["CloudBoard.Api<br/>(Controllers, Middleware)"]
        INFRA["CloudBoard.Infrastructure<br/>(Cosmos, Blob, Functions)"]
        CLIENT["CloudBoard.Client"]
        PORTAL["CloudBoard.AdminPortal"]
        FUNCS["CloudBoard.Functions"]
    end

    subgraph "Middle — Application"
        APP["CloudBoard.Application<br/>(Commands, Queries, DTOs, Interfaces)"]
    end

    subgraph "Inner — Domain"
        DOM["CloudBoard.Domain<br/>(Entities, Value Objects, Enums)"]
    end

    API --> APP
    INFRA --> APP
    APP --> DOM
    API --> INFRA

    style DOM fill:#FFEB3B,stroke:#F57F17,color:#000
    style APP fill:#81D4FA,stroke:#0277BD,color:#000
    style INFRA fill:#C5E1A5,stroke:#558B2F,color:#000
    style API fill:#81C784,stroke:#388E3C,color:#000
```

### The Dependency Rule

> Inner layers **never** reference outer layers.

| Layer | Can reference | Cannot reference |
|---|---|---|
| **Domain** | Nothing | Application, Infrastructure, Api |
| **Application** | Domain | Infrastructure, Api |
| **Infrastructure** | Domain, Application | Api |
| **Api** | Application, Infrastructure | — |

The Application layer defines **interfaces** (e.g., `IWorkspaceRepository`, `IBlobStorageService`).
The Infrastructure layer provides **implementations** (e.g., `CosmosWorkspaceRepository`, `BlobStorageService`).
The Api layer wires them together in `Program.cs` via dependency injection.

---

## Project Dependency Graph

```mermaid
graph TD
    Api["CloudBoard.Api"] --> Infra["CloudBoard.Infrastructure"]
    Api --> App["CloudBoard.Application"]
    Infra --> App
    Infra --> Dom["CloudBoard.Domain"]
    App --> Dom
    Func["CloudBoard.Functions"] --> Infra
    Func --> App

    style Api fill:#81C784,stroke:#388E3C
    style Infra fill:#C5E1A5,stroke:#558B2F
    style App fill:#81D4FA,stroke:#0277BD
    style Dom fill:#FFEB3B,stroke:#F57F17
    style Func fill:#FFB74D,stroke:#F57C00
```

> `CloudBoard.Client` and `CloudBoard.AdminPortal` are standalone front-end apps — they call the API over HTTP and have **no project reference** to any backend project.

---

## Data Model (ER Diagram)

All entities live in a **single Cosmos container** (`WorkspaceData`) with partition key `/workspaceId` and a `docType` discriminator.

```mermaid
erDiagram
    WORKSPACE ||--o{ MEMBERSHIP : has
    WORKSPACE ||--o{ INVITE : issues
    WORKSPACE ||--o{ TASK_ITEM : contains
    WORKSPACE ||--o{ NOTE_ITEM : contains
    WORKSPACE ||--o{ FILE_ITEM : contains
    WORKSPACE ||--o{ TAG : contains
    WORKSPACE ||--o{ COMMENT : contains
    WORKSPACE ||--o{ ACTIVITY_LOG : records
    TASK_ITEM ||--o{ COMMENT : "has (via parentId + parentType)"
    NOTE_ITEM ||--o{ COMMENT : "has (via parentId + parentType)"
    FILE_ITEM ||--o{ COMMENT : "has (via parentId + parentType)"
    TASK_ITEM }o--o{ TAG : "tagged_with (via TagIds list)"
    NOTE_ITEM }o--o{ TAG : "tagged_with (via TagIds list)"
    FILE_ITEM }o--o{ TAG : "tagged_with (via TagIds list)"

    WORKSPACE {
        string id PK
        string docType "= Workspace"
        string name
        string ownerUserId
        datetime createdUtc
        datetime updatedUtc
    }
    MEMBERSHIP {
        string id PK
        string workspaceId FK
        string userId
        RoleInWorkspace roleInWorkspace "Member | WorkspaceAdmin"
        datetime createdUtc
        datetime updatedUtc
    }
    INVITE {
        string id PK
        string workspaceId FK
        string email
        string tokenHash
        datetime expiresUtc
        datetime createdUtc
        datetime updatedUtc
    }
    TASK_ITEM {
        string id PK
        string workspaceId FK
        string title
        bool isDone
        string_array tagIds "denormalised"
        datetime createdUtc
        datetime updatedUtc
    }
    NOTE_ITEM {
        string id PK
        string workspaceId FK
        string title
        string body
        string_array tagIds "denormalised"
        datetime createdUtc
        datetime updatedUtc
    }
    FILE_ITEM {
        string id PK
        string workspaceId FK
        string fileName
        string contentType
        long sizeBytes
        string blobName
        string_array tagIds "denormalised"
        datetime uploadedUtc
        datetime createdUtc
        datetime updatedUtc
    }
    TAG {
        string id PK
        string workspaceId FK
        string name
        datetime createdUtc
        datetime updatedUtc
    }
    COMMENT {
        string id PK
        string workspaceId FK
        ParentType parentType "Task | Note | File"
        string parentId
        string userId
        string body
        datetime createdUtc
        datetime updatedUtc
    }
    ACTIVITY_LOG {
        string id PK
        string workspaceId FK
        string userId
        ActivityAction action
        EntityType entityType
        string entityId
        string detailsJson
        int dateId "YYYYMMDD"
        datetime occurredUtc
        datetime createdUtc
        datetime updatedUtc
    }
```

### Key design decisions

| Decision | Reason |
|---|---|
| Single container with `docType` discriminator | Cosmos free tier allows only 1 container with shared throughput (1000 RU/s). Splitting later is easy because all queries already filter by `docType`. |
| Partition key = `/workspaceId` | All data for a workspace lives in the same logical partition. Queries scoped to a workspace are cheap single-partition reads. |
| Tag IDs stored inline on items (`TagIds` list) | Avoids cross-partition joins. Cosmos has no JOIN across partitions — denormalisation is the standard pattern. |
| Comment uses `ParentType` + `ParentId` | Polymorphic reference avoids three separate comment tables. Common in document databases. |
| `DateId` on ActivityLog (int `YYYYMMDD`) | Enables cheap range filtering without full datetime comparison — useful for daily summaries. |

---

## API Request Flow

### MediatR flow (Workspaces, Notes, Tags)

```mermaid
sequenceDiagram
    participant C as Client / Browser
    participant Ctrl as WorkspacesController
    participant M as MediatR
    participant H as CreateWorkspaceHandler
    participant R as IWorkspaceRepository
    participant DB as Cosmos / InMemory

    C->>Ctrl: POST /api/workspaces { name }
    Ctrl->>M: Send(CreateWorkspaceCommand)
    M->>H: Handle(command, ct)
    H->>R: CreateAsync(workspace, ct)
    R->>DB: UpsertItemAsync
    DB-->>R: workspace
    R-->>H: workspace
    H-->>M: WorkspaceDto
    M-->>Ctrl: WorkspaceDto
    Ctrl-->>C: 201 Created + WorkspaceDto
```

### Hand-rolled flow (Tasks, Files, Comments, etc.)

```mermaid
sequenceDiagram
    participant C as Client / Browser
    participant Ctrl as TasksController
    participant H as CreateTaskHandler
    participant R as ITaskRepository
    participant DB as Cosmos / InMemory

    C->>Ctrl: POST /api/workspaces/{id}/tasks { title }
    Ctrl->>H: HandleAsync(CreateTaskCommand, ct)
    H->>R: CreateAsync(taskItem, ct)
    R->>DB: UpsertItemAsync
    DB-->>R: taskItem
    R-->>H: taskItem
    H-->>Ctrl: TaskDto
    Ctrl-->>C: 201 Created + TaskDto
```

> **Why both patterns?** Students learn MediatR (industry standard) and hand-rolled CQRS (simpler, less magic). This makes it easier to understand what MediatR does under the hood.

---

## CQRS Pattern — MediatR vs. Hand-Rolled

```mermaid
graph LR
    subgraph "MediatR (Workspaces, Notes, Tags)"
        CMD1["IRequest&lt;TResult&gt;"] --> PIPE["MediatR Pipeline"]
        PIPE --> HANDLER1["IRequestHandler&lt;T, TResult&gt;"]
    end

    subgraph "Hand-Rolled (Tasks, Files, Comments, Memberships, Invites, ActivityLog, Admin)"
        CMD2["Plain record"] --> HANDLER2["ICommandHandler&lt;T, TResult&gt;<br/>or IQueryHandler&lt;T, TResult&gt;"]
    end

    style CMD1 fill:#81D4FA,stroke:#0277BD
    style PIPE fill:#B39DDB,stroke:#512DA8
    style HANDLER1 fill:#A5D6A7,stroke:#388E3C
    style CMD2 fill:#FFE082,stroke:#F9A825
    style HANDLER2 fill:#FFCC80,stroke:#EF6C00
```

| Aspect | MediatR | Hand-Rolled |
|---|---|---|
| **Used for** | Workspaces, Notes, Tags | Tasks, Files, Comments, Memberships, Invites, ActivityLog, Admin |
| **Command type** | Implements `IRequest<T>` | Plain `record` class |
| **Handler type** | `IRequestHandler<TCommand, TResult>` | `ICommandHandler<TCommand, TResult>` / `IQueryHandler<TQuery, TResult>` |
| **Registration** | Auto-scanned by `AddMediatR()` | Manual registration in `DependencyInjection.cs` |
| **Pipeline behaviours** | Supports behaviours (validation, logging) | Add manually if needed |
| **Controller call** | `mediator.Send(command)` | `handler.HandleAsync(command, ct)` |

### When to use which

- **MediatR**: Great for large teams, cross-cutting pipeline behaviours, when you want plug-and-play handler discovery.
- **Hand-rolled**: Simpler, more explicit, easier to debug, fewer dependencies.
- **CloudBoard uses both** so students can compare and choose for their own projects.

---

## Authentication & Authorization

### Auth flow overview

```mermaid
sequenceDiagram
    participant User as User (Browser)
    participant WASM as CloudBoard.Client<br/>(Blazor WASM)
    participant Entra as Microsoft Entra<br/>External ID
    participant API as CloudBoard.Api
    participant Cosmos as Cosmos DB

    User->>WASM: Open app
    WASM->>Entra: Redirect to login (MSAL)
    Entra-->>WASM: id_token + access_token
    WASM->>API: GET /api/workspaces<br/>Authorization: Bearer {token}
    API->>API: Validate JWT (issuer, audience, signature)
    API->>Cosmos: Check Membership for workspace
    Cosmos-->>API: Membership record
    API-->>WASM: 200 OK / 403 Forbidden
```

### Role model

```mermaid
graph TD
    subgraph "Global Roles (App Roles in Entra)"
        SA["SystemAdmin"]
        U["User"]
    end

    subgraph "Workspace Roles (Membership in Cosmos)"
        WA["WorkspaceAdmin"]
        M["Member"]
    end

    SA --> |"full access"| ADMIN["AdminController"]
    SA --> |"full access"| ALL["All Workspace Endpoints"]
    U --> |"must check"| MEMBERSHIP{{"Membership<br/>check"}}
    MEMBERSHIP --> WA
    MEMBERSHIP --> M

    WA --> |"can manage members"| MGMT["Memberships, Invites"]
    WA --> |"can CRUD"| ITEMS["Tasks, Notes, Files, Tags, Comments"]
    M --> |"can CRUD"| ITEMS

    style SA fill:#EF9A9A,stroke:#C62828
    style U fill:#81D4FA,stroke:#0277BD
    style WA fill:#FFE082,stroke:#F9A825
    style M fill:#C5E1A5,stroke:#558B2F
```

| Role | Scope | Stored in | Access |
|---|---|---|---|
| `SystemAdmin` | Global | Entra app role | Full admin portal + all workspaces |
| `User` | Global | Entra app role (default) | Must be a workspace member to access workspace data |
| `WorkspaceAdmin` | Per-workspace | Cosmos `Membership` (roleInWorkspace) | Manage members, invites; full CRUD within workspace |
| `Member` | Per-workspace | Cosmos `Membership` (roleInWorkspace) | CRUD tasks, notes, files, tags, comments within workspace |

### Auth configuration per app

| App | Auth Method | Library | Notes |
|---|---|---|---|
| CloudBoard.Client (WASM) | MSAL.js (redirect flow) | `Microsoft.Authentication.WebAssembly.Msal` | Standalone WASM, obtains access tokens for API |
| CloudBoard.AdminPortal (Server) | OpenID Connect (cookie-based) | `Microsoft.Identity.Web` | Server-side, session cookie after login |
| CloudBoard.Api | JWT Bearer validation | `Microsoft.Identity.Web` | Validates tokens from both Client and AdminPortal |

### Entra App Registrations

| Registration | Used by | Notes |
|---|---|---|
| **CloudBoard API** | Api, AdminPortal | Exposes `access_as_user` scope; defines app roles |
| **CloudBoard Client** | Client (WASM) | Requests `access_as_user` scope from API registration |

> Admin Portal **shares** the API registration — it authenticates via OIDC and uses cookies server-side.

---

## Cosmos DB Design

### Container layout

```mermaid
graph LR
    subgraph "Cosmos Account"
        subgraph "Database: CloudBoardDb"
            subgraph "Container: WorkspaceData<br/>Partition key: /workspaceId"
                D1["docType: Workspace"]
                D2["docType: TaskItem"]
                D3["docType: NoteItem"]
                D4["docType: FileItem"]
                D5["docType: Tag"]
                D6["docType: Comment"]
                D7["docType: Membership"]
                D8["docType: Invite"]
                D9["docType: ActivityLog"]
            end
        end
    end

    style D1 fill:#FFEB3B,stroke:#F57F17
    style D2 fill:#81D4FA,stroke:#0277BD
    style D3 fill:#A5D6A7,stroke:#388E3C
    style D4 fill:#FFCC80,stroke:#EF6C00
    style D5 fill:#CE93D8,stroke:#7B1FA2
    style D6 fill:#EF9A9A,stroke:#C62828
    style D7 fill:#B39DDB,stroke:#512DA8
    style D8 fill:#80CBC4,stroke:#00695C
    style D9 fill:#F48FB1,stroke:#AD1457
```

### Query patterns

| Query | Partition used? | Cost |
|---|---|---|
| Get all tasks for workspace X | ✅ Single partition (workspaceId = X, docType = TaskItem) | Low (1–5 RU) |
| Get single task by id + workspaceId | ✅ Point read | Very low (1 RU) |
| Get all workspaces for user | ❌ Cross-partition (fan-out on ownerUserId) | Higher — consider indexing |
| Get all activity logs (Admin) | ❌ Cross-partition scan | Expensive — use sparingly |

### Repository hierarchy

```mermaid
classDiagram
    class CosmosRepositoryBase~T~ {
        #Container : Container
        #ReadItemAsync(id, partitionKey, ct) T?
        #UpsertItemAsync(item, partitionKey, ct) T
        #DeleteItemAsync(id, partitionKey, ct)
        #QueryPagedAsync(queryable, ct) ContinuationResult~T~
    }

    class CosmosWorkspaceRepository {
        +CreateAsync()
        +GetByIdAsync()
        +ListByOwnerAsync()
        +UpdateAsync()
        +DeleteAsync()
    }

    class CosmosTaskRepository {
        +CreateAsync()
        +GetByIdAsync()
        +ListByWorkspaceAsync()
        +UpdateAsync()
        +DeleteAsync()
    }

    CosmosRepositoryBase <|-- CosmosWorkspaceRepository
    CosmosRepositoryBase <|-- CosmosTaskRepository
    CosmosRepositoryBase <|-- CosmosNoteRepository
    CosmosRepositoryBase <|-- CosmosFileRepository
    CosmosRepositoryBase <|-- CosmosTagRepository
    CosmosRepositoryBase <|-- CosmosCommentRepository
    CosmosRepositoryBase <|-- CosmosMembershipRepository
    CosmosRepositoryBase <|-- CosmosInviteRepository
    CosmosRepositoryBase <|-- CosmosActivityLogRepository
```

### Cosmos LINQ warnings

> ⚠️ **Cosmos LINQ ≠ EF LINQ**
> - No lazy loading, no navigation properties.
> - Use `.ToFeedIterator()` not `.ToListAsync()` — the latter pulls everything into memory.
> - Not all LINQ operators are supported (e.g., no `GroupBy`, limited `Select` projections).
> - Always test queries in the Cosmos emulator or Data Explorer to check RU costs.

---

## Blob Storage Design

```mermaid
sequenceDiagram
    participant C as Client
    participant API as CloudBoard.Api
    participant Blob as Azure Blob Storage
    participant Cosmos as Cosmos DB

    Note over C,Cosmos: File Upload
    C->>API: POST /api/workspaces/{id}/files<br/>(multipart/form-data)
    API->>Blob: UploadAsync(stream)
    Blob-->>API: blobName
    API->>Cosmos: Save FileItem metadata<br/>(fileName, contentType, sizeBytes, blobName)
    Cosmos-->>API: FileItem
    API-->>C: 201 Created + FileDto

    Note over C,Cosmos: File Download
    C->>API: GET /api/workspaces/{id}/files/{fileId}/download
    API->>Cosmos: Get FileItem (find blobName)
    Cosmos-->>API: FileItem
    API->>Blob: DownloadAsync(blobName)
    Blob-->>API: Stream
    API-->>C: 200 OK + file stream
```

### Blob naming convention

```
cloudboard-files/
  └── {workspaceId}/
      └── {fileId}/{originalFileName}
```

### Tangent extension points (future weeks)

| Feature | Status | Notes |
|---|---|---|
| Upload / Download | ✅ Implemented | Via `IBlobStorageService` |
| Versioning | 🔲 Placeholder | Append version suffix to blob name |
| Soft delete | 🔲 Placeholder | Set `isDeleted` flag on FileItem |
| SAS sharing links | 🔲 Placeholder | `GenerateSasUrlAsync()` method stub exists |
| Direct-to-blob upload | 🔲 Tangent | Client uploads directly to Blob with SAS; API saves metadata only |

---

## Azure Functions Integration

```mermaid
graph LR
    API["CloudBoard.Api"] -- "HTTP POST + Function Key" --> F1["GenerateDailyWorkspaceSummary"]
    API -- "HTTP POST + Function Key" --> F2["ProcessFileVirusScanPlaceholder"]
    API -- "HTTP POST + Function Key" --> F3["CleanupExpiredInvites"]

    F1 --> COSMOS[("Cosmos DB")]
    F2 --> COSMOS
    F3 --> COSMOS

    style API fill:#81C784,stroke:#388E3C
    style F1 fill:#FFB74D,stroke:#F57C00
    style F2 fill:#FFB74D,stroke:#F57C00
    style F3 fill:#FFB74D,stroke:#F57C00
    style COSMOS fill:#CE93D8,stroke:#7B1FA2
```

| Function | Trigger | Purpose |
|---|---|---|
| `GenerateDailyWorkspaceSummary` | HTTP (POST) | Aggregates ActivityLog entries for a workspace into a daily summary |
| `ProcessFileVirusScanPlaceholder` | HTTP (POST) | Placeholder pipeline — logs that scan occurred, no real scanning |
| `CleanupExpiredInvites` | HTTP (POST) | Deletes Invite documents past their `ExpiresUtc` |

### Calling pattern from API

```csharp
// In the API controller or handler:
await _functionsClient.TriggerDailySummaryAsync(workspaceId, ct);

// IFunctionsClient interface (Application layer)
// FunctionsClient implementation (Infrastructure layer)
// Calls: POST {baseUrl}/api/GenerateDailyWorkspaceSummary?code={functionKey}
```

### Security

- Function keys are stored in `appsettings.json` (local) or Azure Key Vault (production).
- The client app **never** has the function key — only the API does.
- In production, consider adding VNet integration or IP restrictions for defense-in-depth.

---

## Paging Strategy

CloudBoard uses **continuation-token-based paging** instead of offset/skip paging.

```mermaid
sequenceDiagram
    participant C as Client
    participant API as CloudBoard.Api
    participant DB as Cosmos DB

    C->>API: GET /api/workspaces/{id}/tasks?pageSize=20
    API->>DB: Query (page 1)
    DB-->>API: 20 items + continuationToken="abc123"
    API-->>C: { items: [...], continuationToken: "abc123" }

    C->>API: GET /api/workspaces/{id}/tasks?pageSize=20&continuationToken=abc123
    API->>DB: Query (page 2, with token)
    DB-->>API: 15 items + continuationToken=null
    API-->>C: { items: [...], continuationToken: null }
    Note over C: continuationToken is null → no more pages
```

### Why continuation tokens?

| Aspect | Continuation Token | Offset / Skip |
|---|---|---|
| **Cosmos cost** | Low — DB remembers position | High — DB must scan and discard N rows |
| **Consistency** | Stable — new inserts don't shift pages | Unstable — inserts cause duplicate/missing rows |
| **Implementation** | Opaque string from Cosmos SDK | Simple integer math |
| **Client complexity** | Must store token between requests | Just increment page number |

### Response shape

```json
{
  "items": [
    { "id": "...", "title": "My Task", "isDone": false, ... },
    { "id": "...", "title": "Another Task", "isDone": true, ... }
  ],
  "continuationToken": "eyJjb250aW51YXRpb24i..."
}
```

When `continuationToken` is `null`, the client knows there are no more pages.

---

## Feature Flags & Swappable Infrastructure

CloudBoard uses **feature flags** in `appsettings.json` to toggle between in-memory stubs and real Azure services.

```mermaid
graph TD
    FLAGS["appsettings.json<br/>FeatureFlags"]
    FLAGS --> |"UseCosmosDb: false"| MEM_REPO["InMemory Repositories<br/>(week01–03)"]
    FLAGS --> |"UseCosmosDb: true"| COSMOS_REPO["Cosmos Repositories<br/>(week04+)"]
    FLAGS --> |"UseAzureStorage: false"| MEM_BLOB["InMemory Blob Service<br/>(week01–04)"]
    FLAGS --> |"UseAzureStorage: true"| AZURE_BLOB["Azure Blob Service<br/>(week05+)"]
    FLAGS --> |"UseFunctionsClient: false"| NOOP["NoOp Functions Client<br/>(week01–05)"]
    FLAGS --> |"UseFunctionsClient: true"| REAL_FUNC["Real Functions Client<br/>(week06+)"]

    style FLAGS fill:#FFE082,stroke:#F9A825,color:#000
    style MEM_REPO fill:#E0E0E0,stroke:#757575
    style COSMOS_REPO fill:#CE93D8,stroke:#7B1FA2
    style MEM_BLOB fill:#E0E0E0,stroke:#757575
    style AZURE_BLOB fill:#A5D6A7,stroke:#388E3C
    style NOOP fill:#E0E0E0,stroke:#757575
    style REAL_FUNC fill:#FFB74D,stroke:#F57C00
```

```json
{
  "FeatureFlags": {
    "UseCosmosDb": false,
    "UseAzureStorage": false,
    "UseFunctionsClient": false
  }
}
```

This means:
- **Week 01**: All flags `false` — runs entirely in-memory with zero Azure dependencies.
- **Week 04**: Flip `UseCosmosDb` to `true` — now using real Cosmos DB.
- **Week 05**: Flip `UseAzureStorage` to `true` — real Blob Storage for file uploads.
- **Week 06**: Flip `UseFunctionsClient` to `true` — API calls real Azure Functions.

> **Why?** Students can run the full solution locally from day one without installing the Cosmos emulator or creating Azure resources. Each week "turns on" a new integration.
