# Story: newversion-scraper-scraper-scaffolding-3-01

## Context

**Feature:** newversion-scraper-scraper-scaffolding
**Sprint:** 3
**Priority:** high
**Estimate:** M

## User Story

As a developer, I want `PrintingMagic.Scraper` registered in the Aspire AppHost via `aspire add`, so that it appears as a healthy resource in the Aspire dashboard alongside the other services.

## Acceptance Criteria

- [ ] `aspire add PrintingMagic.Scraper` is run from the AppHost project directory, generating the correct project reference and builder call in `apphost.cs`
- [ ] The generated resource builder call is confirmed to compile (no manual csproj edits needed)
- [ ] `Stlflix:Email` and `Stlflix:Password` are injected as Aspire environment variables via `WithEnvironment(...)` or a connection string — confirmed not hardcoded anywhere
- [ ] AppHost builds and starts with `PrintingMagic.Scraper` in `Running` state (no startup exceptions in the Aspire dashboard)
- [ ] Service is visible in the Aspire dashboard resource list with a descriptive name

## Technical Notes

- Run `aspire add` from inside `TargetProjects/newversion/apphost/PrintingMagic.Aspire/`
- Credential injection: use `builder.AddProject<Projects.PrintingMagic_Scraper>("scraper").WithEnvironment("Stlflix__Email", scraperEmail).WithEnvironment("Stlflix__Password", scraperPassword)` pattern (or equivalent via `IResourceBuilder` fluent API)
- For local dev credentials: use `dotnet user-secrets` on the AppHost project to supply `Stlflix:Email` and `Stlflix:Password` values
- **Dependency:** `newversion-apphost-scaffolding` must be at `dev-ready` before this story starts

## Dependencies

- newversion-apphost-scaffolding (AppHost must exist with at least an empty builder)
- newversion-scraper-scraper-scaffolding-2-01 through 2-03 (scraper must compile before it can be registered)
