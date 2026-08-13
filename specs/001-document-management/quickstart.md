# Quickstart: Document Management Feature Validation

**Feature**: Document Upload and Management  
**Date**: 2026-08-13  
**Purpose**: Runnable validation scenarios to verify feature implementation end-to-end

---

## Prerequisites

### Environment Setup
- ✅ .NET 10.0 SDK installed (`dotnet --version` shows 10.0.x)
- ✅ Visual Studio Code or Visual Studio 2022
- ✅ SQLite installed (bundled with EF Core packages)
- ✅ ContosoDashboard running successfully on .NET 10.0 with SQLite
- ✅ Predefined users available: John Smith (ID=1), Sarah Johnson (ID=2), Mike Chen (ID=3), Emily Rodriguez (ID=4)

### Database Setup
```powershell
# From project root: C:\Spec_Driven\Pilot\ContosoDashboard
cd ContosoDashboard

# Create and apply migration for document tables
dotnet ef migrations add AddDocumentManagement
dotnet ef database update

# Verify tables created
sqlite3 ContosoDashboard.db ".tables"
# Expected output includes: Documents, DocumentShares, DocumentTasks
```

### File Storage Setup
```powershell
# Ensure uploads directory exists (created automatically by LocalFileStorageService on first run)
# Add to .gitignore if not already present
if (-not (Test-Path ".gitignore")) { New-Item -ItemType File -Path ".gitignore" }
if (-not (Get-Content ".gitignore" | Select-String "uploads/")) {
    Add-Content -Path ".gitignore" -Value "`nuploads/"
}
```

### Build and Run
```powershell
# Build solution
dotnet build

# Run application
dotnet run

# Expected output:
# info: Microsoft.Hosting.Lifetime[14]
#       Now listening on: http://localhost:5000
# info: Microsoft.Hosting.Lifetime[0]
#       Application started. Press Ctrl+C to shut down.
```

---

## Validation Scenarios

### Scenario 1: Upload Personal Document (P1 - Core MVP)

**Objective**: Verify users can upload documents to personal library with metadata.

**Steps**:
1. Open browser: `http://localhost:5000`
2. Log in as John Smith (Email: `john.smith@contoso.com`, any password)
3. Navigate to "Documents" page from navigation menu
4. Click "Upload Document" button
5. Fill upload form:
   - **Select File**: Choose a PDF file under 25 MB (e.g., sample resume, report)
   - **Title**: "My Personal Resume"
   - **Description**: "Updated resume for 2026"
   - **Category**: Select "Personal Files"
   - **Tags**: "resume, personal, 2026"
   - **Associated Project**: Leave blank (personal document)
6. Click "Upload" button

**Expected Outcomes**:
- ✅ Upload completes within 30 seconds
- ✅ Success message displayed: "Document uploaded successfully"
- ✅ Document appears in "My Documents" list with:
  - Title: "My Personal Resume"
  - Category: "Personal Files"
  - Upload Date: Today's date
  - File Size: Displayed in KB/MB
  - Uploader: John Smith
- ✅ Download link is functional (clicking downloads the PDF)
- ✅ Physical file exists at `uploads/1/personal/{guid}.pdf`
- ✅ Database record created: `sqlite3 ContosoDashboard.db "SELECT * FROM Documents WHERE Title='My Personal Resume';"` returns 1 row

**Performance Validation**:
- Upload time < 30 seconds for files up to 25 MB (FR-041)
- Document list page loads < 2 seconds (FR-042)

---

### Scenario 2: Associate Document with Project (P1 - Core MVP)

**Objective**: Verify project members can upload documents to projects and all members can access them.

**Steps**:
1. Log in as John Smith (project member)
2. Navigate to "Documents" → "Upload Document"
3. Fill upload form:
   - **Select File**: Choose a Word document (e.g., project plan)
   - **Title**: "Q1 Project Plan"
   - **Description**: "Project planning document for Q1 2026"
   - **Category**: "Project Documents"
   - **Tags**: "planning, q1, 2026"
   - **Associated Project**: Select "Website Redesign" (or another active project)
