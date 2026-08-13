# IDocumentService Contract

**Purpose**: Interface defining document management business logic with authorization, validation, and storage orchestration.

**Namespace**: `ContosoDashboard.Services`

**Lifecycle**: Scoped (per HTTP request) - registered in `Program.cs` as `builder.Services.AddScoped<IDocumentService, DocumentService>()`

---

## Interface Definition

```csharp
public interface IDocumentService
{
    // Upload Operations
    Task<Document> UploadDocumentAsync(
        Stream fileStream, 
        string fileName, 
        string contentType, 
        DocumentUploadModel metadata, 
        int currentUserId);

    // Retrieval Operations
    Task<Document> GetDocumentAsync(int documentId, int currentUserId);
    Task<List<Document>> GetUserDocumentsAsync(int userId, int currentUserId);
    Task<List<Document>> GetProjectDocumentsAsync(int projectId, int currentUserId);
    Task<List<Document>> GetSharedDocumentsAsync(int currentUserId);
    Task<List<Document>> SearchDocumentsAsync(string searchTerm, int currentUserId);

    // Download Operations
    Task<(Stream fileStream, string fileName, string contentType)> DownloadDocumentAsync(
        int documentId, 
        int currentUserId);

    // Metadata Management
    Task UpdateDocumentMetadataAsync(
        int documentId, 
        DocumentUpdateModel metadata, 
        int currentUserId);
    Task ReplaceDocumentFileAsync(
        int documentId, 
        Stream fileStream, 
        string fileName, 
        string contentType, 
        int currentUserId);

    // Sharing Operations
    Task ShareDocumentAsync(int documentId, List<int> recipientUserIds, int currentUserId);
    Task ShareDocumentWithProjectTeamAsync(int documentId, int projectId, int currentUserId);
    Task RevokeShareAsync(int documentShareId, int currentUserId);

    // Task Integration
    Task AttachDocumentToTaskAsync(int documentId, int taskId, int currentUserId);
    Task<List<Document>> GetTaskDocumentsAsync(int taskId, int currentUserId);
    Task DetachDocumentFromTaskAsync(int documentId, int taskId, int currentUserId);

    // Deletion Operations
    Task SoftDeleteDocumentAsync(int documentId, int currentUserId);
    Task<List<Document>> GetSoftDeletedDocumentsAsync(); // Admin only
    Task PermanentlyDeleteExpiredDocumentsAsync(); // Background job only

    // Authorization Helpers
    Task<bool> CanUserAccessDocumentAsync(int documentId, int userId);
    Task<bool> CanUserModifyDocumentAsync(int documentId, int userId);
}
```

---

## Method Specifications

### UploadDocumentAsync

**Purpose**: Upload a new document with metadata and store file securely.

**Parameters**:
- `fileStream`: Stream containing file content (max 25 MB)
- `fileName`: Original filename from user's device (e.g., "Budget_2026.pdf")
- `contentType`: MIME type (e.g., "application/pdf")
- `metadata`: User-provided metadata (Title, Description, Category, Tags, ProjectId)
- `currentUserId`: Authenticated user performing upload

**Authorization**:
- User must be authenticated
- If `metadata.ProjectId` is provided, user must be a member of that project

**Validation**:
- File size ≤ 25 MB (26,214,400 bytes)
- MIME type in whitelist: `application/pdf`, `application/vnd.openxmlformats-officedocument.*`, `text/plain`, `image/jpeg`, `image/png`
- File extension matches MIME type (prevent spoofing)
- Title required (1-200 characters)
- Category required (from predefined enum)

**Process**:
1. Validate file size and MIME type → throw `ValidationException` on failure
2. Check project membership authorization → throw `UnauthorizedAccessException` on failure
3. Generate unique FilePath: `{currentUserId}/{projectId-or-personal}/{Guid.NewGuid()}.{extension}`
4. Call `IFileStorageService.UploadAsync()` to save file
5. Create Document entity in database with metadata
6. Send notification to project members if ProjectId is set
7. Return created Document entity

