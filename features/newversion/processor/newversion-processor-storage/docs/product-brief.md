---
feature: newversion-processor-storage
doc_type: product-brief
status: draft
goal: "Establish the evidence-preserving product intelligence layer that turns scraped model assets into reviewable, downstream-ready product records"
key_decisions:
  - Frame this feature as an internal platform capability that serves future UI, listing, and review workflows
  - Preserve source evidence and AI provenance so enriched product records can be trusted, reviewed, and reprocessed
  - Design for downstream projections instead of binding storage directly to a single future consumer
  - Treat compliance screening as an early gate in the enrichment flow
  - Store merchandising-critical outputs such as hero image, gallery ordering, finish state, and generated copy as first-class data
open_questions:
  - Should the persistence boundary for enriched model data live in ApiService or in a standalone processor service?
  - What exact handoff contract should connect scraper output to processor work intake?
  - Which AI provider and policy taxonomy should govern identification, finish classification, and content moderation?
  - What confidence thresholds should require human review before downstream use?
depends_on:
  - newversion-scraper-scraper-scaffolding
blocks: []
updated_at: 2026-04-11T11:14:27Z
---

# Product Brief: Processor Storage Foundation

## Executive Summary

PrintingMagic can already scrape STLFlix collections and model metadata, but it still cannot turn those raw assets into sellable, reviewable catalog entries. The missing layer is not just storage in the narrow sense. It is a processor-owned product intelligence foundation that preserves evidence, captures AI-driven interpretation, and prepares each model for downstream display and listing workflows.

`newversion-processor-storage` fills that gap. It establishes the durable storage and product-record shape needed to ingest model zips, retain extracted image evidence, support visual classification, and persist the fields that matter commercially: what the product is, whether it is painted or multicolor, which image should be primary, which gallery images belong together, what copy should describe the item, and whether the product should be held back for policy reasons. This feature creates the foundation the rest of the processor pipeline can build on without forcing future UI and listing services to reprocess raw assets.

## The Problem

Today the scraper can discover products, but discovery alone does not create a usable catalog. The system has no durable place to hold the visual evidence, AI judgments, confidence scores, or merchandising decisions that sit between "model found" and "product ready." As a result, several critical outcomes remain impossible or unsafe:

- a future UI cannot reliably show the right hero image, gallery order, finish badges, or generated copy,
- a future listing service cannot consume structured product attributes without reinterpreting raw assets,
- low-confidence or policy-sensitive results cannot be separated from trustworthy ones,
- model improvements and prompt changes cannot be replayed cheaply because the evidence trail is not preserved.

Without this feature, scraped models remain raw intake, not product records.

## The Solution

Build the processor storage foundation around a canonical processed-model record backed by durable evidence. For each scraped model, the system should preserve the original asset package and extracted images, then store the interpretation outcomes that future services actually need.

That means the product record must be able to represent:

- product identity inferred from the images,
- generated product copy suitable for selling,
- finish classification such as painted vs multicolor print,
- compliance status and reasons for concern,
- hero image selection,
- coherent gallery selection by background family,
- ordered gallery metadata,
- confidence and provenance for AI-derived fields,
- enough structure to support future projections for both UI and listing use cases.

This is intentionally a foundation feature. It does not need to solve every downstream workflow now, but it must define the data model strongly enough that those workflows can be added without redoing the storage layer.

## Who This Serves

### Primary Internal Users

- The lead operator needs a trustworthy processing backbone so scraped models can become usable product candidates instead of raw imports.
- Future UI consumers need display-ready product records with selected imagery, generated copy, and confidence-aware badges.
- Future listing-service consumers need structured commercial attributes and policy-aware readiness signals.
- Future human reviewers need preserved evidence and confidence metadata so ambiguous or flagged products can be corrected safely.

### Indirect End Benefit

End customers eventually benefit through cleaner listings, more relevant product copy, more consistent galleries, and fewer unsafe or low-quality products reaching publication.

## Why This Approach

### Evidence-Preserving By Design

The processor should keep the original zip and extracted images as durable evidence. That makes reprocessing practical when prompts improve, policies change, or a human reviewer overrides a result.

### Confidence-Aware Instead Of Pretending Certainty