4. Click "Upload"
5. Navigate to "Projects" → "Website Redesign" (project detail page)
6. Scroll to "Documents" section
7. Log out and log in as Sarah Johnson (different project member)
8. Navigate to "Projects" → "Website Redesign" → "Documents" section

**Expected Outcomes**:
- ✅ Document uploads successfully with project association
- ✅ Document appears in project's documents section with correct metadata
- ✅ Other project members (Sarah Johnson) can see the document in project detail page
- ✅ Sarah Johnson can download the document (authorization check passes)
- ✅ Physical file stored at `uploads/1/3/{guid}.docx` (assuming project ID = 3)
- ✅ Database: `ProjectId` field populated with correct project ID

**Authorization Validation**:
- Log in as Mike Chen (NOT a member of Website Redesign project)
- Try navigating directly to project detail page
- ✅ Mike Chen cannot see the document (filtered by authorization)

---

### Scenario 3: Search Documents by Keyword (P2 - Enhanced Usability)

**Objective**: Verify users can search documents by title, description, tags, and results respect authorization.

**Steps**:
1. Upload 5 documents with varying titles, tags, and project associations:
   - "Budget Report 2026" (Personal, tags: "budget, finance")
   - "Marketing Strategy" (Project A, tags: "marketing, strategy")
   - "Team Meeting Notes" (Personal, tags: "meetings, notes")
   - "Q1 Budget Review" (Project B, tags: "budget, q1")
   - "Design Mockups" (Project A, tags: "design, ui")
2. Log in as John Smith (member of Project A, owner of personal docs)
3. Navigate to "Documents" page
4. Enter search term: "budget"
5. Click "Search" button

**Expected Outcomes**:
- ✅ Search returns within 2 seconds (FR-043)
- ✅ Results include:
  - "Budget Report 2026" (owner can see own documents)
  - "Q1 Budget Review" (if John is member of Project B, otherwise excluded)
- ✅ Results do NOT include documents John lacks permission to access
- ✅ Results are ordered by upload date (newest first)
- ✅ Clicking a result opens document details or download

**Performance Validation**:
- Search completes < 2 seconds for 500 documents (FR-022, FR-043)

---

### Scenario 4: Share Document with Specific User (P2 - Collaboration)

**Objective**: Verify document owners can share documents with colleagues and recipients receive notifications.

**Steps**:
1. Log in as John Smith
2. Upload a personal document: "Training Materials"
3. Click "Share" button on the document
4. Select "Share with Users"
5. Choose "Sarah Johnson" from user dropdown
6. Click "Share"
7. Log out and log in as Sarah Johnson
8. Check notifications (bell icon in header)
9. Navigate to "Documents" → "Shared with Me"

**Expected Outcomes**:
- ✅ John Smith successfully shares document
- ✅ Sarah Johnson receives in-app notification: "John Smith shared 'Training Materials' with you"
- ✅ Document appears in Sarah's "Shared with Me" section
- ✅ Sarah can download the document
- ✅ Sarah cannot edit or delete the document (read-only access)
- ✅ Database: `DocumentShares` table has record linking Sarah to the document

**Revoke Share**:
1. Log in as John Smith
2. Navigate to document details
3. Click "Manage Sharing"
4. Remove Sarah Johnson
5. Log in as Sarah Johnson
6. Navigate to "Shared with Me"

**Expected After Revoke**:
- ✅ Document no longer appears in Sarah's "Shared with Me"
- ✅ Sarah cannot download the document (authorization fails)

---

### Scenario 5: Share Document with Project Team (P2 - Team Collaboration)

**Objective**: Verify sharing with project creates snapshot of team membership (FR-051).

