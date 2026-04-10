# Brainstorm: newversion-processor-storage

**Feature:** `newversion-processor-storage`  
**Phase:** preplan  
**Author:** Mary (bmad-agent-analyst)  
**Date:** 2026-04-10  

---

## Problem Statement

The scraper service produces `ModelImportDto` records containing page metadata and a `ImagesZipBlobUrl` pointing to a blob-stored zip of all product photos for a 3D model listing. Currently nothing consumes these. A processor service must:

1. Acquire each scraped model's zip, extract its images, and run AI visual analysis across them.
2. Produce enriched, sale-ready metadata for every model so that two downstream consumers — a **UI display service** and a **product listing service** — can read it without doing any additional AI work.

The "storage" story is specifically: *what data gets persisted, in what shape, and queryable by whom.*

---

## Core Pipeline Stages (what needs storing at each step)

| Stage | What happens | New data produced |
|---|---|---|
| **1. Ingest** | Receive `ModelImportDto` trigger; download + unzip blob | `RawPhoto[]` paths in blob storage |
| **2. Product identification** | Vision AI inspects representative photos → "what is this?" | `Category`, `ProductType`, `AiIdentificationSummary` |
| **3. Copy generation** | LLM writes listing copy from vision context + page text | `ListingTitle`, `ListingDescription`, `Tags[]` |
| **4. Primary photo selection** | AI scores photos; highest visual quality / most representative → primary | `Photo.IsPrimary`, `Photo.QualityScore` |
| **5. Background detection** | Per-photo: is background white, black, or other? | `Photo.BackgroundType` (enum), `Photo.BackgroundScore` |
| **6. Background selection** | Which dominant background best represents the product? Filter + rank matching photos | `Model.DominantBackground`, `Photo.DisplayOrder` (within background group) |
| **7. Finish detection** | Is this hand-painted or a multicolor-print-only finish? Visual inspection | `Model.IsPainted`, `Photo.IsPainted` |
| **8. ToS violation detection** | Does any image show weapons, prohibited content, IP violations? | `Photo.IsViolent`, `Model.PolicyState`, `ModerationCase` records |
| **9. Storage** | Persist fully-enriched model + photos; emit event or update status | `ProcessStatus = Ready` (or `NeedsReview` if flagged) |

---

## Existing Domain Model Inventory (old version)

The old `DomainClasses.cs` already encodes most of this. The new processor storage model should deliberately reuse field semantics (the old Worker validated these shapes over time):

**`Model` fields already battle-tested:**
- `ListingTitle`, `ListingDescription`, `Category`, `Tags`, `ContentNotes`
- `IsPainted`, `IsViolent`
- `PrimaryPhotoId`
- `PolicyState` (`Compliant / NeedsReview / Rejected`)
- `ProcessStatus` (`Unprocessed / Imported / Classified / Ready / Published / NeedsReview / EnrichmentFailed`)
- `AiSource`, `ListingQualityScore`, `ScaleCategory`, `SourcePageText`

**`Photo` fields already battle-tested:**
- `IsPrimary`, `IsPainted`, `IsViolent`
- `OrderInSource`, `DisplayOrder`
- `BlobUrl`, `Description`

**`ModerationCase` + `ModerationAction`:** ToS violation tracking, already structured.

