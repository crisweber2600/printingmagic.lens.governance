# Deployment Guide: PrintingMagic Processor Service

**Feature ID:** newversion-processor-storage  
**Service:** `PrintingMagic.Processor`

---

## Prerequisites

| Requirement | Version |
|-------------|---------|
| .NET SDK | 10.x |
| Postgres | 17+ (via Docker or managed) |
| RabbitMQ | 4.x (via Docker or managed) |
| Aspire SDK | 13.2.2 (for Aspire-managed deployments) |

---

## Environment Variables

The processor expects two connection strings in the standard .NET format (`ConnectionStrings__<name>`):

| Variable | Example | Notes |
|----------|---------|-------|
| `ConnectionStrings__processor-db` | `Host=localhost;Port=5432;Database=processordb;Username=postgres;Password=secret` | Postgres database for product storage |
| `ConnectionStrings__message-broker` | `amqp://guest:guest@localhost:5672` | RabbitMQ broker |

---

## Running via Aspire (recommended)

Aspire provisions and wires all dependencies automatically.

```bash
cd TargetProjects/newversion/apphost/PrintingMagic.Aspire
dotnet run apphost.cs
```

The AppHost will:
1. Start a Postgres 17 container (`processordb`)
2. Start a RabbitMQ 4 container with management UI (`message-broker`)
3. Start the Scraper service
4. Start the Processor service with connection strings injected via env vars
5. Run EF Core migrations automatically on processor startup

Access the Aspire dashboard at `https://localhost:15888`.

### Aspire parameters

The AppHost reads two parameters for database credentials (defined in `aspire.config.json`):

| Parameter | Used for |
|-----------|---------|
| `processorDbUser` | Postgres username passed to the processor |
| `processorDbPassword` | Postgres password passed to the processor |

In local dev, override these in `aspire.config.json`. In CI, set them via environment variables on the AppHost process.

---

## Running standalone (without Aspire)

1. Start Postgres and RabbitMQ manually (or via Docker Compose).
2. Set environment variables (or `appsettings.json`):
   ```json
   {
     "ConnectionStrings": {
       "processor-db": "Host=localhost;Port=5432;Database=processordb;Username=..;Password=..",
       "message-broker": "amqp://guest:guest@localhost:5672"
     }
   }
   ```
3. Run the processor:
   ```bash
   cd TargetProjects/newversion/processor
   dotnet run --project PrintingMagic.Processor
   ```

EF Core migrations run automatically on startup via `await dbContext.Database.MigrateAsync()` in `Program.cs`.

---

## Database Migrations

Migrations are applied automatically on startup. To apply them manually or generate a new migration:

```bash
export DOTNET_ROOT=~/.dotnet
export PATH=$PATH:$DOTNET_ROOT:$DOTNET_ROOT/tools

# Apply latest migration
dotnet ef database update \
  --project PrintingMagic.Processor \
  --connection "Host=localhost;Port=5432;Database=processordb;Username=...;Password=..."

# Add a new migration
dotnet ef migrations add <MigrationName> \
  --project PrintingMagic.Processor
```

---

## Running Tests

### Unit tests (no Docker required)

```bash
cd TargetProjects/newversion/processor
dotnet test PrintingMagic.Processor.Tests
```

Expected: **4/4 pass** — consumer idempotency, consumer happy path, endpoint 200, endpoint 404.

### Integration tests (Docker required)

Integration tests use `DistributedApplicationTestingBuilder` to spin up the full Aspire stack. They require a working Docker daemon.

```bash
dotnet test PrintingMagic.Processor.Tests.Integration
```

Expected: **3 tests** — scraper health running, processor health running, message-broker running.

---

## Health Checks

The processor exposes standard .NET health check endpoints via `MapDefaultEndpoints()` (from `PrintingMagic.ServiceDefaults`):

- `GET /health` — liveness
- `GET /alive` — readiness

---

## Logging

Structured logging via `OpenTelemetry` is configured in `PrintingMagic.ServiceDefaults`. In Aspire deployments, traces and logs stream to the Aspire dashboard. For standalone deployments, configure an OTLP exporter via:

```
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```