**Steps**:
1. Log in as John Smith (project member and document owner)
2. Upload a document associated with "Website Redesign" project
3. Click "Share" → "Share with Project Team" → Select "Website Redesign"
4. Verify all current project members receive access:
   - Sarah Johnson (project member)
   - Mike Chen (project member)
5. Add a new member to the project (Emily Rodriguez)
6. Log in as Emily Rodriguez
7. Navigate to "Shared with Me"

**Expected Outcomes**:
- ✅ Sarah and Mike receive shares and notifications
- ✅ Both can access document in "Shared with Me"
- ✅ Emily Rodriguez does NOT automatically receive access (snapshot approach, FR-051)
- ✅ Database: `DocumentShares` table has records for Sarah and Mike only (not Emily)

**To Grant Emily Access**:
1. John Smith must re-share explicitly or share with her individually

---

### Scenario 6: Attach Document to Task (Task Integration)

**Objective**: Verify documents can be attached to tasks and appear on task detail page.

**Steps**:
1. Log in as John Smith
2. Navigate to "Tasks" page
3. Select an existing task: "Design homepage mockup"
4. Scroll to "Attached Documents" section
5. Click "Attach Document"
6. Select "Design Mockups" from document list
7. Click "Attach"
8. Verify document appears in task's attached documents section
9. Click document link to download

**Expected Outcomes**:
- ✅ Document successfully attached to task
- ✅ Document appears in task detail page under "Attached Documents"
- ✅ Document's `ProjectId` is automatically set to task's parent project (if not already set)
- ✅ Database: `DocumentTasks` join table has record linking document to task
- ✅ Other task assignees can see and download the attached document

---

### Scenario 7: Edit Document Metadata (P3 - Quality of Life)

**Objective**: Verify document owners can update metadata without re-uploading file.

**Steps**:
1. Log in as John Smith (document owner)
2. Navigate to "My Documents"
3. Select document: "My Personal Resume"
4. Click "Edit" button
5. Update fields:
   - **Title**: "My Updated Resume 2026"
   - **Category**: "Reports"
   - **Tags**: "resume, job search, 2026, updated"
6. Click "Save Changes"
7. Verify changes appear in document list

**Expected Outcomes**:
- ✅ Metadata updates successfully
- ✅ Document title reflects new value: "My Updated Resume 2026"
- ✅ Category updated to "Reports"
- ✅ Tags updated and searchable
- ✅ File content remains unchanged (same physical file)
- ✅ Upload date and original filename unchanged

**Authorization Validation**:
- Log in as Sarah Johnson (non-owner)
- Navigate to a shared document from John
- ✅ "Edit" button is disabled or hidden (only owner can edit)

---

### Scenario 8: Replace Document File (P3 - Version Update)

**Objective**: Verify document owners can upload a replacement file while preserving metadata and shares.

**Steps**:
1. Log in as John Smith
2. Upload a document: "Project Budget v1.xlsx"
3. Share document with Sarah Johnson
4. Click "Edit" → "Replace File"
5. Select new file: "Project Budget v2.xlsx"
6. Click "Upload Replacement"
7. Log in as Sarah Johnson
8. Navigate to "Shared with Me"
9. Download the document

**Expected Outcomes**:
- ✅ Old file deleted from storage: `uploads/1/personal/{old-guid}.xlsx` removed
- ✅ New file stored: `uploads/1/personal/{new-guid}.xlsx` created
- ✅ Database: `FilePath`, `FileSize`, `FileType` updated to new file
- ✅ Metadata preserved: Title, Description, Category, Tags unchanged
- ✅ Sharing preserved: Sarah Johnson still has access
- ✅ Sarah downloads new version (v2) when clicking download link

---

### Scenario 9: Preview PDF and Image Documents (P3 - Convenience)

**Objective**: Verify in-browser preview for supported file types (PDF, images).

