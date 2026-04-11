---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
inputDocuments:
  - docs/newversion/processor/newversion-processor-storage/brainstorm.md
  - docs/newversion/processor/newversion-processor-storage/prd.md
  - docs/newversion/processor/newversion-processor-storage/product-brief.md
workflowType: architecture
feature: newversion-processor-storage
doc_type: architecture
status: draft
project_name: PrintingMagic
user_name: cweber
date: '2026-04-11'
goal: "Establish a standalone processor service with EF Core storage and MassTransit intake that persists enriched, evidence-backed product records ready for future AI and listing workflows"
key_decisions:
  - ADR-01 Service SDK is Microsoft.NET.Sdk.Web (Minimal API + MassTransit consumer in one host)
  - ADR-02 MassTransit intake via ModelDiscoveredV1 event in new PrintingMagic.Contracts library
  - ADR-03 Dedicated processor-db on the shared postgres Aspire resource (same instance, isolated database)
  - ADR-04 EF Core migrations applied with MigrateAsync() on startup for local dev; migrations are checked in
  - ADR-05 Processor repo lives at TargetProjects/newversion/processor/ — its own git repo per domain convention
  - ADR-06 PrintingMagic.ServiceDefaults packaged as NuGet (in scope this feature); PrintingMagic.Contracts is the new cross-service contracts library
  - ADR-07 Aspire AppHost references the processor via relative #:project directive; registered as "processor"
open_questions: []
depends_on:
  - newversion-scraper-scraper-scaffolding
blocks: []
updated_at: '2026-04-11T00:00:00Z'
---

# Architecture Decision Document: Processor Storage Foundation

**Feature:** `newversion-processor-storage`  
**Domain:** `newversion`  
**Service:** `processor`  
**Author:** cweber  
**Date:** 2026-04-11  
**Status:** Draft

---

## 1. System Context

```
┌─────────────────────────────────────────────────────────────────┐
│  Aspire AppHost (PrintingMagic.Aspire)                          │
│                                                                   │
│  ┌──────────────────┐   ModelProcessingRequestedV1   ┌────────────────────┐
│  │ PrintingMagic    │ ─── ModelDiscoveredV1 ─────────▶│ PrintingMagic      │
│  │ .Scraper         │          RabbitMQ               │ .Processor         │
│  │                  │                                 │                    │
│  │ (scraper-db)     │                                 │ (processor-db)     │
│  └──────────────────┘                                 │                    │
│                                                        │ GET /products/{id} │
│  ┌─────────────────┐                                  │ POST /imports/model│
│  │ postgres        │◀─────────────────────────────────│ (future consumers) │
│  │ (shared)        │    processor-db                   └────────────────────┘
│  └─────────────────┘
│
│  ┌─────────────────┐
│  │ RabbitMQ        │  (message-broker — existing)
│  └─────────────────┘
└─────────────────────────────────────────────────────────────────┘
```

**Key boundaries:**
- `PrintingMagic.Scraper` publishes; `PrintingMagic.Processor` consumes and owns all enriched product persistence.
- `PrintingMagic.Processor` owns `processor-db` exclusively — no shared table access with scraper or ApiService.
- `PrintingMagic.Contracts` is the message-contract library shared across both services.

---

## 2. Technology Stack

| Concern | Choice | Rationale |
|---------|--------|-----------|
| SDK | `Microsoft.NET.Sdk.Web` | Minimal API needed for FR-03 (`GET /products/{id}`) + intake endpoint; also hosts MassTransit |
| Target framework | `net10.0` | Matches scraper (NFR-01) |
| Database access | EF Core 10 + `Npgsql.EntityFrameworkCore.PostgreSQL` | Matches scraper pattern |
| Aspire DB integration | `Aspire.Npgsql.EntityFrameworkCore.PostgreSQL` | Resolves connection string from Aspire environment |
| Messaging | MassTransit 8.4.0 + `MassTransit.RabbitMQ` | Existing infra in AppHost; MassTransit consumer for intake |
| Service defaults | `PrintingMagic.ServiceDefaults` (NuGet package — in scope this feature) | OpenTelemetry, health checks, service discovery |
| Unit tests | xUnit 2.9.3 + Moq + FluentAssertions | Matches scraper test project (domain constitution: Given/When/Then naming) |
| Integration tests | `Aspire.Hosting.Testing` (`DistributedApplicationTestingBuilder`) | Domain constitution hard gate |

