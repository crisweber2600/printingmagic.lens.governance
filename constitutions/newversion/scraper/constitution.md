---
permitted_tracks: [quickplan, full, hotfix, tech-change, express, hotfix-express]
required_artifacts:
  dev:
    - bdd-tests
gate_mode: hard
additional_review_participants: []
enforce_stories: true
enforce_review: false
---

# newversion/scraper Service Constitution

## Scope

Governance for all features under the `newversion` domain, `scraper` service (Playwright-based background scraper).

## Service-Level TDD/BDD Additions

All org-level and domain-level TDD/BDD requirements apply. This service adds:

### Playwright Browser Automation BDD

For stories that involve browser automation (Playwright):

- Each page-navigation or data-extraction behaviour MUST have a BDD scenario
- Scenarios MUST cover the **happy path** AND at least one **failure / retry path** (network timeout, selector not found, etc.)
- Use `PlaywrightTest` base class or equivalent; do **not** use raw `Playwright.CreateAsync()` in test constructors

### Scrape-Cycle BDD Coverage Requirement

Every scrape cycle path (normal loop, error path, graceful shutdown via `CancellationToken`) requires a dedicated BDD scenario mapping to the story acceptance criteria for that path.

### Polite Delay Contract

BDD scenarios for stories involving rate-limit delays MUST verify that:
1. The configured lower/upper bounds are respected (no request faster than `PoliteDelayLower`)
2. The token-cancellation path exits cleanly without bypassing delays

## Artifact Requirements (additive to org + domain)

| Phase | Artifact  | Gate | Description |
|-------|-----------|------|-------------|
| dev   | bdd-tests | hard | Playwright + scraper-specific BDD coverage (reinforced) |