**Steps**:
1. Log in as John Smith
2. Upload a PDF document: "Report.pdf"
3. Upload an image: "Chart.png"
4. Upload a Word document: "Notes.docx"
5. Click "Preview" button on each document

**Expected Outcomes**:

**PDF Preview**:
- ✅ Modal opens with embedded PDF viewer
- ✅ PDF renders within 3 seconds (FR-040)
- ✅ User can scroll through pages in preview
- ✅ "Download" button available in modal
- ✅ "Close" button or Escape key closes modal

**Image Preview**:
- ✅ Modal opens with full image displayed
- ✅ Image renders within 3 seconds
- ✅ Image scales to fit modal (responsive)
- ✅ Download and close options available

**Word Document** (not supported):
- ✅ "Preview" button disabled or shows message: "Preview not available for this file type"
- ✅ "Download" button still functional

**Keyboard Navigation**:
- ✅ Tab key navigates between Close and Download buttons
- ✅ Escape key closes modal
- ✅ Focus trapped within modal (doesn't tab outside)

---

### Scenario 10: Soft Delete with 30-Day Retention (Deletion & Audit)

**Objective**: Verify soft delete makes documents inaccessible immediately but retains for 30 days.

**Steps**:
1. Log in as John Smith
2. Upload a document: "Old Report.pdf"
3. Share with Sarah Johnson
4. Click "Delete" button
5. Confirm deletion in prompt
6. Attempt to access document directly (refresh page or navigate away and back)
7. Log in as Sarah Johnson
8. Navigate to "Shared with Me"

**Expected Outcomes**:
- ✅ Confirmation prompt appears before deletion
- ✅ Document disappears from "My Documents" list immediately
- ✅ Document no longer appears in search results
- ✅ Direct URL access returns 404 Not Found
- ✅ Sarah Johnson sees message: "This document is no longer available"
- ✅ Database: `IsDeleted = true`, `DeletedDate` set to current timestamp
- ✅ Physical file still exists on disk (not deleted yet)

**Admin Audit View** (FR-050):
1. Log in as Administrator user
2. Navigate to "Admin" → "Deleted Documents"
3. Verify "Old Report.pdf" appears in list with deletion date

**Background Cleanup** (FR-049):
1. Manually run background job or wait 30 days (simulate with test):
   ```csharp
   // In DocumentCleanupService or manual test
   var cutoffDate = DateTime.UtcNow.AddDays(-30);
   var expiredDocs = await _context.Documents
       .IgnoreQueryFilters()
       .Where(d => d.IsDeleted && d.DeletedDate <= cutoffDate)
       .ToListAsync();
   ```
2. Verify physical file deleted after 30 days
3. Verify database record removed

---

### Scenario 11: Authorization and IDOR Protection (Security Validation)

**Objective**: Verify unauthorized users cannot access documents by manipulating IDs (IDOR attack prevention).

**Steps**:
1. Log in as John Smith
2. Upload a personal document (note the Document ID from URL or database)
3. Log out
4. Log in as Mike Chen (different user, not shared with)
5. Manually navigate to: `http://localhost:5000/api/documents/{documentId}/download`

**Expected Outcomes**:
- ✅ Request returns HTTP 403 Forbidden
- ✅ Error message: "You do not have permission to access this document"
- ✅ File content NOT served (authorization check prevents access)
- ✅ No information disclosure (error message doesn't reveal document existence)

**Test Variations**:
- Try accessing shared document after share is revoked → 403 Forbidden
- Try accessing project document after being removed from project → 403 Forbidden
- Try editing document owned by another user → 403 Forbidden or button hidden

**Success Criteria** (SC-006): Zero unauthorized access during validation testing

---

### Scenario 12: Accessibility Validation (WCAG 2.1 Level AA)

**Objective**: Verify document management UI meets accessibility standards (FR-044 through FR-048).

**Steps**:

**Keyboard Navigation Test**:
1. Open "Documents" page
2. Use Tab key only (no mouse) to navigate entire page:
   - Tab through navigation menu
   - Tab to "Upload Document" button → Press Enter to activate
   - Tab through upload form fields
   - Tab to "Upload" button → Press Enter to submit
   - Tab through document list (if clickable rows)
   - Tab to "Preview" and "Download" buttons
3. Open preview modal
4. Use Escape key to close modal

**Expected Outcomes**:
- ✅ All interactive elements reachable via Tab key
- ✅ Focus indicators visible (outline or highlight)
- ✅ Tab order is logical (top-to-bottom, left-to-right)
- ✅ Enter key activates buttons
- ✅ Escape key closes modals
- ✅ Focus trapped in modal (doesn't tab outside)

**Screen Reader Test** (NVDA or VoiceOver):
1. Open "Documents" page with screen reader active
2. Navigate through page with arrow keys

**Expected Outcomes**:
- ✅ Page title announced: "Documents - ContosoDashboard"
- ✅ Form labels announced: "Title", "Select File", "Category"
- ✅ Button labels announced: "Upload Document", "Preview", "Download"
- ✅ ARIA labels present on icon-only buttons
- ✅ Error messages announced when validation fails
- ✅ Success messages announced when upload completes

**Color Contrast Test** (axe DevTools or WAVE):
1. Open "Documents" page
2. Run axe DevTools browser extension
3. Check for contrast violations

**Expected Outcomes**:
- ✅ No contrast violations (4.5:1 for normal text, 3:1 for large text)
- ✅ Bootstrap default colors meet WCAG AA standards
- ✅ Custom styles (if any) validated

**Success Criteria** (SC-011): 100% automated accessibility test pass rate

---

## Performance Benchmarks

| Operation | Target | How to Measure |
|-----------|--------|----------------|
| Upload 25 MB file | < 30 seconds | Browser dev tools → Network tab → upload time |
| Document list load (500 docs) | < 2 seconds | Page load time in Network tab |
| Search results | < 2 seconds | Time from Enter key to results displayed |
| Preview render (PDF/image) | < 3 seconds | Time from click to content visible |

**Tools**:
- Browser DevTools Network tab (F12 → Network)
- Lighthouse performance audit (F12 → Lighthouse)
- Manual stopwatch for user experience timing

---

## Rollback Plan

If critical issues found during validation:

1. **Revert Database Migration**:
   ```powershell
   dotnet ef database update [PreviousMigrationName]
   dotnet ef migrations remove
   ```

2. **Remove Uploaded Files**:
   ```powershell
   Remove-Item -Recurse -Force "uploads/"
   ```

3. **Revert Code Changes**: Git reset to previous commit before feature implementation

---

## Success Criteria Summary

After completing all validation scenarios, verify:

- ✅ SC-001: 70% user adoption (measure after 3 months in production)
- ✅ SC-002: Upload < 30 seconds for 25 MB files
- ✅ SC-003: Locate document < 30 seconds via search
- ✅ SC-004: Document list loads < 2 seconds for 500 documents
- ✅ SC-005: 90% of documents have category assigned (verify in database)
- ✅ SC-006: Zero unauthorized access incidents (all IDOR tests blocked)
- ✅ SC-007: 100% file type validation success (all unsupported types rejected)
- ✅ SC-008: Preview loads < 3 seconds
- ✅ SC-009: Search results < 2 seconds for 95% of queries
- ✅ SC-010: Team members access project documents < 1 minute after project access
- ✅ SC-011: 100% WCAG 2.1 Level AA compliance (axe DevTools pass)

---

## Next Steps

After successful validation:

1. **Commit Feature**: `git add .` → `git commit -m "feat(001): Implement document upload and management"`
2. **Push to GitHub**: `git push origin main`
3. **Update Documentation**: Add feature usage guide to README.md
4. **Training Materials**: Create user guide for document management feature
5. **Celebrate**: Feature complete! 🎉