AI outputs are not deterministic facts. Storing confidence and provenance allows the system to separate publishable results from review-required ones.

### Compliance Before Merchandising

Policy screening belongs early in the flow. That avoids wasting effort generating copy and curated galleries for products that should never proceed.

### Downstream-Ready Without Downstream Coupling

The processor should own one canonical enriched record and project it outward. That gives future UI and listing services stable inputs without forcing them to share one rigid schema or redo image interpretation work.

## Success Criteria

1. Each imported model can be represented as a durable processed-product record with preserved evidence references.
2. The storage model can persist merchandising-critical outputs: identity, copy, finish state, compliance status, hero image, selected gallery, and gallery order.
3. AI-derived fields can store confidence and provenance so low-confidence results are distinguishable from trusted results.
4. Downstream consumers can retrieve display-ready and listing-ready data without reopening raw zips or rerunning visual analysis.
5. The design leaves a clear path for future human review, reprocessing, and policy evolution.

## Scope

### In Scope

- defining the durable storage foundation for processed models,
- shaping the canonical enriched product record,
- preserving evidence needed for visual interpretation and future reprocessing,
- storing the results needed by future UI and listing flows,
- establishing the role of confidence, provenance, and compliance state in the data model.

### Out Of Scope For This Brief

- final selection of AI provider or prompt design,
- reviewer UI implementation,
- listing publication workflow,
- full technical architecture for queues, migrations, or deployment details,
- the complete business-plan or tech-plan decisions that follow this brief.

## Risks And Open Questions

- The biggest architectural decision still open is whether enriched persistence belongs behind ApiService or inside a standalone processor service.
- The scraper-to-processor handoff contract needs to be made explicit so the storage model is shaped around the real intake boundary.
- Compliance categories need to be defined more clearly than "weapons or other violations" before implementation work begins.
- The team needs a policy for what can flow downstream automatically versus what must wait for human review.
- Upstream STLFlix scraping still carries its own legal review gate, which remains adjacent to but separate from this feature.

## Vision

If this feature succeeds, PrintingMagic gains a reusable product intelligence layer rather than a one-off schema. Scraped assets become evidence-backed product records that can power future storefront UI, listing automation, and reviewer tooling. Instead of each downstream service rediscovering what a model is and whether it is safe to publish, the processor becomes the single place where that interpretation is performed, stored, and improved over time.---
feature: newversion-processor-storage
doc_type: product-brief
status: draft
goal: "Scaffold a production-ready data and blob storage layer for the PrintingMagic processor — enabling persistent enriched model records, processing state tracking, and image asset storage in the newversion Aspire architecture"
key_decisions:
  - "Dedicated processor-db (PostgreSQL) — isolated from scraper-db, registered as a new Aspire database resource on the shared postgres container"
  - "Azurite for local dev blob storage, Azure.Storage.Blobs SDK for production-compatible client"
  - "EF Core with Npgsql provider — matches scraper pattern, full ORM for entity management and migrations"
  - "Processing state as columns on the Model entity (not a separate audit table) — lean initial design"
  - "Inline EF Core migration at processor startup (no separate MigrationService project)"
  - "Processor service is a standalone PrintingMagic.Processor Aspire project — NOT embedded in ApiService"
open_questions:
  - "Confirm whether model-level import is handled via the collection batch endpoint or a separate POST /import/model endpoint"
  - "Confirm blob storage container naming convention (e.g., 'model-assets' or 'collections')"
  - "Should the processor read unprocessed models from processor-db directly, or consume RabbitMQ events from the scraper?"
depends_on:
  - newversion-scraper-scraper-scaffolding
blocks: []
updated_at: "2026-04-10T18:30:00Z"
---

# Product Brief: newversion-processor-storage

*Authored by Mary — Strategic Business Analyst | PrePlan phase*

---

## Vision

The `newversion-processor-storage` feature delivers the **data persistence and blob storage scaffold** for a new standalone `PrintingMagic.Processor` service in the newversion Aspire architecture. When this feature is complete, the processor service will have a fully operational database (EF Core, PostgreSQL, Npgsql), a blob storage layer (Azurite locally, Azure Blob in production), and the foundational entity schema needed to receive scraped collection data, persist enriched model records, and track processing state — ready for the enrichment pipeline logic to be layered on top in subsequent features.

