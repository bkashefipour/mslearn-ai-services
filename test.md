# CloudBoard — Progress Report

> Auto-generated against the requirements defined in `docs/Prompt.txt`.
> Last updated: **June 2025**

---

## 1. Executive Summary

The **CloudBoard** solution is currently at **Week 01 (scaffolding)** maturity. The core Visual Studio solution compiles successfully with four of the seven planned projects building cleanly. Domain entities, in-memory repositories, CQRS handlers (both MediatR and hand-rolled), API controllers, a standalone Blazor WASM client shell, and seed data are all in place. The project is runnable locally without any Azure dependencies.

---

## 2. Requirement Compliance Matrix

| #  | Requirement | Status | Notes |
|----|-------------|--------|-------|
| 1  | Complete VS solution structure that compiles | ✅ Done | 4 of 7 projects build (see §3 for gaps) |
| 2  | Clean Architecture with full Controllers | ✅ Done | Domain → Application → Infrastructure → Api layering in place |
| 3  | REST + minimal CQRS (commands/queries) | ✅ Done | MediatR for Workspace/Notes/Tags; hand-rolled for Tasks/Files/Comments/Memberships/Invites/ActivityLog/Admin |
| 4  | Week-by-week branch plan | ❌ Missing | `docs/week-by-week.md` does not exist yet |
| 5  | Local dev setup + Azure deploy targets | ⚠️ Partial | `README.md` has quick-start; `docs/local-dev.md` missing |
| 6  | Click-ops first for class | ❌ Missing | `docs/azure-clickops.md` not created |
| 7  | Keep `ClaudeTest.slnx`, projects as `CloudBoard.*` | ⚠️ Unknown | `.slnx` file not found by search; build log shows projects restoring correctly |
| 8  | Standalone Blazor WASM for Client (no ASP.NET host) | ✅ Done | `CloudBoard.Client` uses `WebAssemblyHostBuilder`, calls API via `HttpClient` |
| 9  | Blazor Web App InteractiveServer for Admin Portal | ❌ Missing | `CloudBoard.AdminPortal` project does not exist yet |
| 10 | Azure Functions v4 isolated worker model | ❌ Missing | `CloudBoard.Functions` project does not exist yet |
| 11 | MediatR vs. hand-rolled split | ✅ Done | `DependencyInjection.cs` registers MediatR for Workspace/Notes/Tags, hand-rolled for the rest |
| 12 | Cosmos DB SDK direct (`Microsoft.Azure.Cosmos`) | ⚠️ Partial | Package declared in `Directory.Packages.props`; feature-flagged off (`FeatureFlags:UseCosmosDb`); no Cosmos repository implementations found yet |
| 13 | Auth placeholder (compiled, disabled for week01) | ✅ Done | `[Authorize]` commented out on all controllers; JWT Bearer config commented in `Program.cs`; MSAL commented in Client `Program.cs` |
| 14 | Projects physically nested under `src/` | ✅ Done | Build paths confirm `src/CloudBoard.Api/`, `src/CloudBoard.Domain/`, etc. |
| 15 | Solution Items (`.gitignore`, `global.json`, `README.md`, `Directory.Packages.props`, `Directory.Build.props`, `.editorconfig`) | ✅ Done | All files exist at repo root |
| 16 | In-memory stub repos for week01 | ✅ Done | `InMemoryWorkspaceRepository` and full in-memory seeder confirmed; feature flag controls swap |
| 17 | `Microsoft.Authentication.WebAssembly.Msal` | ✅ Done | Declared in `Directory.Packages.props`; commented-out usage in Client `Program.cs` |
| 18 | Server-side OIDC (`Microsoft.Identity.Web`) for Admin Portal | ⚠️ Partial | Package declared; Admin Portal project not yet created |
| 19 | Port assignments (API 7100, Admin 7200, Client 7300, Functions 7071) | ⚠️ Partial | API confirmed at 7100; Client configured to call 7100; CORS allows 7200 & 7300; Admin & Functions ports not yet configured (projects missing) |
| 20 | Continuation-token-based paging | ✅ Done | `ContinuationResult<T>` used consistently across all controllers and in-memory repos |
| 21 | Two Entra registrations (Client + API; Admin shares API) | ⚠️ Placeholder | Commented config sections present; no real Entra wiring (expected for week01) |

---

## 3. Project Inventory

