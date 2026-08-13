# Feature Specification: Document Upload and Management

**Feature Branch**: `001-document-management`  
**Created**: 2026-08-13  
**Status**: Draft  
**Input**: Stakeholder requirements from `StakeholderDocs/document-upload-and-management-feature.md`

## Clarifications

### Session 2026-08-13

- Q: How should document tags be stored in the database? → A: Comma-separated text in Document.Tags column (simple implementation, adequate for training, LIKE queries for search)
- Q: What happens when two users simultaneously upload documents with the same title to the same project? → A: Allow both uploads - titles are not unique constraints (DocumentId and GUID-based file paths ensure uniqueness, users distinguish by metadata)
- Q: What accessibility standard should the document upload and preview features meet? → A: WCAG 2.1 Level AA compliance (keyboard navigation, screen reader support, ARIA labels, color contrast)
- Q: How long should deleted documents be retained before permanent removal? → A: 30 days (soft delete with DeletedDate timestamp, background cleanup removes files/records after 30 days)
- Q: When sharing documents with "teams" (FR-024), what does "team" mean in this context? → A: Project teams only (reuse existing ProjectMembers table; sharing with a project creates DocumentShare records for all project members)

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Upload and View Personal Documents (Priority: P1)

As an employee, I want to upload work-related documents to my personal document library so that I can access them from anywhere within the dashboard without searching my local drives.

**Why this priority**: This is the core MVP functionality. Without basic upload and viewing capabilities, no other features can exist. This delivers immediate value by centralizing document storage.

**Independent Test**: Can be fully tested by logging in as any user, uploading a PDF document with title and category, and verifying it appears in the "My Documents" list with correct metadata displayed.

**Acceptance Scenarios**:

1. **Given** I am logged into the dashboard, **When** I navigate to the Documents page and click "Upload Document", **Then** I see an upload form with fields for file selection, title, description, category, and tags
2. **Given** I have selected a PDF file under 25 MB, **When** I provide a title and category then submit, **Then** the file uploads successfully and appears in my document list with upload date, file size, and my name as uploader
3. **Given** I am viewing my documents list, **When** I sort by title or upload date, **Then** documents reorder correctly
4. **Given** I am viewing my documents list, **When** I filter by category, **Then** only documents in that category appear
5. **Given** I am viewing my documents, **When** I click a document's download link, **Then** the file downloads to my computer

---

### User Story 2 - Associate Documents with Projects (Priority: P1)

As a project team member, I want to attach documents to projects I'm working on so that all team members can access project-related files in one central location.

**Why this priority**: Project collaboration is a core dashboard feature. Integrating documents with existing projects creates immediate value for team workflows and is essential for the training scenario.

**Independent Test**: Can be fully tested by logging in as a project member, uploading a document with a project selected, navigating to that project's detail page, and verifying the document appears in the project documents section and is accessible to other project members.

**Acceptance Scenarios**:

1. **Given** I am uploading a document, **When** I select an associated project from the dropdown, **Then** the document is linked to that project
2. **Given** I am viewing a project detail page, **When** I navigate to the documents section, **Then** I see all documents associated with that project
3. **Given** I am a team member on a project, **When** I view the project's documents, **Then** I can download any document uploaded by other team members
4. **Given** I am viewing a task detail page, **When** I attach a document to the task, **Then** the document is automatically associated with the task's parent project
5. **Given** I am a Project Manager, **When** I view my project's documents, **Then** I can upload new documents directly to the project

---

### User Story 3 - Search and Discover Documents (Priority: P2)

As a dashboard user, I want to search for documents by keywords, tags, or uploader name so that I can quickly locate specific documents without browsing through lists.

**Why this priority**: Search significantly improves usability once users have uploaded multiple documents. Not critical for MVP but high-value enhancement for daily use.

**Independent Test**: Can be fully tested by uploading 10 documents with varying titles, tags, and uploaders, then performing searches by different criteria and verifying only authorized matching documents appear in results within 2 seconds.

**Acceptance Scenarios**:

