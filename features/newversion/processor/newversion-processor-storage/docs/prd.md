---
feature: newversion-processor-storage
doc_type: prd
status: draft
goal: "Establish a standalone processor service with a shared data-contracts library that turns scraped model assets into durable, evidence-preserving, downstream-ready product records"
key_decisions:
  - Enriched persistence lives in a standalone processor service — not behind ApiService
  - A common data-contracts library owns the scraper-to-processor handoff types (shared NuGet package)
  - AI enrichment (identification, copy generation, finish classification, compliance screening) is deferred to a future feature; this feature provisions the storage foundation and record shape that the AI feature will populate
  - AI confidence and provenance fields are modeled in the storage schema to avoid a later breaking migration when the AI feature ships
depends_on:
  - newversion-scraper-scraper-scaffolding
blocks: []
open_questions:
  - Which AI provider will govern identification, finish classification, and compliance? (deferred to AI enrichment feature)
  - What confidence thresholds should route outputs to human review vs automatic downstream use? (deferred to AI enrichment feature)
  - What is the deployment topology for the processor service inside the Aspire AppHost? (TechPlan)
  - What migration strategy governs the EF Core schema in a team-facing Aspire environment? (TechPlan)
stepsCompleted:
  - step-01-init
  - step-02-discovery
inputDocuments:
  - docs/newversion/processor/newversion-processor-storage/brainstorm.md
  - docs/newversion/processor/newversion-processor-storage/product-brief.md
workflowType: prd
updated_at: 2026-04-11T00:00:00Z
---

# Product Requirements Document: Processor Storage Foundation

**Feature:** `newversion-processor-storage`  
**Author:** cweber  
**Date:** 2026-04-11  
**Status:** Draft

---

## 1. Overview

`newversion-processor-storage` establishes the durable storage backbone for the PrintingMagic processor pipeline. The scraper already discovers STLFlix models and delivers import payloads to ApiService; this feature builds what comes next — a standalone processor service that receives those imports and produces canonical, evidence-preserving product records ready for display and listing workflows.

This is a foundation feature, not an end-to-end one. It defines the data model, the service boundary, and the intake contract so that future AI enrichment (visual classification, copy generation, compliance screening) can populate and extend the record without redoing the storage layer.

A **common data-contracts library** (`PrintingMagic.Data.Contracts`) is introduced in this feature to publish the shared types that cross the scraper/processor boundary, making the intake contract explicit and versioned.

---

## 2. Problem Statement

Today the scraper delivers `CollectionImportRequest` and `ModelImportDto` payloads, but no durable processor-owned record exists to receive them. Without one:

- **No traceable evidence** — extracted images and zip contents disappear after scrape; reprocessing requires re-scraping.
- **No enrichment home** — there is nowhere to persist visual classification results, generated copy, compliance status, hero image selection, or gallery ordering.
- **Unclear service boundary** — whether enriched persistence belongs in ApiService or a dedicated processor is unresolved, creating architectural drift risk.
- **No downstream contract** — the future UI and listing service cannot build against a concrete product record because none exists.

Every day this gap is open, new models accumulate as raw imports rather than usable catalog entries.

---

## 3. Goals and Non-Goals

### Goals

1. Establish a **standalone processor service** (`PrintingMagic.Processor`) that owns enriched product persistence.
2. Define a **canonical processed-product record** with first-class fields for identity, copy, finish state, compliance, hero image, gallery selection, and gallery order.
3. Create a **`PrintingMagic.Data.Contracts` common library** that holds the shared intake contract types used across the scraper and processor boundary.
4. **Preserve source evidence** — original zip and extracted images stored durably so future AI or human re-review does not require a fresh scrape.
5. **Model AI-readiness** — confidence and provenance fields built into the schema now so the forthcoming AI enrichment feature can populate them without a migration.
6. **Register the service in the Aspire AppHost** so it participates in the local dev environment.

### Non-Goals

- AI enrichment execution (visual identification, classification, copy generation, moderation) — this is a separate future feature.
- Human review UI — separate future feature.
- Product listing publication — separate future feature.
- Defining AI provider selection or prompt design.
- Building the full queue-based processing pipeline — intake mechanics may be simplified for this foundation (TechPlan decision).

---

## 4. User Stories

### 4.1 Operator — Processing Backbone

> As the lead operator, I need scraped model imports to land in a durable processor-owned record so that models progress from raw intake to product candidates without manual intervention or re-scraping.

