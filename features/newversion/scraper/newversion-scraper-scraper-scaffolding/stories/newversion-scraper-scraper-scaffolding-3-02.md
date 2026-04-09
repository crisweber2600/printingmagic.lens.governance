# Story: newversion-scraper-scraper-scaffolding-3-02

## Context

**Feature:** newversion-scraper-scraper-scaffolding
**Sprint:** 3
**Priority:** high
**Estimate:** M

## User Story

As a developer, I want Playwright browser binary provisioning configured for both local dev and the Aspire container environment, so that `PrintingMagic.Scraper` does not crash at startup with a `BrowserNotFound` exception.

## Acceptance Criteria

- [ ] Local dev: a post-restore hook or documented manual step (`playwright install chromium`) is established so that developers can run the scraper locally without reading the internals of the code
- [ ] Aspire container: a `Dockerfile` for `PrintingMagic.Scraper` (or an existing one updated) includes a `RUN playwright install chromium --with-deps` layer after the .NET restore step
- [ ] The approach is documented in `PrintingMagic.Scraper/README.md` (or an equivalent developer onboarding doc)
- [ ] Starting the AppHost after this story is complete results in `PrintingMagic.Scraper` reaching `Running` state — no `BrowserNotFound` or `PlaywrightException` in startup logs

## Technical Notes

- Playwright binary install command for Linux/container: `RUN pwsh ./playwright.ps1 install chromium --with-deps` or `RUN dotnet tool run playwright install chromium --with-deps` depending on how Playwright is surfaced in the project
- For local dev, the simplest path is a `<Target Name="InstallPlaywright" AfterTargets="Build">` MSBuild target that runs `playwright install chromium` once after build — check if already present in the reference Worker's csproj
- Reference Worker csproj: `TargetProjects/oldversion/printingmagic/PrintingMagic.Worker/PrintingMagic.Worker.csproj`
- CI: if integration tests run in CI (L2 from adversarial review), the CI workflow also needs the `playwright install` step — add as a comment/TODO if CI is not in scope for this sprint

## Dependencies

- newversion-scraper-scraper-scaffolding-3-01 (scraper must be registered in Aspire before this can be validated end-to-end)
