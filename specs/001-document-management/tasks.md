# Tasks: Document Upload and Management

**Input**: Design documents from `/specs/001-document-management/`
**Prerequisites**: plan.md (complete), spec.md (complete), research.md (complete), data-model.md (complete), contracts/ (complete)

**Tests**: Not explicitly requested in specification - manual validation via quickstart.md scenarios

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

Per plan.md: Blazor Server application structure with `ContosoDashboard/` as project root.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and Git configuration

- [ ] T001 Add `uploads/` directory to .gitignore in ContosoDashboard/.gitignore
- [ ] T002 [P] Add `appsettings.json` configuration for FileStorage:BasePath = "./uploads"

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Database Models

- [ ] T003 [P] Create Document entity class in ContosoDashboard/Models/Document.cs with all properties per data-model.md
- [ ] T004 [P] Create DocumentShare entity class in ContosoDashboard/Models/DocumentShare.cs with all properties per data-model.md
- [ ] T005 [P] Create DocumentTask entity class in ContosoDashboard/Models/DocumentTask.cs with all properties per data-model.md
- [ ] T006 [P] Create DocumentCategory enum in ContosoDashboard/Models/DocumentCategory.cs (ProjectDocuments, TeamResources, PersonalFiles, Reports, Presentations, Other)

### Database Configuration

- [ ] T007 Update ApplicationDbContext.cs to add DbSet<Document>, DbSet<DocumentShare>, DbSet<DocumentTask>
- [ ] T008 Configure entity relationships and indexes in ApplicationDbContext.OnModelCreating per data-model.md (foreign keys, unique constraints, global query filter for IsDeleted)
- [ ] T009 Create EF Core migration: `dotnet ef migrations add AddDocumentManagement` from ContosoDashboard/ directory
- [ ] T010 Apply migration: `dotnet ef database update` and verify tables created in ContosoDashboard.db

### File Storage Infrastructure

- [ ] T011 [P] Create IFileStorageService interface in ContosoDashboard/Services/IFileStorageService.cs per contracts/IFileStorageService.md
- [ ] T012 Implement LocalFileStorageService in ContosoDashboard/Services/LocalFileStorageService.cs (UploadAsync, DownloadAsync, DeleteAsync, ExistsAsync, GetTemporaryUrlAsync)
- [ ] T013 Register IFileStorageService → LocalFileStorageService in Program.cs with scoped lifetime

### Service Layer Interfaces

- [ ] T014 [P] Create DocumentUploadModel DTO class in ContosoDashboard/Models/DocumentUploadModel.cs
- [ ] T015 [P] Create DocumentUpdateModel DTO class in ContosoDashboard/Models/DocumentUpdateModel.cs
- [ ] T016 [P] Create custom exception classes: NotFoundException, ValidationException in ContosoDashboard/Services/Exceptions.cs
- [ ] T017 Create IDocumentService interface in ContosoDashboard/Services/IDocumentService.cs per contracts/IDocumentService.md (all 18 methods)
- [ ] T018 Register IDocumentService (interface placeholder) in Program.cs with scoped lifetime (implementation in T027)

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - Upload and View Personal Documents (Priority: P1) 🎯 MVP

**Goal**: Users can upload files to personal library and view them in "My Documents" list

**Independent Test**: Log in as any user, upload PDF with title and category, verify it appears in "My Documents" with correct metadata

### Implementation for User Story 1