**Exceptions**:
- `ValidationException`: Invalid file size, type, or metadata
- `UnauthorizedAccessException`: User not authorized to upload to specified project
- `IOException`: File storage failure

**Returns**: Created `Document` entity with populated DocumentId

---

### GetDocumentAsync

**Purpose**: Retrieve a single document with authorization check.

**Parameters**:
- `documentId`: Document to retrieve
- `currentUserId`: Authenticated user requesting document

**Authorization**:
- User owns document (UploadedByUserId = currentUserId) OR
- User is member of document's project OR
- Document is explicitly shared with user (DocumentShare record exists)

**Process**:
1. Query document with `.Include(d => d.Project).ThenInclude(p => p.ProjectMembers)` for authorization
2. Verify authorization → throw `UnauthorizedAccessException` if denied
3. Return Document entity

**Exceptions**:
- `NotFoundException`: Document doesn't exist or is soft-deleted
- `UnauthorizedAccessException`: User lacks permission

**Returns**: `Document` entity

---

### GetUserDocumentsAsync

**Purpose**: Retrieve all documents uploaded by a specific user.

**Parameters**:
- `userId`: User whose documents to retrieve
- `currentUserId`: Authenticated user making request

**Authorization**:
- If `userId == currentUserId`, return all user's documents (My Documents)
- If `userId != currentUserId`, return only documents the current user can access (requires shared or project membership)

**Process**:
1. Query `Documents.Where(d => d.UploadedByUserId == userId)`
2. If `userId != currentUserId`, filter by authorization (shared or project member)
3. Order by UploadDate descending
4. Return list

**Returns**: `List<Document>` ordered by upload date (newest first)

---

### SearchDocumentsAsync

**Purpose**: Search documents by keyword in title, description, tags with authorization filtering.

**Parameters**:
- `searchTerm`: Keyword to search (case-insensitive)
- `currentUserId`: Authenticated user performing search

**Authorization**: Only return documents user has access to (owned, project member, or shared)

**Process**:
1. Query documents where `(Title.Contains(searchTerm) || Description.Contains(searchTerm) || Tags.Contains(searchTerm))`
2. Apply authorization filter (owner, project member, or explicit share)
3. Order by UploadDate descending
4. Return results within 2 seconds (FR-043)

**Performance**: Use indexes on UploadedByUserId, ProjectId; paginate results for large datasets

**Returns**: `List<Document>` matching search criteria and authorization

---

### DownloadDocumentAsync

**Purpose**: Retrieve file stream for download/preview with authorization.

**Parameters**:
- `documentId`: Document to download
- `currentUserId`: Authenticated user requesting download

**Authorization**: Same as `GetDocumentAsync` (owner, project member, or shared)

**Process**:
1. Call `GetDocumentAsync()` to verify authorization
2. Call `IFileStorageService.DownloadAsync(document.FilePath)` to get file stream
3. Return tuple of (stream, FileName, FileType)

**Exceptions**:
- `NotFoundException`: Document or file doesn't exist
- `UnauthorizedAccessException`: User lacks permission
- `IOException`: File read failure

**Returns**: Tuple `(Stream fileStream, string fileName, string contentType)`

---

### ShareDocumentAsync

**Purpose**: Share document with specific users (explicit sharing).

**Parameters**:
- `documentId`: Document to share
- `recipientUserIds`: List of user IDs to share with
- `currentUserId`: Authenticated user initiating share

**Authorization**: Only document owner can share (UploadedByUserId = currentUserId)