This is a **pure scaffold feature**. It produces working infrastructure; it does not implement scraping, Playwright automation, or business logic. Its success criterion is: the processor service starts inside Aspire, connects to its database, runs migrations, connects to blob storage, and can receive a `CollectionImportRequest` payload and persist it.

---

## Background

The PrintingMagic newversion rewrite uses .NET Aspire to orchestrate independent microservices. The scraper service (delivered) harvests STLFlix collection and model metadata and publishes it to an internal API endpoint. The data it delivers — `CollectionSeed` records — is the raw input for the enrichment pipeline.

Enrichment means: visiting each model's source URL via a headless browser to extract page text, downloading cover images, packaging them as a ZIP archive, uploading to blob storage, and recording the blob URL. The oldversion `ModelAutomationService` is the operational reference for this behaviour. The newversion must replicate it as a first-class Aspire service.

Before enrichment logic can be written, the **storage layer must exist**. This feature establishes it.

---

## Target Users / Stakeholders

| Stakeholder | Need |
|-------------|------|
| Developer implementing processor enrichment (next feature) | A working DbContext, entity types, and blob client to write against — no ambiguity about schema or storage details |
| Aspire AppHost | A registered `processor-db` database and `blob-storage` resource with correct references wired to the processor service |
| End users (PrintingMagic platform) | Indirectly — model enrichment (page text, images) powers the product catalog; this storage layer is the prerequisite to that value |

---

## Scope

### In Scope

1. **New Aspire project:** `PrintingMagic.Processor` .NET 9 worker/background-service project, registered in the AppHost via `aspire add` or equivalent project reference
2. **Aspire AppHost wiring:**
   - `processor-db` → new PostgreSQL database on the shared `postgres` container
   - `blob-storage` → Azurite Azure Blob emulator container
   - Processor service references both and waits for them to be ready
3. **EF Core DbContext:** `ProcessorDbContext` with the following entities:
   - `Collection` (Id, SourceUrl, Title, Description, CoverPhotoUrl, PublishedUtc, PlatformCode, DiscoveredUtc, ImportedUtc)
   - `Model` (Id, CollectionId, SourceUrl, Title, CoverPhotoUrl, DiscoveredUtc, ImportedUtc, Status: `Pending|Processing|Complete|Failed`, ProcessingStartedUtc, ProcessingCompletedUtc, ProcessingError, PageTitle, PageText, ImagesZipBlobUrl)
   - `Photo` (Id, ModelId, SourceUrl, BlobPath, DownloadedUtc) — for individual image tracking
4. **EF Core migrations:** Initial migration scaffolded and auto-applied at processor startup
5. **Blob storage client:** `IBlobStorageService` interface + Azurite-backed implementation, registered in DI; wraps `BlobServiceClient` from `Azure.Storage.Blobs`; blob path convention: `collections/{collectionId}/models/{modelId}/images.zip`
6. **Import endpoint (on Processor or ApiService):** `POST /import/collection` receives `CollectionImportRequest`, maps `CollectionSeed` records to `Collection` + `Model` entities, persists to `processor-db` with `Status = Pending`, returns 200 with count of new records created (deduplication by `SourceUrl`)
7. **BDD test scaffold:** xUnit test project (`PrintingMagic.Processor.Tests`) with at minimum:
   - `GivenSolutionBuilt_WhenProcessorProjectInspected_ThenWorkerDerivesFromBackgroundService`
   - `GivenEmptyProcessorDb_WhenMigrationRuns_ThenCollectionAndModelTablesExist`
   - `GivenValidImportRequest_WhenImportEndpointCalled_ThenCollectionsPersistedWithPendingStatus`
8. **Aspire integration test:** Processor service starts and is reachable (satisfies the newversion domain constitution hard gate)

### Out of Scope

- Playwright/headless browser enrichment logic (next feature: `newversion-processor-enrichment`)
- RabbitMQ consumer (next feature after scaffold)
- Admin UI or API for reviewing enriched content
- Azure Blob Storage production configuration (secrets management, connection strings)
- `PrintingMagic.ApiService` changes (unless the import endpoint lives there — see open question)

