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
  - ADR-02 MassTransit intake via ModelProcessingRequestedV1 message added to PrintingMagic.Data.Contracts
  - ADR-03 Dedicated processor-db on the shared postgres Aspire resource (same instance, isolated database)
  - ADR-04 EF Core migrations applied with MigrateAsync() on startup for local dev; migrations are checked in
  - ADR-05 Processor repo lives at TargetProjects/newversion/processor/ — its own git repo per domain convention
  - ADR-06 ServiceDefaults is copied from scraper pattern into the processor repo (not shared as NuGet for this sprint)
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
│  │ PrintingMagic    │ ──────────────────────────────▶│ PrintingMagic      │
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
- `PrintingMagic.Data.Contracts` is the message-contract library shared across both services.

---

## 2. Technology Stack

| Concern | Choice | Rationale |
|---------|--------|-----------|
| SDK | `Microsoft.NET.Sdk.Web` | Minimal API needed for FR-03 (`GET /products/{id}`) + intake endpoint; also hosts MassTransit |
| Target framework | `net10.0` | Matches scraper (NFR-01) |
| Database access | EF Core 10 + `Npgsql.EntityFrameworkCore.PostgreSQL` | Matches scraper pattern |
| Aspire DB integration | `Aspire.Npgsql.EntityFrameworkCore.PostgreSQL` | Resolves connection string from Aspire environment |
| Messaging | MassTransit 8.4.0 + `MassTransit.RabbitMQ` | Existing infra in AppHost; MassTransit consumer for intake |
| Service defaults | `PrintingMagic.ServiceDefaults` (copied from scraper) | OpenTelemetry, health checks, service discovery |
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

### ADR-02 — Intake Mechanism: MassTransit Consumer — `ModelProcessingRequestedV1`

**Context:** OQ-01 left the intake mechanism as a TechPlan decision. Options were HTTP push or queue-based async.

**Decision:** Async queue intake via MassTransit consumer. A new message `ModelProcessingRequestedV1` is added to `PrintingMagic.Data.Contracts`. The scraper publishes this message after the model zip is available. The processor defines a MassTransit consumer that creates `ProcessedProduct` records on receipt.

```csharp
// PrintingMagic.Data.Contracts — new message in Contracts.V1.cs
public record ModelProcessingRequestedV1(
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
- The scraper already has `IEventPublisher` + MassTransit plumbing for publishing events; adding this message is additive.
- The `POST /imports/model` HTTP endpoint remains available as a manual/testing intake path (backed by the same handler logic as the consumer).

**Dev task produced:** Scraper must publish `ModelProcessingRequestedV1` after zip URL is stored (implement in Dev phase, scraper side).

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

**Cross-repo reference for `Data.Contracts`:** The processor references `Data.Contracts` via a relative path reference — `../../scraper/PrintingMagic.Data.Contracts/PrintingMagic.Data.Contracts.csproj`. This works in local dev and CI (both repos checked out). A NuGet package version of `Data.Contracts` is a future upgrade (not in scope).

---

### ADR-06 — ServiceDefaults: Copied into Processor Repo

**Context:** `PrintingMagic.ServiceDefaults` is in the scraper repo. ServiceDefaults is a shared Aspire library providing OpenTelemetry, health checks, and service discovery.

**Decision:** Copy `PrintingMagic.ServiceDefaults` into the processor repo (`TargetProjects/newversion/processor/PrintingMagic.ServiceDefaults/`). This is consistent with the scraper pattern. Converting to a NuGet package is a future concern outside this feature's scope.

---

### ADR-07 — Aspire Registration: `"processor"` resource, port auto-assigned

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
    public bool ListingReady { get; set; } = false;

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
// PrintingMagic.Processor/Consumers/ModelProcessingRequestedConsumer.cs
public class ModelProcessingRequestedConsumer : IConsumer<ModelProcessingRequestedV1>
{
    private readonly ProcessorDbContext _db;
    private readonly ILogger<ModelProcessingRequestedConsumer> _logger;

    public ModelProcessingRequestedConsumer(ProcessorDbContext db, ILogger<ModelProcessingRequestedConsumer> logger)
    {
        _db = db;
        _logger = logger;
    }

    public async Task Consume(ConsumeContext<ModelProcessingRequestedV1> context)
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
    x.AddConsumer<ModelProcessingRequestedConsumer>();
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
│   │   └── ModelProcessingRequestedConsumer.cs
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
│       └── ModelProcessingRequestedConsumerTests.cs   ← unit tests (BDD naming)
│
├── PrintingMagic.Processor.Tests.Integration/
│   ├── PrintingMagic.Processor.Tests.Integration.csproj  ← Aspire.Hosting.Testing
│   └── ProcessorAppHostTests.cs                ← DistributedApplicationTestingBuilder
│
├── PrintingMagic.ServiceDefaults/              ← copied from scraper, IsAspireSharedProject
│   ├── PrintingMagic.ServiceDefaults.csproj
│   └── Extensions.cs
│
└── PrintingMagic.Data.Contracts/               ← REFERENCE ONLY (relative, from scraper repo)
    (not checked in here — referenced via ../../scraper/PrintingMagic.Data.Contracts/)
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

## 9. Data.Contracts Changes

Add the following to `TargetProjects/newversion/scraper/PrintingMagic.Data.Contracts/Contracts.V1.cs`:

```csharp
/// <summary>
/// Published by the scraper after a model's zip blob URL is available.
/// Consumed by PrintingMagic.Processor to create a ProcessedProduct record.
/// </summary>
public record ModelProcessingRequestedV1(
    int ModelId,
    int CollectionId,
    string SourceUrl,
    string ImagesZipBlobUrl,
    DateTime DiscoveredUtc);
