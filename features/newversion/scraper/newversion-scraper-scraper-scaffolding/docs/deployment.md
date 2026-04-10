# Deployment: PrintingMagic.Scraper

**Feature ID:** newversion-scraper-scraper-scaffolding

## Local Development

### Prerequisites

1. **.NET 10 SDK**
2. **Docker** — for PostgreSQL and RabbitMQ containers
3. **PowerShell** — for Playwright browser install (`pwsh`)
4. **Aspire CLI** — `aspire run`

### Environment Setup (once per machine)

Add to `~/.bashrc` (or shell profile):

```bash
# Aspire dev-certs trust
export SSL_CERT_DIR="$SSL_CERT_DIR:/usr/lib/ssl/certs:/home/$USER/.aspnet/dev-certs/trust"

# Docker Engine container tunnel
export ASPIRE_ENABLE_CONTAINER_TUNNEL=true
```

Run `aspire doctor` to verify — all 3 checks should pass.

### Playwright Browser Install

After building the scraper project, install the Chromium headless shell:

```bash
cd TargetProjects/newversion/scraper/PrintingMagic.Scraper
dotnet build
pwsh bin/Debug/net10.0/playwright.ps1 install chromium
```

### Credentials

Supply credentials via Aspire config parameters. Either:

**Option A — `aspire.config.json`** (local-only, do not commit real values):
```json
{
  "parameters": {
    "stlflix-email": "your@email.com",
    "stlflix-password": "your-password"
  }
}
```

**Option B — environment variables** (preferred for CI/shared machines):
```bash
export Parameters__stlflix-email="your@email.com"
export Parameters__stlflix-password="your-password"
```

### Running Locally

```bash
cd /path/to/workspace
aspire run --apphost TargetProjects/newversion/apphost/PrintingMagic.Aspire/apphost.cs
```

The Aspire dashboard URL and resource endpoints are printed to stdout on startup.

## Aspire Resources

| Resource | Type | Lifetime |
|----------|------|---------|
| `postgres` | PostgreSQL 17 container | Persistent |
| `scraper-db` | Database on `postgres` | — |
| `message-broker` | RabbitMQ 4.2-management container | Persistent |
| `scraper` | .NET project | — |

## Environment Variables (injected by Aspire)

| Variable | Source |
|----------|--------|
| `Stlflix__Email` | Aspire parameter `stlflix-email` |
| `Stlflix__Password` | Aspire parameter `stlflix-password` |
| `ConnectionStrings__scraper-db` | Aspire PostgreSQL reference |
| `ConnectionStrings__message-broker` | Aspire RabbitMQ reference |

## Rollback Notes

- All containers use `ContainerLifetime.Persistent` — stop them manually with `aspire stop` or `docker stop` if needed.
- No database migrations are run automatically at startup; run EF Core migrations manually if schema changes are needed.

## Production Deployment

> **Blocked** — Legal review of STLFlix Terms of Service must be completed before production deployment (story 3-04). Do not deploy to production until this gate is cleared.
