# Story: newversion-scraper-scraper-scaffolding-1-01

## Context

**Feature:** newversion-scraper-scraper-scaffolding
**Sprint:** 1
**Priority:** high
**Estimate:** M

## User Story

As a developer, I want a `PrintingMagic.Scraper` .NET 9 worker project with Microsoft.Playwright installed and configured, so that subsequent browser-automation stories have a compiling project to build on.

## Acceptance Criteria

- [ ] `PrintingMagic.Scraper` project exists in the new solution with a `BackgroundService`-derived `Worker` class skeleton
- [ ] `Microsoft.Playwright` NuGet package is referenced in the project file
- [ ] A `playwright install chromium` post-restore instruction is documented in the project README (or added as a build target stub for sprint 3 to wire up)
- [ ] Project builds with no warnings on .NET 9
- [ ] `ActivitySource` named `"PrintingMagic.Scraper"` is declared in `Worker.cs` following the reference pattern
- [ ] Polite delay lower/upper bounds are declared as `IConfiguration`-backed values with constants `2000` / `5000` ms as defaults (M3 from adversarial review)
- [ ] Scrape cycle delay is declared as an `IConfiguration`-backed value defaulting to `5` minutes

## Technical Notes

- Reference: `TargetProjects/oldversion/printingmagic/PrintingMagic.Worker/Worker.cs` and `Program.cs`
- Project type: `Microsoft.NET.Sdk.Worker`
- Do NOT port `IModelAutomationService`, `IFileCleanupService`, or their implementations — out of scope for this feature
- The `Worker.ExecuteAsync` skeleton should have the same outer loop structure (cleanup → test session → process drops → wait) but the body can be stubbed with `await Task.Delay(cycleDelay, stoppingToken)` until sprint 2 stories fill it in
- Leave `IDropProcessingService` and `IModelAutomationService` DI registrations as TODO comments for sprint 2

## Dependencies

- None (first story in sprint 1)
