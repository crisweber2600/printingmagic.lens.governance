# Adversarial Review: newversion-processor-storage / techplan

**Reviewed:** 2026-04-11T00:00:00Z
**Source:** phase-complete
**Overall Rating:** pass-with-warnings

---

## Summary

The architecture for `newversion-processor-storage` is well-structured and covers all 19 functional requirements and 7 NFRs. The data model, consumer registration, API surface, Aspire integration, and project layout are coherent and implementation-ready. All 9 findings (2 High, 4 Medium, 3 Low) have been reviewed and resolved. Both architecture documents (governance + staged docs/) have been updated accordingly. The phase is clear to advance to `/finalizeplan`.

**Recommendation:** advance to `/finalizeplan`.

---

## Findings

### Critical

*None.*

---

### High

| # | Dimension | Finding | Decision |
|---|-----------|---------|----------|
| H-1 | Logic Flaw | The staged `docs/` architecture references `PrintingMagic.Contracts` + `ModelDiscoveredV1` while the governance copy used `PrintingMagic.Data.Contracts` + `ModelProcessingRequestedV1`. | **🔵 b — RESOLVED:** Governance architecture updated to match staged. `PrintingMagic.Contracts` (new library in `TargetProjects/newversion/contracts/`) and `ModelDiscoveredV1` are now the authoritative names in both copies. Consumer renamed `ProcessModelConsumer`. |
| H-2 | Cross-Feature Dependency | The scraper-side publish step was undocumented with no owner. | **🔵 b — RESOLVED:** New §Cross-Repo Dev Tasks section added to both architecture documents. Three scraper-side tasks listed, all owned by cweber. |

---

### Medium

| # | Dimension | Finding | Decision |
|---|-----------|---------|----------|
| M-1 | Coverage Gap | `listing_ready` had no defined set condition. | **🟡 a — RESOLVED:** `listing_ready` is always `false` in this foundation sprint; AI enrichment feature is responsible for setting it `true`. Comment added to entity definition in both architecture docs. |
| M-2 | Assumptions & Blind Spots | Endpoints were open/unauthenticated with no acknowledgement. | **🟡 a — RESOLVED:** Explicit security note added to API surface sections in both architecture docs: “Endpoints are intentionally unauthenticated in this foundation sprint.” |
| M-3 | Coverage Gap | ServiceDefaults copy drift risk was untracked. | **🔵 b — RESOLVED:** `PrintingMagic.ServiceDefaults` promoted to NuGet package, IN SCOPE for this feature. ADR-06 (governance) and ADR-07 (staged) updated. Project structure updated to show `servicedefaults/` repo. Scraper update task added to Cross-Repo Dev Tasks. |
| M-4 | Logic Flaw | Architecture §14 referenced stale `/devproposal`. | **🟡 a — RESOLVED:** Governance §14 was already fixed (previous commit). Staged §Next Phase added as new §14 with `/finalizeplan`. |

---

### Low

| # | Dimension | Finding | Decision |
|---|-----------|---------|----------|
| L-1 | Coverage Gap | `ProcessedProduct.UpdatedAt` had no documented update rule (only an insert default). | **🟡 a — RESOLVED:** Rule added to §10 consistency rules (governance) and EF Core notes (staged): callers must explicitly set `entity.UpdatedAt = DateTime.UtcNow` before every `SaveChangesAsync` call on update. DB default is insert-only. |
| L-2 | Coverage Gap | `POST /imports/model` HTTP intake path had no specified test. | **🟡 a — RESOLVED:** AC added to testing strategy in both architecture docs for a unit test of the HTTP intake path (`GivenValidModelImportDto_WhenPostToImportsModel_ThenReturns201WithProductId`). |
| L-3 | Complexity & Risk | CI workspace layout for relative `<ProjectReference>` paths was undocumented. | **🟡 a — RESOLVED:** Workspace layout requirement block added to ADR-05 (governance) and §4 Project Structure blockquote (staged) documenting expected checkout layout and CI enforcement requirement. |

---

## Party-Mode Challenge Round

**Winston (Architect):** The governance architecture is solid — but I'm bothered by the scraper-publish task living in a footnote at the end of ADR-02 with no owner. In a sprint with only cweber, it's probably fine. In a team, this uncategorized cross-repo task is the thing that falls through. Have you decided whether the scraper publish is its own story or just a note inside the processor epic?

**John (PM):** `listing_ready` is a ticking bomb. Every feature downstream — the listing service, the UI — is going to ask "how do I know when a product is ready to show?" and the answer right now is "we'll figure it out later." That's fine for a storage foundation, but the PRD says it's a `bool` with business meaning. If you don't define the rule now, the AI enrichment feature will define it for you under time pressure. Is the rule just `compliance_status == clear`? Pin it.

**Quinn (QA):** My concern is the HTTP intake path. The consumer is unit-tested. The `POST /imports/model` endpoint is the fallback test/manual path, and it uses "the same handler logic as the consumer" — that phrase is doing a lot of work. Does that mean it calls the consumer logic directly? Via shared service? If there's shared logic, both paths should be tested. If they diverge, you have two intake paths with one test. Which is it?

### Blind-Spot Challenge Questions

1. **If the RabbitMQ `message-broker` resource fails to start in Aspire, what happens to the processor?** MassTransit will fail consumer registration — does the processor start at all, and is `WaitFor(rabbitMq)` handling this correctly?
2. **How does the processor uniqueness constraint behave under a partial duplicate?** `ScraperModelId` has a unique index. What happens if `SaveChangesAsync` throws a unique-key violation because the scraper publishes the same message twice before idempotency check completes? Is there a retry that could cause a double-insert window?
3. **What happens to messages published before the processor is first deployed?** In a fresh environment where the processor's queue doesn't exist yet, does MassTransit auto-create it, or are messages from the scraper lost?
4. **Who owns `ProcessorDbContext` migrations lock?** If two developers run the processor simultaneously in local dev, `MigrateAsync()` on startup could race. Is there an EF migration lock mechanism (e.g., Npgsql advisory locks)?
5. **Is the `Data.Contracts` relative path reference tested in the current CI pipeline for the scraper?** If the scraper CI runs in isolation (no `processor` sibling checkout), does its build still pass?

---

## Constitutional Compliance

| Gate | Status | Evidence |
|------|--------|---------|
| xUnit BDD Given/When/Then naming | ✅ PASS | §10 consistency rule; `ProcessModelConsumerTests.cs` naming convention stated |
| Aspire integration test (hard gate) | ✅ PASS | `ProcessorAppHostTests.cs` with `DistributedApplicationTestingBuilder` specified |
| TDD / Red-Green-Refactor | ✅ PASS (informational) | Consumer unit test + integration test stubs specified; full TDD execution is dev-phase |
| BDD artifacts (bdd-tests.md) | ⚠️ DEFERRED | Required at dev phase; not yet due at techplan |
| Stories | ⚠️ DEFERRED | Required at finalizeplan/dev phase |

---

## Verdict

**pass-with-warnings**

Findings H-1 and H-2 are high severity but do not represent logic failures in the architecture design itself — they represent documentation inconsistency and a missing dependency tracking record that must be resolved before story authoring. The architecture fully covers the PRD requirements, honors the constitutional constraints, and provides an implementation-ready scaffold.

**Caller instruction (phase-complete): advance to `techplan-complete` is permitted.** Resolve H-1 (staged/governance doc alignment) and H-2 (scraper publish task ownership) before FinalizePlan creates dev stories.
