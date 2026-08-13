# Research & Technology Decisions

**Feature**: Document Upload and Management  
**Date**: 2026-08-13  
**Status**: Complete

## Research Areas

### 1. Blazor Server File Upload Patterns

**Decision**: Use `InputFile` component with streaming API for file uploads

**Rationale**:
- `InputFile` component is the Blazor standard for file uploads with built-in browser integration
- Streaming API (`IBrowserFile.OpenReadStream()`) handles large files efficiently without loading entire file into memory
- Built-in support for multiple file selection, size limits, and MIME type detection
- Compatible with Blazor Server's SignalR connection (requires `maxBufferSize` configuration)

**Implementation Pattern**:
```csharp
<InputFile OnChange="HandleFileSelected" multiple accept=".pdf,.docx,.xlsx,.pptx,.txt,.jpg,.jpeg,.png" />

private async Task HandleFileSelected(InputFileChangeEventArgs e)
{
    foreach (var file in e.GetMultipleFiles(maxAllowedFiles: 10))
    {
        if (file.Size > 25 * 1024 * 1024) // 25 MB
        {
            // Show error
            continue;
        }
        
        using var stream = file.OpenReadStream(maxAllowedSize: 25 * 1024 * 1024);
        await _documentService.UploadDocumentAsync(stream, file.Name, file.ContentType, metadata);
    }
}
```

**Alternatives Considered**:
- **HTML `<input type="file">` with JavaScript**: Rejected because it breaks Blazor's component model and requires manual interop
- **Third-party upload libraries (BlazorFileUpload, etc.)**: Rejected to minimize dependencies and keep training simple
- **Direct HTTP endpoint for uploads**: Rejected because it bypasses Blazor's authentication/authorization model