- [ ] T019 [P] [US1] Implement UploadDocumentAsync method in ContosoDashboard/Services/DocumentService.cs (authorization, validation, file storage, DB record creation)
- [ ] T020 [P] [US1] Implement GetUserDocumentsAsync method in ContosoDashboard/Services/DocumentService.cs (query user's documents with authorization filter)
- [ ] T021 [P] [US1] Implement GetDocumentAsync method in ContosoDashboard/Services/DocumentService.cs (single document retrieval with authorization check)
- [ ] T022 [P] [US1] Implement DownloadDocumentAsync method in ContosoDashboard/Services/DocumentService.cs (authorization + file stream retrieval)
- [ ] T023 [P] [US1] Implement CanUserAccessDocumentAsync helper method in ContosoDashboard/Services/DocumentService.cs (owner/project/share check)
- [ ] T024 [US1] Complete DocumentService.cs registration in Program.cs (replace interface-only registration from T018 with concrete implementation)
- [ ] T025 [P] [US1] Create Documents.razor page in ContosoDashboard/Pages/Documents.razor with "My Documents" list (title, category, date, size columns with sort/filter)
- [ ] T026 [P] [US1] Create DocumentUpload.razor component in ContosoDashboard/Pages/DocumentUpload.razor (form with InputFile, title, description, category, tags, project dropdown)
- [ ] T027 [US1] Add download endpoint handler in Documents.razor.cs for secure file download (/api/documents/{id}/download route)
- [ ] T028 [US1] Add "Documents" navigation link in ContosoDashboard/Shared/NavMenu.razor
- [ ] T029 [US1] Add [Authorize] attribute to Documents.razor page for authentication enforcement

**Checkpoint**: User Story 1 complete - users can upload and view personal documents independently

---

## Phase 4: User Story 2 - Associate Documents with Projects (Priority: P1) 🎯 MVP

**Goal**: Users can attach documents to projects; project members can view project documents

**Independent Test**: Upload document with project selected, navigate to project detail page, verify document appears and is downloadable by project members

### Implementation for User Story 2

- [ ] T030 [P] [US2] Implement GetProjectDocumentsAsync method in ContosoDashboard/Services/DocumentService.cs (query documents by ProjectId with authorization)
- [ ] T031 [P] [US2] Update DocumentUpload.razor to populate project dropdown from user's projects (load from IProjectService)
- [ ] T032 [US2] Add project documents section to ContosoDashboard/Pages/ProjectDetails.razor (display documents table, download links)
- [ ] T033 [US2] Implement AttachDocumentToTaskAsync method in ContosoDashboard/Services/DocumentService.cs (create DocumentTask record, auto-associate with project)
- [ ] T034 [US2] Implement GetTaskDocumentsAsync method in ContosoDashboard/Services/DocumentService.cs (query documents attached to specific task)
- [ ] T035 [US2] Add document attachment UI to ContosoDashboard/Pages/Tasks.razor (attach button, document list for task detail view)

**Checkpoint**: User Story 2 complete - documents integrated with projects and tasks

---

## Phase 5: User Story 3 - Search and Discover Documents (Priority: P2)

**Goal**: Users can search documents by keyword with authorization-filtered results

**Independent Test**: Upload 10 documents with varying titles/tags, search by keyword, verify only authorized documents appear within 2 seconds

### Implementation for User Story 3

- [ ] T036 [P] [US3] Implement SearchDocumentsAsync method in ContosoDashboard/Services/DocumentService.cs (LIKE query on title/description/tags with authorization filter)
- [ ] T037 [US3] Add search input and button to Documents.razor page header
- [ ] T038 [US3] Add search results display section in Documents.razor (reuse documents list table with search filter applied)
- [ ] T039 [US3] Add client-side sort functionality to Documents.razor (sort by title, date, size with ascending/descending toggle)
- [ ] T040 [US3] Add category filter dropdown to Documents.razor (filter documents by selected category)

**Checkpoint**: User Story 3 complete - search and filtering fully functional

---

## Phase 6: User Story 4 - Share Documents with Team Members (Priority: P2)

**Goal**: Document owners can share with specific users or project teams; recipients get notifications

**Independent Test**: Upload document, share with specific user, log in as that user, verify document in "Shared with Me" and notification received

### Implementation for User Story 4

- [ ] T041 [P] [US4] Implement ShareDocumentAsync method in ContosoDashboard/Services/DocumentService.cs (create DocumentShare records, send notifications)
- [ ] T042 [P] [US4] Implement ShareDocumentWithProjectTeamAsync method in ContosoDashboard/Services/DocumentService.cs (iterate ProjectMembers, create shares snapshot)
- [ ] T043 [P] [US4] Implement GetSharedDocumentsAsync method in ContosoDashboard/Services/DocumentService.cs (query DocumentShares for current user)
- [ ] T044 [P] [US4] Implement RevokeShareAsync method in ContosoDashboard/Services/DocumentService.cs (delete DocumentShare record, authorization check)
- [ ] T045 [US4] Create share modal component in Documents.razor (select users or project team dropdown, share/revoke buttons)
- [ ] T046 [US4] Add "Shared with Me" view/tab in Documents.razor page (display shared documents list)
- [ ] T047 [US4] Integrate with INotificationService to send in-app notifications when document is shared

**Checkpoint**: User Story 4 complete - sharing and collaboration features working

---

## Phase 7: User Story 5 - Manage and Update Document Metadata (Priority: P3)

**Goal**: Document owners can edit metadata and replace files while preserving shares

**Independent Test**: Upload document, edit title/category, verify changes persist; upload replacement file, verify new file served

### Implementation for User Story 5

- [ ] T048 [P] [US5] Implement UpdateDocumentMetadataAsync method in ContosoDashboard/Services/DocumentService.cs (update title, description, category, tags with authorization)
- [ ] T049 [P] [US5] Implement ReplaceDocumentFileAsync method in ContosoDashboard/Services/DocumentService.cs (delete old file, upload new file, update DB record, preserve shares)
- [ ] T050 [P] [US5] Implement CanUserModifyDocumentAsync helper method in ContosoDashboard/Services/DocumentService.cs (owner-only authorization check)
- [ ] T051 [US5] Create edit modal component in Documents.razor (editable form for title, description, category, tags)
- [ ] T052 [US5] Add "Replace File" functionality to edit modal (InputFile component, upload replacement handler)
- [ ] T053 [US5] Add "Edit" button to document list rows in Documents.razor (only visible for document owners)

**Checkpoint**: User Story 5 complete - metadata management and file replacement working

---

## Phase 8: User Story 6 - Preview Common Document Types (Priority: P3)

**Goal**: Users can preview PDFs and images in browser without downloading

**Independent Test**: Upload PDF and image, click preview, verify they render in modal within 3 seconds

### Implementation for User Story 6

- [ ] T054 [P] [US6] Create DocumentPreview.razor component in ContosoDashboard/Pages/DocumentPreview.razor (modal with iframe for PDF, img tag for images)
- [ ] T055 [US6] Add "Preview" button to document list rows in Documents.razor (only for PDF and image file types)
- [ ] T056 [US6] Add preview modal trigger logic in Documents.razor.cs (open modal, load document URL with authorization)
- [ ] T057 [US6] Implement keyboard navigation for preview modal (Escape to close, Tab navigation, ARIA labels per FR-047)

**Checkpoint**: User Story 6 complete - preview functionality working for supported file types

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories and final validation

- [ ] T058 [P] [US1] Implement SoftDeleteDocumentAsync method in ContosoDashboard/Services/DocumentService.cs (set IsDeleted=true, DeletedDate=now, authorization check)
- [ ] T059 [P] Implement PermanentlyDeleteExpiredDocumentsAsync method in ContosoDashboard/Services/DocumentService.cs (background cleanup for 30-day retention)
- [ ] T060 [P] Implement GetSoftDeletedDocumentsAsync method in ContosoDashboard/Services/DocumentService.cs (admin-only audit view with IgnoreQueryFilters)
- [ ] T061 Create DocumentCleanupService as IHostedService in ContosoDashboard/Services/DocumentCleanupService.cs (timer trigger calls PermanentlyDeleteExpiredDocumentsAsync)
- [ ] T062 Register DocumentCleanupService as hosted service in Program.cs
- [ ] T063 Add "Recent Documents" widget to ContosoDashboard/Pages/Index.razor dashboard (last 5 user uploads)
- [ ] T064 Add document count to dashboard summary cards in Index.razor (integrate with IDashboardService)
- [ ] T065 Add error handling UI components in Documents.razor (display validation errors, upload errors, authorization errors with user-friendly messages)
- [ ] T066 [P] Add accessibility attributes to all document forms and modals (ARIA labels, keyboard nav, screen reader support per FR-044-048)
- [ ] T067 [P] Add loading indicators to document upload and search operations (progress bars, spinners per FR-010)
- [ ] T068 Validate color contrast in Documents.razor and modals (WCAG 2.1 AA compliance, 4.5:1 ratio minimum per FR-048)
- [ ] T069 Add confirmation prompts for delete operations (modal with "Are you sure?" message per FR-030)
- [ ] T070 Run all 12 validation scenarios from specs/001-document-management/quickstart.md
- [ ] T071 Fix any issues discovered during quickstart validation
- [ ] T072 Update README.md with document management feature overview and usage instructions

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phases 3-8)**: All depend on Foundational phase completion
  - User Story 1 (Phase 3): Can start after Foundational - No dependencies on other stories
  - User Story 2 (Phase 4): Can start after Foundational - Depends on US1 (T027 download endpoint pattern)
  - User Story 3 (Phase 5): Can start after US1 completion (reuses Documents.razor from T025)
  - User Story 4 (Phase 6): Can start after US1 completion (reuses Documents.razor from T025)
  - User Story 5 (Phase 7): Can start after US1 completion (reuses Documents.razor from T025)
  - User Story 6 (Phase 8): Can start after US1 completion (integrates with Documents.razor from T025)
