---
feature: newversion-scraper-scraper-scaffolding
doc_type: tech-plan
status: draft
goal: "Port PrintingMagic.Worker scraping pipeline into a standalone .NET 9 Aspire-registered background service using Microsoft.Playwright headless browser automation, baselining from the reference codebase"
key_decisions:
  - Microsoft.Playwright (.NET) as the headless browser — same as reference, avoids new technology risk
  - Isolated project PrintingMagic.Scraper rather than embedding in Worker or ApiService
  - Aspire registration via aspire CLI (aspire add), not manual AppHost csproj edit
  - Credentials injected via Aspire environment variables / connection strings — no hardcoded values
  - All data writes routed through IInternalApiClient — no direct database access from the scraper
  - Shared browser session pattern (BrowserSessionService singleton) preserved from reference
open_questions:
  - Playwright browser binary provisioning strategy for Aspire Dockerfile vs local dev (playwright install vs bundled binary)
depends_on:
  - newversion-apphost-scaffolding
blocks: []
updated_at: "2026-04-09T18:31:00Z"
adr_count: 6
---

# Tech Plan: newversion-scraper-scraper-scaffolding

## Technical Summary

The scraper is a .NET 9 `BackgroundService`-based worker project (`PrintingMagic.Scraper`) that ports the authenticated web-scraping pipeline from `PrintingMagic.Worker` (oldversion reference codebase) into a standalone, first-class Aspire resource. It uses Microsoft.Playwright (headless Chromium, .NET bindings) to authenticate against the STLFlix web UI, discover all drop/collection pages, extract collection and model metadata, and hand the result to a local `IInternalApiClient` for storage. The project is registered in the Aspire AppHost using the `aspire add` CLI command and receives credentials and configuration at runtime through Aspire environment injection. No new scraping technology is introduced — the baseline is the validated reference implementation.

## Architecture Overview

```
Aspire AppHost (PrintingMagic.Aspire)
│
├── PrintingMagic.ApiService      (existing — HTTP target for scraped data)
│
└── PrintingMagic.Scraper         (NEW — this feature)
      ├── Worker (BackgroundService loop, 5-min cycle)
      │     ├── BrowserSessionService (singleton Playwright session + login)
      │     ├── DropProcessingService (discover + scrape all unprocessed drops)
      │     └── CollectionHarvesterJob (scrape → IExternalScraper → IInternalApiClient)
      │
      ├── Scrapers/
      │     └── StlflixScraper : IExternalScraper
      │           └── (Playwright page navigation, selector-based extraction)
      │
      └── IInternalApiClient / InternalApiClient
            └── HTTP → PrintingMagic.ApiService (Aspire service discovery URL)
```

**Components affected:** PrintingMagic.Aspire (AppHost — gains a new resource reference)

**Integration points:**
- Aspire AppHost: `aspire add` registers the scraper project; Aspire injects OTEL endpoint and service URLs
- PrintingMagic.ApiService: ingestion endpoint (POST `/import/collection`, POST `/import/model`) — same contract as reference
- STLFlix web platform: outbound Playwright HTTP via `platform.stlflix.com`

## Design Decisions (ADRs)

### ADR-1: Microsoft.Playwright as the headless browser

**Decision:** Use Microsoft.Playwright (.NET) for headless browser automation.

**Rationale:** The reference codebase uses Playwright successfully in production. It provides reliable authenticated session management, stable selector APIs, and .NET 9 compatibility. Switching to a different browser automation library introduces unvalidated risk and rewrite effort.

**Alternatives rejected:**
- `PuppeteerSharp` — fewer .NET-native APIs, less alignment with the reference implementation
- `Selenium` — heavier, requires external WebDriver server, no advantage over Playwright for this use case
- HTTP API scraping — STLFlix does not expose a documented API; web UI scraping is the only available approach

### ADR-2: Isolated `PrintingMagic.Scraper` project

**Decision:** The scraper lives in its own project, not embedded in `PrintingMagic.Worker` or `PrintingMagic.ApiService`.

**Rationale:** Aspire service isolation and the single-responsibility principle. A separate project allows independent deployment, independent scaling, and independent configuration. It also mirrors the Aspire architecture pattern.

**Alternatives rejected:**
- Embedding in ApiService — violates separation of concerns; API service should not run background browser sessions
- Single Worker project combining scraper + model automation — the oldversion Worker was a known source of coupling; new version separates these explicitly

### ADR-3: Register via `aspire add`

**Decision:** Register `PrintingMagic.Scraper` in the AppHost using `aspire add PrintingMagic.Scraper`.

**Rationale:** The user explicitly requested Aspire CLI integration. `aspire add` generates the correct project reference and resource builder call in `apphost.cs` without manual csproj editing, reducing human error.

**Alternatives rejected:**
- Manual `apphost.cs` edit — error-prone; bypasses Aspire's own project reference tooling

### ADR-4: Credentials via Aspire environment injection

**Decision:** `Stlflix:Email` and `Stlflix:Password` are injected as Aspire environment variables, not hardcoded.

**Rationale:** Security. The reference codebase had a default email hardcoded as a fallback — this is removed in the new version. Credentials are supplied at runtime via Aspire's connection string / environment mechanism and optionally stored in user secrets for local dev.

**Alternatives rejected:**
- Hardcoded fallback credentials — violates OWASP A02:2021 (Cryptographic Failures / secrets in code)
- Config file only — config files may be committed accidentally; Aspire environment injection is preferred

### ADR-5: All data writes through IInternalApiClient

**Decision:** The scraper never writes directly to the database. All scraped data is submitted via `IInternalApiClient` to `PrintingMagic.ApiService`.

### ADR-6: Aspire service discovery for internal API base URL

