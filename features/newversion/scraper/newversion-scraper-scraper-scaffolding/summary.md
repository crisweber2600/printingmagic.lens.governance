# Summary: newversion-scraper-scraper-scaffolding

**Phase:** sprintplan
**Goal:** Deliver a standalone .NET scraper service — ported from the reference Worker — that harvests STLFlix collection and model metadata and stores it ready for downstream processing
**Status:** draft
**Updated:** 2026-04-09T18:33:00Z

## Key Decisions

- Baseline from PrintingMagic.Worker (oldversion) rather than greenfield to reduce risk and preserve validated logic
- Scope includes both drop/collection discovery AND individual model metadata scraping
- Microsoft.Playwright (.NET) as the headless browser — same as reference, avoids new technology risk
- Isolated project PrintingMagic.Scraper rather than embedding in Worker or ApiService
- Aspire registration via aspire CLI (aspire add), not manual AppHost csproj edit
- Credentials injected via Aspire environment variables / connection strings — no hardcoded values
- All data writes routed through IInternalApiClient — no direct database access from the scraper

## Open Questions

- Legal review of STLFlix ToS must be completed before production deployment (tracked as story 3-04)
- Do new-version data contracts match old-version schema exactly? (tracked as story 1-03)
- Internal API URL resolution strategy (Aspire service discovery vs. config) — decision in story 2-04
- Playwright browser binary provisioning approach for Aspire container (tracked as story 3-02)

## Artifacts Present

- business-plan.md
- tech-plan.md
- adversarial-review.md
- sprint-plan.md
- stories/ (11 stories: 1-01, 1-02, 1-03, 2-01, 2-02, 2-03, 2-04, 3-01, 3-02, 3-03, 3-04)