- **Polish (Phase 9)**: Depends on all desired user stories being complete (at minimum US1 + US2 for MVP)

### User Story Dependencies

- **User Story 1 (P1)**: Foundational → US1 (MVP core)
- **User Story 2 (P1)**: Foundational → US1 → US2 (MVP complete with projects integration)
- **User Story 3 (P2)**: Foundational → US1 → US3 (enhances discoverability)
- **User Story 4 (P2)**: Foundational → US1 → US4 (adds collaboration)
- **User Story 5 (P3)**: Foundational → US1 → US5 (quality of life)
- **User Story 6 (P3)**: Foundational → US1 → US6 (convenience feature)

### Within Each User Story

- **User Story 1**: Models (T003-006) → DB Config (T007-010) → File Storage (T011-013) → Service Interface (T014-018) → Service Implementation (T019-024) → UI Components (T025-029)
- **User Story 2**: Service methods (T030, T033-034) → UI integration (T031-032, T035)
- **User Story 3**: Service method (T036) → UI components (T037-040)
- **User Story 4**: Service methods (T041-044) → UI components (T045-046) → Notification integration (T047)
- **User Story 5**: Service methods (T048-050) → UI components (T051-053)
- **User Story 6**: Preview component (T054) → UI integration (T055-057)

### Parallel Opportunities

