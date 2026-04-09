# Story: newversion-scraper-scraper-scaffolding-2-01

## Context

**Feature:** newversion-scraper-scraper-scaffolding
**Sprint:** 2
**Priority:** high
**Estimate:** L

## User Story

As a developer, I want `IExternalScraper` / `StlflixScraper` ported to `PrintingMagic.Scraper`, so that the system can discover all STLFlix drop URLs and scrape each drop's collection and model metadata into a `CollectionImportRequest`.

## Acceptance Criteria

- [ ] `IExternalScraper` interface is ported: `Task<CollectionImportRequest> ScrapeAsync(CancellationToken ct)`
- [ ] `StlflixScraper` is ported to `Scrapers/StlflixScraper.cs` with all navigation, discovery, and per-drop scraping logic
- [ ] `DropsPageUrl`, `MaxNavigationRetries`, `NavigationTimeoutMs`, `MinDelayBetweenPagesMs`, and `MaxDelayBetweenPagesMs` are read from `IConfiguration` (with old-version constants as defaults) — polite delay M3 requirement
- [ ] Session invalidation + re-login retry on login-page redirect is preserved faithfully from the reference
- [ ] Unit test: given an HTML fixture for the drops page, `DiscoverDropUrls` returns the expected URL list
- [ ] Unit test: given an HTML fixture for a single drop page, `ScrapeOneDropWithRetry` returns the expected `CollectionSeed`
- [ ] `ActivitySource` span added to `ScrapeAsync` following the reference pattern
- [ ] Legal comment from reference (`StlflixScraper.cs` summary XML) is preserved: "Must not go live in production until legal review of target platform's ToS is complete"

## Technical Notes

- Reference: `TargetProjects/oldversion/printingmagic/PrintingMagic.Worker/Scrapers/StlflixScraper.cs`
- All CSS/ARIA selectors should be declared as private constants at the top of the class (not inline strings) to make future UI-change fixes easy
- `PoliteDelay` should use the configuration-backed bounds from story 1-01
- Keep the `MaxItemRetries = 3` and `MaxNavigationRetries = 3` as configurable values

## Dependencies

- newversion-scraper-scraper-scaffolding-1-02 (BrowserSessionService must be available)
- newversion-scraper-scraper-scaffolding-1-03 (CollectionImportRequest type must exist)
