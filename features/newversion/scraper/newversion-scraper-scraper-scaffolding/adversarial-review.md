# Adversarial Review: newversion-scraper-scraper-scaffolding

**Reviewed:** 2026-04-09T18:32:00Z
**Business Plan SHA:** (see governance main — business-plan.md)
**Tech Plan SHA:** (see governance main — tech-plan.md)
**Overall Rating:** pass-with-warnings

## Summary

Both plans are internally consistent and well-grounded in the reference codebase. The business plan accurately scopes the harvest pipeline and the tech plan faithfully maps onto the reference architecture. Three High-severity issues require resolution before sprint planning locks in sprint 1: the `PrintingMagic.Data.Contracts` dependency may not exist in the new solution (hard blocker for sprint 1), Playwright browser binary provisioning in the Aspire context is unaddressed (hard to fix mid-sprint), and the legal gate on production deployment is not explicitly represented as a story or acceptance criterion. No Critical blockers were found. All High items should be resolved or formally accepted before sprint 1 begins.

---

## Findings

### Critical

*None.*

---

### High

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| H1 | Coverage Gap | `PrintingMagic.Data.Contracts` — the DTO types `CollectionImportRequest`, `CollectionSeed`, and `ModelImportDto` are open questions in both plans but neither plan makes their existence a sprint 1 prerequisite story. If they don't exist, the scraper project cannot compile. | Add a story in sprint 1: "Verify or create `PrintingMagic.Data.Contracts` project and confirm DTO schema alignment with old version." Gate all other sprint 1 stories on this story. |
| H2 | Complexity & Risk | Playwright browser binary provisioning for Aspire is listed as an open question but has no resolution path or story. `playwright install` is not a standard .NET project build step — it must be invoked explicitly after restore, and in a containerised Aspire run it requires a Dockerfile layer or a post-build hook. If this is skipped, the service crashes at startup with a `BrowserNotFound` exception. | Add a story in sprint 3: "Configure Playwright browser binary installation for both local dev (post-restore hook) and Aspire container (Dockerfile step)." Define the approach before sprint 1 starts so the tech plan reflects it. |
| H3 | Assumptions & Blind Spots | The legal gate (STLFlix ToS review required before production deployment) exists as a comment in the reference codebase (`StlflixScraper.cs` legal note) and as a risk in the business plan, but neither plan translates it into an explicit acceptance criterion or a blocking story. Without a story, it is invisible to sprint tracking and could be silently bypassed. | Add a story or explicit sprint blocker: "Legal review of STLFlix ToS approved before production release." This story is `Won't Start` until sign-off is received; it blocks the production deployment story, not dev/test. |

---

### Medium

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| M1 | Coverage Gap | The tech plan calls for OTEL instrumentation but the testing strategy has no test for observability signal presence. If ActivitySource spans are dropped (e.g., SDK not wired to Aspire), it won't be caught until the dashboard is checked manually. | Add one smoke-test assertion: "Aspire dashboard shows at least one trace span from `PrintingMagic.Scraper` after a scrape cycle." |
| M2 | Logic Flaw | Internal API URL injection strategy is an open question (Aspire service discovery vs. hardcoded config). In the reference codebase this is a config value — if the new version uses Aspire service discovery, the `IInternalApiClient` base URL resolution logic must change. This is not represented as a design decision or story. | Resolve before sprint 2 starts: confirm whether to use `http+http://apiservice` (Aspire service discovery) or explicit config. Record as ADR-6 in tech plan. |
| M3 | Complexity & Risk | The polite delay (2–5s) is hardcoded in the reference codebase as constants (`MinDelayBetweenPagesMs = 2000`, `MaxDelayBetweenPagesMs = 5000`). The tech plan states it will be "configurable" but no configuration key or schema is defined. If the scraper is too aggressive in a specific environment, there is no lever to tune it without a code change. | Add a story in sprint 1 or 2: "Expose polite delay bounds as IConfiguration values with the reference codebase constants as defaults." |
| M4 | Cross-Feature Dependencies | The business plan lists `newversion-apphost-scaffolding` as a dependency and a blocked feature. However, the `feature-index.yaml` shows `newversion-apphost-scaffolding` at status `preplan` with no planning documents. If apphost scaffolding slips, sprint 3 (Aspire integration) is blocked. | Track the dependency explicitly in the sprint plan: sprint 3 has a hard dependency on `newversion-apphost-scaffolding` being at `dev-ready`. |

---

### Low

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| L1 | Coverage Gap | `StorageState` serialisation is used in `BrowserSessionService` (`_storageState = "{}"`) but the format is not documented in the tech plan. If the default changes between Playwright versions, the session may silently fail. | Document the expected format in a code comment; add a defensive check that `_storageState` is valid JSON before passing it to `CreateContextAsync`. |
| L2 | Assumptions | The test strategy lists a "live STLFlix" integration test. This requires live credentials in the CI environment. The tech plan does not mention credential provisioning for CI, which could block automated integration testing. | Note in the testing strategy: integration tests require `Stlflix:Email` and `Stlflix:Password` as CI secrets; add as a sprint 3 sub-task. |
| L3 | Complexity | `CollectionHarvesterJob.RunAsync` uses `SemaphoreSlim(5)` for concurrency control but the parallelism limit is hardcoded. In a resource-constrained Aspire container, 5 concurrent Playwright operations may be too many. | Expose the concurrency limit as a configuration value with `5` as the default, same as the polite delay recommendation (M3). |
