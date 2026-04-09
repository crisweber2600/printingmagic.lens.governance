# Story: newversion-scraper-scraper-scaffolding-2-04

## Context

**Feature:** newversion-scraper-scraper-scaffolding
**Sprint:** 2
**Priority:** high
**Estimate:** S

## User Story

As a developer, I want the internal API URL resolution strategy decided and recorded as ADR-6 in the tech plan, so that `InternalApiClient` (story 2-03) has a clear, unambiguous way to locate `PrintingMagic.ApiService` at runtime in both local dev and Aspire container environments.

## Acceptance Criteria

- [ ] Decision is made: Aspire service discovery (`http+http://apiservice`) OR explicit `IConfiguration` key (`"ApiService:BaseUrl"`)
- [ ] Decision is recorded as ADR-6 in `TargetProjects/lens/lens-governance/features/newversion/scraper/newversion-scraper-scraper-scaffolding/tech-plan.md`
- [ ] `InternalApiClient` (story 2-03) is updated (or its TODO comment resolved) to use the decided approach
- [ ] If Aspire service discovery is chosen: confirm that `PrintingMagic.ApiService` is registered in the AppHost under the name `"apiservice"` (coordinate with `newversion-apphost-scaffolding`)
- [ ] If explicit config is chosen: add a `"ApiService:BaseUrl"` entry to `appsettings.Development.json` with the local dev URL

## Technical Notes

- Aspire service discovery is the preferred approach per the tech plan architecture overview — use it unless there is a confirmed blocker
- Aspire service discovery URL pattern for .NET: register `HttpClient` with `AddServiceDiscovery()` and configure the base address as `http+http://apiservice`
- Reference: the old version uses a hardcoded config value — the new version should use Aspire service discovery if the AppHost is live

## Dependencies

- newversion-scraper-scraper-scaffolding-2-01 (enough context exists to make this decision)
- Coordination with newversion-apphost-scaffolding owner to confirm service registration name