---

## 3. Architectural Decisions

### ADR-01 — Service SDK: `Microsoft.NET.Sdk.Web` with Minimal API

**Context:** The processor must serve `GET /products/{id}` (FR-03) and accept intake requests (FR-01). The scraper uses `Microsoft.NET.Sdk.Worker` (no HTTP endpoints). The processor is different: it exposes HTTP and processes messages.

**Decision:** Use `Microsoft.NET.Sdk.Web`. A Web host supports `WebApplication.CreateBuilder` which allows both `app.MapGet(...)` Minimal API routes and `AddMassTransit(...)` hosted services. No need for a separate Worker process.

**Consequences:**
- Ports auto-assigned by Aspire in dev.
- `AddServiceDefaults()` works identically on both `Host` and `WebApplication` builders.
- MassTransit consumer runs as an `IHostedService` inside the same web process.

---

### ADR-02 — Intake Mechanism: MassTransit Consumer — `ModelDiscoveredV1`

**Context:** OQ-01 left the intake mechanism as a TechPlan decision. Options were HTTP push or queue-based async.

**Decision:** Async queue intake via MassTransit consumer. `ModelDiscoveredV1` lives in `PrintingMagic.Contracts`. The scraper publishes this event after the model zip is available. The processor defines a MassTransit consumer that creates `ProcessedProduct` records on receipt.

```csharp
// PrintingMagic.Contracts — Events/ModelDiscoveredV1.cs
public record ModelDiscoveredV1(
    int ModelId,
    int CollectionId,
    string SourceUrl,
    string ImagesZipBlobUrl,
    DateTime DiscoveredUtc);
```

**Rationale:**
- Decouples processor startup/availability from scraper runs.
- Reuses existing RabbitMQ `message-broker` resource — zero new infrastructure.
- No direct HTTP service-to-service dependency between scraper and processor (resilient, independently deployable).
- The scraper already has MassTransit plumbing for publishing events; `ModelDiscoveredV1` is moved from `PrintingMagic.Scraper.Events.Contracts` to `PrintingMagic.Contracts`.
- The `POST /imports/model` HTTP endpoint remains available as a manual/testing intake path (backed by the same handler logic as the consumer).

**Dev task produced:** Scraper must be updated to publish `ModelDiscoveredV1` from `PrintingMagic.Contracts` after zip URL is stored. See §Cross-Repo Dev Tasks.

---

### ADR-03 — Database: Dedicated `processor-db` on Shared `postgres`

**Context:** OQ-02 left DB isolation as a TechPlan decision. Options are shared DB with scraper or separate.

**Decision:** Dedicated `processor-db` database on the same shared `postgres` container that the scraper uses. Aspire's `AddDatabase()` creates a logical database within the container.

```csharp
// apphost.cs addition
var processorDb = postgres.AddDatabase("processor-db");

builder.AddProject<Projects.PrintingMagic_Processor>("processor")
    .WithReference(processorDb)
    .WaitFor(processorDb)
    .WithReference(rabbitMq)
    .WaitFor(rabbitMq);
```

**Rationale:**
- Zero new infrastructure in local dev (postgres container already persistent).
- Schema isolation: `ProcessorDbContext` only ever touches `processor-db`.
- Avoids EF migration conflicts between scraper and processor.
- Upgrade path: move to separate postgres resource with no code changes (just change Aspire resource reference).

---

### ADR-04 — EF Core Migration Strategy

**Context:** NFR-04 requires checked-in migrations runnable from clean state.

**Decision:**
- Migrations generated via `dotnet ef migrations add` and checked into the processor repo.
- On startup (development/test environments) call `await context.Database.MigrateAsync()` in `Program.cs`.
- For CI/production: migrations applied via `dotnet ef database update` in the deployment pipeline (not auto-applied in release mode).

