# BDD Test Scenarios — newversion-scraper-scraper-scaffolding

> **Governance artifact** — required by org/domain/service constitutions (`gate_mode: hard`)
>
> Every scenario below traces directly to a story acceptance criterion.
> Scenarios are written in Gherkin (Given/When/Then) and map to xUnit tests using
> FluentAssertions + method-name BDD conventions.
>
> **TDD contract:** Each scenario header MUST exist as a failing test before the
> corresponding implementation is written.

---

## Sprint 1 — Worker Scaffold

### Story 1-01 — Worker project with Playwright and configuration

```gherkin
Feature: PrintingMagic.Scraper project scaffold

  Scenario: Worker project exists with BackgroundService skeleton
    Given the PrintingMagic.Scraper project has been built from source
    When the Worker class is inspected via reflection
    Then it derives from BackgroundService
    And its namespace is PrintingMagic.Scraper

  Scenario: Microsoft.Playwright NuGet package is referenced
    Given the PrintingMagic.Scraper project file
    When its PackageReference elements are read
    Then a reference to Microsoft.Playwright exists

  Scenario: Project builds with no warnings on .NET 9
    Given the full solution
    When dotnet build is executed with -warnaserror
    Then the exit code is 0

  Scenario: ActivitySource named PrintingMagic.Scraper is declared in Worker
    Given the Worker class is loaded
    When its static fields are inspected
    Then an ActivitySource field with Name "PrintingMagic.Scraper" is present

  Scenario: Polite delay lower bound defaults to 2000 ms
    Given a Worker instance with empty IConfiguration
    When the polite delay lower bound constant is read
    Then it equals 2000

  Scenario: Polite delay upper bound defaults to 5000 ms
    Given a Worker instance with empty IConfiguration
    When the polite delay upper bound constant is read
    Then it equals 5000

  Scenario: Polite delay lower bound is IConfiguration-backed
    Given an IConfiguration with key Worker:PoliteDelayLowerMs set to 1000
    When a Worker is constructed with that configuration
    Then the effective polite delay lower bound is 1000

  Scenario: Scrape cycle delay defaults to 5 minutes
    Given a Worker instance with empty IConfiguration
    When the scrape cycle delay is read
    Then it equals TimeSpan 5 minutes
```

---

### Story 1-02 — BrowserSessionService and LoginService

```gherkin
Feature: BrowserSessionService concurrency and session state

  Scenario: GetBrowserAsync calls PerformLoginAsync when session is invalidated
    Given a BrowserSessionService with a mock ILoginService
    And the session has been invalidated
    When GetBrowserAsync is called
    Then ILoginService.PerformLoginAsync is called exactly once

  Scenario: GetBrowserAsync does not re-login when session is valid
    Given a BrowserSessionService with a mock ILoginService
    And the session is valid and logged in
    When GetBrowserAsync is called
    Then ILoginService.PerformLoginAsync is NOT called

  Scenario: SemaphoreSlim concurrency guard prevents concurrent login
    Given two concurrent calls to GetBrowserAsync on a BrowserSessionService
    When both calls proceed simultaneously
    Then only one PerformLoginAsync call is made

  Scenario: ActivitySource span is created for GetBrowserAsync
    Given a BrowserSessionService with OpenTelemetry configured
    When GetBrowserAsync is called
    Then an activity span named GetBrowserAsync is recorded

Feature: LoginService credential handling

  Scenario: LoginService throws InvalidOperationException when Stlflix:Email is missing
    Given an IConfiguration with only Stlflix:Password configured
    When LoginService is constructed
    Then an InvalidOperationException is thrown mentioning Stlflix:Email

  Scenario: LoginService throws InvalidOperationException when Stlflix:Password is missing
    Given an IConfiguration with only Stlflix:Email configured
    When LoginService is constructed
    Then an InvalidOperationException is thrown mentioning Stlflix:Password

  Scenario: LoginService reads email from IConfiguration without hardcoded fallback
    Given an IConfiguration with Stlflix:Email set to "test@example.com"
    And Stlflix:Password set to "secret"
    When LoginService reads the email
    Then the email is "test@example.com" with no hardcoded fallback applied
```

---

### Story 1-03 — Data contracts / DTOs

```gherkin
Feature: PrintingMagic.Data.Contracts DTOs

  Scenario: CollectionImportRequest holds a list of CollectionSeed items
    Given a CollectionImportRequest constructed with two CollectionSeed items
    When its Collections property is read
    Then it contains exactly two items

  Scenario: CollectionSeed has Title and ModelUrls properties
    Given a CollectionSeed with Title "TestCollection" and two model URLs
    When its properties are read
    Then Title equals "TestCollection"
    And ModelUrls contains two entries

  Scenario: PrintingMagic.Scraper references PrintingMagic.Data.Contracts and builds cleanly
    Given the full solution
    When dotnet build is executed
    Then the exit code is 0
    And no CS assembly reference errors appear in the build output
```

