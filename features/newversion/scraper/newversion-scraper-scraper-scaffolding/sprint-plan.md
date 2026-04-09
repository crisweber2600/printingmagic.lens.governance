---
feature: newversion-scraper-scraper-scaffolding
doc_type: sprint-plan
status: draft
goal: "Port the STLFlix scraper from the reference Worker into a standalone PrintingMagic.Scraper Aspire service across three sprints: project foundation, harvest pipeline, and Aspire integration"
key_decisions:
  - Sprint 1 is gated on DTO schema alignment (H1 from adversarial review) — no scraper code starts until contracts are confirmed
  - Sprint 3 has a hard dependency on newversion-apphost-scaffolding reaching dev-ready
  - Polite delay bounds and concurrency limit are configuration values from sprint 1
open_questions:
  - Internal API URL resolution strategy (Aspire service discovery vs. config) must be decided before sprint 2 starts
  - Playwright browser binary provisioning approach must be confirmed before sprint 1 ends
depends_on:
  - newversion-apphost-scaffolding
blocks: []
updated_at: "2026-04-09T18:33:00Z"
---

# Sprint Plan: newversion-scraper-scraper-scaffolding

## Sprint Overview

| Sprint | Goal | Stories | Complexity | Key Risk |
|--------|------|---------|------------|----------|
| 1 | Project foundation + Playwright session infrastructure | S1-01, S1-02, S1-03 | M + L + S | Data.Contracts may not exist (H1) |
| 2 | Scraping pipeline + API client | S2-01, S2-02, S2-03, S2-04 | L + M + M + S | Internal API URL strategy must be decided |
| 3 | Aspire integration, observability, legal gate, smoke test | S3-01, S3-02, S3-03, S3-04 | M + M + S + S | Playwright binary provisioning (H2); apphost dependency (M4) |

## Definition of Done

Applied to all stories in this plan:
- Code compiles with no warnings on .NET 9
- All new interfaces have at least one implementation
- All ported logic is functionally equivalent to the reference codebase unless explicitly noted otherwise
- Unit tests cover the new/ported code at the level specified in the story's Technical Notes
- No hardcoded credentials, URLs, or magic numbers (all configuration values backed by `IConfiguration`)
- OTEL `ActivitySource` span added per the reference codebase pattern
- Pull request reviewed and merged to the feature branch before story is marked done

---

## Sprint 1 — Project Foundation + Playwright Session Infrastructure

**Goal:** Create `PrintingMagic.Scraper` project, confirm data contracts, and stand up a working Playwright browser session with login — no scraping yet, just the infrastructure that all subsequent stories depend on.

**Stories:**

| ID | Title | Estimate |
|----|-------|----------|
| newversion-scraper-scraper-scaffolding-1-01 | Create PrintingMagic.Scraper project + Playwright setup | M |
| newversion-scraper-scraper-scaffolding-1-02 | Port BrowserSessionService + LoginService | L |
| newversion-scraper-scraper-scaffolding-1-03 | Verify or create PrintingMagic.Data.Contracts (H1 gate) | S |

**Sprint 1 Acceptance Summary:**
- `PrintingMagic.Scraper` project exists, builds, and references Playwright
- `BrowserSessionService` initialises a headless browser and performs a successful login against STLFlix dev credentials
- `CollectionImportRequest`, `CollectionSeed`, and `ModelImportDto` are confirmed or created and schema-aligned with the old version

**Risks:**
- H1: If `Data.Contracts` project does not exist, story 1-03 must create it — estimate may grow from S to M
- H2: Playwright binary provisioning strategy must be decided by end of sprint 1 (decision feeds into sprint 3)

---

## Sprint 2 — Scraping Pipeline + API Client

**Goal:** Deliver the full scraping pipeline — `StlflixScraper`, `CollectionHarvesterJob`, `DropProcessingService`, and `InternalApiClient` — and verify that scraped data passes to the API service successfully.

**Stories:**

| ID | Title | Estimate |
|----|-------|----------|
| newversion-scraper-scraper-scaffolding-2-01 | Port StlflixScraper (IExternalScraper implementation) | L |
| newversion-scraper-scraper-scaffolding-2-02 | Port DropProcessingService | M |
| newversion-scraper-scraper-scaffolding-2-03 | Port CollectionHarvesterJob + IInternalApiClient | M |
| newversion-scraper-scraper-scaffolding-2-04 | Resolve internal API URL strategy (ADR-6) | S |

**Sprint 2 Acceptance Summary:**
- A full scrape cycle runs against STLFlix (dev credentials): drops discovered, collections scraped, `CollectionImportRequest` produced
- `InternalApiClient` successfully POSTs the result to `PrintingMagic.ApiService`
- Internal API URL uses Aspire service discovery or explicit config — decision recorded as ADR-6 in tech plan

**Risks:**
- M3: Polite delay and concurrency limit are now configurable — confirmed as part of story 2-01 and 2-03
- M2: ADR-6 decision must happen before 2-03 starts

---

## Sprint 3 — Aspire Integration, Observability, Legal Gate + Smoke Test

**Goal:** Register the scraper in the Aspire AppHost, wire OTEL, confirm browser binary provisioning, establish the legal gate story, and run an end-to-end smoke test through the Aspire dashboard.

**Dependency:** `newversion-apphost-scaffolding` must be at `dev-ready` before sprint 3 starts (M4 from adversarial review).

**Stories:**

| ID | Title | Estimate |
|----|-------|----------|
| newversion-scraper-scraper-scaffolding-3-01 | Register PrintingMagic.Scraper in Aspire AppHost via aspire add | M |
| newversion-scraper-scraper-scaffolding-3-02 | Configure Playwright binary provisioning (local dev + Aspire container) | M |
| newversion-scraper-scraper-scaffolding-3-03 | Wire OTEL ActivitySource + Aspire telemetry + dashboard smoke test | S |
| newversion-scraper-scraper-scaffolding-3-04 | Create legal gate story — production deployment blocked until ToS sign-off | S |

**Sprint 3 Acceptance Summary:**
- Aspire AppHost starts with `PrintingMagic.Scraper` resource in `Running` state
- Aspire dashboard shows at least one trace span from `PrintingMagic.Scraper` after a scrape cycle (M1 from adversarial review)
- `playwright install chromium` runs automatically on post-restore (local) and in container image (Dockerfile layer)
- Legal gate story exists in governance as a `Won't Start` blocker story, linked to production deployment

**Risks:**
- H2: Playwright binary provisioning is a sprint 3 story; if the approach chosen in sprint 1 turns out to be incompatible with the Aspire container, this story may expand
- M4: If `newversion-apphost-scaffolding` is not ready, sprint 3 stories 3-01 and 3-02 are blocked; 3-03 and 3-04 can proceed independently

---

## Cross-Sprint Dependencies

| Blocking Story | Blocked Story | Reason |
|----------------|---------------|--------|
| 1-03 (Data.Contracts) | All sprint 2 stories | Without DTO types the scraper code cannot compile |
| 1-02 (BrowserSessionService) | 2-01 (StlflixScraper) | Scraper depends on `IBrowserSessionService` |
| 2-01 (StlflixScraper) | 2-02 (DropProcessingService) | Drop processor calls into `IExternalScraper` |
| 2-04 (ADR-6 URL strategy) | 2-03 (InternalApiClient) | API client base URL must be decided before implementation |
| newversion-apphost-scaffolding dev-ready | 3-01, 3-02 | `aspire add` requires an AppHost to add into |
| 2-01–2-03 complete | 3-03 (smoke test) | Cannot run a scrape cycle from Aspire until pipeline exists |
