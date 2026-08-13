# IFileStorageService Contract

**Purpose**: Interface abstraction for file storage operations, enabling seamless migration from local filesystem (training) to Azure Blob Storage (production).

**Namespace**: `ContosoDashboard.Services`

**Lifecycle**: Scoped (per HTTP request) - registered in `Program.cs` as:
- Training: `builder.Services.AddScoped<IFileStorageService, LocalFileStorageService>()`
- Production: `builder.Services.AddScoped<IFileStorageService, AzureBlobStorageService>()`

---

## Interface Definition

```csharp
public interface IFileStorageService
{
    /// <summary>
    /// Upload a file and return the storage path identifier.
    /// </summary>
    /// <param name="fileStream">Stream containing file content</param>
    /// <param name="fileName">Original filename with extension</param>
    /// <param name="contentType">MIME type of the file</param>
    /// <returns>Relative file path or blob identifier (e.g., "1/personal/guid.pdf")</returns>
    Task<string> UploadAsync(Stream fileStream, string fileName, string contentType);
    
    /// <summary>
    /// Download a file and return a stream.
    /// </summary>
    /// <param name="filePath">File path or blob identifier returned by UploadAsync</param>
    /// <returns>Stream containing file content</returns>
    Task<Stream> DownloadAsync(string filePath);
    
    /// <summary>
    /// Delete a file permanently.
    /// </summary>
    /// <param name="filePath">File path or blob identifier to delete</param>
    Task DeleteAsync(string filePath);
    
    /// <summary>
    /// Check if a file exists at the specified path.
    /// </summary>
    /// <param name="filePath">File path or blob identifier to check</param>
    /// <returns>True if file exists, false otherwise</returns>
    Task<bool> ExistsAsync(string filePath);
    
    /// <summary>
    /// Generate a temporary access URL with expiration (for future use with Azure Blob SAS tokens).
    /// </summary>
    /// <param name="filePath">File path or blob identifier</param>
    /// <param name="expiration">How long the URL should remain valid</param>
    /// <returns>Temporary URL string (local implementation returns file path; Azure returns SAS URL)</returns>
    Task<string> GetTemporaryUrlAsync(string filePath, TimeSpan expiration);
}
```

---

## Method Specifications

### UploadAsync

**Purpose**: Store a file and return its storage identifier.

**Parameters**:
- `fileStream`: Stream containing file content (caller responsible for opening/closing)
- `fileName`: Original filename from user's device (e.g., "Budget_2026.pdf")
- `contentType`: MIME type (e.g., "application/pdf")

**Returns**: Relative file path or blob identifier (e.g., `"1/personal/abc123-def456.pdf"`)

**Process** (LocalFileStorageService):
1. Extract file extension from `fileName`
2. Generate unique storage path: `{userId}/{projectId-or-personal}/{Guid.NewGuid()}.{extension}`
3. Ensure directory exists: `Directory.CreateDirectory(Path.GetDirectoryName(fullPath))`
4. Copy stream to file: `await using var fileStream = File.Create(fullPath); await fileStream.CopyToAsync(fileStream);`
5. Return relative path (without base directory)

**Process** (AzureBlobStorageService):
1. Generate blob name: `{userId}/{projectId-or-personal}/{Guid.NewGuid()}.{extension}`
2. Get container client: `var containerClient = _blobServiceClient.GetBlobContainerClient("documents");`
3. Upload blob: `await containerClient.UploadBlobAsync(blobName, fileStream, new BlobHttpHeaders { ContentType = contentType });`
4. Return blob name

**Exceptions**:
- `IOException`: Disk full, permission denied, or network error (Azure)
- `ArgumentException`: Invalid file path characters

**Thread Safety**: Safe for concurrent calls (each operation creates unique file path)

---

### DownloadAsync

**Purpose**: Retrieve file content as a stream.

**Parameters**:
- `filePath`: Path returned by `UploadAsync` (e.g., `"1/personal/abc123.pdf"`)

**Returns**: `Stream` containing file content (caller responsible for disposing)

**Process** (LocalFileStorageService):
1. Combine base directory with relative path: `var fullPath = Path.Combine(_basePath, filePath);`
2. Validate path to prevent directory traversal: `if (!Path.GetFullPath(fullPath).StartsWith(_basePath)) throw SecurityException;`
3. Check file exists: `if (!File.Exists(fullPath)) throw FileNotFoundException;`
4. Open read stream: `return File.OpenRead(fullPath);`