**Guard:** `MigrateAsync()` is gated on `app.Environment.IsDevelopment()` to prevent accidental auto-migration in production.

---

### ADR-05 — Repository Layout: Separate Git Repo at `TargetProjects/newversion/processor/`

**Context:** Per domain convention (see repo memory), each service has its own git repo under `TargetProjects/newversion/`.

**Decision:** The processor lives at `TargetProjects/newversion/processor/` with its own `.git`. The AppHost references it via `#:project ../../processor/PrintingMagic.Processor/PrintingMagic.Processor.csproj`.

**Workspace layout requirement:** CI and local dev must check out repos under a common parent so relative references resolve correctly:
```
<workspace-root>/
  apphost/
  contracts/    ← PrintingMagic.Contracts
  scraper/
  processor/
```
All relative `<ProjectReference>` paths (e.g., `../../contracts/PrintingMagic.Contracts/`) assume this layout. CI pipelines must enforce it.

---

### ADR-06 — ServiceDefaults: NuGet Package (in scope)

**Context:** `PrintingMagic.ServiceDefaults` is currently in the scraper repo and was previously copied per-service. Copying creates silent drift risk across services.

**Decision:** `PrintingMagic.ServiceDefaults` is packaged as a local NuGet package in this feature sprint. The csproj is moved to `TargetProjects/newversion/servicedefaults/` (own git repo, follows domain pattern) with NuGet packaging metadata. Both scraper and processor reference it via a local NuGet feed or `<ProjectReference>` during dev. The copy in the scraper repo is retired once the package is published.

**Trigger for sync (until NuGet packaging lands):** If NuGet packaging is not completed this sprint, a copy is acceptable as a fallback — but a sync task must be added to any ServiceDefaults PR in the scraper to propagate changes to all copies.

---

### ADR-07 — Aspire Registration: `"processor"` resource, port auto-assigned

> **Note:** `PrintingMagic.Contracts` is a library, not a runnable project — no `#:project` directive is added for it in AppHost. It is referenced transitively via scraper and processor project references.

**Context:** OQ-03 asked for Aspire registration name and port conventions.

**Decision:**
- Aspire resource name: `"processor"` (matches the domain convention: `scraper` → `processor`).
- Port: auto-assigned by Aspire (no fixed port binding in development).
- Service discovery name (for future consumers): `http+http://processor`.

---

## 4. Data Model — EF Core Entities

### 4.1 Entity Overview

```
ProcessedProduct (1) ──────────────────── (N) ImageEvidence
     │
     └──────────────────────────────────── (N) GalleryImage
```

### 4.2 `ProcessedProduct`

```csharp
public class ProcessedProduct
{
    public Guid Id { get; set; }                        // PK
    public int ScraperModelId { get; set; }             // FK ref to scraper's model ID
    public int CollectionId { get; set; }
    public string SourceUrl { get; set; } = string.Empty;
    public string ImagesZipBlobUrl { get; set; } = string.Empty;

    // Processing state
    public ProcessingStatus ProcessingStatus { get; set; } = ProcessingStatus.Pending;

    // Merchandising outputs (nullable — populated by AI enrichment feature)
    public string? ProductTitle { get; set; }
    public string? CopyTitle { get; set; }
    public string? CopyDescription { get; set; }
    public FinishType? FinishType { get; set; }
    public string? HeroImageUrl { get; set; }
    public ComplianceStatus? ComplianceStatus { get; set; }
    public bool ListingReady { get; set; } = false;  // Always false in this foundation sprint; AI enrichment feature sets to true

    // AI provenance (nullable — populated by AI enrichment feature)
    public decimal? Confidence { get; set; }
    public string? AiProvider { get; set; }
    public string? AiModel { get; set; }
    public DateTime? ProcessedAt { get; set; }

    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }

    public ICollection<ImageEvidence> Images { get; set; } = new List<ImageEvidence>();
    public ICollection<GalleryImage> GalleryImages { get; set; } = new List<GalleryImage>();
}

public enum ProcessingStatus { Pending, Processing, Enriched, ReviewRequired, Blocked }
public enum FinishType { Painted, MulticolorPrint, Unknown }
public enum ComplianceStatus { Clear, Flagged, PendingReview, Blocked }
```