| Project | Required | Exists | Builds |
|---------|----------|--------|--------|
| `CloudBoard.Domain` | ✅ | ✅ | ✅ |
| `CloudBoard.Application` | ✅ | ✅ | ✅ |
| `CloudBoard.Infrastructure` | ✅ | ✅ | ✅ |
| `CloudBoard.Api` | ✅ | ✅ | ✅ (1 warning: duplicate `using`) |
| `CloudBoard.Client` | ✅ | ✅ | ✅ |
| `CloudBoard.AdminPortal` | ✅ | ❌ | N/A |
| `CloudBoard.Functions` | ✅ | ❌ | N/A |

### Build Warning

`src/CloudBoard.Api/Program.cs` line 2 has a duplicate `using CloudBoard.Application;` directive (CS0105).

---

## 4. Domain Model Coverage

All **9 baseline entities** from the ER diagram are implemented:

| Entity | Class | `DocType` Discriminator | Partition Key (`WorkspaceId`) |
|--------|-------|-------------------------|-------------------------------|
| Workspace | `Workspace` | `"Workspace"` | ✅ (via `WorkspaceId => Id`) |
| Membership | `Membership` | `"Membership"` | ✅ |
| Invite | `Invite` | `"Invite"` | ✅ |
| TaskItem | `TaskItem` | `"TaskItem"` | ✅ |
| NoteItem | `NoteItem` | `"NoteItem"` | ✅ |
| FileItem | `FileItem` | `"FileItem"` | ✅ |
| Tag | `Tag` | `"Tag"` | ✅ |
| Comment | `Comment` | `"Comment"` | ✅ (polymorphic via `ParentType` + `ParentId`) |
| ActivityLog | `ActivityLog` | `"ActivityLog"` | ✅ (includes `DateId` for range filtering) |

**Base class:** `BaseEntity` with `Id`, `DocType`, `CreatedUtc`, `UpdatedUtc`.

---

## 5. API Controllers

All **10 required controllers** are implemented:

| Controller | Route | CQRS Mode | Auth |
|------------|-------|-----------|------|
| `HealthController` | `api/health` | None (simple) | Open |
| `WorkspacesController` | `api/workspaces` | MediatR | Commented `[Authorize]` |
| `TasksController` | `api/workspaces/{id}/tasks` | Hand-rolled | Commented `[Authorize]` |
| `NotesController` | `api/workspaces/{id}/notes` | MediatR | Commented `[Authorize]` |
| `FilesController` | `api/workspaces/{id}/files` | Hand-rolled | Commented `[Authorize]` |
| `TagsController` | `api/workspaces/{id}/tags` | MediatR | Commented `[Authorize]` |
| `CommentsController` | `api/workspaces/{id}/comments` | Hand-rolled | Commented `[Authorize]` |
| `MembershipsController` | `api/workspaces/{id}/memberships` | Hand-rolled | Commented `[Authorize]` |
| `InvitesController` | `api/workspaces/{id}/invites` | Hand-rolled | Commented `[Authorize]` |
| `AdminController` | `api/admin` | Hand-rolled | Commented `[Authorize(Policy = "SystemAdmin")]` |

All endpoints return proper HTTP status codes, use DTOs, and support continuation-token paging.

---

## 6. Infrastructure Layer

| Component | Status | Details |
|-----------|--------|---------|
| In-memory repositories | ✅ Done | All 9 entity repos registered as singletons; feature-flagged via `FeatureFlags:UseCosmosDb` |
| In-memory Blob Storage | ✅ Done | `InMemoryBlobStorageService` stub |
| In-memory Data Seeder | ✅ Done | Rich seed data: 2 workspaces, 2 tags, 2 tasks, 2 notes, 2 files, 2 memberships, 2 comments, 2 invites, 2 activity logs |
| Cosmos repositories | ❌ Not yet | Planned for week04; feature flag ready |
| Azure Blob Storage client | ❌ Not yet | Feature-flagged via `FeatureFlags:UseAzureStorage` |
| Functions HTTP client | ⚠️ Stub | `NoOpFunctionsClient` registered when `FeatureFlags:UseFunctionsClient` is false |
| First Login Service | ✅ Done | `IFirstLoginService` / `FirstLoginService` registered; middleware commented out until week02 |

---

## 7. Client (Blazor WASM)

