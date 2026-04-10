# PrintingMagic.Scraper

**Feature ID:** newversion-scraper-scraper-scaffolding
**Domain:** newversion / scraper
**Delivered:** 2026-04-10

## Overview

`PrintingMagic.Scraper` is a standalone .NET background worker service that harvests STLFlix collection and model metadata using Microsoft Playwright headless browser automation. It is registered as an Aspire project resource alongside PostgreSQL and RabbitMQ, and publishes harvested data for downstream processing.

> **Legal gate:** This service must not serve production traffic until a legal review of the STLFlix Terms of Service confirms that automated scraping is permitted. See story 3-04.

## How It Works

1. **Worker loop** — `Worker` runs on a configurable polling interval, invoking `DropProcessingService` each cycle.
2. **Browser session** — `BrowserSessionService` manages a persistent Playwright Chromium session with automatic login and session-validity testing.
3. **Login** — `LoginService` performs STLFlix login using credentials from configuration (`Stlflix:Email`, `Stlflix:Password`). Retries up to 3 times.
4. **Drop discovery** — `StlflixScraper` navigates to the drops listing page and scrapes available drop URLs.
5. **Collection harvest** — `CollectionHarvesterJob` iterates discovered drops, scrapes model metadata for each, and publishes `CollectionImportRequest` events.
6. **API client** — `IInternalApiClient` / `InternalApiClient` POSTs harvest results to the internal API service. No direct database writes from the scraper.

## Configuration

| Key | Default | Required | Description |
|-----|---------|----------|-------------|
| `Stlflix:Email` | — | ✓ | STLFlix account email (injected via Aspire parameter `stlflix-email`) |
| `Stlflix:Password` | — | ✓ | STLFlix account password (injected via Aspire parameter `stlflix-password`) |
| `Scraper:PollingIntervalMinutes` | `5` | — | Interval between scraper cycles |
| `Scraper:BrowserOptions:Headless` | `true` | — | Run Chromium headlessly |
| `Scraper:StlflixOptions:MinDelayMs` | `2000` | — | Minimum polite delay between page requests |
| `Scraper:StlflixOptions:MaxDelayMs` | `5000` | — | Maximum polite delay between page requests |

## Project Structure

```
PrintingMagic.Scraper/
├── BrowserSessionService.cs    # Playwright session lifecycle, login, session validity
├── LoginService.cs             # STLFlix authentication with retry
├── Worker.cs                   # IHostedService — main polling loop
├── DropProcessingService.cs    # Coordinates drop discovery and harvest
├── Harvest/
│   └── CollectionHarvesterJob.cs   # Scrapes collection metadata per drop
├── Scrapers/
│   └── StlflixScraper.cs       # DOM navigation — drops listing + model pages
├── Events/                     # RabbitMQ event contracts
├── Data/                       # EF Core DbContext for local state
├── Configuration/              # Strongly-typed config classes
└── appsettings.json
```

## Dependencies

| Dependency | Purpose |
|------------|---------|
| `Microsoft.Playwright` | Headless Chromium browser automation |
| `RabbitMQ.Client` | Event publishing |
| `Npgsql.EntityFrameworkCore.PostgreSQL` | Local scraper state persistence |
| `PrintingMagic.Data.Contracts` | Shared DTOs (`CollectionImportRequest`, `ModelImportDto`) |
| `PrintingMagic.ServiceDefaults` | Aspire service defaults (OTEL, health checks) |

## Known Limitations

- The Playwright Chromium binary must be installed manually on first setup (`pwsh playwright.ps1 install chromium`). This is not automated.
- Production deployment is blocked by legal gate (story 3-04).
- Internal API URL resolution uses configuration fallback — Aspire service discovery integration is pending (story 2-04).