### 4.3 `ImageEvidence`

```csharp
public class ImageEvidence
{
    public Guid Id { get; set; }
    public Guid ProductId { get; set; }          // FK → ProcessedProduct
    public string Url { get; set; } = string.Empty;
    public string? ExtractionPath { get; set; }
    public ProcessedProduct Product { get; set; } = null!;
}
```

### 4.4 `GalleryImage`

```csharp
public class GalleryImage
{
    public Guid Id { get; set; }
    public Guid ProductId { get; set; }          // FK → ProcessedProduct
    public string Url { get; set; } = string.Empty;
    public string? BackgroundFamily { get; set; }
    public int SortOrder { get; set; }
    public bool IsHero { get; set; } = false;
    public ProcessedProduct Product { get; set; } = null!;
}
```

### 4.5 `ProcessorDbContext`

```csharp
public class ProcessorDbContext : DbContext
{
    public ProcessorDbContext(DbContextOptions<ProcessorDbContext> options) : base(options) { }

    public DbSet<ProcessedProduct> ProcessedProducts => Set<ProcessedProduct>();
    public DbSet<ImageEvidence> ImageEvidences => Set<ImageEvidence>();
    public DbSet<GalleryImage> GalleryImages => Set<GalleryImage>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<ProcessedProduct>(e =>
        {
            e.HasKey(x => x.Id);
            e.HasIndex(x => x.ScraperModelId).IsUnique();
            e.Property(x => x.SourceUrl).HasMaxLength(2048).IsRequired();
            e.Property(x => x.ImagesZipBlobUrl).HasMaxLength(2048).IsRequired();
            e.Property(x => x.ProcessingStatus).HasConversion<string>();
            e.Property(x => x.FinishType).HasConversion<string>();
            e.Property(x => x.ComplianceStatus).HasConversion<string>();
            e.Property(x => x.CreatedAt).HasDefaultValueSql("now() at time zone 'utc'");
            e.Property(x => x.UpdatedAt).HasDefaultValueSql("now() at time zone 'utc'");
        });

        modelBuilder.Entity<ImageEvidence>(e =>
        {
            e.HasKey(x => x.Id);
            e.Property(x => x.Url).HasMaxLength(2048).IsRequired();
            e.HasOne(x => x.Product).WithMany(p => p.Images)
             .HasForeignKey(x => x.ProductId).OnDelete(DeleteBehavior.Cascade);
        });

        modelBuilder.Entity<GalleryImage>(e =>
        {
            e.HasKey(x => x.Id);
            e.Property(x => x.Url).HasMaxLength(2048).IsRequired();
            e.HasOne(x => x.Product).WithMany(p => p.GalleryImages)
             .HasForeignKey(x => x.ProductId).OnDelete(DeleteBehavior.Cascade);
        });
    }
}
```

---

## 5. API Surface (Minimal API)

| Method | Route | Description |
|--------|-------|-------------|
| `POST` | `/imports/model` | Manual/test intake. Accepts `ModelImportDto`, creates `ProcessedProduct`. Returns `201 Created` with product ID. |
| `GET` | `/products/{id:guid}` | Returns full `ProcessedProductDto` for the given product ID. Returns `404` if not found. |
| `GET` | `/health` | ASP.NET health check endpoint (from `AddServiceDefaults`). |

> **Security note:** All endpoints are intentionally unauthenticated in this foundation sprint. Authentication and authorization are a future feature concern. This is an acknowledged, scoped decision — not an oversight.

### Request/Response DTOs

