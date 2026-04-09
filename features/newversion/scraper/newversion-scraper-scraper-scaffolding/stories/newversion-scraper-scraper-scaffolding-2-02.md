# Story: newversion-scraper-scraper-scaffolding-2-02

## Context

**Feature:** newversion-scraper-scraper-scaffolding
**Sprint:** 2
**Priority:** high
**Estimate:** M

## User Story

As a developer, I want `IDropProcessingService` / `DropProcessingService` ported to `PrintingMagic.Scraper`, so that all unprocessed STLFlix drops can be discovered and imported in a single service cycle.

## Acceptance Criteria

- [ ] `IDropProcessingService` interface is ported: `Task<DropImportResult?> ProcessDropsAsync(CancellationToken token)`
- [ ] `DropProcessingService` is ported with the Playwright-based drops-page navigation, drop URL discovery, and `IInternalApiClient` call sequence
- [ ] `SingleDropMode` configuration value is preserved (`"Worker:SingleDropMode"` key, default `false` for "process all drops")
- [ ] Navigation retry loop (up to 3 attempts) and login-redirect detection are ported faithfully from the reference
- [ ] Unit test: when `SingleDropMode = true`, only the first discovered drop is processed
- [ ] Unit test: when navigation redirects to login, `IBrowserSessionService.InvalidateSession` is called and the retry loop continues
- [ ] `ActivitySource` span added to `ProcessDropsAsync`

## Technical Notes

- Reference: `TargetProjects/oldversion/printingmagic/PrintingMagic.Worker/DropProcessingService.cs`
- `DropImportResult` record is defined in this file (`public record DropImportResult(int CollectionId, IReadOnlyList<int> ModelIds)`) — port it alongside the service
- `IInternalApiClient` is mocked in unit tests; the real implementation comes in story 2-03
- Do not port `IModelAutomationService` references — that is out of scope for this feature

## Dependencies

- newversion-scraper-scraper-scaffolding-2-01 (IExternalScraper must exist for the DI graph to resolve)
- newversion-scraper-scraper-scaffolding-1-02 (IBrowserSessionService must be available)
