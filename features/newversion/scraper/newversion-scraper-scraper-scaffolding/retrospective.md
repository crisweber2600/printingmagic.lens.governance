# Retrospective: newversion-scraper-scraper-scaffolding

**Feature:** newversion-scraper-scraper-scaffolding
**Completed:** 2026-04-10T00:00:00Z

## What Went Well

- **Port fidelity was high.** The ported code from `PrintingMagic.Worker` (oldversion) mapped cleanly to the new project structure. The existing `BrowserSessionService`, `LoginService`, `StlflixScraper`, `DropProcessingService`, and `CollectionHarvesterJob` all carried over with minimal adaptation.

- **Data.Contracts already existed.** Story 1-03 (H1 gate) resolved quickly — `PrintingMagic.Data.Contracts` was present and schema-compatible, unblocking sprint 2 immediately.

- **Aspire integration worked end-to-end.** The file-based `apphost.cs` approach with `#:project`, `#:package`, and `#:sdk` directives proved clean. PostgreSQL and RabbitMQ containers started healthy on first run.

- **Playwright binary install resolved simply.** The headless shell was not pre-installed, but `pwsh playwright.ps1 install chromium` resolved it immediately with no code changes.

- **Credential injection via Aspire parameters** worked as designed — `AddParameter` / `WithEnvironment` cleanly avoids hardcoded values while surfacing the configuration requirement visibly in the apphost.

## What Didn't Go Well

- **`#:project` directive syntax was wrong at launch.** The initial `apphost.cs` included `name path` syntax (`#:project PrintingMagic.Scraper ../../scraper/...`) which is not valid — the directive only accepts a path. This caused a build failure that required investigation against Aspire CLI docs.

- **Missing `#:package` directives.** `AddPostgres` and `AddRabbitMQ` are not included by default in the file-based AppHost SDK. `Aspire.Hosting.PostgreSQL` and `Aspire.Hosting.RabbitMQ` packages had to be explicitly declared, which was non-obvious.

- **`aspire doctor` warnings were unaddressed at project start.** Both `SSL_CERT_DIR` and `ASPIRE_ENABLE_CONTAINER_TUNNEL` were unconfigured, causing docker tunnel issues and certificate warnings on first boot. These should be pre-configured in dev environment setup.

- **Playwright browser binary not provisioned in Aspire context.** Story 3-02 was planned but the binary install step was left to a manual post-run step rather than automated in the apphost or CI. This gap means a fresh dev environment will hit the same `Executable doesn't exist` error on first boot.

## Key Learnings

- **File-based Aspire AppHost `#:project` takes only a path, not `name path`.** The project type name is derived automatically from the `.csproj` filename. Document this for future services.

- **All hosting integrations require explicit `#:package` declarations in file-based apphosts.** There is no auto-inclusion. Each `Add*` method must have a corresponding package reference.

- **Run `aspire doctor` and fix all warnings before starting dev** — the two shell profile env vars (`SSL_CERT_DIR`, `ASPIRE_ENABLE_CONTAINER_TUNNEL`) need to be set once per machine and don't require code changes.

- **Playwright binary provisioning needs to be part of the dev setup script or Aspire resource command**, not left to post-boot manual steps.

## Metrics

- Planned duration: 3 sprints (S1–S3)
- Actual duration: ~1 day (scaffolding delivered as single accelerated pass)
- Stories completed: All 11 (1-01 through 3-04 — including legal gate story 3-04 which remains open pending external review)
- Bugs found post-merge: 2 (apphost directive syntax, missing package declarations)

## Action Items

| Item | Owner | Priority |
|------|-------|----------|
| Automate Playwright binary install in dev setup / `aspire exec` hook | crisweber2600 | Medium |
| Add `aspire doctor` to onboarding checklist in repo README | crisweber2600 | Low |
| Resolve legal gate (story 3-04) before enabling production traffic | crisweber2600 | High |
| Document `#:project` path-only syntax in project conventions | crisweber2600 | Low |