**`DerivedAsset`:** Hero + Thumbnail generated assets — relevant for later but out of scope for this feature (that's post-processing generation).

**`PhotoAnalysis` record (old Worker):**
- `FileName, IsPrimary, IncludeInListing, IsViolent, IsPainted`
- Gap: no `BackgroundType` — **this is net-new**

---

## Net-New Fields (not in old domain)

| Entity | New field | Type | Why |
|---|---|---|---|
| `Photo` | `BackgroundType` | enum (`White / Black / Mixed / Other`) | Background selection requires per-photo classification |
| `Photo` | `BackgroundScore` | float (0–1) | Confidence of background classification for tie-breaking |
| `Model` | `DominantBackground` | enum (`White / Black / Mixed / None`) | Which background type was selected for the listing |
| `Photo` | `QualityScore` | float (0–1) | AI primary photo ranking input |
| `Photo` | `IncludeInListing` | bool | Whether this photo should appear in the listing (matches old PhotoAnalysis but wasn't in DomainClasses Photo entity) |
| `Model` | `AiIdentificationSummary` | string | Raw vision model output for what the product is (auditable) |
| `Model` | `ProcessingJobId` | Guid | Traceability back to the processing job that produced this record |

---

## Consumer Contract Design

Two downstream services will query this data. Their requirements drive the schema:

### UI Display Service (future)
Needs:
- All photos grouped by background type with display order
- Primary photo URL
- Human-readable listing copy
- PolicyState to know if model is flagged
- IsPainted flag for "painted models" filter facet

### Product Listing Service (future)
Needs:
- ListingTitle, ListingDescription, Tags
- Category
- DominantBackground (determines which photo set to publish at which marketplace)
- PolicyState === Compliant gate before listing
- ModerationCase records if NeedsReview
- ProcessStatus === Ready gate

---

## Key Open Questions

1. **Trigger mechanism**: Does the processor subscribe to a message queue/event from the scraper (preferred for decoupling) or does it poll the scraper DB? If queue: which broker (Azure Service Bus / RabbitMQ / Aspire-native)?

2. **Blob storage for extracted images**: Where do the unzipped photos live after extraction? Same storage account as scraper zip? Per-model container? Naming convention?

3. **AI provider**: Which vision model? Options:
   - **Azure OpenAI GPT-4o / GPT-4 Vision** — structured JSON output, strong product identification
   - **Azure AI Vision** — cheaper but narrower; might add dedicated ToS classifier
   - **Hybrid**: GPT-4o for identification + copy; dedicated classifier for IsPainted/background detection
   
4. **Single-pass vs. multi-pass AI**: Old Worker did one pass per photo (Playwright + LLM). Can we batch-infer per model with a single multi-image prompt? Cost vs. quality tradeoff.

5. **BackgroundType depth**: Is "white background" sufficient, or do we need finer-grained scene classification (e.g., gradient, studio, lifestyle)? MVP: white/black/other.

6. **ToS ruleset**: Where does the prohibited content taxonomy live? Hardcoded prompt rules? External policy file? This touches the future moderation workflow.

7. **Scale**: How many models will be processed concurrently? Old Worker used `SemaphoreSlim(5)`. New service should expose this as a configurable Aspire setting.

8. **Idempotency**: What happens if the processor is triggered twice for the same model? Should re-processing overwrite, version, or no-op?

9. **ProcessStatus state machine**: Should the processor write intermediate statuses (`Importing → Analyzing → Classifying → Ready`) or just final state? Intermediate states help with failure recovery.

---

## Storage Architecture Options

### Option A: Extend scraper DB (shared DB anti-pattern)
Processor writes enriched fields directly into the scraper's `ScrapedModel` table.
- ❌ Tight coupling; processor is a write-peer of the scraper
- ❌ Violates service boundary

### Option B: Dedicated processor schema in shared DB (tolerable short-term)
Processor owns its own tables (`ProcessedModel`, `ProcessedPhoto`) in the same DB instance (different schema).
- ✅ Simpler infra for MVP
- ⚠️ Still shares physical DB — acceptable for early new version

### Option C: Dedicated DB per service (target architecture)
Processor has its own EF Core DbContext backed by its own DB.
- ✅ Clean DDD service boundary
- ✅ Can migrate independently
- ⚠️ Requires Aspire to provision a second DB resource or separate container
- 🎯 Recommended for this clean-sheet rewrite

### Option D: Event-sourced / projection (overkill for MVP)
- ❌ Out of scope

**Recommendation**: Option C. The new architecture should not repeat old-version shortcut of sharing the DB. Each service gets its own Aspire-provisioned DB. The processor DB contract becomes a first-class versioned API.

---

## Initial Entity Sketch

```
ProcessedModel
  Id (int, PK)
  ScrapedModelId (int, FK → scraper)
  ProcessingJobId (Guid)
  Status (ProcessStatus enum)
  Category (string)
  AiIdentificationSummary (string)
  ListingTitle (string)
  ListingDescription (string)
  Tags (string[])
  IsPainted (bool)
  IsViolent (bool)
  PolicyState (PolicyState enum)
  DominantBackground (BackgroundType enum)
  PrimaryPhotoId (int, FK)
  ListingQualityScore (float?)
  ContentNotes (string?)
  AiSource (string)
  ProcessedUtc (DateTimeOffset)
  UpdatedUtc (DateTimeOffset)

ProcessedPhoto
  Id (int, PK)
  ProcessedModelId (int, FK)
  SourceFileName (string)
  BlobUrl (string)
  OrderInSource (int)
  DisplayOrder (int)
  IsPrimary (bool)
  IncludeInListing (bool)
  IsPainted (bool)
  IsViolent (bool)
  BackgroundType (BackgroundType enum)
  BackgroundScore (float)
  QualityScore (float)
  Description (string?)

ModerationCase
  Id (int, PK)
  ProcessedModelId (int, FK)
  Reason (string)
  Severity (enum)
  CreatedUtc (DateTimeOffset)
  ResolvedUtc (DateTimeOffset?)
  Resolution (string?)
```

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| AI vision misidentifies product category | Medium | Medium | Human review queue (`NeedsReview` status + UI workflow) |
| Background detection incorrect for ambiguous photos | High | Low | Conservative: if confidence < threshold → `Other`, exclude from background group |
| ToS classifier false positive (flags legitimate product) | Medium | High | Never auto-reject; always set `NeedsReview` + ModerationCase, human confirms |
| Processing cost exceeds budget (GPT-4o vision per photo) | Medium | Medium | Batch images per model, cache results, cap retries |
| Zip extraction failure (corrupt archive from scraper) | Low | Medium | Dead-letter queue; set `EnrichmentFailed` status, alert |

---

## Summary

The `newversion-processor-storage` feature is the heart of the new AI enrichment pipeline. It defines the **data contract** that everything downstream depends on. The key design decisions are:

1. **Dedicated DB per service** (Option C) — don't repeat old-version DB coupling mistake
2. **Extend old field semantics** — `IsPainted`, `IsViolent`, `PolicyState`, `DisplayOrder` etc. are battle-proven; adopt them verbatim
3. **Net-new fields**: `BackgroundType`, `BackgroundScore`, `DominantBackground`, `QualityScore`, `IncludeInListing` on photos; `AiIdentificationSummary` on model
4. **Never auto-reject on ToS** — always NeedsReview + ModerationCase for human triage
5. **Configurable concurrency** via Aspire settings (not hardcoded SemaphoreSlim)
