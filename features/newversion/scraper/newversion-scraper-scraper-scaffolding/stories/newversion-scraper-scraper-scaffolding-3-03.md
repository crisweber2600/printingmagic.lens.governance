# Story: newversion-scraper-scraper-scaffolding-3-03

## Context

**Feature:** newversion-scraper-scraper-scaffolding
**Sprint:** 3
**Priority:** medium
**Estimate:** S

## User Story

As an operator, I want `PrintingMagic.Scraper` to emit OpenTelemetry traces through Aspire so that I can observe scrape cycles, diagnose failures, and confirm the service is active in the Aspire dashboard.

## Acceptance Criteria

- [ ] `ActivitySource` named `"PrintingMagic.Scraper"` is registered with the OTEL SDK in `Program.cs` via `services.AddOpenTelemetry().WithTracing(t => t.AddSource("PrintingMagic.Scraper"))`
- [ ] Aspire OTEL endpoint (`OTEL_EXPORTER_OTLP_ENDPOINT`) is wired automatically by Aspire registration (no manual config needed)
- [ ] After running one scrape cycle (or stubbing `Worker.ExecuteAsync` to start a single iteration), the Aspire dashboard shows at least one trace span with source `PrintingMagic.Scraper` (M1 from adversarial review)
- [ ] Structured logs from `ILogger<T>` implementations are visible in the Aspire dashboard console log view for `scraper`
- [ ] Smoke test: AppHost starts, scraper resource reaches `Running`, at least one log line from the scraper appears in the dashboard within 30 seconds

## Technical Notes

- Reference OTEL pattern: `TargetProjects/oldversion/printingmagic/PrintingMagic.Worker/Worker.cs` — the `ActivitySource` declarations on each service class are already in place from sprint 2 stories; this story just wires the SDK registration
- Aspire `ServiceDefaults` extension (if present in the new solution) may already include `AddOpenTelemetry` — check `PrintingMagic.ServiceDefaults` before writing new registration code
- If `PrintingMagic.ServiceDefaults` exists in the new solution: use `builder.AddServiceDefaults()` in `Program.cs` instead of a manual OTEL setup

## Dependencies

- newversion-scraper-scraper-scaffolding-3-01 (Aspire AppHost registration must be complete)
- newversion-scraper-scraper-scaffolding-3-02 (service must reach Running state before traces can appear)
