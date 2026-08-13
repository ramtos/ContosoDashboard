# Implementation Plan: Document Upload and Management

**Branch**: `main` | **Date**: 2026-08-13 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-document-management/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

Implement document upload and management capabilities for ContosoDashboard training application. Users upload files (PDF, Office docs, images) to personal libraries or projects, with metadata (title, category, tags), search/filtering, sharing, and preview. Technical approach uses Blazor Server UI components, Entity Framework Core for metadata persistence, local file system storage via `IFileStorageService` abstraction (enabling future Azure Blob migration), service-layer authorization to prevent IDOR attacks, and WCAG 2.1 AA accessible UI. Soft-delete with 30-day retention provides audit trail before permanent removal.

## Technical Context

**Language/Version**: C# 13 / .NET 10.0  
**Primary Dependencies**: ASP.NET Core 10.0 (Blazor Server), Entity Framework Core 10.0 (SQLite provider), Bootstrap 5.3, Bootstrap Icons  
**Storage**: SQLite database (local file for training), file system for document storage (local directory, future Azure Blob Storage)  
**Testing**: Manual testing for training scenario (unit tests for service layer recommended but optional)  
**Target Platform**: Cross-platform (Windows/Linux/macOS), ASP.NET Core Kestrel web server  
**Project Type**: Web application (Blazor Server with cookie-based authentication)  
**Performance Goals**: Document uploads complete in <30s for 25MB files; list pages load in <2s for 500 documents; search results in <2s; previews render in <3s  
**Constraints**: Offline-first (no external APIs), 25MB file size limit, training-appropriate security (mock authentication with production authorization patterns), WCAG 2.1 Level AA accessibility  
**Scale/Scope**: Training scenario with 4 predefined users, ~10-50 documents per user, 3-5 active projects, 500 documents max per project

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Gate | Status | Notes |
|-----------|------|--------|-------|
| **I. Specification-First Development** | Feature specification completed with user stories, requirements, and success criteria before design begins | ✅ PASS | `spec.md` complete with 6 user stories, 51 functional requirements, 11 success criteria, and 5 clarifications resolved |
| **II. Training-Appropriate Security** | Authorization implemented at page and service layers; IDOR protection verified; no sensitive data exposure | ✅ PASS (Phase 1) | `IDocumentService` defines authorization checks for all operations: owner, project member, or explicit share verification before access/modify. Service methods throw `UnauthorizedAccessException` on failure. Authorization helper methods `CanUserAccessDocumentAsync` and `CanUserModifyDocumentAsync` encapsulate permission logic. |
| **III. Infrastructure Abstraction** | External dependencies use interface abstractions enabling cloud migration | ✅ PASS (Phase 1) | `IFileStorageService` interface defined with `LocalFileStorageService` (training) and `AzureBlobStorageService` (production) implementations. File paths are implementation-agnostic. Database access via EF Core (swappable connection strings). |
| **IV. Clean Service Architecture** | Business logic in service classes with interfaces; no business logic in pages/controllers | ✅ PASS (Phase 1) | `IDocumentService` interface defined with all business logic: validation, authorization, file storage orchestration, notification triggers. Blazor pages call service methods only; no business logic in UI components. |
| **V. Explicit Over Implicit** | Code is self-documenting with clear naming and explicit error handling | ✅ PASS (Phase 1) | Service methods have descriptive names (`UploadDocumentAsync`, `AuthorizeDocumentAccessAsync`). Custom exceptions defined (`NotFoundException`, `UnauthorizedAccessException`, `ValidationException`). Authorization checks are explicit (not hidden in filters). |

**Re-evaluation Complete**: All constitution principles satisfied in Phase 1 design. Proceeding to implementation is authorized.

## Project Structure

### Documentation (this feature)

```text
specs/001-document-management/
├── spec.md              # Feature specification (complete)
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (technology decisions)
├── data-model.md        # Phase 1 output (database schema)
├── quickstart.md        # Phase 1 output (validation scenarios)
├── contracts/           # Phase 1 output (service interfaces)
│   └── IDocumentService.md
│   └── IFileStorageService.md
└── tasks.md             # Phase 2 output (/speckit.tasks - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
ContosoDashboard/
├── Models/
│   ├── Document.cs                  # [NEW] Document entity
│   ├── DocumentShare.cs             # [NEW] DocumentShare entity  
│   └── DocumentTask.cs              # [NEW] Document-Task join table
├── Data/
│   └── ApplicationDbContext.cs      # [MODIFY] Add DbSet<Document>, DbSet<DocumentShare>, DbSet<DocumentTask>
├── Services/
│   ├── IDocumentService.cs          # [NEW] Document business logic interface
│   ├── DocumentService.cs           # [NEW] Document business logic implementation
│   ├── IFileStorageService.cs       # [NEW] File storage abstraction
│   └── LocalFileStorageService.cs   # [NEW] Local filesystem implementation
├── Pages/
│   ├── Documents.razor              # [NEW] Main documents list page
│   ├── DocumentUpload.razor         # [NEW] Upload form component
│   ├── DocumentPreview.razor        # [NEW] Preview modal component
│   ├── ProjectDetails.razor         # [MODIFY] Add project documents section
│   ├── Tasks.razor                  # [MODIFY] Add document attachment to tasks
│   └── Index.razor                  # [MODIFY] Add "Recent Documents" dashboard widget
├── Shared/
│   └── NavMenu.razor                # [MODIFY] Add "Documents" navigation link
└── wwwroot/
    └── uploads/                     # [IGNORED BY GIT] Local file storage directory

Migrations/
└── YYYYMMDDHHMMSS_AddDocumentTables.cs  # [NEW] EF Core migration for Document tables
```

**Structure Decision**: Web application structure with Blazor Server pages calling service layer. Document files stored outside `wwwroot` in a dedicated `uploads/` directory (created at runtime if missing) to prevent direct web access and enforce authorization. Services registered in `Program.cs` with scoped lifetime for per-request dependency injection.

## Complexity Tracking

> **No constitution violations requiring justification**

All design decisions align with constitution principles:
- Infrastructure abstraction via `IFileStorageService` (Principle III)
- Service layer architecture with `IDocumentService` (Principle IV)  
- Authorization at service level (Principle II)
- Explicit error handling and naming conventions (Principle V)
- Specification-first approach followed (Principle I)