**Decision:** `InternalApiClient` resolves `PrintingMagic.ApiService` via Aspire service discovery. The `HttpClient` base address is configured as `http+http://apiservice` and the `IServiceCollection` registration calls `.AddServiceDiscovery()`.

**Rationale:** Aspire service discovery is the architecturally preferred approach (see `key_decisions` in the frontmatter). It works transparently in both local Aspire dev (the AppHost injects the resolved endpoint as an environment variable) and any container deployment that retains the Aspire infrastructure. No manual URL management is needed.

**Implementation:**
- `PrintingMagic.ApiService` is registered in the AppHost under the name `"apiservice"` (story 3-01 confirms this).
- `InternalApiClient` is registered with `AddHttpClient<IInternalApiClient, InternalApiClient>(c => c.BaseAddress = new Uri("http+http://apiservice"))` followed by `.AddServiceDiscovery()`.
- No `ApiService:BaseUrl` key is added to `appsettings.json`; the service discovery address is the single source of truth.

**Alternatives rejected:**
- Explicit `IConfiguration` key (`"ApiService:BaseUrl"`) — manual URL management; breaks in Aspire container environments where the host/port are not known at config-authoring time.
- Hardcoded URL — violates separation of configuration and code; non-portable across environments.

**Rationale:** Maintains the same architectural boundary as the reference. The API service owns the data model; the scraper is a pure harvesting agent.

**Alternatives rejected:**
- Direct EF Core / database writes from scraper — creates tightly coupled dependency on data layer; bypasses API validation

## API Contracts

### Outbound: STLFlix web platform (via Playwright)

No formal contract. Navigation targets:
- `https://platform.stlflix.com/sign-in` — authentication
- `https://platform.stlflix.com/drops` — drop discovery
- Per-drop URLs discovered dynamically during scraping

**Breaking change:** false (the scraper reads from STLFlix; it does not expose an inbound API)

### Inbound → PrintingMagic.ApiService (unchanged from reference)

| Endpoint | Method | Body | Notes |
|----------|--------|------|-------|
| `/import/collection` | POST | `CollectionImportRequest` | Array of `CollectionSeed` (title + model URLs) |
| `/import/model` | POST | `ModelImportDto` | (id, empty strings in reference — may be extended) |

**Breaking change:** false — no new endpoints; existing contract reused

## Data Model Changes

No new tables or schema changes. The scraper produces `CollectionImportRequest` → `CollectionSeed[]` → posted to the API service. The API service owns all persistence.

**Schema alignment check required (open question):** Verify `PrintingMagic.Data.Contracts` DTO shapes in new solution match old version before integration.

## Dependencies

- `Microsoft.Playwright` NuGet — headless browser bindings
- `Microsoft.Playwright.MSTest` or equivalent — for selector-level tests
- `PrintingMagic.Data.Contracts` — DTO project (must exist or be created before sprint 1 completes)
- Aspire AppHost project (`newversion-apphost-scaffolding`) — must be registered before Aspire resource registration
- `Microsoft.Extensions.Hosting` — `BackgroundService` base class
- `OpenTelemetry.*` — OTEL SDK packages for Aspire telemetry integration

## Rollout Strategy

**Phase:** Local dev only → confirm scraper runs and Aspire shows it healthy.  
**Feature flag:** None needed (it is a new, additive service with no existing callers).  
**Canary:** Not applicable at this stage.  
**Rollback plan:** If the Aspire resource causes AppHost instability, remove the resource from `apphost.cs` (revert the `aspire add` registration) and delete the `PrintingMagic.Scraper` project reference. No data rollback needed — the scraper only posts to the API; it does not own any tables.

## Testing Strategy

| Level | Scope | Approach |
|-------|-------|----------|
| Unit | `StlflixScraper` selectors | Page fixture HTML + Playwright test harness; assert URLs and titles extracted correctly |
| Unit | `BrowserSessionService` | Mock `IPlaywright`; assert re-login triggered on session invalidation |
| Unit | `CollectionHarvesterJob` | Mock `IExternalScraper` + `IInternalApiClient`; assert concurrency gate (SemaphoreSlim(5)) is respected |
| Integration | Full scrape cycle | Single cycle against live STLFlix (dev credentials); assert `CollectionImportRequest` is non-empty and API posts succeed |
| Aspire smoke test | Service starts | Aspire AppHost launches; `PrintingMagic.Scraper` resource reaches `Running` state with no startup exceptions |

## Observability

Following the reference codebase `ActivitySource` pattern:

```csharp
private static readonly ActivitySource s_activitySource = new("PrintingMagic.Scraper");
```

| Signal | Detail |
|--------|--------|
| Traces | `ActivitySource` spans on `ExecuteAsync`, `ProcessDropsAsync`, `ScrapeAsync`, `LoginAsync`, `GetBrowserAsync` |
| Logs | Structured `ILogger<T>` logs at Info/Warning/Error — same log points as reference |
| Metrics | Aspire default resource metrics (CPU, memory) — no custom counters in scope for this sprint |
| Alerts | None in scope — operator monitors Aspire dashboard |

## Open Questions

1. **Data.Contracts project** — Does `PrintingMagic.Data.Contracts` exist in the new solution? If not, it must be created as a minimal DTO project in sprint 1 before `CollectionImportRequest` and related types can be referenced.
2. **Playwright browser binary provisioning** — For containerized Aspire deployment, `playwright install chromium` must run as part of the container image build. Strategy for local dev: `playwright install` post-project-restore hook vs. manual step?
3. **Internal API URL** — Is `PrintingMagic.ApiService` URL injected via Aspire service discovery (`http+http://apiservice`) or a hardcoded config value? Aspire service discovery is preferred.