**Validation**:
- `recipientUserIds` cannot include `currentUserId` (can't share with yourself)
- All recipient user IDs must exist in Users table

**Process**:
1. Verify current user owns document → throw `UnauthorizedAccessException` otherwise
2. For each recipient:
   - Check if DocumentShare already exists → skip if duplicate
   - Create DocumentShare record with SharedByUserId = currentUserId
3. Send notification to each recipient
4. Save changes

**Exceptions**:
- `NotFoundException`: Document doesn't exist
- `UnauthorizedAccessException`: User doesn't own document
- `ValidationException`: Invalid recipient user IDs

**Returns**: `Task` (void)

---

### ShareDocumentWithProjectTeamAsync

**Purpose**: Share document with all current members of a project (snapshot approach per FR-051).

**Parameters**:
- `documentId`: Document to share
- `projectId`: Project whose members receive access
- `currentUserId`: Authenticated user initiating share

**Authorization**: Only document owner can share

**Process**:
1. Verify current user owns document
2. Query all ProjectMembers for specified project
3. For each project member (excluding owner):
   - Create DocumentShare record (snapshot at share time)
4. Send notification to all recipients
5. Save changes

**Note**: New members added to project later do NOT automatically get access (FR-051)

**Returns**: `Task` (void)

---

### SoftDeleteDocumentAsync

**Purpose**: Mark document as deleted (soft delete) with 30-day retention.

**Parameters**:
- `documentId`: Document to delete
- `currentUserId`: Authenticated user initiating deletion

**Authorization**:
- User owns document (UploadedByUserId = currentUserId) OR
- User is Project Manager on document's project

**Process**:
1. Verify authorization → throw `UnauthorizedAccessException` if denied
2. Set `IsDeleted = true` and `DeletedDate = DateTime.UtcNow`
3. Save changes
4. Document immediately becomes inaccessible due to global query filter

**Note**: Physical file remains on disk; permanent deletion handled by background job after 30 days

**Returns**: `Task` (void)

---

### PermanentlyDeleteExpiredDocumentsAsync

**Purpose**: Background job to permanently delete documents soft-deleted more than 30 days ago.

**Authorization**: Called by background hosted service only (no user context)

**Process**:
1. Query documents with `.IgnoreQueryFilters().Where(d => d.IsDeleted && d.DeletedDate <= DateTime.UtcNow.AddDays(-30))`
2. For each expired document:
   - Call `IFileStorageService.DeleteAsync(document.FilePath)` to remove physical file
   - Call `_context.Documents.Remove(document)` to delete database record
   - Cascade deletes DocumentShare and DocumentTask records automatically
3. Save changes

**Returns**: `Task` (void)

---

## Data Transfer Objects (DTOs)

### DocumentUploadModel

```csharp
public class DocumentUploadModel
{
    [Required, MaxLength(200)]
    public string Title { get; set; }
    
    [MaxLength(1000)]
    public string? Description { get; set; }
    
    [Required]
    public DocumentCategory Category { get; set; }
    
    [MaxLength(500)]
    public string? Tags { get; set; } // Comma-separated
    
    public int? ProjectId { get; set; }
}

public enum DocumentCategory
{
    ProjectDocuments,
    TeamResources,
    PersonalFiles,
    Reports,
    Presentations,
    Other
}
```

### DocumentUpdateModel

```csharp
public class DocumentUpdateModel
{
    [Required, MaxLength(200)]
    public string Title { get; set; }
    
    [MaxLength(1000)]
    public string? Description { get; set; }
    
    [Required]
    public DocumentCategory Category { get; set; }
    
    [MaxLength(500)]
    public string? Tags { get; set; }
}
```

---

## Error Handling

All service methods follow consistent error handling:

- **NotFoundException**: Resource doesn't exist (HTTP 404)
- **UnauthorizedAccessException**: User lacks permission (HTTP 403)
- **ValidationException**: Invalid input (HTTP 400)
- **IOException**: File system error (HTTP 500)

Blazor pages catch these exceptions and display appropriate user-friendly messages.

---

## Implementation Notes

- **Dependency Injection**: Service requires `ApplicationDbContext`, `IFileStorageService`, `INotificationService`, `IHttpContextAccessor`
- **Transactions**: Use explicit transactions for multi-step operations (upload file → create DB record)
- **Async/Await**: All I/O operations are async to prevent thread blocking
- **Null Safety**: Enable nullable reference types; validate all inputs
- **Logging**: Log authorization failures and file operations for audit trail