---

## Sprint 2 — Scraper Core Logic

### Story 2-01 — StlflixScraper

```gherkin
Feature: StlflixScraper URL discovery and page scraping

  Scenario: DiscoverDropUrls extracts expected URLs from a drops-page HTML fixture
    Given an HTML fixture representing the STLFlix drops page with 3 drop links
    When StlflixScraper.DiscoverDropUrls is called with that HTML
    Then it returns exactly 3 drop URLs

  Scenario: ScrapeOneDropWithRetry extracts correct CollectionSeed from a single-drop HTML fixture
    Given an HTML fixture for a single STLFlix drop page with title "Drop42"
    And the page contains 2 model URLs
    When ScrapeOneDropWithRetry is called with that fixture
    Then a CollectionSeed with Title "Drop42" and 2 ModelUrls is returned

  Scenario: Session invalidation triggers re-login and retry on login-page redirect
    Given a StlflixScraper whose IBrowserSessionService is mocked
    And the first navigation redirects to the login page
    And the second navigation succeeds
    When ScrapeAsync is called
    Then IBrowserSessionService.InvalidateSession is called once
    And the scrape completes successfully on retry

  Scenario: DropsPageUrl is read from IConfiguration
    Given IConfiguration with key Scraper:DropsPageUrl set to "https://example.com/drops"
    When StlflixScraper is constructed
    Then its effective DropsPageUrl is "https://example.com/drops"

  Scenario: CSS selectors are declared as constants, not inline strings
    Given the StlflixScraper source file
    When its private string constants are enumerated
    Then at least one constant matching a CSS/ARIA selector pattern is present

  Scenario: ActivitySource span is created for ScrapeAsync
    Given a StlflixScraper with OpenTelemetry configured
    When ScrapeAsync is called
    Then an activity span named ScrapeAsync is recorded with source "PrintingMagic.Scraper"

  Scenario: Legal comment on automated scraping is preserved in the class summary
    Given the StlflixScraper source file
    When its XML doc summary is read
    Then it contains the text "legal review"
    And it contains the text "ToS"
```

---

### Story 2-02 — DropProcessingService

```gherkin
Feature: DropProcessingService drop processing

  Scenario: Only the first drop is processed when SingleDropMode is true
    Given DropProcessingService configured with SingleDropMode = true
    And a mock IBrowserSessionService that returns 3 drop URLs
    When ProcessDropsAsync is called
    Then IInternalApiClient.ImportCollectionAsync is called exactly once

  Scenario: All drops are processed when SingleDropMode is false
    Given DropProcessingService configured with SingleDropMode = false
    And a mock IBrowserSessionService that returns 3 drop URLs
    When ProcessDropsAsync is called
    Then IInternalApiClient.ImportCollectionAsync is called 3 times

  Scenario: InvalidateSession is called and retry loop continues on login-page redirect
    Given DropProcessingService with a mock IBrowserSessionService
    And the first page navigation redirects to the login page
    And the second navigation succeeds
    When ProcessDropsAsync is called
    Then IBrowserSessionService.InvalidateSession is called once
    And the process completes without throwing

  Scenario: ActivitySource span is created for ProcessDropsAsync
    Given a DropProcessingService with OpenTelemetry configured
    When ProcessDropsAsync is called
    Then an activity span named ProcessDropsAsync is recorded
```

---

### Story 2-03 — InternalApiClient and CollectionHarvesterJob

```gherkin
Feature: CollectionHarvesterJob model processing

  Scenario: ProcessModelAsync is called exactly once per model ID returned
    Given mock IExternalScraper returning 1 collection with 5 model IDs
    And mock IInternalApiClient
    When CollectionHarvesterJob.RunAsync is called
    Then IInternalApiClient.ProcessModelAsync is called exactly 5 times
    And each call uses a distinct model ID from the collection

  Scenario: Exception from ScrapeAsync propagates from RunAsync
    Given mock IExternalScraper that throws InvalidOperationException
    When CollectionHarvesterJob.RunAsync is called
    Then the exception propagates without being swallowed

  Scenario: ModelConcurrency limit is respected
    Given IConfiguration with Worker:ModelConcurrency set to 2
    And a collection with 10 model IDs
    When RunAsync is called
    Then at most 2 ProcessModelAsync calls are in-flight simultaneously

  Scenario: ModelConcurrency defaults to 5
    Given IConfiguration with no Worker:ModelConcurrency key
    When CollectionHarvesterJob is constructed
    Then the effective concurrency limit is 5

  Scenario: ActivitySource span is created for RunAsync
    Given a CollectionHarvesterJob with OpenTelemetry configured
    When RunAsync is called
    Then an activity span named RunAsync is recorded
```

---

### Story 2-04 — API URL strategy (ADR-6)

> **Note:** Story 2-04 is an architecture decision record (ADR) story; its ACs are
> verified by inspection (code review / governance doc check) rather than automated tests.
> The BDD scenario below is a documentation-completeness check only.

