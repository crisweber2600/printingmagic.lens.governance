# Retrospective: newversion-processor-storage

**Feature:** newversion-processor-storage  
**Completed:** 2026-04-11T00:00:00Z  
**Owner:** cweber

---

## What Went Well

- **Multi-repo structure held up** — the contracts → servicedefaults → scraper → processor → apphost dependency chain was clean once all repos were set up. Each service's concerns remained well-isolated.
- **MassTransit idempotency pattern was straightforward** — catching `PostgresException { SqlState: PostgresErrorCodes.UniqueViolation }` via `DbUpdateException` inner exception worked exactly as designed. No false faults.
- **EF Core migrations were painless** — `dotnet-ef` generated the `InitialCreate` migration correctly on the first run. Schema matched entities exactly.
- **Aspire registration was clean** — the script-based apphost format (`#:sdk`, `#:project`, `#:package` directives) is elegant for simple use cases. Build via `dotnet build apphost.cs` worked as expected.
- **Minimal Aspire csproj wrapper worked** — creating `PrintingMagic.Aspire.csproj` alongside `apphost.cs` elegantly solved the integration test project reference problem without duplicating the apphost entrypoint.
- **4/4 unit tests passed on first complete run** — consumer tests (idempotency + happy path) and endpoint tests (found + not-found) all green with clear BDD names.
- **Connection string naming caught early** — the `"rabbitmq"` vs `"message-broker"` mismatch was caught during S7 review before end-to-end testing.

## What Didn't Go Well

- **Git commit to wrong repo in S7** — the S7 apphost commit landed in the workspace root git repo instead of the apphost's embedded `.git` repo at `apphost/PrintingMagic.Aspire/.git`. This required remediation at the start of the next session. Root cause: `git add -A` from the workspace root staged workspace-level artifacts rather than the intended `apphost.cs` change.
- **Session context limit required mid-sprint summary** — the implementation spanned two sessions due to token budget constraints. The conversation summary accurately preserved all context, but this added coordination overhead.
- **`MassTransit.Testing` namespace confusion** — the namespace exists inside the main `MassTransit` package, not a separate `MassTransit.Testing` NuGet. A non-obvious API surface that took extra investigation.
- **`IConsumerTestHarness<T>` API** — `harness.Consumer<T>()` doesn't exist; the correct API is `harness.GetConsumerHarness<T>()`. The test harness API is not well-documented relative to its power.
- **Scoped DbContext in tests** — resolving a scoped service from the root `IServiceProvider` throws. Required `provider.CreateAsyncScope()` pattern, which is correct but not obvious.
- **`ResourceNotificationService.WaitForResourceAsync`** — the Aspire 13.x API for waiting on resource state is on `ResourceNotificationService`, not directly on `DistributedApplication` as suggested in the story spec. Took extra investigation to discover the correct pattern.

## Key Learnings

1. **Git ops discipline is critical in multi-repo setups** — always `cd` into the correct repo before `git add` / `git commit`. Never use workspace-root `git add -A` when targeting an embedded repo.
2. **`MassTransit.Testing` is in the main package** — no separate NuGet. Use `AddMassTransitTestHarness` + `harness.GetConsumerHarness<T>()`.
3. **Aspire integration tests need a `.csproj`** — script-based apphosts (`apphost.cs`) need a thin `.csproj` wrapper (`<Compile Remove="apphost.cs" />` + explicit project references) for integration test `ProjectReference` to work.
4. **Aspire 13.x resource waiting** — use `_app.Services.GetRequiredService<ResourceNotificationService>().WaitForResourceAsync("name", KnownResourceStates.Running)`.
5. **`DOTNET_ROOT` required for `dotnet-ef`** — global tool needs `DOTNET_ROOT=/home/cweber/.dotnet` exported; not always set in CI-like environments.
6. **`PostgresException` direct construction works** — `new PostgresException(msg, severity, invariantSeverity, sqlState)` is a valid public constructor in Npgsql 10.0.2 for test harness use.

## Metrics

- **Planned duration:** 1 sprint (~1 day)
- **Actual duration:** ~2 sessions (split by context limit)
- **Stories completed:** 8/8 (S1–S8), all 26 points
- **Tests delivered:** 7 unit tests total (4 processor, 3 integration test scaffolded — requires Docker to run), 0 failures
- **Scraper regression:** 16/16 existing scraper tests unaffected by S3 migration
- **Git remediations needed:** 1 (S7 wrong-repo commit)
- **Bugs found post-merge:** 0 known

## Action Items

- [ ] Run integration tests in a Docker-capable CI environment to validate `ProcessorStartupTests` (requires Postgres + RabbitMQ containers)
- [ ] Update `sprint-status.yaml` to reflect all stories as `complete`
- [ ] Evaluate extracting the `ThrowingDbContext` test helper to a shared test utilities package if more services need it
