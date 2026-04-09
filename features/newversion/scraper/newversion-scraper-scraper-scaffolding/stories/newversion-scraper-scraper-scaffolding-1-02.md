# Story: newversion-scraper-scraper-scaffolding-1-02

## Context

**Feature:** newversion-scraper-scraper-scaffolding
**Sprint:** 1
**Priority:** high
**Estimate:** L

## User Story

As a developer, I want `IBrowserSessionService` / `BrowserSessionService` and `ILoginService` / `LoginService` ported to `PrintingMagic.Scraper`, so that all subsequent scraping stories have a reliable, re-authenticating headless browser session available.

## Acceptance Criteria

- [ ] `IBrowserSessionService` interface is ported with all methods: `GetBrowserAsync`, `GetStorageStateAsync`, `CreateContextAsync`, `CreatePageAsync`, `TestSessionValidityAsync`, `InvalidateSession`
- [ ] `BrowserSessionService` implementation is ported, maintaining the `SemaphoreSlim(1,1)` concurrency guard and the `_sessionInvalidated` / `_isLoggedIn` state machine
- [ ] `ILoginService` / `LoginService` are ported; credentials (`Stlflix:Email`, `Stlflix:Password`) are read from `IConfiguration` with no hardcoded fallback email (reference had a hardcoded fallback — remove it; throw `InvalidOperationException` if email is missing, matching password behaviour)
- [ ] Unit test: mock `ILoginService`; assert `BrowserSessionService.GetBrowserAsync` calls `PerformLoginAsync` when `_sessionInvalidated = true`
- [ ] Unit test: assert `LoginService` throws `InvalidOperationException` when `Stlflix:Email` is missing (not just `Stlflix:Password`)
- [ ] `ActivitySource` spans added to `GetBrowserAsync`, `GetStorageStateAsync`, `CreateContextAsync` following the reference pattern
- [ ] Services registered in DI (`Program.cs`) as scoped/singleton following the reference registration pattern

## Technical Notes

- Reference: `TargetProjects/oldversion/printingmagic/PrintingMagic.Worker/BrowserSessionService.cs`
- Reference: `TargetProjects/oldversion/printingmagic/PrintingMagic.Worker/LoginService.cs`
- Reference: `TargetProjects/oldversion/printingmagic/PrintingMagic.Worker/IBrowserSessionService.cs`
- **Security fix vs. reference:** The reference `LoginService` has a hardcoded fallback for `Stlflix:Email` (`"cris@weber.center"`). This must NOT be ported — both email and password must come from configuration, and both must throw if missing.
- Playwright browser initialisation (`InitializeBrowserAsync`) should use `Headless = true` for Aspire deployment; expose as `IConfiguration`-backed `"Playwright:Headless"` bool defaulting to `true`

## Dependencies

- newversion-scraper-scraper-scaffolding-1-01 (project must exist and compile)