**Process** (AzureBlobStorageService):
1. Get blob client: `var blobClient = _containerClient.GetBlobClient(filePath);`
2. Download stream: `var response = await blobClient.DownloadAsync(); return response.Value.Content;`

**Exceptions**:
- `FileNotFoundException` / `RequestFailedException`: File/blob doesn't exist
- `SecurityException`: Path traversal attempt detected
- `IOException`: Read error

**Note**: Caller MUST dispose the returned stream after use

---

### DeleteAsync

**Purpose**: Permanently remove a file from storage.

**Parameters**:
- `filePath`: Path returned by `UploadAsync`

**Returns**: `Task` (void)

**Process** (LocalFileStorageService):
1. Combine base directory with relative path
2. Validate path to prevent directory traversal
3. Check file exists (optional: succeed silently if not exists for idempotency)
4. Delete file: `File.Delete(fullPath);`

**Process** (AzureBlobStorageService):
1. Get blob client
2. Delete blob: `await blobClient.DeleteIfExistsAsync();`

**Exceptions**:
- `IOException`: File locked or permission denied
- `SecurityException`: Path traversal attempt

**Idempotency**: Succeeds silently if file doesn't exist (no exception thrown)

---

### ExistsAsync

**Purpose**: Check if a file exists without downloading it.

**Parameters**:
- `filePath`: Path to check

**Returns**: `bool` - true if file exists, false otherwise

**Process** (LocalFileStorageService):
1. Combine base directory with relative path
2. Validate path to prevent directory traversal
3. Return `File.Exists(fullPath)`

**Process** (AzureBlobStorageService):
1. Get blob client
2. Return `await blobClient.ExistsAsync()`

**Exceptions**: None (returns false on errors for safety)

---

### GetTemporaryUrlAsync

**Purpose**: Generate a time-limited URL for direct file access (primarily for Azure Blob SAS tokens; local implementation placeholder).

**Parameters**:
- `filePath`: Path to generate URL for
- `expiration`: How long URL should remain valid (e.g., `TimeSpan.FromMinutes(15)`)

**Returns**: `string` - URL for temporary access

**Process** (LocalFileStorageService):
1. Return the file path as-is (placeholder implementation)
2. Note: In training, files are served through authenticated download endpoint, not direct URLs

**Process** (AzureBlobStorageService):
1. Get blob client
2. Generate SAS token: `var sasToken = blobClient.GenerateSasUri(BlobSasPermissions.Read, DateTime.UtcNow.Add(expiration));`
3. Return SAS URL as string

**Use Case**: Future enhancement for direct browser access to files (e.g., preview PDFs directly from Azure Blob without proxying through app server)

**Note**: Local implementation doesn't support true temporary URLs; authorization remains at download endpoint

---

## LocalFileStorageService Implementation

**Configuration** (`appsettings.json`):
```json
{
  "FileStorage": {
    "BasePath": "C:\\Spec_Driven\\Pilot\\ContosoDashboard\\uploads",
    "BasePathRelative": "./uploads"
  }
}
```

**Constructor**:
```csharp
public class LocalFileStorageService : IFileStorageService
{
    private readonly string _basePath;
    private readonly ILogger<LocalFileStorageService> _logger;
    
    public LocalFileStorageService(IConfiguration configuration, ILogger<LocalFileStorageService> logger)
    {
        _basePath = configuration["FileStorage:BasePath"] 
            ?? Path.Combine(Directory.GetCurrentDirectory(), "uploads");
        _logger = logger;
        
        // Ensure base directory exists
        if (!Directory.Exists(_basePath))
        {
            Directory.CreateDirectory(_basePath);
            _logger.LogInformation("Created file storage directory: {BasePath}", _basePath);
        }
    }
    
    // ... implement interface methods
}
```

**Directory Structure**:
```
uploads/
├── 1/                      # User ID 1
│   ├── personal/           # Personal documents
│   │   ├── abc123-def456.pdf
│   │   └── xyz789-ghi012.docx
│   └── 3/                  # Project ID 3
│       ├── jkl345-mno678.xlsx
│       └── pqr901-stu234.jpg
└── 2/                      # User ID 2
    ├── personal/
    │   └── vwx567-yza890.png
    └── 5/                  # Project ID 5
        └── bcd123-efg456.pptx
```

**Security**:
- Files stored outside `wwwroot` to prevent direct web access
- Path validation prevents directory traversal attacks
- File permissions set to read/write for application user only
- Add `uploads/` to `.gitignore` to prevent committing user files

