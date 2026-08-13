# Data Model: Document Management

**Feature**: Document Upload and Management  
**Date**: 2026-08-13  
**Status**: Complete

## Entity Relationship Diagram

```
User (existing)
 │
 ├─── 1:N ──► Document (owns/uploads)
 │              │
 │              ├─── N:1 ──► Project (optional association)
 │              │
 │              ├─── 1:N ──► DocumentShare (shared with users)
 │              │              │
 │              │              └─── N:1 ──► User (recipient)
 │              │
 │              └─── N:N ──► TaskItem (via DocumentTask join table)
 │
 └─── 1:N ──► DocumentShare (recipient)

ProjectMember (existing)
 │
 └─── Used to resolve project team membership for document access authorization
```

## Entities

### Document

**Purpose**: Represents an uploaded file with metadata, ownership, and optional project association.

**Attributes**:

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `DocumentId` | `int` | PRIMARY KEY, IDENTITY | Unique document identifier |
| `Title` | `nvarchar(200)` | NOT NULL | User-provided document title |
| `Description` | `nvarchar(1000)` | NULL | Optional user-provided description |
| `Category` | `nvarchar(50)` | NOT NULL | Document category: "Project Documents", "Team Resources", "Personal Files", "Reports", "Presentations", "Other" |
| `FileName` | `nvarchar(255)` | NOT NULL | Original filename from user's device (e.g., "Budget_2026.pdf") |
| `FilePath` | `nvarchar(500)` | NOT NULL, UNIQUE | Storage path: "{userId}/{projectId-or-personal}/{guid}.{ext}" |
| `FileSize` | `bigint` | NOT NULL | File size in bytes (max 25 MB = 26,214,400 bytes) |
| `FileType` | `nvarchar(255)` | NOT NULL | MIME type (e.g., "application/pdf", "image/jpeg") |
| `Tags` | `nvarchar(500)` | NULL | Comma-separated tags (e.g., "budget,2026,draft") for LIKE queries |
| `UploadDate` | `datetime2` | NOT NULL, DEFAULT GETUTCDATE() | UTC timestamp when document was uploaded |
| `UploadedByUserId` | `int` | NOT NULL, FOREIGN KEY → User.UserId | Owner/uploader of the document |
| `ProjectId` | `int` | NULL, FOREIGN KEY → Project.ProjectId | Optional project association |
| `IsDeleted` | `bit` | NOT NULL, DEFAULT 0 | Soft delete flag (0 = active, 1 = deleted) |
| `DeletedDate` | `datetime2` | NULL | UTC timestamp when document was soft-deleted; NULL if active |

**Indexes**:
- `IX_Document_UploadedByUserId` (non-clustered) - for "My Documents" queries
- `IX_Document_ProjectId` (non-clustered) - for project documents queries
- `IX_Document_UploadDate` (non-clustered) - for sorting by date
- `IX_Document_IsDeleted` (filtered, where IsDeleted = 0) - for active document queries

**Validation Rules**:
- `Title` must be 1-200 characters (enforced in service layer)
- `Description` must be 0-1000 characters
- `Category` must be one of predefined values (enum in code)
- `FileSize` must be ≤ 26,214,400 bytes (25 MB)
- `FileType` must match whitelisted MIME types
- `Tags` format: comma-separated, no validation on individual tag structure (service layer handles split/join)
- `FilePath` must be globally unique (prevents file overwrites)
- `IsDeleted` and `DeletedDate` correlation: if IsDeleted = 1, DeletedDate must be non-NULL