**Phase 1 (Setup)**: All tasks [P] can run in parallel (T001, T002)

**Phase 2 (Foundational)**: Within database models, file storage, DTOs:
- T003, T004, T005, T006 can run in parallel (different entity files)
- T011 can run in parallel with T003-006 (different subsystem)
- T014, T015, T016 can run in parallel (different DTO files)
- Sequential: T007-010 must follow T003-006 (DB config needs entities); T012-013 must follow T011 (implementation needs interface); T017-018 must follow T014-016 (interface needs DTOs)

**Phase 3 (User Story 1)**: Within service implementation and UI:
- T019, T020, T021, T022, T023 can run in parallel after T017 (different service methods)
- T025, T026 can run in parallel after T024 (different Razor components)
- Sequential: T024 must follow T019-023; T027-029 must follow T025-026

**Phase 4 (User Story 2)**:
- T030 can run in parallel with T033-034
- T031 can run in parallel with T032, T035

**Phase 5 (User Story 3)**:
- T037-040 can run in parallel after T036

**Phase 6 (User Story 4)**:
- T041, T042, T043, T044 can run in parallel (different service methods)
- T045, T046 can run in parallel after T041-044

**Phase 7 (User Story 5)**:
- T048, T049, T050 can run in parallel
- T051, T052, T053 can run in parallel after T048-050

**Phase 8 (User Story 6)**:
- T055, T056, T057 can run in parallel after T054