---

## AzureBlobStorageService Implementation (Production)

**Configuration** (`appsettings.json`):
```json
{
  "AzureBlobStorage": {
    "ConnectionString": "DefaultEndpointsProtocol=https;AccountName=...",
    "ContainerName": "documents"
  }
}
```

**Constructor**:
```csharp
public class AzureBlobStorageService : IFileStorageService
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly BlobContainerClient _containerClient;
    private readonly ILogger<AzureBlobStorageService> _logger;
    
    public AzureBlobStorageService(IConfiguration configuration, ILogger<AzureBlobStorageService> logger)
    {
        var connectionString = configuration["AzureBlobStorage:ConnectionString"];
        var containerName = configuration["AzureBlobStorage:ContainerName"] ?? "documents";
        
        _blobServiceClient = new BlobServiceClient(connectionString);
        _containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        _logger = logger;
        
        // Ensure container exists
        _containerClient.CreateIfNotExists(PublicAccessType.None);
    }
    
    // ... implement interface methods
}
```

**Container Structure**: Same logical structure as local (user ID / project ID / filename)

**Security**:
- Container has `PublicAccessType.None` (no anonymous access)
- SAS tokens used for time-limited direct access
- Azure AD authentication for app service → blob storage
- CORS configured for preview scenarios (if needed)

---

## Migration Path

### Step 1: Training (Local Storage)
```csharp
// Program.cs
builder.Services.AddScoped<IFileStorageService, LocalFileStorageService>();
```

### Step 2: Production (Azure Blob)
```csharp
// Program.cs
if (builder.Environment.IsProduction())
{
    builder.Services.AddScoped<IFileStorageService, AzureBlobStorageService>();
}
else
{
    builder.Services.AddScoped<IFileStorageService, LocalFileStorageService>();
}
```

### Step 3: Data Migration (one-time)
```csharp
// Migration script to copy files from local to Azure
public async Task MigrateLocalFilesToAzure()
{
    var localService = new LocalFileStorageService(localConfig, logger);
    var azureService = new AzureBlobStorageService(azureConfig, logger);
    
    var documents = await _context.Documents.ToListAsync();
    
    foreach (var doc in documents)
    {
        using var stream = await localService.DownloadAsync(doc.FilePath);
        var newPath = await azureService.UploadAsync(stream, doc.FileName, doc.FileType);
        
        // FilePath stays the same (logical path is consistent)
        // Only implementation changes
    }
}
```

**Key Design**: File paths are implementation-agnostic (e.g., `"1/personal/guid.pdf"`), so database records don't need updates when migrating storage backends.

---

## Testing Considerations

### Unit Tests
```csharp
[Fact]
public async Task UploadAsync_CreatesUniqueFilePath()
{
    var service = new LocalFileStorageService(config, logger);
    using var stream = new MemoryStream(Encoding.UTF8.GetBytes("test content"));
    
    var path1 = await service.UploadAsync(stream, "test.txt", "text/plain");
    stream.Position = 0;
    var path2 = await service.UploadAsync(stream, "test.txt", "text/plain");
    
    Assert.NotEqual(path1, path2); // Paths must be unique
}

[Fact]
public async Task DownloadAsync_ThrowsSecurityException_OnPathTraversal()
{
    var service = new LocalFileStorageService(config, logger);
    
    await Assert.ThrowsAsync<SecurityException>(() => 
        service.DownloadAsync("../../etc/passwd"));
}
```

### Integration Tests
- Mock `IFileStorageService` in service layer tests
- Use in-memory test storage for integration tests (no real filesystem/Azure)
- Verify file cleanup on transaction rollback

---

## Performance Considerations

- **Async I/O**: All operations are async to prevent thread blocking
- **Streaming**: Files are streamed, not loaded into memory (supports large files)
- **Concurrency**: Each upload generates unique path (no locking needed)
- **Caching**: No caching at storage layer (caching handled by CDN or app layer if needed)
- **Partitioning**: Files partitioned by user ID and project ID for better filesystem/blob performance

---

## Conclusion

`IFileStorageService` provides a clean abstraction that:
- Enables offline training with local filesystem
- Supports seamless cloud migration to Azure Blob Storage
- Follows Constitution Principle III (Infrastructure Abstraction)
- Maintains security through path validation and authorization
- Scales from single-user training to multi-tenant production