1. **Given** I am on the documents page, **When** I enter a keyword in the search box, **Then** I see all documents whose title, description, or tags match the keyword
2. **Given** I search for documents, **When** results are returned, **Then** only documents I have permission to access appear in the results
3. **Given** I search for a document, **When** I filter search results by category or date range, **Then** results update to show only matching documents
4. **Given** multiple documents match my search, **When** I sort results by relevance or date, **Then** documents reorder appropriately
5. **Given** I search for documents, **When** the search completes, **Then** results appear within 2 seconds

---

### User Story 4 - Share Documents with Team Members (Priority: P2)

As a document owner, I want to share specific documents with colleagues or teams so that they can access files even if they're not members of the associated project.

**Why this priority**: Enables cross-team collaboration and flexible sharing beyond project boundaries. Valuable for productivity but not essential for basic document management.

**Independent Test**: Can be fully tested by uploading a document, sharing it with a specific user, logging in as that user, verifying the document appears in their "Shared with Me" section, and confirming they receive an in-app notification.

**Acceptance Scenarios**:

1. **Given** I own a document, **When** I click the "Share" button and select individual users or a project team, **Then** those users receive access to the document (sharing with a project team creates shares for all current ProjectMembers)
2. **Given** a document is shared with me, **When** I check my notifications, **Then** I see a notification that someone shared a document
3. **Given** a document is shared with me, **When** I navigate to "Shared with Me", **Then** I see the shared document and can download it
4. **Given** I own a document, **When** I revoke sharing for a specific user, **Then** they no longer see the document in their "Shared with Me" section
5. **Given** I am a recipient of a shared document, **When** I try to edit or delete the document, **Then** the system prevents me (only owner can modify)

---

### User Story 5 - Manage and Update Document Metadata (Priority: P3)

As a document owner, I want to edit document details (title, description, category, tags) and replace files with updated versions so that I can keep documents current without deleting and re-uploading.

**Why this priority**: Quality-of-life improvement that enhances long-term usability. Users can initially manage documents by deleting and re-uploading, making this lower priority.

**Independent Test**: Can be fully tested by uploading a document, editing its title and category, verifying the changes persist, then uploading a replacement file and confirming the new version is served when downloaded.

**Acceptance Scenarios**:

1. **Given** I own a document, **When** I click "Edit" and change the title or description, **Then** the updated metadata is saved and displayed
2. **Given** I own a document, **When** I upload a replacement file, **Then** the new file replaces the old one while preserving metadata and permissions
3. **Given** I own a document, **When** I add or remove tags, **Then** search results reflect the updated tags
4. **Given** I own a document, **When** I change its category, **Then** it appears in the correct category filter
5. **Given** I don't own a document (shared or project document), **When** I try to edit metadata, **Then** the system prevents editing

---

### User Story 6 - Preview Common Document Types (Priority: P3)

As a dashboard user, I want to preview PDFs and images directly in the browser so that I can quickly view content without downloading files.

**Why this priority**: Nice-to-have convenience feature. Users can still accomplish their goals by downloading and opening files locally. Enhances experience but not critical for training objectives.

**Independent Test**: Can be fully tested by uploading a PDF and an image, clicking the preview button for each, and verifying they render in the browser within 3 seconds without requiring a download.

**Acceptance Scenarios**:

1. **Given** I am viewing a PDF document, **When** I click "Preview", **Then** the PDF renders in a browser modal within 3 seconds
2. **Given** I am viewing an image document, **When** I click "Preview", **Then** the image displays in a browser modal
3. **Given** I am viewing a document that cannot be previewed (e.g., Excel), **When** I look for the preview option, **Then** only a download button is available
4. **Given** a preview is open, **When** I click "Download" within the preview modal, **Then** the file downloads
5. **Given** I don't have permission to view a document, **When** I try to access the preview URL directly, **Then** the system returns an authorization error

---

### Edge Cases

