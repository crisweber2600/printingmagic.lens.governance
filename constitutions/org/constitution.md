---
permitted_tracks: [quickplan, full, hotfix, tech-change, express, hotfix-express]
required_artifacts:
  planning:
    - business-plan
    - tech-plan
  dev:
    - stories
    - bdd-tests
gate_mode: hard
additional_review_participants: []
enforce_stories: true
enforce_review: false
---

# Org-Level Governance Constitution

## Test-Driven Development (TDD) — Mandatory

All code produced across every domain and service MUST follow **Test Driven Development (TDD)**:

1. **Red** — Write a failing test that reflects an acceptance criterion or behaviour before writing any implementation code.
2. **Green** — Write the minimum implementation code to make the test pass.
3. **Refactor** — Clean up code and tests while keeping tests green.

No story may be considered `done` until all its acceptance criteria have corresponding passing tests written *before* the implementation.

## Behaviour Driven Development (BDD) — Mandatory

All automated tests MUST be expressed in **BDD style** (Given / When / Then), and each BDD scenario MUST directly trace to a story's acceptance criteria:

- **Given** — the precondition or system state
- **When** — the action or event
- **Then** — the expected outcome, matching an acceptance criterion

BDD test files MUST be checked in as `bdd-tests.md` (feature-level) or in a `bdd-tests/` directory (per-story breakdown) alongside the feature's story artifacts. This file documents the BDD scenarios mapped to each story's acceptance criteria.

### Acceptance Criteria Format — Mandatory

Every acceptance criterion written anywhere in this project (stories, PRDs, feature plans, or any planning artifact) MUST be expressed in **Given / When / Then** format:

```
Given {precondition or initial system state}
When {action or event}
Then {expected observable outcome}
```

Single-line checkbox form is permitted where brevity is required, provided the Given/When/Then structure is still legible:

```
- [ ] Given {state}, When {action}, Then {outcome}
```

Free-text acceptance criteria that cannot be mapped to a Given/When/Then structure MUST be rewritten before the artifact is promoted. This rule applies at every planning phase and is enforced at story creation time.

### BDD → AC Traceability Rule

Each acceptance criterion checkbox in a story file (`- [ ] ...`) MUST have a corresponding `Scenario:` in the BDD test suite. The scenario title SHOULD quote or paraphrase the acceptance criterion verbatim to make traceability explicit.

**Example mapping:**

Story AC:
```
- [ ] `PrintingMagic.Scraper` project exists with a `BackgroundService`-derived `Worker` class skeleton
```

BDD Scenario:
```gherkin
Scenario: Worker project exists with BackgroundService skeleton
  Given the solution has been built from source
  When the PrintingMagic.Scraper project is inspected
  Then a class derived from BackgroundService named Worker is present
```

## Artifact Requirements

| Phase    | Artifact      | Gate  | Description |
|----------|---------------|-------|-------------|
| planning | business-plan | hard  | Business justification and goals |
| planning | tech-plan     | hard  | Technical design and approach |
| dev      | stories       | hard  | User stories with acceptance criteria |
| dev      | bdd-tests     | hard  | BDD scenarios mapped to story acceptance criteria |

## Enforcement

- `gate_mode: hard` — compliance failures **block** workflow promotion
- `enforce_stories: true` — the `stories` artifact is always required in `dev` phase
- Missing `bdd-tests` artifact at the start of `dev` phase MUST be created before the first story is coded