```csharp
// Response DTO — maps from ProcessedProduct entity
public record ProcessedProductDto(
    Guid Id,
    int ScraperModelId,
    string SourceUrl,
    string ImagesZipBlobUrl,
    string ProcessingStatus,
    string? ProductTitle,
    string? CopyTitle,
    string? CopyDescription,
    string? FinishType,
    string? HeroImageUrl,
    string? ComplianceStatus,
    bool ListingReady,
    decimal? Confidence,
    string? AiProvider,
    string? AiModel,
    DateTime? ProcessedAt,
    DateTime CreatedAt,
    DateTime UpdatedAt,
    IReadOnlyList<ImageEvidenceDto> Images,
    IReadOnlyList<GalleryImageDto> GalleryImages);

public record ImageEvidenceDto(Guid Id, string Url, string? ExtractionPath);
public record GalleryImageDto(Guid Id, string Url, string? BackgroundFamily, int SortOrder, bool IsHero);
```

---

## 6. MassTransit Consumer

```csharp
// PrintingMagic.Processor/Consumers/ProcessModelConsumer.cs
public class ProcessModelConsumer : IConsumer<ModelDiscoveredV1>
{
    private readonly ProcessorDbContext _db;
    private readonly ILogger<ProcessModelConsumer> _logger;

    public ProcessModelConsumer(ProcessorDbContext db, ILogger<ProcessModelConsumer> logger)
    {
        _db = db;
        _logger = logger;
    }

    public async Task Consume(ConsumeContext<ModelDiscoveredV1> context)
    {
        var msg = context.Message;
        // Idempotent — skip if product already exists for this model ID
        var existing = await _db.ProcessedProducts
            .AnyAsync(p => p.ScraperModelId == msg.ModelId, context.CancellationToken);
        if (existing) return;

        var product = new ProcessedProduct
        {
            Id = Guid.NewGuid(),
            ScraperModelId = msg.ModelId,
            CollectionId = msg.CollectionId,
            SourceUrl = msg.SourceUrl,
            ImagesZipBlobUrl = msg.ImagesZipBlobUrl,
            ProcessingStatus = ProcessingStatus.Pending,
            CreatedAt = DateTime.UtcNow,
            UpdatedAt = DateTime.UtcNow
        };
        _db.ProcessedProducts.Add(product);
        await _db.SaveChangesAsync(context.CancellationToken);
        _logger.LogInformation("Created ProcessedProduct {Id} for model {ModelId}", product.Id, msg.ModelId);
    }
}
```

MassTransit registration in `Program.cs`:
```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<ProcessModelConsumer>();
    x.UsingRabbitMq((context, cfg) =>
    {
        var connectionString = builder.Configuration.GetConnectionString("message-broker");
        if (!string.IsNullOrEmpty(connectionString)) cfg.Host(new Uri(connectionString));
        cfg.ConfigureEndpoints(context);
    });
});
```

---

## 7. Project Structure

```
TargetProjects/newversion/processor/            ← git repo root
├── .git/
├── .gitignore
├── PrintingMagic.Processor.sln
│
├── PrintingMagic.Processor/
│   ├── PrintingMagic.Processor.csproj          ← Microsoft.NET.Sdk.Web, net10.0
│   ├── Program.cs
│   ├── Consumers/
│   │   └── ProcessModelConsumer.cs
│   ├── Data/
│   │   ├── ProcessorDbContext.cs
│   │   ├── Entities/
│   │   │   ├── ProcessedProduct.cs
│   │   │   ├── ImageEvidence.cs
│   │   │   └── GalleryImage.cs
│   │   ├── Enums/
│   │   │   ├── ProcessingStatus.cs
│   │   │   ├── FinishType.cs
│   │   │   └── ComplianceStatus.cs
│   │   └── Migrations/                         ← EF Core migrations (checked in)
│   │       └── ...
│   ├── Dtos/
│   │   ├── ProcessedProductDto.cs
│   │   ├── ImageEvidenceDto.cs
│   │   └── GalleryImageDto.cs
│   ├── Routes/
│   │   ├── ImportsRoutes.cs                    ← POST /imports/model
│   │   └── ProductsRoutes.cs                   ← GET /products/{id}
│   ├── appsettings.json
│   └── appsettings.Development.json
│
├── PrintingMagic.Processor.Tests/
│   ├── PrintingMagic.Processor.Tests.csproj    ← xUnit, no Aspire hosting
│   └── Consumers/
│       └── ProcessModelConsumerTests.cs        ← unit tests (BDD naming)
│
├── PrintingMagic.Processor.Tests.Integration/
│   ├── PrintingMagic.Processor.Tests.Integration.csproj  ← Aspire.Hosting.Testing
│   └── ProcessorAppHostTests.cs                ← DistributedApplicationTestingBuilder
│
└── PrintingMagic.ServiceDefaults/              ← NuGet package (in scope this feature)
    ├── PrintingMagic.ServiceDefaults.csproj    ← NuGet metadata added
    └── Extensions.cs

(PrintingMagic.Contracts lives at TargetProjects/newversion/contracts/ — separate repo, referenced via project reference)
```