```

This is an additive change — no existing types are modified (NFR-05 forward-compatibility).

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
| MassTransit | Consumer endpoint name auto-configured via `ConfigureEndpoints(context)` — do not hardcode queue names |

---

## 11. Open Questions — Resolved by This Document

| ID | Question | Resolution |
|----|----------|------------|
| OQ-01 | HTTP vs queue intake | **Queue (MassTransit consumer)** via `ModelProcessingRequestedV1`; HTTP endpoint remains available for manual/test use |
| OQ-02 | Same PostgreSQL instance or separate? | **Same `postgres` Aspire resource**, separate `processor-db` logical database |
| OQ-03 | Aspire registration name and port | **`"processor"`** resource name; port auto-assigned by Aspire |

---

## 12. Requirements Coverage Validation

| Req | Covered By |
|-----|------------|
| FR-01 | `ModelProcessingRequestedConsumer` + `POST /imports/model` route |
| FR-02 | `ProcessedProduct.ImagesZipBlobUrl` + `ImageEvidence` collection |
| FR-03 | `GET /products/{id}` Minimal API route returning `ProcessedProductDto` |
| FR-04 | All merchandising fields nullable on `ProcessedProduct` entity |
| FR-05 | All AI provenance fields nullable on `ProcessedProduct` entity |
| FR-06 | `ProcessingStatus` enum on `ProcessedProduct` |
| FR-07 | `AddProject<Projects.PrintingMagic_Processor>("processor")` in AppHost |
| FR-08 | `ProcessorAppHostTests.cs` with `DistributedApplicationTestingBuilder` |
| FR-09 | `GalleryImage` entity with `Url`, `BackgroundFamily`, `SortOrder` |
| FR-10 | `ModelProcessingRequestedV1` + existing `ModelImportDto`/`CollectionImportRequest` in `Data.Contracts` |
| FR-11 | `PrintingMagic.Processor.csproj` references `Data.Contracts` via relative path |
| FR-12 | `Data.Contracts` csproj has no cross-project references (unchanged) |
| NFR-01 | `net10.0`, Aspire SDK 13.2.2, MassTransit 8.4.0, EF Core 10 |
| NFR-02 | `Given_When_Then` test naming enforced in consistency rules |
| NFR-03 | `PrintingMagic.Processor.Tests.Integration` project with `DistributedApplicationTestingBuilder` |
| NFR-04 | EF Core migrations in `Data/Migrations/`; `MigrateAsync()` in `Program.cs` gated to development |
| NFR-05 | `ModelProcessingRequestedV1` is additive; no existing `Data.Contracts` types modified |
| NFR-06 | `ImagesZipBlobUrl` + `ImageEvidence` persisted on first intake; no re-scraping needed for later reads |
| NFR-07 | Processor has its own git repo, own DB, own process — fully independent from ApiService |

**All 19 functional requirements and 7 NFRs are architecturally covered.** ✅

---

## 13. Domain Constitution Compliance

| Gate | Status | Evidence |
|------|--------|---------|
| xUnit BDD Given/When/Then naming | ✅ Required | Consistency rule §10; `ModelProcessingRequestedConsumerTests.cs` |
| Aspire integration test | ✅ Required | `ProcessorAppHostTests.cs` with `DistributedApplicationTestingBuilder` |
| `net10.0` / Aspire SDK | ✅ | ADR-01, ADR-07 |

---

## 14. Next Phase

On completion of this architecture document, advance to `/finalizeplan` to consolidate all planning artifacts, run the final adversarial review, and prepare the dev-ready PR handoff.

