# Product Brief: newversion-processor-storage

**Feature:** `newversion-processor-storage`  
**Service:** `newversion/processor`  
**Prepared by:** Mary (bmad-agent-analyst)  
**Phase:** preplan  
**Date:** 2026-04-10  

---

## Executive Summary

The `processor` service transforms raw scraped model records (zip archives + page metadata produced by the `scraper` service) into fully enriched, sale-ready product listings through an AI-powered visual and text analysis pipeline. The `storage` feature defines exactly what data is produced, how it is persisted, and under what contract it is exposed to the two downstream consumers: the **UI display service** and the **product listing service**.

---

## Problem

The scraper reliably harvests 3D model listings and stores their photos in zip archives. However, the raw data alone is insufficient to list products for sale or display them meaningfully to customers:

- There is no auto-generated product title, description, or tags
- No photo has been selected as the primary / hero image
- Photos with mixed backgrounds make listings visually inconsistent; no background type classification exists
- No determination exists of whether a model is hand-painted vs. a multicolor print
- No ToS screening has been applied (weapons, prohibited content will exist in the raw data)
- Downstream services cannot consume scraper data directly — they need a clean, enriched contract

Without this pipeline, no product can move from "scraped" to "listable."

---

## Goals

| # | Goal | Success Metric |
|---|---|---|
| G1 | All scraped models are automatically enriched without human input | 100% of new `ModelImportDto` records trigger processing |
| G2 | AI-generated listing copy is good enough for direct publishing | Listing quality score ≥ threshold on first pass (baseline TBD via review) |
| G3 | Primary photo and background-filtered photo set selected per model | `IsPrimary` set on exactly one photo; `DisplayOrder` populated for all included photos |
| G4 | Finish type identified (painted vs. multicolor print) | `IsPainted` populated on all processed photos and model |
| G5 | ToS violations surfaced for human review — never auto-published | All flagged models reach `NeedsReview` status + `ModerationCase` created; no auto-rejection |
| G6 | Data ready for UI + listing service consumption with no further AI calls | ProcessedModel API returns all required fields; downstream services need no AI access |

---

## Non-Goals (this feature)

- **DerivedAsset generation** (Hero/Thumbnail renders) — post-processing, separate feature
- **UI display service** — separate service / feature
- **Product listing service** (marketplace publishing) — separate service / feature
- **Human review workflow** (moderation UI) — separate feature
- **Re-processing / reprocessing triggers** — v2 concern; MVP is one-pass per model
- **BackgroundType beyond white/black/other** — lifestyle/studio scene classification is future scope

---

## Users / Consumers

| Consumer | Relationship | What they need from processor storage |
|---|---|---|
| **UI display service** | Future downstream | Photos by background + display order; primary photo URL; listing copy; `PolicyState`; `IsPainted` facet |
| **Product listing service** | Future downstream | `ListingTitle`, `ListingDescription`, `Tags`; `Category`; `DominantBackground`; `PolicyState === Compliant` gate; `ProcessStatus === Ready` gate |
| **Moderation team (future)** | Human actors via future UI | `ModerationCase` list; `PolicyState`; reason + severity for each flagged model |
| **cweber (operator)** | Developer / operator | `ProcessStatus`, `ProcessingJobId`, error details for observability |

---

## Core Capabilities Required

### 1. Pipeline Orchestration
Accept a trigger for each new `ModelImportDto`, orchestrate all enrichment stages end-to-end, and persist the result atomically. Support configurable concurrency (Aspire config, not hardcoded).

### 2. Zip Extraction & Photo Storage
Download `ImagesZipBlobUrl`, extract photos, store to blob storage under processor-owned naming convention, and register each photo record.

### 3. AI Product Identification
Use vision AI to answer: "What is this product?" Populate `Category`, `AiIdentificationSummary`, inform copy generation. Must handle figurines, terrain pieces, tools, props, NSFW-adjacent items, etc.

### 4. Listing Copy Generation
From vision context + `PageText` from the scraper: generate `ListingTitle`, `ListingDescription`, `Tags[]`. Output must be suitable for direct use by product listing service (i.e., not a draft).

### 5. Primary Photo Selection
Rank photos by visual quality and representativeness. Exactly one photo per model is marked `IsPrimary = true`.

### 6. Background Type Classification
Per-photo: classify background as `White`, `Black`, `Mixed`, or `Other` with a confidence score. Then select the `DominantBackground` for the model (the background type with the most consistently clear photos) and apply `DisplayOrder` to photos of the dominant type.

### 7. Finish Detection
Determine if the model is hand-painted (`IsPainted = true`) or multicolor-print-only (`IsPainted = false`). Set at both photo level and model level.

### 8. ToS Screening
Identify content that violates terms of service (weapons, prohibited objects, IP violations). Never reject automatically. Set `PolicyState = NeedsReview` and create a `ModerationCase` record for each concern.