**Acceptance Criteria:**
- [ ] Given a `ModelImportDto` is posted to the intake endpoint, When the request is processed, Then a `ProcessedProduct` record is created and tied to the originating scrape import ID.
- [ ] Given a `ProcessedProduct` record exists, When it is retrieved by product ID, Then the original zip URL and extracted image URLs are present on the record.
- [ ] Given a `ProcessedProduct` record has been created, When a `GET /products/{id}` request is made, Then the full record is returned.

### 4.2 Future AI Feature — Writeable Evidence Fields

> As the forthcoming AI enrichment feature, I need confidence and provenance columns to already exist on the product record so I can populate them without issuing a breaking migration.

**Acceptance Criteria:**
- [ ] Given a `ProcessedProduct` record is being created, When the AI enrichment feature populates the provenance fields, Then `confidence`, `ai_provider`, `ai_model`, and `processed_at` are accepted without error and persisted.
- [ ] Given an existing `ProcessedProduct` created without enrichment, When the AI enrichment feature later writes provenance fields, Then the record is updated without requiring a schema migration.

### 4.3 Future UI Consumer — Display-Ready Data Shape

> As the future UI service, I need to query a complete product record that includes hero image, gallery metadata, finish classification, and generated copy so I can render listings without re-processing assets.

**Acceptance Criteria:**
- [ ] Given a `ProcessedProduct` record has been enriched, When a `GET /products/{id}` request is made, Then `hero_image_url`, `gallery_images` (ordered), `finish_type`, `copy_title`, `copy_description`, and `compliance_status` are all present in the response.
- [ ] Given a `ProcessedProduct` with no enrichment, When a `GET /products/{id}` request is made, Then those fields are returned as null and the response is still valid.

### 4.4 Future Listing Consumer — Structured Commercial Attributes

> As the future listing service, I need structured commercial attributes and a policy readiness signal so I can determine whether a product is safe to publish without reopening asset files.

**Acceptance Criteria:**
- [ ] Given a `ProcessedProduct` record exists, When a listing service queries it, Then `finish_type` is a typed enum value (`painted`, `multicolor_print`, or `unknown`) — not a free-text string.
- [ ] Given a `ProcessedProduct` record exists, When a listing service queries it, Then `compliance_status` is one of `clear`, `flagged`, `pending_review`, or `blocked`.
- [ ] Given a `ProcessedProduct` record has `compliance_status = clear` and all required fields populated, When the listing service reads `listing_ready`, Then it is `true` without the listing service needing to reopen any raw asset files.

### 4.5 Contracts Library — Shared Intake Types

> As a developer working on both the scraper and processor services, I need a shared library for intake contract types so that payload changes propagate without diverging copies.

**Acceptance Criteria:**
- [ ] Given `PrintingMagic.Scraper` references `PrintingMagic.Data.Contracts`, When `ModelImportDto` is used in the scraper, Then no local duplicate type definition exists in the scraper project.
- [ ] Given `PrintingMagic.Processor` references `PrintingMagic.Data.Contracts`, When `ModelImportDto` is used in the processor, Then no local duplicate type definition exists in the processor project.
- [ ] Given the `PrintingMagic.Data.Contracts` csproj, When it is built in isolation, Then it compiles without referencing any other PrintingMagic project (no circular dependencies).

---

## 5. Functional Requirements

### 5.1 Processor Service (`PrintingMagic.Processor`)

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-01 | Expose an intake endpoint (mechanism TBD in TechPlan — HTTP or queue) that accepts a `ModelImportDto` and creates a `ProcessedProduct` record | Must Have |
| FR-02 | Persist the original asset zip URL and extracted image URLs on the product record | Must Have |
| FR-03 | Expose a `GET /products/{id}` endpoint returning the full product record | Must Have |
| FR-04 | Persist `finish_type`, `compliance_status`, `listing_ready`, `hero_image_url`, `gallery_images`, `copy_title`, `copy_description` as nullable/defaulted fields | Must Have |
| FR-05 | Persist `confidence`, `ai_provider`, `ai_model`, `processed_at` as nullable AI provenance fields | Must Have |
| FR-06 | Support a `processing_status` state machine: `pending`, `processing`, `enriched`, `review_required`, `blocked` | Must Have |
| FR-07 | Register in the Aspire AppHost so the service starts and is reachable in local dev | Must Have |
| FR-08 | Include at least one Aspire integration test validating the service starts | Must Have |
| FR-09 | `gallery_images` stores ordered image entries with `url`, `background_family`, and `sort_order` | Should Have |