```gherkin
Feature: ADR-6 API URL strategy decision

  Scenario: ADR-6 entry exists in tech-plan
    Given the feature's tech-plan.md at governance path
    When the document is scanned for ADR-6
    Then a section or entry labelled "ADR-6" is present
    And it contains a decision for either Aspire service discovery or explicit config

  Scenario: InternalApiClient uses the decided URL strategy without a TODO placeholder
    Given the InternalApiClient source file after story 2-03 is complete
    When the file is scanned for the string "TODO: wire ADR-6 URL"
    Then no such TODO remains (it has been resolved per the ADR decision)
```

---

## Sprint 3 — Aspire Integration and Observability

### Story 3-01 — Aspire registration

```gherkin
Feature: PrintingMagic.Scraper Aspire registration

  Scenario: aspire add generates a valid project reference in AppHost
    Given the PrintingMagic.Aspire AppHost project
    When assembled project references are inspected
    Then a reference to PrintingMagic.Scraper exists

  Scenario: Stlflix:Email is injected via Aspire environment variable (not hardcoded)
    Given the AppHost builder configuration
    When Stlflix-related environment variable wiring is inspected
    Then WithEnvironment for Stlflix__Email is configured to read from a secret or parameter
    And no string literal matching an email address appears in the builder code

  Scenario: Stlflix:Password is injected via Aspire environment variable (not hardcoded)
    Given the AppHost builder configuration
    When Stlflix-related environment variable wiring is inspected
    Then WithEnvironment for Stlflix__Password is configured to read from a secret or parameter
    And no string literal matching a password appears in the builder code

  Scenario: AppHost starts with scraper service in Running state
    Given a fully configured AppHost with PrintingMagic.Scraper registered
    When DistributedApplicationTestingBuilder starts the app
    Then the resource named "scraper" reaches Running state within 60 seconds
```

---

### Story 3-02 — Playwright browser install

```gherkin
Feature: Playwright Chromium installation for local dev and containers

  Scenario: AppHost starts scraper without BrowserNotFound exception after playwright install
    Given the Playwright Chromium browser is installed
    When the AppHost is started and PrintingMagic.Scraper initialises
    Then no PlaywrightException or BrowserNotFoundError appears in startup logs

  Scenario: README documents the playwright install step for developers
    Given the PrintingMagic.Scraper README.md
    When it is read
    Then it contains the string "playwright install chromium"
```

---

### Story 3-03 — OpenTelemetry wiring

```gherkin
Feature: OpenTelemetry tracing and structured logging via Aspire

  Scenario: ActivitySource PrintingMagic.Scraper is registered with OTEL SDK
    Given the PrintingMagic.Scraper Program.cs
    When the OpenTelemetry service registration is inspected
    Then AddSource("PrintingMagic.Scraper") is present in the WithTracing configuration

  Scenario: At least one trace span appears in Aspire dashboard after a scrape cycle
    Given the AppHost is running with scraper registered
    When Worker.ExecuteAsync completes one scrape iteration (or stub iteration)
    Then the Aspire dashboard reports at least one trace with source "PrintingMagic.Scraper"

  Scenario: Structured logs from scraper are visible in Aspire dashboard within 30 seconds
    Given the AppHost starts with PrintingMagic.Scraper in Running state
    When 30 seconds elapse
    Then at least one structured log line from the scraper appears in the Aspire UI console view
```

---

### Story 3-04 — Legal compliance (governance story — no automated tests)

> Story 3-04 is a governance/process artifact only. Its acceptance criteria are verified
> by peer review and documented sign-off, not automated tests.

---

## Test Organisation Reference

| Test class | Story coverage | Framework |
|---|---|---|
| `WorkerConfigurationTests` | 1-01 | xUnit + FluentAssertions |
| `BrowserSessionServiceTests` | 1-02 | xUnit + NSubstitute |
| `LoginServiceTests` | 1-02 | xUnit + FluentAssertions |
| `DataContractsTests` | 1-03 | xUnit |
| `StlflixScraperTests` | 2-01 | xUnit + NSubstitute + HTML fixture files |
| `DropProcessingServiceTests` | 2-02 | xUnit + NSubstitute |
| `CollectionHarvesterJobTests` | 2-03 | xUnit + NSubstitute |
| `Adr6UrlStrategyTests` | 2-04 | xUnit (inspection-based) |
| `AspireRegistrationTests` | 3-01 | xUnit + Aspire.Testing |
| `PlaywrightInstallTests` | 3-02 | xUnit (documentation check) |
| `OtelWiringTests` | 3-03 | xUnit + Aspire.Testing |

All test projects MUST be placed under `PrintingMagic.Scraper.Tests/` with a project reference to `PrintingMagic.Scraper`. Integration tests that start the Aspire AppHost should be in a separate `PrintingMagic.Scraper.IntegrationTests/` project.
