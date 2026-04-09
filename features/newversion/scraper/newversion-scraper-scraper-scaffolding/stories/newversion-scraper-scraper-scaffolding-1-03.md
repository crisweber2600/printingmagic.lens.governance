# Story: newversion-scraper-scraper-scaffolding-1-03

## Context

**Feature:** newversion-scraper-scraper-scaffolding
**Sprint:** 1
**Priority:** critical (H1 gate — all sprint 2 stories are blocked until this is done)
**Estimate:** S (may expand to M if Data.Contracts project does not exist)

## User Story

As a developer, I want `PrintingMagic.Data.Contracts` to exist in the new solution with `CollectionImportRequest`, `CollectionSeed`, and `ModelImportDto` schema-aligned with the old version, so that the scraper project can reference these types without compilation errors.

## Acceptance Criteria

- [ ] Determine whether `PrintingMagic.Data.Contracts` (or an equivalent DTO project) exists in the new solution
- [ ] If it exists: verify that `CollectionImportRequest`, `CollectionSeed`, and `ModelImportDto` have the same shapes as the old version — document any differences as an ADR addition to the tech plan
- [ ] If it does NOT exist: create a minimal `PrintingMagic.Data.Contracts` project with these three DTOs matching the old version exactly
- [ ] `PrintingMagic.Scraper` project references `PrintingMagic.Data.Contracts` and the build is clean
- [ ] A comment is added to each DTO noting the old-version source file for future alignment checks
- [ ] Outcome (exists-and-aligned / exists-needs-update / created-new) is recorded as a note in the sprint plan open questions section

## Technical Notes

- Old version reference: `TargetProjects/oldversion/printingmagic/PrintingMagic.Data/` (check for Contracts namespace)
- DTOs used by the scraper:
  - `CollectionImportRequest(IReadOnlyList<CollectionSeed> Collections)`
  - `CollectionSeed` — at minimum: `string Title`, `IReadOnlyList<string> ModelUrls`
  - `ModelImportDto(int Id, string ?, string ?, string ?)` — check reference `CollectionHarvesterJob.cs` for parameter shapes
- This story is intentionally small if the project already exists; the acceptance criteria are a checklist, not a build exercise

## Dependencies

- newversion-scraper-scraper-scaffolding-1-01 (project must exist to accept the reference)