**References**:
- [ASP.NET Core Blazor file uploads](https://learn.microsoft.com/en-us/aspnet/core/blazor/file-uploads)
- [InputFile Component Documentation](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.components.forms.inputfile)

---

### 2. File Storage Abstraction Pattern

**Decision**: Create `IFileStorageService` interface with `LocalFileStorageService` implementation for training, designed for Azure Blob Storage migration

**Rationale**:
- Constitution Principle III (Infrastructure Abstraction) requires interface-based external dependencies
- Local file storage enables offline training without Azure subscription
- Interface contract supports future migration to Azure Blob Storage without changing business logic
- Separation allows mocking in tests and easy environment-specific configuration

**Interface Design**:
```csharp
public interface IFileStorageService
{
    Task<string> UploadAsync(Stream fileStream, string fileName, string contentType);
    Task<Stream> DownloadAsync(string filePath);
    Task DeleteAsync(string filePath);
    Task<bool> ExistsAsync(string filePath);
    Task<string> GetTemporaryUrlAsync(string filePath, TimeSpan expiration); // For future Azure SAS tokens
}
```

**LocalFileStorageService Implementation**:
- Store files in `{BaseDirectory}/{userId}/{projectId-or-personal}/{guid}.{extension}`
- BaseDirectory configurable via `IConfiguration` (e.g., `FileStorage:BasePath`)
- Use `Path.Combine()` for cross-platform path handling
- Validate paths to prevent directory traversal attacks (`Path.GetFullPath()` check)
- Create directories on-demand with proper permissions

**Azure Blob Storage Migration Path**:
- Create `AzureBlobStorageService : IFileStorageService`
- Use Azure.Storage.Blobs NuGet package
- Container structure mirrors directory structure
- `GetTemporaryUrlAsync` generates SAS tokens for secure time-limited access
- Switch implementation in `Program.cs` based on environment variable

**Alternatives Considered**:
- **Direct file system access in service layer**: Rejected because it violates Infrastructure Abstraction principle and prevents cloud migration
- **Repository pattern over file storage**: Rejected as over-engineering; files are not domain entities requiring repository abstraction
- **Third-party storage abstraction libraries**: Rejected to keep training dependencies minimal

**References**:
- [Azure Blob Storage for .NET](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-dotnet-get-started)
- [Dependency Injection in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection)

---

### 3. Authorization and IDOR Prevention

**Decision**: Implement service-level authorization with explicit permission checks before all document operations

**Rationale**:
- Constitution Principle II (Training-Appropriate Security) requires authorization at both page and service levels
- IDOR (Insecure Direct Object Reference) attacks occur when users can access resources by manipulating IDs
- Service-level checks ensure authorization even if page-level protection is bypassed
- Explicit permission verification makes security visible and auditable

**Authorization Pattern**:
```csharp
public async Task<Document> GetDocumentAsync(int documentId, int currentUserId)
{
    var document = await _context.Documents
        .Include(d => d.Project)
        .ThenInclude(p => p.ProjectMembers)
        .FirstOrDefaultAsync(d => d.DocumentId == documentId && !d.IsDeleted);
    
    if (document == null)
        throw new NotFoundException("Document not found");
    
    // Authorization: User owns document OR is project member OR has explicit share
    bool isOwner = document.UploadedByUserId == currentUserId;
    bool isProjectMember = document.ProjectId.HasValue && 
        document.Project.ProjectMembers.Any(pm => pm.UserId == currentUserId);
    bool hasExplicitShare = await _context.DocumentShares
        .AnyAsync(ds => ds.DocumentId == documentId && ds.SharedWithUserId == currentUserId);
    
    if (!isOwner && !isProjectMember && !hasExplicitShare)
        throw new UnauthorizedAccessException("You do not have permission to access this document");
    
    return document;
}
```

**Key Patterns**:
- Every service method that retrieves/modifies documents includes authorization check
- Use custom exceptions (`NotFoundException`, `UnauthorizedAccessException`) for clear error handling
- Never return documents from database without authorization verification
- Filter queries with `.Where(d => !d.IsDeleted)` to exclude soft-deleted documents
- Use `IHttpContextAccessor` to get current user ID in services

**Alternatives Considered**:
- **Page-level authorization only**: Rejected because it doesn't prevent direct API calls or service misuse
- **Policy-based authorization**: Rejected as over-engineering for training scenario; explicit checks are more educational
- **Resource-based authorization (ASP.NET Core)**: Rejected because it requires more boilerplate and is less explicit for training

**References**:
- [Authorization in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/introduction)
- [OWASP: Insecure Direct Object References](https://owasp.org/www-project-top-ten/2017/A5_2017-Broken_Access_Control)

---

### 4. Soft Delete Implementation

**Decision**: Add `IsDeleted` boolean and `DeletedDate` timestamp to Document entity; filter all queries with `.Where(d => !d.IsDeleted)`; implement background cleanup job

**Rationale**:
- Specification requires 30-day retention for audit and recovery
- Soft delete is simple to implement and understand for training
- Background cleanup demonstrates scheduled task patterns
- Query filters prevent accidentally exposing deleted documents

**Implementation Pattern**:
```csharp
// Entity
public class Document
{
    // ... other properties
    public bool IsDeleted { get; set; } = false;
    public DateTime? DeletedDate { get; set; }
}

// Service method
public async Task DeleteDocumentAsync(int documentId, int currentUserId)
{
    var document = await GetDocumentAsync(documentId, currentUserId); // Authorization check included
    
    if (document.UploadedByUserId != currentUserId)
        throw new UnauthorizedAccessException("Only document owner can delete");
    
    document.IsDeleted = true;
    document.DeletedDate = DateTime.UtcNow;
    await _context.SaveChangesAsync();
}

// Global query filter (ApplicationDbContext.OnModelCreating)
modelBuilder.Entity<Document>().HasQueryFilter(d => !d.IsDeleted);

// Background cleanup job (hosted service)
public class DocumentCleanupService : IHostedService
{
    public async Task CleanupExpiredDocuments()
    {
        var cutoffDate = DateTime.UtcNow.AddDays(-30);
        var expiredDocs = await _context.Documents
            .IgnoreQueryFilters() // Bypass global filter to find deleted docs
            .Where(d => d.IsDeleted && d.DeletedDate <= cutoffDate)
            .ToListAsync();
        
        foreach (var doc in expiredDocs)
        {
            await _fileStorageService.DeleteAsync(doc.FilePath); // Delete file
            _context.Documents.Remove(doc); // Delete DB record
        }
        await _context.SaveChangesAsync();
    }
}
```

**Alternatives Considered**:
- **Hard delete immediately**: Rejected because spec requires 30-day retention
- **Move to archive table**: Rejected as over-engineering; single table with flag is simpler for training
- **Database triggers for cleanup**: Rejected because SQLite has limited trigger capabilities and external job is more portable

**References**:
- [EF Core Global Query Filters](https://learn.microsoft.com/en-us/ef/core/querying/filters)
- [Background tasks in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services)

---

### 5. Accessibility Implementation (WCAG 2.1 Level AA)

**Decision**: Use semantic HTML, ARIA labels, keyboard navigation, and sufficient color contrast for all document management UI

**Rationale**:
- Specification requires WCAG 2.1 Level AA compliance (FR-044 through FR-048)
- Blazor's component model supports accessible HTML generation
- Bootstrap 5.3 provides accessible default styling (meets contrast requirements)
- Accessibility is a core web development skill for training

**Key Patterns**:

**File Upload Form**:
```html
<EditForm Model="@model" OnValidSubmit="HandleUpload">
    <div class="mb-3">
        <label for="documentTitle" class="form-label">Title *</label>
        <InputText id="documentTitle" class="form-control" @bind-Value="model.Title" 
                   aria-required="true" aria-describedby="titleHelp" />
        <div id="titleHelp" class="form-text">Provide a descriptive title for your document</div>
        <ValidationMessage For="@(() => model.Title)" />
    </div>
    
    <div class="mb-3">
        <label for="fileInput" class="form-label">Select File *</label>
        <InputFile id="fileInput" class="form-control" OnChange="HandleFileSelected" 
                   aria-required="true" aria-describedby="fileHelp"
                   accept=".pdf,.docx,.xlsx,.pptx,.txt,.jpg,.jpeg,.png" />
        <div id="fileHelp" class="form-text">Maximum file size: 25 MB. Supported formats: PDF, Office documents, text, images.</div>
    </div>
    
    <button type="submit" class="btn btn-primary" aria-label="Upload document">
        <i class="bi bi-upload" aria-hidden="true"></i> Upload
    </button>
</EditForm>
```

**Document Preview Modal**:
```html
<div class="modal" tabindex="-1" role="dialog" aria-labelledby="previewModalTitle" aria-hidden="@(!showModal)">
    <div class="modal-dialog modal-lg" role="document">
        <div class="modal-content">
            <div class="modal-header">
                <h5 class="modal-title" id="previewModalTitle">@documentTitle</h5>
                <button type="button" class="btn-close" @onclick="CloseModal" 
                        aria-label="Close preview"></button>
            </div>
            <div class="modal-body">
                <iframe src="@previewUrl" title="Document preview" class="w-100" style="height: 600px;"></iframe>
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-secondary" @onclick="CloseModal">Close (Esc)</button>
                <a href="@downloadUrl" class="btn btn-primary" download aria-label="Download document">
                    <i class="bi bi-download" aria-hidden="true"></i> Download
                </a>
            </div>
        </div>
    </div>
</div>

@code {
    protected override void OnAfterRender(bool firstRender)
    {
        if (showModal)
        {
            // Focus modal on open for screen reader announcement
            JSRuntime.InvokeVoidAsync("focusElement", ".modal");
        }
    }
}
```

**Keyboard Navigation**:
- All interactive elements (buttons, links, form inputs) are keyboard accessible via Tab
- Modal dialogs trap focus (prevent tabbing outside modal)
- Escape key closes modals
- Enter key activates buttons and form submission
- Arrow keys navigate lists and dropdown menus (native browser behavior)

**Color Contrast**:
- Use Bootstrap's default color scheme (meets WCAG AA contrast ratios)
- Verify custom colors with contrast checker tool
- Minimum ratios: 4.5:1 for normal text, 3:1 for large text (18pt+ or 14pt+ bold)

**Screen Reader Support**:
- Use semantic HTML elements (`<button>`, `<nav>`, `<main>`, `<form>`)
- Add `aria-label` for icon-only buttons
- Add `aria-describedby` to link help text to form fields
- Use `aria-hidden="true"` on decorative icons
- Provide `alt` text for all images

**Testing**:
- Automated: axe DevTools browser extension (catches 30-50% of issues)
- Manual keyboard testing: Navigate entire feature using only keyboard
- Screen reader testing: NVDA (Windows) or VoiceOver (macOS) basic walkthrough

**Alternatives Considered**:
- **Third-party accessible component libraries**: Rejected to keep training focused on standard Bootstrap and Blazor
- **WCAG AAA standard**: Rejected as too strict for training; AA is industry standard

**References**:
- [WCAG 2.1 Level AA Guidelines](https://www.w3.org/WAI/WCAG21/quickref/?versions=2.1&levels=aa)
- [Bootstrap Accessibility](https://getbootstrap.com/docs/5.3/getting-started/accessibility/)
- [Blazor Accessibility](https://learn.microsoft.com/en-us/aspnet/core/blazor/accessibility)
- [axe DevTools](https://www.deque.com/axe/devtools/)

---

### 6. Document Preview Implementation

**Decision**: Use in-browser rendering for PDFs (`<embed>` or `<iframe>`) and images (`<img>` tag) via secure download endpoint

**Rationale**:
- Modern browsers natively support PDF rendering (Chrome, Edge, Firefox, Safari)
- Images render natively in all browsers
- No third-party libraries required (keeps training simple)
- Authorization enforced at download endpoint before serving content

**Implementation Pattern**:
```csharp
// Blazor component
<button @onclick="() => ShowPreview(document)">Preview</button>

@if (showPreviewModal)
{
    <div class="modal show d-block" tabindex="-1">
        <div class="modal-dialog modal-xl">
            <div class="modal-content">
                <div class="modal-header">
                    <h5 class="modal-title">@previewDocument.Title</h5>
                    <button type="button" class="btn-close" @onclick="ClosePreview"></button>
                </div>
                <div class="modal-body">
                    @if (previewDocument.FileType.StartsWith("image/"))
                    {
                        <img src="/api/documents/@previewDocument.DocumentId/download" 
                             alt="@previewDocument.Title" class="img-fluid" />
                    }
                    else if (previewDocument.FileType == "application/pdf")
                    {
                        <embed src="/api/documents/@previewDocument.DocumentId/download" 
                               type="application/pdf" width="100%" height="600px" />
                    }
                </div>
            </div>
        </div>
    </div>
}

// Download endpoint (Razor Page or API Controller)
[Authorize]
public async Task<IActionResult> OnGetDownload(int id)
{
    var userId = GetCurrentUserId();
    
    try
    {
        var (stream, fileName, contentType) = await _documentService.DownloadDocumentAsync(id, userId);
        return File(stream, contentType, fileName);
    }
    catch (UnauthorizedAccessException)
    {
        return Forbid();
    }
    catch (NotFoundException)
    {
        return NotFound();
    }
}
```

**Security Considerations**:
- Always pass through authorization endpoint (never direct file URLs)
- Use `Content-Disposition: inline` for preview, `attachment` for download
- Set `X-Content-Type-Options: nosniff` header to prevent MIME type sniffing
- Validate file MIME type matches expected type before serving

**Browser Compatibility**:
- PDF preview: Chrome/Edge/Firefox/Safari (95%+ market share)
- Fallback: Show "Download to view" button for unsupported browsers
- Image preview: Universal support

**Alternatives Considered**:
- **PDF.js library**: Rejected to avoid additional dependencies and keep training simple
- **Office document preview (Word, Excel, PowerPoint)**: Rejected because it requires Office Online integration or third-party libraries
- **Server-side conversion to images**: Rejected as over-engineering and adds processing overhead

**References**:
- [HTML <embed> Element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/embed)
- [Content-Disposition Header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Disposition)

---

## Summary of Decisions

| Area | Decision | Training Implementation | Production Migration Path |
|------|----------|------------------------|---------------------------|
| **File Upload** | Blazor `InputFile` component with streaming | `InputFile` + validation | Same |
| **File Storage** | `IFileStorageService` abstraction | `LocalFileStorageService` (filesystem) | `AzureBlobStorageService` (Azure Blob) |
| **Authorization** | Service-level permission checks | Explicit ownership/membership checks | Same pattern + Azure AD roles |
| **Soft Delete** | `IsDeleted` flag + background cleanup | In-process hosted service | Azure Function timer trigger |
| **Accessibility** | WCAG 2.1 Level AA compliance | Semantic HTML + ARIA labels + keyboard nav | Same |
| **Preview** | In-browser rendering via download endpoint | Native browser PDF/image rendering | Same + CDN for faster delivery |

## Open Questions

None - all technical approaches are confirmed and align with constitution principles.