### 5.2 Data Contracts Library (`PrintingMagic.Data.Contracts`)

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-10 | Define `ModelImportDto` and `CollectionImportRequest` as the canonical cross-service intake types | Must Have |
| FR-11 | Both `PrintingMagic.Scraper` and `PrintingMagic.Processor` reference `PrintingMagic.Data.Contracts`; no local duplicate type definitions | Must Have |
| FR-12 | Library is structured to be independently packageable (proper csproj, no circular dependencies) | Must Have |

---

## 6. Non-Functional Requirements

| ID | Requirement | Category |
|----|-------------|----------|
| NFR-01 | Services follow the Aspire-based .NET 9 architecture established by `newversion-scraper-scraper-scaffolding` | Architecture |
| NFR-02 | BDD scenarios written in xUnit with Given/When/Then naming (domain constitution requirement — hard gate) | Testing |
| NFR-03 | At least one Aspire integration test using `DistributedApplicationTestingBuilder` (domain constitution requirement — hard gate) | Testing |
| NFR-04 | EF Core migrations are checked in and runnable from a clean state | Data |
| NFR-05 | Intake contract types in `Data.Contracts` are forward-compatible — additive changes only for minor versions | Compatibility |
| NFR-06 | No raw asset re-processing needed to answer any query against an already-persisted record | Performance |
| NFR-07 | The processor service is independently runnable and independently testable from ApiService | Isolation |

---

## 7. Data Model (Conceptual)

```
ProcessedProduct
  id                  : uuid, PK
  scrape_import_id    : string (reference to scraper's import ID)
  collection_id       : string
  model_id            : string
  source_zip_url      : string (durable evidence)
  extracted_images    : ImageEvidence[] (durable evidence)
  processing_status   : enum (pending | processing | enriched | review_required | blocked)

  # Merchandising outputs (nullable — populated by AI enrichment feature)
  product_title        : string?
  copy_title           : string?
  copy_description     : string?
  finish_type          : enum? (painted | multicolor_print | unknown)
  hero_image_url       : string?
  gallery_images       : GalleryImage[]
  compliance_status    : enum? (clear | flagged | pending_review | blocked)
  listing_ready        : bool (computed/stored)

  # AI provenance (nullable — populated by AI enrichment feature)
  confidence           : decimal?
  ai_provider          : string?
  ai_model             : string?
  processed_at         : datetime?

  created_at           : datetime
  updated_at           : datetime

ImageEvidence
  id                   : uuid, PK
  product_id           : uuid, FK
  url                  : string
  extraction_path      : string?

GalleryImage
  id                   : uuid, PK
  product_id           : uuid, FK
  url                  : string
  background_family    : string?
  sort_order           : int
  is_hero              : bool
```

---

## 8. Out of Scope (Explicit Deferrals)

| Item | Deferred To |
|------|-------------|
| AI provider selection and prompt design | Future AI enrichment feature |
| Visual identification, finish classification, copy generation, compliance screening execution | Future AI enrichment feature |
| Confidence thresholds and human-review routing logic | Future AI enrichment feature |
| Review UI for flagged products | Future review feature |
| Listing publication workflow | Future listing feature |
| Queue/messaging infrastructure selection | TechPlan |

---

## 9. Success Metrics

1. Each scraped model import can be round-tripped into a `ProcessedProduct` record with preserved evidence references — verifiable via integration test.
2. A `GET /products/{id}` call returns all merchandising and provenance fields without re-reading raw assets.
3. The `PrintingMagic.Data.Contracts` library is the single definition of intake types — no type duplication verified by build.
4. All hard-gate domain constitution requirements pass: BDD unit tests + Aspire integration test.
5. The AI enrichment feature can populate confidence and provenance fields without issuing a breaking schema migration.

---

## 10. Open Questions and Deferred Decisions

| ID | Question | Owner | Status |
|----|----------|-------|--------|
| OQ-01 | HTTP vs queue-based intake mechanism for processor — synchronous pull from ApiService or async message? | TechPlan | Open |
| OQ-02 | Database selection — same PostgreSQL instance as ApiService or separate? | TechPlan | Open |
| OQ-03 | Processor Aspire registration name and port conventions | TechPlan | Open |
| OQ-04 | AI provider selection (Azure OpenAI, OpenAI, local) | AI feature | Deferred |
| OQ-05 | Confidence thresholds for routing to human review | AI feature | Deferred |

---

## 11. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `newversion-scraper-scraper-scaffolding` | Upstream | Provides `ModelImportDto` / `CollectionImportRequest`; contracts library must match scraper output |
| `PrintingMagic.Aspire` AppHost | Infrastructure | Processor registers here |
| `PrintingMagic.ServiceDefaults` | Infrastructure | Standard Aspire service defaults |
| Future AI enrichment feature | Downstream | Will populate all AI-provenance and merchandising fields |