---

## Success Criteria

| # | Criterion | Validation |
|---|-----------|-----------|
| 1 | `PrintingMagic.Processor` project exists, builds, and registers in Aspire AppHost | `aspire run` starts without errors; processor service appears in Aspire dashboard |
| 2 | `processor-db` PostgreSQL database is provisioned and connected | EF Core migrations run cleanly on startup; `Collection` and `Model` tables exist |
| 3 | Azurite blob storage container is reachable from processor | Health check passes; `BlobServiceClient` can create and list containers |
| 4 | `POST /import/collection` accepts a `CollectionImportRequest` and persists records | BDD test verifies 200 response + DB rows with `Status = Pending` |
| 5 | Deduplication works — re-importing same `SourceUrl` returns 200, 0 new records, no error | BDD test validates upsert/skip behaviour |
| 6 | All xUnit BDD tests pass in CI | `dotnet test` exits 0 |
| 7 | Aspire integration test confirms processor starts and is reachable | `DistributedApplicationTestingBuilder` test passes |

---

## Key Constraints

| Constraint | Source |
|-----------|--------|
| .NET 10 target framework | `PrintingMagic.Data.Contracts` already targets net10.0 |
| PostgreSQL via Npgsql — no SQL Server | Established by scraper pattern; domain constitution allows full track |
| Aspire service isolation — dedicated `processor-db` | Architectural principle; scraper-db precedent |
| BDD tests mandatory (hard gate) | newversion domain constitution §Domain-Level TDD/BDD Additions |
| Aspire integration test mandatory (hard gate) | newversion domain constitution §Aspire Integration Tests |
| No direct DB access from Scraper → processor-db | ADR-5 from scraper: all cross-service data via HTTP, not shared DB access |

---

## Dependencies

| Feature / Service | Relationship | Status |
|------------------|-------------|--------|
| `newversion-scraper-scraper-scaffolding` | Delivers `CollectionImportRequest` / `CollectionSeed` contracts; AppHost PostgreSQL + RabbitMQ resources — both reused | Archived / complete |
| `newversion-apphost-scaffolding` | AppHost must accept a new project reference and new Aspire resources | In preplan |
| `PrintingMagic.Data.Contracts` (scraper project) | `CollectionSeed` and `CollectionImportRequest` types must be accessible to processor | Shared project (currently under scraper/ — evaluate whether to move to a shared contracts repo) |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| `PrintingMagic.Data.Contracts` is in the scraper project repo — processor can't reference it without circular repo dependency | **High** | High | Evaluate moving to a shared `PrintingMagic.Contracts` NuGet package or a shared project directory before implementation begins |
| Import endpoint ownership unclear (ApiService vs. Processor) — causes scope conflict with `newversion-apphost-scaffolding` | Med | Med | Resolve in product-brief review: Processor owns the import endpoint |
| Azurite Aspire hosting package version compatibility with net10.0 AppHost | Low | Low | Verify in story 1; fallback to Docker-compose Azurite if Aspire package unavailable |
| Processor polling `processor-db` at startup before migrations finish | Med | Low | `WaitFor(processorDb)` + inline `MigrateAsync()` in hosted service startup eliminates race |

---

## Track and Timeline

| Field | Value |
|-------|-------|
| Track | full |
| Phase | preplan |
| Priority | medium |
| Next phase | businessplan |

This feature follows the **full lifecycle track** — businessplan, techplan, adversarial review, sprintplan, BDD tests, and dev stories are all required before implementation begins.

---

## Handoff Notes for BusinessPlan

1. **Resolve the `PrintingMagic.Data.Contracts` ownership question first** — this is the most likely implementation blocker and should be a businessplan ADR.
2. Confirm import endpoint location (Processor vs. ApiService) as a point of the businessplan architecture section.
3. The blob storage blob-path convention from the oldversion (`collections/{id}/models/{id}/images.zip`) should be preserved to reduce migration risk if data movement between old and new becomes necessary.
4. `newversion-apphost-scaffolding` may need to close or be folded into this feature's apphost changes — coordinate with that feature owner.