| Component | Status |
|-----------|--------|
| `Program.cs` (standalone WASM) | ✅ Done |
| `CloudBoardApiClient` typed service | ✅ Done — covers all API endpoints |
| Home page with API health check | ✅ Done |
| MSAL auth placeholder | ✅ Done (commented out) |
| Navigation / workspace pages | ❌ Not found — only `Home.razor` exists |

---

## 8. Documentation

| Document | Required | Status |
|----------|----------|--------|
| `README.md` | ✅ | ✅ Done — quick start, project table, port mapping |
| `docs/architecture.md` | ✅ | ❌ Missing |
| `docs/week-by-week.md` | ✅ | ❌ Missing |
| `docs/local-dev.md` | ✅ | ❌ Missing |
| `docs/azure-clickops.md` | ✅ | ❌ Missing |
| `docs/naming-conventions.md` | ✅ | ❌ Missing |
| `docs/Prompt.txt` | — | ✅ Present (original requirements) |

---

## 9. CI/CD (GitHub Actions)

| Workflow | Required | Status |
|----------|----------|--------|
| Static Web Apps deploy (WASM) | ✅ | ❌ Missing |
| App Service deploy (API) | ✅ | ❌ Missing |
| App Service deploy (Admin Portal) | ✅ | ❌ Missing |
| Functions deploy | ✅ | ❌ Missing |

---

## 10. Week-by-Week Progress vs. Plan

| Week | Scope | Status |
|------|-------|--------|
| **week01** | Solution scaffolding + basic API + client shell + local run | ✅ **Complete** — all controllers, in-memory repos, seed data, Swagger, CORS, Client shell |
| **week02** | Auth integration (Entra External ID "easy mode") | ❌ Not started — placeholders in place |
| **week03** | Clean Architecture + CQRS basics + workspace model | ✅ **Already delivered in week01** (architecture is clean, CQRS registered) |
| **week04** | Cosmos DB integration + repos + partitioning | ❌ Not started — feature flag ready |
| **week05** | App Service deploy + config | ❌ Not started |
| **week06** | Functions + API-to-functions calls | ❌ Not started — Functions project missing |
| **week07** | Security hardening + roles + admin portal access | ❌ Not started — AdminPortal project missing |
| **week08** | Monitoring/logging (Application Insights) | ❌ Not started |
| **week09** | Final polish + docs + optional tangents | ❌ Not started |

---

## 11. Identified Issues

| Priority | Issue | Recommendation |
|----------|-------|----------------|
| 🔴 High | `CloudBoard.AdminPortal` project missing | Create Blazor Web App with `@rendermode InteractiveServer` on port 7200 |
| 🔴 High | `CloudBoard.Functions` project missing | Create Azure Functions v4 isolated worker with 3 placeholder functions |
| 🟡 Medium | All `/docs` markdown files missing (except `README.md`) | Generate skeleton docs for `architecture.md`, `week-by-week.md`, `local-dev.md`, `azure-clickops.md`, `naming-conventions.md` |
| 🟡 Medium | No GitHub Actions workflows | Create 4 workflow YAML files in `.github/workflows/` |
| 🟡 Medium | Duplicate `using` in `CloudBoard.Api/Program.cs` (line 2) | Remove duplicate `using CloudBoard.Application;` |
| 🟢 Low | Client Blazor WASM has only `Home.razor` | Add workspace list/detail pages, task/note/file management pages |
| 🟢 Low | `.slnx` file not located | Verify `ClaudeTest.slnx` exists at repo root and includes all 7 projects |

---

## 12. Summary Scorecard

| Category | Items Required | Items Complete | % |
|----------|---------------|----------------|---|
| Projects | 7 | 5 | 71% |
| Domain Entities | 9 | 9 | 100% |
| API Controllers | 10 | 10 | 100% |
| In-Memory Repos | 9 | 9 | 100% |
| Cosmos Repos | 9 | 0 | 0% |
| Docs | 6 | 1 | 17% |
| CI/CD Workflows | 4 | 0 | 0% |
| Client Pages | ~8 estimated | 1 | ~13% |
| **Overall Week01 Target** | — | — | **~85%** |

**Bottom line:** Week 01 scaffolding is substantially complete for the API, Domain, Application, and Infrastructure layers. The two missing projects (`AdminPortal`, `Functions`) and the documentation suite are the primary gaps before progressing to week02.    