- **What happens when a user uploads a file exceeding 25 MB?** System rejects the upload before processing and displays a clear error message: "File size exceeds 25 MB limit. Please select a smaller file."
- **What happens when a user uploads an unsupported file type (e.g., .exe)?** System validates file extension against a whitelist and rejects with error: "File type not supported. Please upload PDF, Office documents, text files, or images."
- **What happens when a user tries to download a document they don't have permission to access?** Service-level authorization check denies access and returns HTTP 403 Forbidden with message: "You don't have permission to access this document."
- **What happens when file storage disk is full?** System catches the IOException during file write and displays error: "Unable to save document. Please try again later or contact support."
- **What happens when a user deletes a document that's shared with others?** System soft-deletes the document (sets IsDeleted=true, DeletedDate=now). Shared users see a message: "This document is no longer available" in their "Shared with Me" section. Document and file are permanently removed after 30 days.
- **What happens when a user navigates directly to a deleted document URL?** System returns HTTP 404 Not Found: "Document not found" (soft-deleted documents are not accessible).
- **What happens when a project is deleted that has associated documents?** Documents remain in the system but project association is cleared. Document owners retain access; orphaned documents show "No associated project."
- **What happens when virus scanning detects malware in an uploaded file?** (For training: Virus scanning is simulated) System rejects the upload and displays: "File failed security scan. Upload blocked for safety."
- **What happens when two users upload documents with identical filenames?** System generates unique GUID-based storage filenames to prevent collisions. Users see their chosen display names, but physical files have unique names like `abc123-def456.pdf`.
- **What happens when two users simultaneously upload documents with the same title to the same project?** Both uploads succeed. Document titles are not unique constraints; each document has a unique DocumentId and GUID-based FilePath. Users distinguish documents by uploader name, upload date, and other metadata in the document list.
- **What happens when a user's role changes (e.g., removed from a project)?** Authorization checks enforce current permissions. User immediately loses access to project documents they could previously view.

## Requirements *(mandatory)*

### Functional Requirements

#### Upload and Storage
- **FR-001**: System MUST allow authenticated users to select one or multiple files from their device for upload
- **FR-002**: System MUST support PDF, Microsoft Office documents (Word .docx, Excel .xlsx, PowerPoint .pptx), text files (.txt), and image files (JPEG, PNG) with a maximum file size of 25 MB per file
- **FR-003**: System MUST require users to provide a document title and category (from predefined list: Project Documents, Team Resources, Personal Files, Reports, Presentations, Other) before upload
- **FR-004**: System MUST allow users to optionally provide a description, associated project, and custom tags during upload
- **FR-005**: System MUST automatically capture and store upload date/time, uploader user ID, file size in bytes, and MIME type (up to 255 characters)
- **FR-006**: System MUST generate unique GUID-based filenames and store files in a secure directory structure: `{userId}/{projectId or "personal"}/{guid}.{extension}`
- **FR-007**: System MUST save file to disk BEFORE creating database record to prevent orphaned database entries
- **FR-008**: System MUST validate file extension against a whitelist before accepting upload
- **FR-009**: System MUST reject files exceeding 25 MB with a clear error message before processing begins
- **FR-010**: System MUST display upload progress indicator and success/error messages after upload completes

#### Authorization and Security
- **FR-011**: System MUST enforce authorization at service layer: users can only upload documents to projects where they are members or to their personal library
- **FR-012**: System MUST prevent direct file access by storing files outside web-accessible directories (not in `wwwroot`)
- **FR-013**: System MUST implement authorization checks in download endpoint: users can only download documents they own, are project members for, or have been explicitly shared with them
- **FR-014**: System MUST use GUID-based filenames to prevent path traversal attacks and filename guessing
- **FR-015**: System MUST validate MIME types match file extensions to prevent file type spoofing
- **FR-016**: System MUST implement IDOR protection: verify user authorization before returning any document metadata or file content

#### Document Organization and Discovery
- **FR-017**: System MUST display a "My Documents" view showing all documents uploaded by the current user with columns: title, category, upload date, file size, associated project
- **FR-018**: Users MUST be able to sort documents by title, upload date, category, or file size (ascending/descending)
- **FR-019**: Users MUST be able to filter documents by category, associated project, or date range
- **FR-020**: System MUST display project documents on the project detail page, visible to all project members
- **FR-021**: System MUST provide search functionality that queries document title, description, tags, uploader name, and associated project name
- **FR-022**: System MUST return search results within 2 seconds and show only documents the user is authorized to access
- **FR-023**: System MUST allow users to download any document they have access to via a secure download endpoint

