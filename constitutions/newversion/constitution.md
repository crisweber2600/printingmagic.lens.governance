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

# newversion Domain Constitution

## Scope

Governance for all features under the `newversion` domain (Aspire-based .NET 9 rewrite).

## Domain-Level TDD/BDD Additions

All org-level TDD/BDD requirements apply. This domain adds:

### .NET / xUnit BDD Convention

For .NET services in this domain, BDD scenarios MUST be implemented using **xUnit** with a naming convention that reflects the Given/When/Then structure:

```csharp
// Recommended: FluentAssertions + xUnit
public class WorkerBackgroundServiceTests
{
    [Fact]
    public async Task GivenSolutionBuilt_WhenScraperProjectInspected_ThenWorkerDerivesFromBackgroundService()
    {
        // Arrange (Given)
        // Act (When)
        // Assert (Then — maps to AC)
    }
}
```

### Aspire Integration Tests

Features that register Aspire resources MUST include at least one integration test that validates the resource starts and is reachable via Aspire's `DistributedApplicationTestingBuilder`.

## artifact Requirements (additive to org)

| Phase | Artifact   | Gate | Description |
|-------|------------|------|-------------|
| dev   | bdd-tests  | hard | BDD scenarios (reinforced at domain level) |