### 9. Persistence & Status Management
Store all enriched data in a processor-owned DB. Manage `ProcessStatus` transitions. Expose processed data via internal API for downstream consumption.

---

## Data Contract (Key Entities)

### ProcessedModel
The top-level enriched product record.

| Field | Type | Source |
|---|---|---|
| `Id` | int | processor |
| `ScrapedModelId` | int | FK → scraper |
| `ProcessingJobId` | Guid | processor |
| `Status` | `ProcessStatus` enum | processor |
| `Category` | string | AI |
| `AiIdentificationSummary` | string | AI (raw vision output, auditable) |
| `ListingTitle` | string | AI |
| `ListingDescription` | string | AI |
| `Tags` | string[] | AI |
| `IsPainted` | bool | AI |
| `IsViolent` | bool | AI |
| `PolicyState` | `PolicyState` enum | AI |
| `DominantBackground` | `BackgroundType` enum | AI |
| `PrimaryPhotoId` | int | AI |
| `ListingQualityScore` | float? | AI |
| `ContentNotes` | string? | AI |
| `AiSource` | string | processor |
| `ProcessedUtc` | DateTimeOffset | processor |

### ProcessedPhoto
One record per photo in the zip archive.

| Field | Type | Source |
|---|---|---|
| `Id` | int | processor |
| `ProcessedModelId` | int | FK |
| `SourceFileName` | string | zip |
| `BlobUrl` | string | blob storage |
| `OrderInSource` | int | zip ordering |
| `DisplayOrder` | int | AI |
| `IsPrimary` | bool | AI |
| `IncludeInListing` | bool | AI |
| `IsPainted` | bool | AI |
| `IsViolent` | bool | AI |
| `BackgroundType` | `BackgroundType` enum | AI |
| `BackgroundScore` | float | AI |
| `QualityScore` | float | AI |
| `Description` | string? | AI |

### New Enums
- `BackgroundType`: `White, Black, Mixed, Other`
- `ProcessStatus` (inherited semantics): `Unprocessed, Imported, Classified, Ready, Published, NeedsReview, EnrichmentFailed`
- `PolicyState` (inherited semantics): `Compliant, NeedsReview, Rejected`

---

## Architecture Decisions

| Decision | Choice | Rationale |
|---|---|---|
| DB isolation | Processor-owned dedicated DB | Clean service boundary; does not repeat old-version shared-DB coupling |
| Trigger mechanism | Message queue consumer (Service Bus / RabbitMQ) | Decoupled from scraper; scraper publishes event on new `ModelImportDto` |
| AI provider | Azure OpenAI GPT-4o Vision (primary) | Strong structured JSON output; handles multi-image prompts per model |
| Concurrency control | Aspire config-driven (`ProcessorSettings:MaxConcurrency`) | Not hardcoded; operator-tunable per environment |
| ToS handling | `NeedsReview` + `ModerationCase` only — no auto-reject | Safety-first; false positives must not suppress valid products |
| Background selection strategy | `DominantBackground` = type with highest count of high-confidence photos | Conservative; ties broken by `WhiteBackground > BlackBackground > Mixed` |

---

## Open Questions for Business Plan Phase

1. **Queue broker selection**: Azure Service Bus vs. RabbitMQ? Aspire has native support for both.
2. **Blob container naming**: Processor-owned container per model, or per collection?
3. **AI call batching**: All photos in one multi-image prompt, or sequential per-photo? (cost/quality tradeoff)
4. **ListingQualityScore definition**: What score threshold gates `ProcessStatus = Ready` vs. `NeedsReview`?
5. **ModerationCase severity tiers**: What are the severity levels? (e.g., `Low = ContentWarning, High = PossibleWeapon, Critical = ExplicitViolation`)
6. **Idempotency policy**: Can a model be re-processed? What changes if re-processed (overwrite vs. version)?
7. **IncludeInListing rule**: Is every non-violent photo with matched background included, or is there a maximum count per listing?

---

## Dependencies

| Dependency | Direction | Status |
|---|---|---|
| `newversion/scraper` — `ModelImportDto` contract | Upstream input | Implemented (scraper service exists) |
| `newversion/scraper` — `ImagesZipBlobUrl` blob | Upstream input | Implemented |
| Azure Blob Storage (processor-owned) | Infrastructure | Provisioned by Aspire AppHost |
| Message broker | Infrastructure | TBD (see open questions) |
| Azure OpenAI Vision | External AI | Needs provisioning / key management |
| `newversion/processor` AppHost registration | Infrastructure | TBD (processor service not yet in AppHost) |

---

## Definition of Done (for preplan phase)

- [x] `brainstorm.md` committed to governance docs
- [x] `product-brief.md` committed to governance docs
- [ ] Open questions logged (above)
- [ ] Phase advanced to `businessplan` in `feature.yaml`