**Business Rules**:
- All queries MUST include `WHERE IsDeleted = 0` filter (global query filter in EF Core)
- Authorization: User can access if (UploadedByUserId = current user) OR (ProjectId in user's projects) OR (explicit DocumentShare exists)
- Deletion: Only owner can soft-delete; soft-deleted documents remain for 30 days before permanent removal
- File storage: Physical file deleted only when database record is permanently deleted

---

### DocumentShare

**Purpose**: Represents explicit sharing of a document with a specific user (beyond project-based access).

**Attributes**:

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `DocumentShareId` | `int` | PRIMARY KEY, IDENTITY | Unique share identifier |
| `DocumentId` | `int` | NOT NULL, FOREIGN KEY → Document.DocumentId | Document being shared |
| `SharedWithUserId` | `int` | NOT NULL, FOREIGN KEY → User.UserId | Recipient user |
| `SharedByUserId` | `int` | NOT NULL, FOREIGN KEY → User.UserId | User who initiated the share |
| `SharedDate` | `datetime2` | NOT NULL, DEFAULT GETUTCDATE() | UTC timestamp when share was created |

**Indexes**:
- `IX_DocumentShare_DocumentId_SharedWithUserId` (unique, composite) - prevents duplicate shares
- `IX_DocumentShare_SharedWithUserId` (non-clustered) - for "Shared with Me" queries

**Validation Rules**:
- `SharedWithUserId` must differ from `SharedByUserId` (can't share with yourself)
- `SharedByUserId` must be document owner (UploadedByUserId) - enforced in service layer
- Unique constraint on (DocumentId, SharedWithUserId) prevents duplicate share records

**Business Rules**:
- Sharing with a project team: Service layer iterates `ProjectMembers` and creates individual `DocumentShare` records (snapshot at share time)
- New project members added later do NOT automatically receive access (FR-051)
- Revoke share: Delete DocumentShare record; recipient immediately loses access
- Cascade: If Document is permanently deleted, all DocumentShare records are deleted (ON DELETE CASCADE)
- Soft-deleted documents: DocumentShare records remain but are inaccessible (global filter blocks document retrieval)

---

### DocumentTask (Join Table)

**Purpose**: Many-to-many relationship between documents and tasks; allows attaching documents to tasks for context.

**Attributes**:

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `DocumentId` | `int` | PRIMARY KEY (composite), FOREIGN KEY → Document.DocumentId | Document attached to task |
| `TaskId` | `int` | PRIMARY KEY (composite), FOREIGN KEY → TaskItem.TaskId | Task receiving the attachment |
| `AttachedDate` | `datetime2` | NOT NULL, DEFAULT GETUTCDATE() | UTC timestamp when document was attached to task |
| `AttachedByUserId` | `int` | NOT NULL, FOREIGN KEY → User.UserId | User who attached the document |

**Indexes**:
- Primary key (DocumentId, TaskId) provides clustering
- `IX_DocumentTask_TaskId` (non-clustered) - for querying task documents

**Validation Rules**:
- User must have access to both Document and Task to create attachment (service layer authorization)
- Task and Document's ProjectId should match (soft validation: warning if mismatch, but not prevented)

**Business Rules**:
- Attaching a document to a task automatically associates document with task's parent project (if document has no ProjectId)
- Deleting a Task permanently removes DocumentTask records (ON DELETE CASCADE)
- Soft-deleting a Document leaves DocumentTask records intact but document is inaccessible
- Permanently deleting a Document removes DocumentTask records (ON DELETE CASCADE)

---

## Migration Notes

### Creating Tables

**Migration Command**:
```bash
dotnet ef migrations add AddDocumentManagement
dotnet ef database update
```

**Migration Contents** (Entity Framework Core):
```csharp
public partial class AddDocumentManagement : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Document table
        migrationBuilder.CreateTable(
            name: "Documents",
            columns: table => new
            {
                DocumentId = table.Column<int>(nullable: false)
                    .Annotation("Sqlite:Autoincrement", true),
                Title = table.Column<string>(maxLength: 200, nullable: false),
                Description = table.Column<string>(maxLength: 1000, nullable: true),
                Category = table.Column<string>(maxLength: 50, nullable: false),
                FileName = table.Column<string>(maxLength: 255, nullable: false),
                FilePath = table.Column<string>(maxLength: 500, nullable: false),
                FileSize = table.Column<long>(nullable: false),
                FileType = table.Column<string>(maxLength: 255, nullable: false),
                Tags = table.Column<string>(maxLength: 500, nullable: true),
                UploadDate = table.Column<DateTime>(nullable: false, defaultValueSql: "datetime('now')"),
                UploadedByUserId = table.Column<int>(nullable: false),
                ProjectId = table.Column<int>(nullable: true),
                IsDeleted = table.Column<bool>(nullable: false, defaultValue: false),
                DeletedDate = table.Column<DateTime>(nullable: true)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Documents", x => x.DocumentId);
                table.ForeignKey(
                    name: "FK_Documents_Users_UploadedByUserId",
                    column: x => x.UploadedByUserId,
                    principalTable: "Users",
                    principalColumn: "UserId",
                    onDelete: ReferentialAction.Restrict);
                table.ForeignKey(
                    name: "FK_Documents_Projects_ProjectId",
                    column: x => x.ProjectId,
                    principalTable: "Projects",
                    principalColumn: "ProjectId",
                    onDelete: ReferentialAction.SetNull);
            });

        // Unique constraint on FilePath
        migrationBuilder.CreateIndex(
            name: "IX_Documents_FilePath",
            table: "Documents",
            column: "FilePath",
            unique: true);

        // Indexes for performance
        migrationBuilder.CreateIndex(
            name: "IX_Documents_UploadedByUserId",
            table: "Documents",
            column: "UploadedByUserId");

        migrationBuilder.CreateIndex(
            name: "IX_Documents_ProjectId",
            table: "Documents",
            column: "ProjectId");

        migrationBuilder.CreateIndex(
            name: "IX_Documents_UploadDate",
            table: "Documents",
            column: "UploadDate");

        // DocumentShare table
        migrationBuilder.CreateTable(
            name: "DocumentShares",
            columns: table => new
            {
                DocumentShareId = table.Column<int>(nullable: false)
                    .Annotation("Sqlite:Autoincrement", true),
                DocumentId = table.Column<int>(nullable: false),
                SharedWithUserId = table.Column<int>(nullable: false),
                SharedByUserId = table.Column<int>(nullable: false),
                SharedDate = table.Column<DateTime>(nullable: false, defaultValueSql: "datetime('now')")
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_DocumentShares", x => x.DocumentShareId);
                table.ForeignKey(
                    name: "FK_DocumentShares_Documents_DocumentId",
                    column: x => x.DocumentId,
                    principalTable: "Documents",
                    principalColumn: "DocumentId",
                    onDelete: ReferentialAction.Cascade);
                table.ForeignKey(
                    name: "FK_DocumentShares_Users_SharedWithUserId",
                    column: x => x.SharedWithUserId,
                    principalTable: "Users",
                    principalColumn: "UserId",
                    onDelete: ReferentialAction.Cascade);
                table.ForeignKey(
                    name: "FK_DocumentShares_Users_SharedByUserId",
                    column: x => x.SharedByUserId,
                    principalTable: "Users",
                    principalColumn: "UserId",
                    onDelete: ReferentialAction.Restrict);
            });

        // Unique index on (DocumentId, SharedWithUserId)
        migrationBuilder.CreateIndex(
            name: "IX_DocumentShares_DocumentId_SharedWithUserId",
            table: "DocumentShares",
            columns: new[] { "DocumentId", "SharedWithUserId" },
            unique: true);

        migrationBuilder.CreateIndex(
            name: "IX_DocumentShares_SharedWithUserId",
            table: "DocumentShares",
            column: "SharedWithUserId");

        // DocumentTask join table
        migrationBuilder.CreateTable(
            name: "DocumentTasks",
            columns: table => new
            {
                DocumentId = table.Column<int>(nullable: false),
                TaskId = table.Column<int>(nullable: false),
                AttachedDate = table.Column<DateTime>(nullable: false, defaultValueSql: "datetime('now')"),
                AttachedByUserId = table.Column<int>(nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_DocumentTasks", x => new { x.DocumentId, x.TaskId });
                table.ForeignKey(
                    name: "FK_DocumentTasks_Documents_DocumentId",
                    column: x => x.DocumentId,
                    principalTable: "Documents",
                    principalColumn: "DocumentId",
                    onDelete: ReferentialAction.Cascade);
                table.ForeignKey(
                    name: "FK_DocumentTasks_TaskItems_TaskId",
                    column: x => x.TaskId,
                    principalTable: "TaskItems",
                    principalColumn: "TaskId",
                    onDelete: ReferentialAction.Cascade);
                table.ForeignKey(
                    name: "FK_DocumentTasks_Users_AttachedByUserId",
                    column: x => x.AttachedByUserId,
                    principalTable: "Users",
                    principalColumn: "UserId",
                    onDelete: ReferentialAction.Restrict);
            });

        migrationBuilder.CreateIndex(
            name: "IX_DocumentTasks_TaskId",
            table: "DocumentTasks",
            column: "TaskId");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "DocumentTasks");
        migrationBuilder.DropTable(name: "DocumentShares");
        migrationBuilder.DropTable(name: "Documents");
    }
}
```

### ApplicationDbContext Changes

**Required Updates**:
```csharp
public class ApplicationDbContext : DbContext
{
    // Existing DbSets...
    
    public DbSet<Document> Documents { get; set; }
    public DbSet<DocumentShare> DocumentShares { get; set; }
    public DbSet<DocumentTask> DocumentTasks { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        // Existing configurations...
        
        // Document configuration
        modelBuilder.Entity<Document>(entity =>
        {
            entity.HasKey(d => d.DocumentId);
            entity.Property(d => d.Title).HasMaxLength(200).IsRequired();
            entity.Property(d => d.Description).HasMaxLength(1000);
            entity.Property(d => d.Category).HasMaxLength(50).IsRequired();
            entity.Property(d => d.FileName).HasMaxLength(255).IsRequired();
            entity.Property(d => d.FilePath).HasMaxLength(500).IsRequired();
            entity.Property(d => d.FileType).HasMaxLength(255).IsRequired();
            entity.Property(d => d.Tags).HasMaxLength(500);
            entity.Property(d => d.IsDeleted).HasDefaultValue(false);
            
            entity.HasIndex(d => d.FilePath).IsUnique();
            entity.HasIndex(d => d.UploadedByUserId);
            entity.HasIndex(d => d.ProjectId);
            entity.HasIndex(d => d.UploadDate);
            
            entity.HasOne(d => d.UploadedBy)
                .WithMany()
                .HasForeignKey(d => d.UploadedByUserId)
                .OnDelete(DeleteBehavior.Restrict);
            
            entity.HasOne(d => d.Project)
                .WithMany()
                .HasForeignKey(d => d.ProjectId)
                .OnDelete(DeleteBehavior.SetNull);
            
            // Global query filter to exclude soft-deleted documents
            entity.HasQueryFilter(d => !d.IsDeleted);
        });
        
        // DocumentShare configuration
        modelBuilder.Entity<DocumentShare>(entity =>
        {
            entity.HasKey(ds => ds.DocumentShareId);
            
            entity.HasIndex(ds => new { ds.DocumentId, ds.SharedWithUserId }).IsUnique();
            entity.HasIndex(ds => ds.SharedWithUserId);
            
            entity.HasOne(ds => ds.Document)
                .WithMany(d => d.DocumentShares)
                .HasForeignKey(ds => ds.DocumentId)
                .OnDelete(DeleteBehavior.Cascade);
            
            entity.HasOne(ds => ds.SharedWith)
                .WithMany()
                .HasForeignKey(ds => ds.SharedWithUserId)
                .OnDelete(DeleteBehavior.Cascade);
            
            entity.HasOne(ds => ds.SharedBy)
                .WithMany()
                .HasForeignKey(ds => ds.SharedByUserId)
                .OnDelete(DeleteBehavior.Restrict);
        });
        
        // DocumentTask configuration
        modelBuilder.Entity<DocumentTask>(entity =>
        {
            entity.HasKey(dt => new { dt.DocumentId, dt.TaskId });
            
            entity.HasIndex(dt => dt.TaskId);
            
            entity.HasOne(dt => dt.Document)
                .WithMany(d => d.DocumentTasks)
                .HasForeignKey(dt => dt.DocumentId)
                .OnDelete(DeleteBehavior.Cascade);
            
            entity.HasOne(dt => dt.Task)
                .WithMany(t => t.DocumentTasks)
                .HasForeignKey(dt => dt.TaskId)
                .OnDelete(DeleteBehavior.Cascade);
            
            entity.HasOne(dt => dt.AttachedBy)
                .WithMany()
                .HasForeignKey(dt => dt.AttachedByUserId)
                .OnDelete(DeleteBehavior.Restrict);
        });
    }
}
```

---

## Seed Data (Optional, for Testing)

```csharp
// In ApplicationDbContext.OnModelCreating or separate seeding class
if (env.IsDevelopment())
{
    // Seed sample documents for each predefined user
    modelBuilder.Entity<Document>().HasData(
        new Document
        {
            DocumentId = 1,
            Title = "Project Plan 2026",
            Description = "Annual project planning document",
            Category = "Project Documents",
            FileName = "project_plan_2026.pdf",
            FilePath = "1/1/guid-abc123.pdf",
            FileSize = 524288, // 512 KB
            FileType = "application/pdf",
            Tags = "planning,2026,strategy",
            UploadDate = DateTime.UtcNow.AddDays(-10),
            UploadedByUserId = 1, // John Smith
            ProjectId = 1,
            IsDeleted = false
        },
        new Document
        {
            DocumentId = 2,
            Title = "Team Photo",
            Description = "Team building event photo",
            Category = "Team Resources",
            FileName = "team_photo.jpg",
            FilePath = "2/personal/guid-def456.jpg",
            FileSize = 1048576, // 1 MB
            FileType = "image/jpeg",
            Tags = "team,events,2026",
            UploadDate = DateTime.UtcNow.AddDays(-5),
            UploadedByUserId = 2, // Sarah Johnson
            ProjectId = null,
            IsDeleted = false
        }
    );
}
```

---

## Query Patterns

### User's Documents (My Documents)
```csharp
var myDocuments = await _context.Documents
    .Where(d => d.UploadedByUserId == currentUserId)
    .OrderByDescending(d => d.UploadDate)
    .ToListAsync();
// Note: IsDeleted filter applied automatically via global query filter
```

### Project Documents
```csharp
var projectDocuments = await _context.Documents
    .Where(d => d.ProjectId == projectId)
    .Include(d => d.UploadedBy)
    .OrderByDescending(d => d.UploadDate)
    .ToListAsync();
```

### Shared with Me
```csharp
var sharedDocuments = await _context.DocumentShares
    .Where(ds => ds.SharedWithUserId == currentUserId)
    .Include(ds => ds.Document)
    .ThenInclude(d => d.UploadedBy)
    .Select(ds => ds.Document)
    .OrderByDescending(d => d.UploadDate)
    .ToListAsync();
```

### Search Documents (with Authorization)
```csharp
var searchResults = await _context.Documents
    .Where(d => 
        (d.Title.Contains(searchTerm) || d.Description.Contains(searchTerm) || d.Tags.Contains(searchTerm)) &&
        (d.UploadedByUserId == currentUserId || 
         d.Project.ProjectMembers.Any(pm => pm.UserId == currentUserId) ||
         d.DocumentShares.Any(ds => ds.SharedWithUserId == currentUserId)))
    .Include(d => d.UploadedBy)
    .OrderByDescending(d => d.UploadDate)
    .ToListAsync();
```

### Soft-Deleted Documents (Admin View)
```csharp
var deletedDocuments = await _context.Documents
    .IgnoreQueryFilters() // Bypass global filter
    .Where(d => d.IsDeleted && d.DeletedDate <= DateTime.UtcNow.AddDays(-30))
    .ToListAsync();
```

---

## Performance Considerations

- **Indexes**: All foreign keys and frequently queried columns have indexes
- **Global Filter**: `IsDeleted = 0` filter applied automatically prevents fetching deleted records
- **Pagination**: Implement for document lists (`.Skip(pageSize * pageIndex).Take(pageSize)`)
- **Selective Loading**: Use `.Select()` to load only needed columns for list views (avoid loading Description, Tags if not displayed)
- **Caching**: Consider caching user project memberships to avoid repeated authorization queries
- **File Operations**: Async I/O for all file system operations to prevent blocking

---

## Security Notes

- **IDOR Protection**: All queries include authorization checks (owner, project member, or explicit share)
- **Path Validation**: `FilePath` validated to prevent directory traversal (Path.GetFullPath check)
- **File Storage**: Files stored outside `wwwroot` to prevent direct web access
- **Soft Delete**: Cascade rules prevent orphaned shares; physical file deletion deferred until permanent removal
- **MIME Type Validation**: `FileType` validated against whitelist before storage
