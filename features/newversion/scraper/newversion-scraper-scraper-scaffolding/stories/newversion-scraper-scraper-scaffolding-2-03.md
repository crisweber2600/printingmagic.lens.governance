# Story: newversion-scraper-scraper-scaffolding-2-03

## Context

**Feature:** newversion-scraper-scraper-scaffolding
**Sprint:** 2
**Priority:** high
**Estimate:** M

## User Story

As a developer, I want `CollectionHarvesterJob` and `IInternalApiClient` / `InternalApiClient` ported to `PrintingMagic.Scraper`, so that scraped collection data is posted to `PrintingMagic.ApiService` and individual models are handed off for processing.

## Acceptance Criteria

- [ ] `IInternalApiClient` interface is ported with at least: `ImportCollectionAsync(CollectionImportRequest)` and `ProcessModelAsync(ModelImportDto)`
- [ ] `InternalApiClient` is ported; the base URL is resolved using the strategy decided in story 2-04 (Aspire service discovery or explicit config) — use a `TODO: wire ADR-6 URL` comment if 2-04 is not yet complete
- [ ] `CollectionHarvesterJob` is ported with the `SemaphoreSlim`-bounded concurrent model processing loop
- [ ] Concurrency limit extracted to `IConfiguration` key `"Worker:ModelConcurrency"` defaulting to `5` (L3 from adversarial review)
- [ ] Unit test: mock `IExternalScraper` returns 3 collections; mock `IInternalApiClient` returns 5 model IDs; assert `ProcessModelAsync` is called exactly 5 times with the correct IDs
- [ ] Unit test: assert that if `IExternalScraper.ScrapeAsync` throws, the exception propagates (existing reference behaviour)
- [ ] `ActivitySource` span added to `CollectionHarvesterJob.RunAsync`

## Technical Notes

- Reference: `TargetProjects/oldversion/printingmagic/PrintingMagic.Worker/CollectionHarvesterJob.cs`
- Reference: `TargetProjects/oldversion/printingmagic/PrintingMagic.Worker/InternalApiClient.cs`
- Reference: `TargetProjects/oldversion/printingmagic/PrintingMagic.Worker/ApiClient.cs`
- The `InternalApiClient` constructs `ModelImportDto(id, string.Empty, string.Empty, string.Empty)` in the reference — preserve this shape until the data contract story confirms the correct fields

## Dependencies

- newversion-scraper-scraper-scaffolding-2-01 (IExternalScraper must exist)
- newversion-scraper-scraper-scaffolding-2-04 (ADR-6 URL strategy must be decided)
- newversion-scraper-scraper-scaffolding-1-03 (CollectionImportRequest and ModelImportDto must exist)