---

## 8. AppHost Changes

The following additions are required in `TargetProjects/newversion/apphost/PrintingMagic.Aspire/apphost.cs`:

```csharp
// Add at top — new project reference directive
#:project ../../processor/PrintingMagic.Processor/PrintingMagic.Processor.csproj

// Add after scraperDb declaration
var processorDb = postgres.AddDatabase("processor-db");

// Add after scraper service registration
builder.AddProject<Projects.PrintingMagic_Processor>("processor")
    .WithReference(processorDb)
    .WaitFor(processorDb)
    .WithReference(rabbitMq)
    .WaitFor(rabbitMq);
```

---

## 9. Contracts Changes

Add the following to `TargetProjects/newversion/contracts/PrintingMagic.Contracts/Events/ModelDiscoveredV1.cs`:

```csharp
/// <summary>
/// Published by the scraper after a model's zip blob URL is available.
/// Consumed by PrintingMagic.Processor to create a ProcessedProduct record.
/// </summary>
public record ModelDiscoveredV1(
    int ModelId,
    int CollectionId,
    string SourceUrl,
    string ImagesZipBlobUrl,
    DateTime DiscoveredUtc);
```

This event is moved from `PrintingMagic.Scraper.Events.Contracts` into `PrintingMagic.Contracts.Events`. Scraper code must update its namespace import — additive at the consumer side (NFR-05 forward-compatibility).

---

## 10. Implementation Consistency Rules

| Category | Rule |
|----------|------|
| Naming — entities | PascalCase class names; snake_case column names via EF fluent config or `HasColumnName()` if different |
| Naming — endpoints | Lowercase kebab-case routes: `/imports/model`, `/products/{id}` |
| Naming — tests | `Given{State}_When{Action}_Then{Outcome}` for all `[Fact]` methods (domain constitution hard gate) |
| Enums | Stored as strings in the database (`.HasConversion<string>()`) |
| Nullable fields | All merchandising and AI provenance fields are nullable `C# ?` types |
| Error handling | Endpoint returns `Results.NotFound()` for missing products; consumer is idempotent (duplicate check before insert) |
| DTOs | Separate `Dto` record types for HTTP responses — entities are never serialized directly |
| Migrations | Never delete existing migration files; additive-only for this feature's schema |
| Timestamps | `DateTime.UtcNow` throughout; database defaults via `HasDefaultValueSql("now() at time zone 'utc'")` |
| `UpdatedAt` on updates | Callers must explicitly set `entity.UpdatedAt = DateTime.UtcNow` before every `SaveChangesAsync` call that modifies a record. The DB default only fires on insert. |
| MassTransit | Consumer endpoint name auto-configured via `ConfigureEndpoints(context)` — do not hardcode queue names |
| `POST /imports/model` testing | The HTTP intake path must have at least one unit test covering the happy path (see FR-08). The consumer and HTTP handler share the same internal service logic to avoid a coverage gap. |

---

## 11. Open Questions — Resolved by This Document

