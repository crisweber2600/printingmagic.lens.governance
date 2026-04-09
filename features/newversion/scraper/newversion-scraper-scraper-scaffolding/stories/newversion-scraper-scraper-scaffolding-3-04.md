# Story: newversion-scraper-scraper-scaffolding-3-04

## Context

**Feature:** newversion-scraper-scraper-scaffolding
**Sprint:** 3
**Priority:** high
**Estimate:** S

## User Story

As the operator, I want a legal gate story to exist as an explicit blocking item in the governance tracker, so that the scraper service is never deployed to production before legal review of the STLFlix Terms of Service is complete.

## Acceptance Criteria

- [ ] This story is created in the governance stories directory with status `Won't Start` until legal sign-off is received
- [ ] The story explicitly states that `PrintingMagic.Scraper` must not serve production traffic until a legal review of STLFlix's ToS has confirmed that automated scraping is permitted
- [ ] The story links to the relevant section in `StlflixScraper.cs` where the existing legal note is preserved (from story 2-01 acceptance criteria)
- [ ] Any future "production deployment" story for this feature lists this story as a dependency
- [ ] The acceptance criterion for this story is: written confirmation from the authorised reviewer that automated scraping is approved under the applicable ToS

## Technical Notes

- This story is a governance/process artifact, not a code change
- Status on creation: `Won't Start` (blocked on external action)
- This story does NOT block dev, build, or local test activities — it blocks only production deployment
- Reference: `StlflixScraper.cs` XML summary legal note: "Must not go live in production until legal review of target platform's ToS is complete (PRD Section 9)"

## Dependencies

- None (this story can be created independently)
