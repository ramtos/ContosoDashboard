<!--
Sync Impact Report:
Version: 0.0.0 → 1.0.0
Justification: Initial constitution establishment for training project
Modified principles: None (initial creation)
Added sections:
  - Core Principles (5 principles)
  - Technology Standards
  - Development Workflow
  - Governance
Follow-up TODOs: None
-->

# ContosoDashboard Training Project Constitution

This constitution defines the development principles and practices for the ContosoDashboard training application, which teaches Spec-Driven Development (SDD) using GitHub Spec Kit.

## Core Principles

### I. Specification-First Development (NON-NEGOTIABLE)

Every feature MUST follow the complete Spec Kit workflow before implementation:

1. **Specify** (`speckit.specify`): Create detailed feature specification with user stories, acceptance criteria, and requirements
2. **Plan** (`speckit.plan`): Generate technical design, architecture decisions, and implementation strategy
3. **Tasks** (`speckit.tasks`): Break down into dependency-ordered, actionable tasks
4. **Implement** (`speckit.implement`): Execute tasks systematically with verification

**Rationale**: This enforces disciplined development practices students must learn. Skipping specification leads to scope creep, unclear requirements, and educational goals not being met. The Spec Kit workflow is the primary learning objective.

### II. Training-Appropriate Security

All features MUST implement security patterns suitable for production, even when using mock implementations:

- Authorization checks at both page level (`[Authorize]`) and service level
- IDOR (Insecure Direct Object Reference) protection in all data access
- Input validation and sanitization
- Proper error handling without information disclosure
- Security headers and CSP policies

**Rationale**: Students learn correct security patterns they'll use in production. Mock authentication is acceptable for offline training, but authorization logic must be production-grade. This prepares students for real-world requirements while maintaining training simplicity.

### III. Infrastructure Abstraction

All external dependencies MUST use interface abstractions to enable seamless migration from local to cloud implementations:

- Database access through Entity Framework Core (swappable connection strings)
- File storage through `IFileStorageService` interface (LocalFileStorage ↔ AzureBlobStorage)
- Authentication through middleware (Cookie-based ↔ Azure AD/Entra ID)
- Configuration through `IConfiguration` (appsettings.json ↔ Azure App Configuration)

**Rationale**: Teaches proper dependency injection and separation of concerns. Students learn that infrastructure is swappable, business logic is portable, and cloud migration should not require rewrites.

### IV. Clean Service Architecture

All business logic MUST reside in service classes with clear interfaces, separate from UI and data layers:

- Services implement interfaces (`ITaskService`, `IProjectService`, etc.)
- Services contain authorization logic to prevent unauthorized access
- Services handle business rules and validation
- Controllers/Pages orchestrate services, never contain business logic
- Data models are separate from view models

**Rationale**: Enforces separation of concerns and testability. Students learn layered architecture patterns essential for maintainable enterprise applications.

### V. Explicit Over Implicit

Code MUST be explicit and self-documenting:

- Meaningful variable and method names over comments
- Explicit null checks rather than relying on nullable reference warnings
- Clear exception messages with context
- README documentation for architectural decisions
- Inline comments only for non-obvious business rules

**Rationale**: Training code should be readable by students with varying experience levels. Explicit code teaches clarity and reduces cognitive load during learning.

## Technology Standards

### Approved Technology Stack

**Current (Training):**
- **Framework**: ASP.NET Core 10.0 with Blazor Server
- **Database**: SQLite (Entity Framework Core)
- **Authentication**: Cookie-based mock authentication
- **UI**: Bootstrap 5.3 with Bootstrap Icons
- **Styling**: CSS (site.css)

**Production Migration Path:**
- **Database**: Azure SQL Database or Cosmos DB
- **File Storage**: Azure Blob Storage
- **Authentication**: Microsoft Entra ID (Azure AD)
- **Hosting**: Azure App Service or Container Apps

### Technology Constraints

- **No external API dependencies** - Training must work offline
- **No paid services required** - Maximize training accessibility
- **No complex build toolchains** - Keep setup simple for students
- **Version consistency** - All team members use same SDK versions

**Rationale**: Offline-first design ensures training works in any environment. Simplicity reduces friction and keeps focus on SDD learning objectives.

## Development Workflow

### Feature Development Process

1. **Stakeholder input** placed in `StakeholderDocs/` as markdown
2. **Run speckit.specify** with feature description from stakeholder docs
3. **Review spec.md** in `specs/NNN-feature-name/` directory
4. **Run speckit.plan** to generate technical design
5. **Run speckit.tasks** to create implementation checklist
6. **Run speckit.implement** to execute tasks systematically
7. **Commit and push** with descriptive commit messages referencing spec

### Commit Standards

- Commit messages MUST reference the feature spec number (e.g., "feat(001): Add document upload")
- Changes MUST include relevant tests (when applicable for training)
- Database migration scripts MUST be included with schema changes
- README updates MUST accompany architectural changes

### Branching Strategy

- **main**: Stable training code, always buildable
- **Feature branches**: Optional for complex features, use spec number (e.g., `001-document-upload`)
- **No long-lived branches**: Merge features promptly to keep training linear

**Rationale**: Keeps repository history clear for educational review. Students can trace feature development through commit history and Spec Kit artifacts.

## Governance

### Constitution Authority

This constitution supersedes all other development practices for the ContosoDashboard training project. When conflicts arise between this constitution and other guidance, the constitution takes precedence.

### Amendment Process

1. Propose amendment with clear rationale in GitHub issue
2. Document educational impact and alternative approaches considered
3. Update constitution with version bump per semantic versioning:
   - **MAJOR**: Backward-incompatible principle changes or removals
   - **MINOR**: New principles or substantial expansions
   - **PATCH**: Clarifications, wording, typo fixes
4. Commit changes with `docs(constitution): <description>` message

### Compliance

- All pull requests MUST demonstrate compliance with relevant principles
- Spec Kit workflow completion is verified before merge
- Security patterns are validated in code review
- Architecture abstractions are checked for consistency

### Educational Exceptions

Principles may be relaxed when explicitly justified for training purposes:
- Mock authentication instead of production identity providers
- In-memory data instead of persistent storage for demos
- Simplified error handling for clarity
- Reduced logging for readability

All exceptions MUST be documented in code comments and README with `[TRAINING ONLY]` markers.

**Version**: 1.0.0 | **Ratified**: 2026-08-12 | **Last Amended**: 2026-08-12
