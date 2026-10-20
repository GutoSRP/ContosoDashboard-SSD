# ContosoDashboard Constitution

## Core Principles

### I. Security by Default (NON-NEGOTIABLE)

- Every Blazor page that displays or changes data MUST declare `@attribute [Authorize]` (or a
  named role policy); only the Login/Logout Razor Pages and error pages MAY allow anonymous
  access.
- Every service method that reads or changes user-owned or project-scoped data MUST receive the
  requesting user's ID (taken from the authenticated user's claims) and MUST verify ownership,
  project membership, or role before returning or changing data (IDOR protection).
- Unauthorized requests MUST return `null`, `false`, or an empty result and MUST NOT reveal
  whether the resource exists. Hiding UI controls MUST NOT be the only authorization control.
- Role checks MUST use the existing hierarchy (Employee < TeamLead < ProjectManager <
  Administrator) and the named policies in `Program.cs`; ad-hoc role strings MUST NOT be added.
- All user input MUST be validated on the server. User-supplied file names MUST NOT be used in
  storage paths (GUID-based names only), and uploads MUST be checked against an extension
  allow-list and a maximum size.
- Protected files MUST be stored outside `wwwroot` and served only through endpoints or
  services that apply the same authorization checks as the owning feature.
- The security headers set in `Program.cs` (CSP, X-Frame-Options, X-Content-Type-Options,
  Referrer-Policy) MUST NOT be weakened without a justification recorded in `plan.md`.
- Secrets MUST NOT be committed; connection strings, storage paths, and limits MUST come from
  configuration (`appsettings*.json`).

**Rationale**: ContosoDashboard teaches secure patterns. A single missing check leaks data
across users, so authorization is enforced in depth: page attribute plus service check.

### II. Offline-First with Cloud-Ready Abstractions

- The application MUST build, run, and pass every acceptance scenario without internet access,
  cloud subscriptions, or external services.
- Training infrastructure is fixed: SQLite through EF Core for data, the local file system
  (outside `wwwroot`, e.g. `AppData/uploads`) for files, and the cookie-based mock login for
  identity.
- Every infrastructure dependency (file storage, virus scanning, e-mail, external APIs) MUST
  be consumed through an interface registered in DI (e.g. `IFileStorageService`); pages and
  business services MUST depend only on that interface.
- New code MUST NOT add cloud SDK packages (e.g. `Azure.Storage.Blobs`) to the training
  implementation.
- Moving to Azure (Azure SQL Database, Blob Storage, Microsoft Entra ID) MUST require only new
  infrastructure implementations, the EF Core provider switch, and configuration/DI changes,
  with no changes to business logic, pages, or the database schema.
- Persisted file locations MUST be relative and portable, e.g.
  `{userId}/{projectId|personal}/{guid}.{ext}`.

**Rationale**: Training must run anywhere with zero setup, while the abstractions model the
production migration path.

### III. Consistent Layered Architecture

- Code MUST follow the existing layers: `Models/` (EF Core entities with data annotations),
  `Data/ApplicationDbContext.cs` (relationships, indexes, seed data), `Services/` (an
  `I{Name}Service` interface plus implementation, registered as scoped in `Program.cs`), and
  `Pages/` + `Shared/` (Blazor components and Razor Pages).
- Pages and components MUST NOT use `ApplicationDbContext` directly; data access and business
  rules MUST live in services.
- New features MUST extend the current architecture without rewriting existing features, and
  the existing pages (Dashboard, Tasks, Projects, Project Details, Team, Notifications,
  Profile) MUST keep working.
- Entity primary keys MUST be integers, consistent with `User`, `Project`, and `TaskItem`.
- String properties MUST declare `[MaxLength]`; columns used for filtering, sorting, or joins
  MUST be indexed in `ApplicationDbContext`.
- The schema is created by `EnsureCreated()` with seed data. Schema changes MUST keep the seed
  data valid, and `plan.md` MUST state that the local `ContosoDashboard.db` must be recreated.
- Simplicity first (YAGNI): new layers, frameworks, or patterns (e.g. repositories, CQRS,
  MediatR) MUST NOT be introduced unless justified in the plan's Complexity Tracking table.

**Rationale**: A predictable structure keeps the training codebase learnable and lets Spec Kit
plans map work to known locations.

### IV. Efficient, Measurable Performance

- All database and file I/O MUST be asynchronous (`async`/`await`, `ToListAsync`,
  `SaveChangesAsync`, stream APIs); `.Result` and `.Wait()` MUST NOT be used.
- Related data MUST be loaded with `.Include()`/`.ThenInclude()` or projections to avoid N+1
  queries.
- New list and search features MUST filter, sort, and page in the database query, not in
  memory.
- Every feature spec MUST state measurable performance targets (e.g. list pages < 2 s for 500
  records, search < 2 s, 25 MB upload < 30 s), and those targets MUST be checked during
  acceptance testing.
- File uploads and downloads MUST be streamed or bounded by an explicit maximum size;
  unbounded reads into memory MUST NOT be used.

**Rationale**: Blazor Server shares server resources across all connected circuits; blocking
calls and over-fetching degrade every user at once.