#### Document Sharing
- **FR-024**: Document owners MUST be able to share documents with specific users or project teams; when sharing with a project team, system creates DocumentShare records for all current ProjectMembers of that project
- **FR-025**: System MUST create a "Shared with Me" view showing all documents explicitly shared with the current user
- **FR-026**: System MUST send an in-app notification to recipients when a document is shared with them
- **FR-027**: Document owners MUST be able to revoke sharing access for specific users
- **FR-051**: When a user is added to a project team after a document was shared with that project, they do NOT automatically gain access (sharing is snapshot of team membership at share time)

#### Document Management
- **FR-028**: Document owners MUST be able to edit document metadata: title, description, category, tags
- **FR-029**: Document owners MUST be able to upload a replacement file that preserves metadata and sharing relationships
- **FR-030**: Document owners MUST be able to delete documents they uploaded after confirmation prompt
- **FR-031**: Project Managers MUST be able to delete any document associated with their projects after confirmation
- **FR-032**: System MUST soft-delete documents by setting IsDeleted flag and DeletedDate timestamp; soft-deleted documents are excluded from all queries and inaccessible to users
- **FR-049**: System MUST implement a background cleanup process that permanently removes soft-deleted documents (database records and physical files) after 30 days from DeletedDate
- **FR-050**: Administrators MUST be able to view a list of soft-deleted documents for audit purposes (within the 30-day retention window)

#### Task Integration
- **FR-033**: System MUST allow users to attach documents to tasks from the task detail page
- **FR-034**: System MUST automatically associate task-attached documents with the task's parent project
- **FR-035**: System MUST display attached documents on the task detail page

#### Dashboard Integration
- **FR-036**: System MUST add a "Recent Documents" widget to the dashboard home page showing the user's last 5 uploaded documents
- **FR-037**: System MUST add document count to dashboard summary cards

#### Preview Capability
- **FR-038**: System MUST provide in-browser preview for PDF files without requiring download
- **FR-039**: System MUST provide in-browser preview for image files (JPEG, PNG) without requiring download
- **FR-040**: System MUST load previews within 3 seconds and enforce authorization before rendering content

#### Performance
- **FR-041**: Document upload MUST complete within 30 seconds for files up to 25 MB on typical network connections
- **FR-042**: Document list pages MUST load within 2 seconds for up to 500 documents
- **FR-043**: Document search MUST return results within 2 seconds

#### Accessibility
- **FR-044**: System MUST comply with WCAG 2.1 Level AA accessibility standards for all document management UI components
- **FR-045**: Upload form MUST be fully keyboard navigable with proper tab order and focus indicators
- **FR-046**: All interactive elements (buttons, links, form controls) MUST have descriptive ARIA labels for screen readers
- **FR-047**: Document preview modals MUST support keyboard navigation (Escape to close, Tab to navigate controls)
- **FR-048**: Color contrast ratios MUST meet WCAG 2.1 AA standards (4.5:1 for normal text, 3:1 for large text) for all document management UI elements

### Key Entities

- **Document**: Represents an uploaded file with metadata. Attributes: DocumentId (integer primary key), Title, Description, Category (text: "Project Documents", "Personal Files", etc.), FileName (original user-provided name), FilePath (GUID-based storage path), FileSize (bytes), FileType (MIME type up to 255 chars), UploadDate, UploadedByUserId (foreign key to User), ProjectId (nullable foreign key to Project), Tags (comma-separated text string, e.g., "budget,2026,draft"), IsDeleted (soft delete flag, default false), DeletedDate (nullable DateTime, set when IsDeleted=true, used for 30-day retention cleanup). Tag searching uses LIKE queries (e.g., WHERE Tags LIKE '%budget%'), and tag editing/display requires string split/join in service layer. All queries MUST filter WHERE IsDeleted = false to exclude soft-deleted documents.

- **DocumentShare**: Represents a sharing relationship between a document and a user. Attributes: DocumentShareId (integer primary key), DocumentId (foreign key to Document), SharedWithUserId (foreign key to User), SharedByUserId (foreign key to User), SharedDate. When a document is shared with a project team, individual DocumentShare records are created for each ProjectMember at the time of sharing (snapshot approach; new team members added later do not automatically gain access).

