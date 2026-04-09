---
feature: newversion-scraper-scraper-scaffolding
doc_type: business-plan
status: draft
goal: "Deliver a standalone .NET scraper service — ported from the reference Worker — that harvests STLFlix collection and model metadata and stores it ready for downstream processing"
key_decisions:
  - Baseline from PrintingMagic.Worker (oldversion) rather than greenfield to reduce risk and preserve validated logic
  - Scope includes both drop/collection discovery AND individual model metadata scraping
open_questions:
  - Legal review of STLFlix ToS must be completed before production deployment (note already present in reference codebase)
  - Do new-version data contracts match old-version schema exactly, or do they require schema updates?
  - What is the target scrape cycle delay for the new version — keep 5-minute reference value or make it configurable?
depends_on: []
blocks:
  - newversion-apphost-scaffolding
updated_at: "2026-04-09T18:30:00Z"
---

# Business Plan: newversion-scraper-scraper-scaffolding

## Executive Summary

PrintingMagic requires automated harvest of 3D printing model metadata from STLFlix — a subscription platform for curated 3D model drops. This scraper service provides the data foundation for the catalog, recommendation engine, and import pipeline by authenticating as a user, navigating the STLFlix web UI with a headless .NET browser (Microsoft.Playwright), and extracting collection titles, model URLs, and associated metadata. The implementation is not greenfield: it ports the proven `PrintingMagic.Worker` scraping pipeline from the reference codebase into a clean, isolated service registered in the new Aspire distributed application host.

## Business Context

The old version of PrintingMagic included `PrintingMagic.Worker` — a background service that used Microsoft.Playwright to scrape STLFlix, harvest drop/collection data, and feed it into the API processing pipeline. That implementation is production-validated. The new version requires the same capability, rebuilt as a first-class isolated service in the Aspire-hosted architecture.

Without this service:
- The metadata ingestion pipeline has no source of truth.
- Downstream features (catalog, search, recommendations, model processing) have no data to operate on.
- The Aspire AppHost distributed application is incomplete as a runnable system.

## Stakeholders

| Stakeholder | Role | Interest |
|-------------|------|----------|
| crisweber2600 | Lead developer + operator | Needs automated harvest; operates the system |
| PrintingMagic API service | Internal consumer | Depends on scraped metadata for catalog and processing pipeline |
| STLFlix (external platform) | Target platform | Terms of Service constraints on automated access must be respected |

## Success Criteria

1. The scraper service starts as an Aspire resource and appears healthy in the Aspire dashboard with no startup errors.
2. A full scrape cycle runs end-to-end: all drops on the STLFlix drops page are discovered and their collection + model metadata is extracted.
3. Scraped metadata persists to the store in a schema the API service can process without transformation.
4. Session management is reliable: the service re-authenticates automatically when the browser session expires, with no manual intervention.
5. Polite delay (2–5 seconds) is enforced between page navigations — no hammering the target platform.
6. Scraped metadata passes data validation before handoff to the API layer.
7. The service recovers gracefully from navigation failures using the same retry pattern as the reference codebase.

## Scope

**In scope:**
- Port `IBrowserSessionService` / `BrowserSessionService` — shared headless browser session with login state
- Port `ILoginService` / `LoginService` — authenticated login to STLFlix via Playwright
- Port `IExternalScraper` / `StlflixScraper` — full drop discovery and per-drop collection scraping
- Port `CollectionHarvesterJob` — orchestrates scrape → import pipeline with concurrency gating
- Port `IDropProcessingService` / `DropProcessingService` — processes all unprocessed drops
- Port `IInternalApiClient` / `InternalApiClient` — bridges scraped data to the API service
- Register the scraper project as an Aspire resource via `aspire add`
- Credentials injected via Aspire environment / connection strings (no hardcoded values)
- Configurable scrape cycle delay, retry limits, and polite delay range
- OpenTelemetry instrumentation following the same `ActivitySource` pattern as the reference

**Out of scope:**
- Model automation / zip upload (separate concern; `IModelAutomationService` is not ported here)
- File cleanup service (`IFileCleanupService` is not ported here)
- Direct database writes (all writes route through the internal API layer)
- Any new UI or user-facing changes
- STLFlix API integration (the platform exposes a web UI only — no official API is available)

## Risks and Mitigations

| Risk | Probability | Mitigation |
|------|-------------|------------|
| STLFlix ToS violation in production | Medium | Legal gate already present in reference codebase; same gate applies — production deployment requires legal sign-off |
| STLFlix UI layout changes breaking selectors | High | Centralise all CSS/role selectors within `StlflixScraper`; write selector-level unit tests against known HTML fixtures |
| Playwright browser binary unavailable at Aspire startup | Low | Pin Playwright version in project file; provision browser binary in Aspire resource setup / Dockerfile |
| Session expiry mid-scrape cycle | Medium | Reference `InvalidateSession` + re-login loop already handles this; port it faithfully |
| Rate limiting or IP blocking by STLFlix | Medium | Polite delay (2–5s jitter) from reference retained; add configurable per-run delay cap |
| Data contract mismatch between old and new versions | Low–Medium | Explicit schema comparison of `CollectionImportRequest`, `CollectionSeed`, and `ModelImportDto` before integration sprint; tracked as open question |

## Dependencies

- `newversion-apphost-scaffolding` — Aspire AppHost project must exist before `aspire add` can register the scraper
- STLFlix platform availability (external, uncontrolled)
- Microsoft.Playwright NuGet package + Playwright browser binary provisioned at runtime
- PrintingMagic.Data.Contracts — `CollectionImportRequest`, `CollectionSeed`, `ModelImportDto` (must exist in new solution or be created as part of this feature)

## Open Questions

1. **ToS legal status** — Is automated scraping of STLFlix approved for production? (gates live deployment, not dev scaffolding or local testing)
2. **Data contract alignment** — Do `CollectionImportRequest`, `CollectionSeed`, and `ModelImportDto` in the new solution match the old version exactly, or do they require schema updates before integration?
3. **Cycle delay** — Reference uses 5-minute inter-cycle delay. Should this be environment-configurable in the new version, or keep the reference constant?

## Timeline Expectations

No external deadline imposed. This feature is blocking the full-system integration sprint for the Aspire AppHost; development should begin immediately after `newversion-apphost-scaffolding` is unblocked.