| ID | Question | Resolution |
|----|----------|------------|
| OQ-01 | HTTP vs queue intake | **Queue (MassTransit consumer)** via `ModelDiscoveredV1` (from `PrintingMagic.Contracts`); HTTP endpoint remains available for manual/test use |
| OQ-02 | Same PostgreSQL instance or separate? | **Same `postgres` Aspire resource**, separate `processor-db` logical database |
| OQ-03 | Aspire registration name and port | **`"processor"`** resource name; port auto-assigned by Aspire |

---

## 12. Requirements Coverage Validation

| Req | Covered By |
|-----|------------|
| FR-01 | `ProcessModelConsumer` + `POST /imports/model` route |
| FR-02 | `ProcessedProduct.ImagesZipBlobUrl` + `ImageEvidence` collection |
| FR-03 | `GET /products/{id}` Minimal API route returning `ProcessedProductDto` |
| FR-04 | All merchandising fields nullable on `ProcessedProduct` entity |
| FR-05 | All AI provenance fields nullable on `ProcessedProduct` entity |
| FR-06 | `ProcessingStatus` enum on `ProcessedProduct` |
| FR-07 | `AddProject<Projects.PrintingMagic_Processor>("processor")` in AppHost |
| FR-08 | `ProcessorAppHostTests.cs` with `DistributedApplicationTestingBuilder`; `ProcessModelConsumerTests.cs` covers `POST /imports/model` unit path |
| FR-09 | `GalleryImage` entity with `Url`, `BackgroundFamily`, `SortOrder` |
| FR-10 | `ModelDiscoveredV1` in `PrintingMagic.Contracts` (new shared library); `ModelImportDto` in processor |
| FR-11 | `PrintingMagic.Processor.csproj` references `PrintingMagic.Contracts` via project reference |
| FR-12 | `PrintingMagic.Contracts` csproj has no cross-project references (standalone) |
| NFR-01 | `net10.0`, Aspire SDK 13.2.2, MassTransit 8.4.0, EF Core 10 |
| NFR-02 | `Given_When_Then` test naming enforced in consistency rules |
| NFR-03 | `PrintingMagic.Processor.Tests.Integration` project with `DistributedApplicationTestingBuilder` |
| NFR-04 | EF Core migrations in `Data/Migrations/`; `MigrateAsync()` in `Program.cs` gated to development |
| NFR-05 | `ModelDiscoveredV1` is new; `PrintingMagic.Contracts` is a new library — no existing types modified |
| NFR-06 | `ImagesZipBlobUrl` + `ImageEvidence` persisted on first intake; no re-scraping needed for later reads |
| NFR-07 | Processor has its own git repo, own DB, own process — fully independent from ApiService |

**All 19 functional requirements and 7 NFRs are architecturally covered.** ✅

---

## 13. Domain Constitution Compliance

| Gate | Status | Evidence |
|------|--------|---------|
| xUnit BDD Given/When/Then naming | ✅ Required | Consistency rule §10; `ProcessModelConsumerTests.cs` |
| Aspire integration test | ✅ Required | `ProcessorAppHostTests.cs` with `DistributedApplicationTestingBuilder` |
| `net10.0` / Aspire SDK | ✅ | ADR-01, ADR-07 |

---

## 14. Cross-Repo Dev Tasks

The following dev tasks span repos and must be explicitly owned before sprint planning:

| Task | Repo | Owner | Notes |
|------|------|-------|-------|
| Publish `ModelDiscoveredV1` from scraper after zip URL stored | `scraper` | cweber | Scraper must update import from `PrintingMagic.Scraper.Events.Contracts` → `PrintingMagic.Contracts.Events` and publish on the MassTransit bus |
| Retire `PrintingMagic.Data.Contracts` stub (or remove) | `scraper` | cweber | After `PrintingMagic.Contracts` is created and referenced |
| Update `PrintingMagic.Scraper.csproj` to reference `PrintingMagic.Contracts` | `scraper` | cweber | Replace old `Data.Contracts` and `Scraper.Events.Contracts` project references |

---

## 15. Next Phase

On completion of this architecture document, advance to `/finalizeplan` to consolidate all planning artifacts, run the final adversarial review, and prepare the dev-ready PR handoff.