**Phase 9 (Polish)**:
- T058, T059, T060 can run in parallel
- T063, T064 can run in parallel
- T065, T066, T067, T068, T069 can run in parallel

---

## Parallel Example: Phase 2 (Foundational)

```bash
# Launch all entity models together:
Task T003: "Create Document entity in Models/Document.cs"
Task T004: "Create DocumentShare entity in Models/DocumentShare.cs"
Task T005: "Create DocumentTask entity in Models/DocumentTask.cs"
Task T006: "Create DocumentCategory enum in Models/DocumentCategory.cs"

# While models are being created, create file storage interface:
Task T011: "Create IFileStorageService interface in Services/IFileStorageService.cs"

# While models are being created, create DTOs:
Task T014: "Create DocumentUploadModel in Models/DocumentUploadModel.cs"
Task T015: "Create DocumentUpdateModel in Models/DocumentUpdateModel.cs"
Task T016: "Create exception classes in Services/Exceptions.cs"

# After models complete, configure database:
Task T007: "Update ApplicationDbContext with DbSets"
Task T008: "Configure entity relationships in OnModelCreating"
Task T009: "Create EF migration"
Task T010: "Apply migration"

# After IFileStorageService interface complete:
Task T012: "Implement LocalFileStorageService"
Task T013: "Register in Program.cs"

# After DTOs complete, create service interface:
Task T017: "Create IDocumentService interface"
Task T018: "Register in Program.cs"
```

---

## Implementation Strategy

### MVP First (User Stories 1 + 2 Only)

1. Complete Phase 1: Setup (2 tasks)
2. Complete Phase 2: Foundational (16 tasks - CRITICAL foundation)
3. Complete Phase 3: User Story 1 (11 tasks - personal document upload/view)
4. Complete Phase 4: User Story 2 (6 tasks - project integration)
5. **STOP and VALIDATE**: Run quickstart scenarios for US1 + US2
6. Deploy/demo MVP with core functionality

**MVP Checkpoint**: After Phase 4, users can upload documents to personal library or projects, and project members can access project documents. This is a functional training demonstration.

### Incremental Delivery

1. **Foundation** (Phases 1-2): 18 tasks → Database, services, file storage ready
2. **MVP** (Phases 3-4): 17 tasks → Upload, view, project integration working
3. **Enhanced** (Phase 5): 5 tasks → Add search and discovery
4. **Collaborative** (Phase 6): 7 tasks → Add sharing with teams
5. **Polished** (Phases 7-8): 10 tasks → Add metadata editing and preview
6. **Production-Ready** (Phase 9): 15 tasks → Polish, accessibility, validation

Each increment adds value and can be validated independently.

### Parallel Team Strategy

With multiple developers:

1. **Foundation Sprint** (all developers together): Complete Phases 1-2 (18 tasks, ~2-3 days)
2. **Feature Sprint** (parallel work after foundation):
   - Developer A: Phase 3 (User Story 1 - 11 tasks)
   - Developer B: Start Phase 4 prep (review Project/Task models)
   - Once Phase 3 complete: Developer A starts Phase 5, Developer B starts Phase 4
3. **Polish Sprint** (all developers): Phase 9 together (15 tasks, ~2 days)

---

## Notes

- **Tests not included**: Specification does not explicitly request TDD approach; validation via quickstart.md manual scenarios (T070-071)
- **[P] tasks**: Different files, no dependencies, safe for parallel execution
- **[Story] labels**: Map tasks to user stories for traceability and independent completion
- **File paths**: All paths are exact and based on plan.md project structure
- **MVP scope**: Phases 1-4 (35 tasks total) deliver functional MVP for training
- **Full feature**: All 9 phases (72 tasks total) deliver complete specification
- **Accessibility**: FR-044 through FR-048 compliance addressed in T066, T068, T057
- **Security**: IDOR protection (FR-016) implemented throughout service layer (T023, T050)
- **Constitution alignment**: All tasks follow clean architecture (services before UI), infrastructure abstraction (IFileStorageService), and explicit security patterns
