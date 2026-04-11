---
feature: newversion-processor-storage
doc_type: brainstorm
status: draft
goal: "Define the processor storage and enrichment problem space for AI-driven model analysis, compliance screening, photo curation, and downstream-ready persistence"
key_decisions:
  - Treat each model zip as a durable product-processing unit rather than an ephemeral archive
  - Use image evidence as the primary source for product identification, finish classification, photo selection, and policy checks
  - Persist a canonical processed-model record that can be projected to future UI and listing-service contracts
  - Capture AI-derived outputs with confidence and provenance so uncertain results can be reviewed or reprocessed
  - Gate merchandising work behind early compliance screening
open_questions:
  - Is the persistence boundary for this feature inside ApiService or in a standalone processor service?
  - What exact handoff contract should connect scraper output to processor work intake?
  - Which blob and relational storage primitives should the processor standardize on in the Aspire environment?
  - Which AI provider will own product identification, finish classification, and compliance decisions?
  - What thresholds should route outputs to future human review instead of automatic downstream use?
depends_on:
  - newversion-scraper-scraper-scaffolding
blocks: []
updated_at: 2026-04-11T11:09:53Z
---

# Brainstorm: newversion-processor-storage

## Purpose

This brainstorm defines the product and storage problem space for the processor layer that follows the scraper. The goal is to turn raw model assets into durable, reviewable product intelligence that can support a future UI service and a future product-listing service.

## Context Snapshot

The upstream scraper is already responsible for harvesting STLFlix collection and model metadata and posting internal import payloads to `PrintingMagic.ApiService`. Governance artifacts confirm that the scraper publishes `CollectionImportRequest` and `ModelImportDto` payloads through existing `/import/collection` and `/import/model` endpoints and does not own the downstream persistence model for enriched product data.

That leaves a clear gap: the system can discover models, but it still lacks a storage and processing foundation for what happens after a model has been scraped.

## Core Need

For each model zip, the system must be able to:

1. preserve the original assets,
2. inspect images visually,
3. determine what the product is,
4. classify whether the item is painted or a multicolor print,
5. detect terms-of-service concerns such as weapons or other restricted content,
6. choose the best primary photo,
7. select the best coherent image set by background,
8. generate product copy suitable for selling,
9. store all of that in a form that future services can consume without reprocessing.

## First-Principles Findings

### Product Assets Are Evidence

The zip and its extracted images are not incidental files. They are the evidence base for every important downstream decision. The processor should therefore preserve original evidence durably instead of treating extraction as a transient step.

### Images Matter More Than Titles

Visual evidence should be treated as the primary source of truth for product identity, finish classification, background grouping, hero-image selection, and compliance screening. Scraped titles and URLs help with context, but the photos are what the buyer will see and what the system must judge.

### The System Has Two Downstream Consumers

The future UI service needs rich display data such as ordered images, hero image, formatted copy, and confidence-backed badges. The future listing service needs structured commercial data such as classification, flags, finish state, and selected image order. The processor should store a canonical record that can project into both shapes.

### AI Outputs Are Probabilistic

Identification, copy generation, finish classification, and moderation are all probabilistic outputs. The storage model therefore needs confidence, provenance, and reprocessing metadata rather than only storing final labels.

### Compliance Is A Gate

If a product appears to violate policy, the system should stop merchandising work early. Compliance should not be an afterthought added after copy generation and image curation.

## Brainstorm Themes

### Durable Evidence And Reprocessing

- Preserve the original zip and extracted images in durable storage.
- Keep enough provenance to rerun improved prompts or models later without depending on a fresh scrape.
- Avoid designs that lose evidence after a single processing attempt.

### Canonical Product Record

- Store one processor-owned canonical record for the enriched model.
- Represent child concepts separately: images, generated copy, compliance findings, and processing attempts or status.
- Use projections for downstream consumers instead of maintaining multiple drifting data blobs.

### Photo Curation As Grouping And Ranking

- Background detection is not just a label; it is a selection mechanism.
- The processor should identify coherent image families such as white-background, black-background, or other recognizable sets.
- The chosen hero image should come from the selected family, and the rest of that family should receive an explicit listing order.

### Finish And Merchandising Attributes

- Painted vs multicolor print should be stored as a typed commercial attribute, not a free-text comment.
- The system should retain evidence references or explanation fields for why the finish was classified that way.
- Product copy generation should consume the visual classification results rather than operating independently of them.

### Reviewability And Operations

- Low-confidence results should be easy to route into a future review workflow.
- Processing failures must be visible and recoverable rather than silently dropped.
- The design should support reprocessing after prompt changes, model upgrades, or manual corrections.

## Candidate Pipeline

```text
scraper handoff
  -> processor intake
  -> durable zip persistence
  -> image extraction
  -> early compliance screening
  -> product identification
  -> finish classification
  -> background grouping and photo ranking
  -> product copy generation
  -> canonical processed-model persistence
  -> downstream projections for UI and listing services
  -> optional future review queue for low-confidence cases
```

## Option Space Still Open

Several implementation-shaping questions came out of the session and should be settled in follow-on planning:

- whether the persistence boundary belongs in ApiService or in a standalone processor service,
- what storage topology best fits the Aspire deployment model,
- whether work intake should remain simple and pull-based at first or move toward a queue-driven design,
- how much audit history belongs in the initial schema versus a follow-on feature.

These are real planning questions, not blockers to understanding the product need.

## Risks To Design Against

- Running compliance late would waste cost and create unsafe downstream artifacts.
- Throwing away confidence scores would make automation unsafe and human review impossible to prioritize.
- Coupling processor storage directly to one downstream service would make later service separation expensive.
- Treating images as database blobs or disposable temp files would make both queryability and reprocessing worse.
- Silently dropping failed processing attempts would create invisible catalog gaps.

## Outcome

The processor-storage feature is not only a schema task. It is the foundation for a catalog-intelligence pipeline that must preserve evidence, support AI-assisted interpretation, surface uncertainty, and prepare product data for future presentation and listing flows. The product brief should position this feature as the enabling layer that turns raw scraped models into reviewable, sellable product records.