- **Document-Task Relationship**: Links documents to specific tasks. Attributes: DocumentId (foreign key to Document), TaskId (foreign key to TaskItem)

- **IFileStorageService**: Interface abstraction for file storage operations. Must support UploadAsync(stream, filename, contentType) → returns file path, DeleteAsync(filePath), DownloadAsync(filePath) → returns stream, GetUrlAsync(filePath, expiration) → returns temporary URL (for future Azure Blob integration)

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Within 3 months of launch, 70% of active dashboard users upload at least one document
- **SC-002**: Users can upload a document from file selection to confirmation in under 30 seconds for files up to 25 MB
- **SC-003**: Users can locate a specific document through search or filtering in under 30 seconds (measured from task start to document download click)
- **SC-004**: Document list pages load and display within 2 seconds for up to 500 documents
- **SC-005**: 90% of uploaded documents have a category assigned (demonstrating users understand and use the categorization feature)
- **SC-006**: Zero unauthorized document access incidents during testing (all authorization checks prevent IDOR attacks)
- **SC-007**: System successfully prevents upload of all unsupported file types (100% validation success rate in testing)
- **SC-008**: Users can preview PDF and image documents in under 3 seconds
- **SC-009**: Document search returns results within 2 seconds for 95% of queries
- **SC-010**: Project team members can successfully attach and access project documents within 1 minute of receiving access to a project
- **SC-011**: Document management UI components pass WCAG 2.1 Level AA automated accessibility testing (100% compliance with axe or similar validator)

## Assumptions

- Users have reliable internet connectivity for uploading documents up to 25 MB
- The local filesystem (for training) or Azure Blob Storage (for production) provides sufficient storage capacity for uploaded documents
- SQLite database (training) or Azure SQL (production) can handle document metadata queries efficiently for up to 10,000 documents per project
- Users understand file organization concepts (categories, tags, projects)
- The existing mock authentication system provides reliable user identity for authorization checks
- Virus/malware scanning will be simulated in training (placeholder validation) and implemented with real scanning service in production
- MIME type validation and file extension whitelist provide adequate security for training purposes (production would add additional scanning layers)
- Preview functionality is limited to PDF and images; Office document preview requires additional libraries not included in training scope

## Out of Scope

The following are explicitly NOT included in this feature specification:

- **Real-time collaboration**: Multiple users editing the same document simultaneously (e.g., Google Docs-style editing)
- **Version history**: Tracking and reverting to previous versions of a document (only latest version stored)
- **Document approval workflows**: Requiring manager approval before document publishing
- **Advanced document processing**: OCR, text extraction, automated tagging, content indexing
- **Email attachments**: Uploading documents via email integration
- **External sharing**: Sharing documents with users outside the organization via public links
- **Folder hierarchy**: Nested folder structures for document organization (flat structure with categories only)
- **Bulk operations**: Downloading multiple documents as a ZIP archive, bulk metadata editing
- **Document templates**: Pre-built document templates for common use cases
- **Integration with external document systems**: SharePoint, Google Drive, Dropbox sync
- **Mobile app**: Native mobile application for document access (web responsive only)
- **Offline access**: Downloading documents for offline use with sync capability
- **Document analytics**: Tracking view counts, most popular documents, access patterns (basic activity logging only for auditing)
- **Advanced search**: Full-text search within document content, fuzzy matching, natural language queries (basic metadata search only)

## Notes

This feature implements the Infrastructure Abstraction principle from the constitution by using `IFileStorageService` interface, enabling seamless migration from local file storage (training) to Azure Blob Storage (production) without changing business logic or UI code. File paths are stored as relative paths in the database to maintain portability across storage implementations.

Authorization follows the constitution's Training-Appropriate Security principle with checks at both page level (`[Authorize]` attribute) and service level (explicit permission verification before data access), preventing IDOR vulnerabilities.

The feature adheres to Clean Service Architecture by implementing `IDocumentService` that handles all business logic, validation, and authorization separately from the Blazor UI components.