### V. Spec-Driven, Verifiable Delivery

- Every feature MUST follow the Spec Kit flow: `/speckit.specify` → `/speckit.clarify` (when
  the spec has open questions) → `/speckit.plan` → `/speckit.tasks` → `/speckit.implement`.
  Code MUST NOT be written for requirements that are not in `spec.md`.
- `spec.md` MUST describe what and why without frameworks, schemas, or code; technical
  decisions belong in `plan.md` and MUST pass the Constitution Check.
- Every user story MUST be prioritized (P1, P2, ...), independently testable, and include
  Given-When-Then acceptance scenarios, with at least one negative authorization scenario.
- Delivery MUST be MVP-first: the P1 story is implemented and verified end to end before work
  on lower-priority stories begins.
- A story is done only when every acceptance scenario has been verified in the running
  application; manual verification with the seeded users is acceptable, and automated tests
  are written when the spec or plan requires them.
- Production-only concerns (real identity provider, cloud services, rate limiting, account
  lockout) MUST be listed as out of scope or future work, not implemented, unless the spec
  explicitly requires them.

**Rationale**: Specs keep the implementation aligned with stakeholder requirements, and
verifiable acceptance criteria make "done" objective.

## Technology Stack & Constraints

- **Runtime**: .NET 10 (`net10.0`), ASP.NET Core Blazor Server, and Razor Pages for
  login/logout; nullable reference types and implicit usings enabled.
- **Data**: Entity Framework Core 10 with SQLite (`Data Source=ContosoDashboard.db`), created
  by `EnsureCreated()` with seed data.
- **SQLite rules**: timestamps MUST be stored as UTC `DateTime`; `decimal` and
  `DateTimeOffset` MUST NOT be used in filtered or sorted columns; text search MUST be
  case-insensitive (e.g. `EF.Functions.Like` or a normalized column), because SQLite string
  comparisons are case-sensitive by default.
- **UI**: Bootstrap 5.3 and Bootstrap Icons from the CDN already allowed by the CSP. New pages
  MUST use `MainLayout`, MUST appear in `NavMenu` when user-facing, and every form field MUST
  have a label and a validation message.
- **File uploads (Blazor)**: read the file metadata before opening the stream, call
  `OpenReadStream(maxAllowedSize)` with the configured limit, copy to a `MemoryStream`
  immediately, clear the `IBrowserFile` reference, and reset `InputFile` with `@key`. The
  upload order MUST be: generate unique path → save file → save database record.
- **Dependencies**: every new NuGet package MUST be justified in `plan.md`; packages with known
  vulnerabilities (NuGet audit NU1901–NU1904) MUST NOT be added.
- **Logging**: use `ILogger<T>`. Logs MUST NOT contain secrets or file contents, and
  security-relevant events required by a spec (e.g. upload, download, delete, share) MUST be
  recorded.
- **Accepted training limitations**: mock authentication, no rate limiting, no account
  lockout, and the `unsafe-inline`/`unsafe-eval` CSP directives required by Blazor Server are
  documented and accepted. New features MUST NOT add limitations without documenting them in
  `README.md`.

## Development Workflow & Quality Gates

- Each feature is developed on its own branch created by Spec Kit (`###-feature-name`), with
  its artifacts in `specs/###-feature-name/`.
- Commits MUST be small and use Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`).
- Before a user story is marked done, all of the following MUST pass:
  1. `dotnet build` succeeds with zero errors and no new compiler warnings.
  2. The app starts with `dotnet run` on a freshly created database (delete
     `ContosoDashboard.db*` first when the schema changed).
  3. Every acceptance scenario of the story is verified with the seeded users: Ni Kang
     (Employee), Floris Kregel (Team Lead), Camille Nicole (Project Manager), and System
     Administrator (Administrator).
  4. A cross-user check confirms that a user without access cannot see or change the new
     data, including through a direct URL.
  5. A smoke test confirms that Dashboard, Tasks, Projects, Team, Notifications, and Profile
     still work.
- `/speckit.analyze` SHOULD run after `/speckit.tasks` and before `/speckit.implement`; every
  CRITICAL finding MUST be resolved before implementation starts.

## Governance

- This constitution supersedes other development practices for ContosoDashboard. When a spec,
  plan, or task conflicts with it, the constitution prevails unless it is amended first.
- Amendments are made by updating `.specify/memory/constitution.md` (normally through
  `/speckit.constitution`), describing the change and its impact, and committing it before
  dependent specs or plans rely on it.
- Versioning follows semantic versioning: MAJOR for removing or redefining a principle, MINOR
  for adding a principle or section or materially expanding guidance, PATCH for
  clarifications and wording.
- Compliance is verified at the Constitution Check in `plan.md` (before Phase 0 research and
  after Phase 1 design), by `/speckit.analyze`, and in code review, which MUST explicitly
  confirm Principle I for every new page and service method. Justified deviations MUST be
  recorded in the plan's Complexity Tracking table.
- `README.md` and `StakeholderDocs/` provide runtime context; this constitution is the
  authority on rules.

**Version**: 1.0.0 | **Ratified**: 2026-10-20 | **Last Amended**: 2026-10-20
