# PrintingMagic Processor Service

**Feature ID:** newversion-processor-storage  
**Domain:** newversion / processor  
**Delivered:** 2026-04-11

---

## Overview

The Processor service is a standalone .NET 10 microservice that consumes `ScraperModelPublished` events from RabbitMQ and persists persisted product records to a Postgres database. It also exposes a minimal REST API for downstream consumers to query stored products.

The service was introduced as part of the `newversion-processor-storage` epic, which also migrated the Scraper service to publish structured events via MassTransit and extracted shared contract types into a `PrintingMagic.Contracts` library.

---

## How It Works

```
Scraper service
   │
   │  (publishes ScraperModelPublished via MassTransit / RabbitMQ)
   ▼
message-broker (RabbitMQ)
   │
   │  (consumes ProcessModelConsumer)
   ▼
PrintingMagic.Processor
   ├── ProcessModelConsumer
   │     ├── Idempotency: catches UniqueViolation on ScraperModelId
   │     └── Persists ProcessModel to Postgres via EF Core
   │
   └── GET /products/{id}
         └── Returns ProcessModel by ID (200) or 404
```

### Key Components

| Component | Location | Purpose |
|-----------|----------|---------|
| `ProcessModel` | `Models/ProcessModel.cs` | EF Core entity — stores scraped product data |
| `ProcessorDbContext` | `Data/ProcessorDbContext.cs` | EF Core context for Postgres, uses `HasIndex(x => x.ScraperModelId).IsUnique()` |
| `ProcessModelConsumer` | `Consumers/ProcessModelConsumer.cs` | MassTransit consumer — idempotent via `DbUpdateException` inner `PostgresException` check |
| `ProductsEndpoints` | `Endpoints/ProductsEndpoints.cs` | `GET /products/{id}` — minimal API |
| `InitialCreate` migration | `Migrations/` | Auto-runs on startup via `MigrateAsync()` in `Program.cs` |

---

## Configuration

All configuration is injected via Aspire connection string environment variables. No manual `appsettings.json` entries are required in production.

| Variable | Purpose | Set by |
|----------|---------|--------|
| `ConnectionStrings__processor-db` | Postgres connection string | Aspire AppHost `processorDb` resource + parameter |
| `ConnectionStrings__message-broker` | RabbitMQ connection string | Aspire AppHost `message-broker` resource |

In non-Aspire environments (e.g., standalone test runs), set these as standard environment variables or `appsettings.json` entries.

---

## Usage

### Running locally via Aspire

```bash
cd TargetProjects/newversion/apphost/PrintingMagic.Aspire
dotnet run apphost.cs
```

The Aspire dashboard will be available at `https://localhost:15888`. The processor service and its Postgres + RabbitMQ dependencies start automatically.

### REST API

**GET /products/{id}**

Returns a stored `ProcessModel` by its primary key.

- `200 OK` — product found, returns JSON body
- `404 Not Found` — no product with that ID

Example:
```
GET /products/42
```

---

## Known Limitations

- The `GET /products/{id}` endpoint queries by `ProcessModel.Id` (internal primary key), not by `ScraperModelId`. A future story should add a `GET /products/by-scraper-id/{scraperModelId}` endpoint if upstream consumers need to correlate by original scraper reference.
- Integration tests (`ProcessorStartupTests`) require Docker to run (Aspire starts Postgres + RabbitMQ as containers). These are excluded from the normal unit test run.
- No pagination is implemented on the products endpoint. Currently a single record fetch only.
