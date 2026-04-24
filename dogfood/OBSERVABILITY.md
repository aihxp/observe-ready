# Mirror Box observability wiring (dogfood for observe-ready v1.0.6)

**Target:** [aihxp/mirror-box v1.0.0](https://github.com/aihxp/mirror-box)
**Backend:** Honeycomb (free tier), `mirror-box` dataset
**Mode:** wire-from-plan (the service has tracing code; observability posture needs SLOs, dashboards, and alerts to close the loop)
**Version:** 1.0
**Last updated:** 2026-04-23
**Owner:** h (dogfood)

## What this is

observe-ready's first pass against Mirror Box. The service already emits OTel spans with six attributes per PRD R-02; the tracing pipe is live. What's missing is the observability layer on top: SLOs tied to PRD NFRs, a minimal dashboard, and alert wiring.

Purpose: close the observe-ready half of the Mirror Box executional gap from the suite VALIDATION report.

## Pre-flight state

- **Tracing:** live. OTel Node SDK + OTLP/gRPC exporter to Honeycomb. Six attributes per span (http.method, http.route, http.status_code, user.ip hashed, request.size_bytes, response.size_bytes).
- **Structured logs:** live. Fastify's pino-backed logger. `ip_hash` instead of raw IP; request/response pairs with reqId correlation; sub-5ms timing data.
- **Metrics:** none. No Prometheus exporter, no OTel metrics, no custom counters. Honeycomb can derive metrics from spans for simple cases, which suffices at Mirror Box's scale.
- **Dashboards:** none.
- **SLOs:** none defined.
- **Alerts:** Fly's built-in health check (30s interval on `/healthz`) is the only live alert.

## SLO design

observe-ready Step 4: SLOs from PRD NFRs.

Mirror Box's PRD NFR table (relevant rows):

| Dimension | PRD threshold |
|---|---|
| Performance | p95 end-to-end <200ms for 1KB POSTs |
| Scale | 50 req/s sustained on free tier |
| Availability | 99% monthly (non-SLA-backed) |

Two SLOs worth defining; one pending:

### SLO-1: Availability

- **Statement:** 99% of requests over a rolling 30-day window succeed (return non-5xx).
- **SLI:** `count(http.status_code < 500) / count(*)` over 30 days.
- **Target:** 0.99 (99%).
- **Error budget:** 1% = 72 minutes / month of downtime or error bursts.
- **Burn rate alerts:** two multiwindow burn rates (below).

### SLO-2: Latency

- **Statement:** 95% of `/echo` requests complete in under 200ms over a rolling 30-day window.
- **SLI:** `histogram_quantile(0.95, duration_ms{http.route="/echo"})` over 30 days.
- **Target:** 200ms at p95.
- **Error budget:** 5% of requests can exceed 200ms.
- **Burn rate alerts:** one fast-burn (below).

### SLO-3: Tracing freshness (not an SLO yet)

- **Statement (proposed):** 99% of requests produce a span ingested by Honeycomb within 60 seconds.
- **Status:** not measured. Would require a synthetic request + Honeycomb-query check; not worth the plumbing at Mirror Box's scale. Listed here so future operators know what's missing.

## Alert wiring

observe-ready Step 6: the minimum alert set that's load-bearing without creating noise.

### Alert 1: Availability fast-burn (SLO-1)

- **Condition:** `error_rate > 2%` over 5 minutes AND over 1 hour.
- **Rationale:** Google SRE multiwindow rule: both short and long windows must exceed 2x the error budget consumption rate to fire. Prevents single-minute blips from paging.
- **Route:** PagerDuty or Slack or (for Mirror Box's unattended dogfood posture) a GitHub Issue auto-opened via a Fly webhook. See "On-call posture" below.
- **Runbook link:** `docs/runbooks/availability-burn.md` in aihxp/mirror-box (not yet written; to be added as part of this dogfood).

### Alert 2: Availability slow-burn (SLO-1)

- **Condition:** `error_rate > 0.5%` over 6 hours AND over 24 hours.
- **Rationale:** slow-burn catches sustained degradation that's too subtle for fast-burn. Fires after ~24 hours of low-grade errors.
- **Route:** same as Alert 1, but lower severity (ticket, not page).

### Alert 3: Latency fast-burn (SLO-2)

- **Condition:** `p95(duration_ms{http.route="/echo"}) > 400ms` over 10 minutes.
- **Rationale:** 400ms is 2x the PRD p95 target (200ms). Sustained above-target means something structural has changed (Fly region outage, Honeycomb ingest blocking request path, memory pressure, etc.).
- **Route:** same as Alert 1.

### Alert 4: Fly health check (built-in)

- **Condition:** `/healthz` fails 3 consecutive 30-second intervals (90s unhealthy).
- **Rationale:** already wired by Fly. Routes to Fly's default notification channel.
- **Not maintained by observe-ready;** just noted for completeness.

### Alerts NOT wired (intentionally)

- **Span export failures.** Exporter is by-design non-blocking per PRD R-02 acceptance. Export failures log a warning; they don't degrade the service. Alerting on them would be noise.
- **Traffic-volume thresholds** (50+ req/s). Below the PRD scale ceiling; not load-bearing.
- **CPU/memory saturation.** Fly handles via auto-restart. No custom alert needed.
- **Dependency latency.** Mirror Box has no dependencies beyond Honeycomb; dependency latency doesn't affect request path.

## Dashboard (proposed)

observe-ready Step 5: dashboard-sprawl discipline says 7 charts above the fold, max. Mirror Box needs 4:

1. **Error rate** (time series, 1h / 24h / 7d). Primary availability view.
2. **Request latency p50 / p95 / p99** (time series, 1h / 24h). Primary latency view.
3. **Request volume by route** (stacked area, 1h / 24h). Traffic shape.
4. **Recent errors** (table, last 100 5xx responses with route, error detail, trace_id link).

Above the fold. No below-the-fold. If a fifth chart would be useful, it's a signal that a new SLO is emerging, not a dashboard-bloat invitation.

Honeycomb query URLs (example templates, operator fills in `<team>`):

- https://ui.honeycomb.io/<team>/datasets/mirror-box/result/COUNT_GROUP/http.route
- https://ui.honeycomb.io/<team>/datasets/mirror-box/result/HEATMAP/duration_ms?filter=http.route:/echo
- etc.

The dashboard lives in Honeycomb's UI; observe-ready doesn't own the visual config beyond declaring which four queries matter.

## Instrumentation gaps (if any)

Mirror Box's six required attributes cover the observable surface for its scope. Three attribute expansions would be useful but are not blocking:

1. **Body schema attribute.** The `/echo` handler could tag the span with the top-level keys of the request body (e.g., `echo.body_keys=["k","n"]`) to help with debugging. Opt-in; not PII-safe by default (top-level key names can reveal intent). Left out at v1.
2. **Server version.** Already in `package.json`, exposed in response bodies, but not tagged on the span. Would help version-slicing in the dashboard. One line to add; worth it in v1.1.
3. **Exporter health.** A low-cardinality attribute (`otel.exporter.healthy=true|false`) would let the dashboard show exporter health separately from request-handling health. Not adding yet.

None of these are Tier 1 blockers; all are observe-ready Tier 2+ polish.

## Incident response posture

Mirror Box is an unattended weekend-project target. Observability has to work under the constraint that nobody is on-call.

### On-call posture: no rotation

No paging rotation. Alerts route to:
- **Fly's default email** for health-check failures (built-in).
- **A private Slack channel or GitHub Issue** for SLO burn-rate alerts (operator chooses).

This is acceptable for a weekend-project dogfood target. It is NOT acceptable if Mirror Box is ever promoted to a teaching target that workshops depend on; a teaching-target promotion would require a real on-call rotation or a supervised SLO.

### Runbook stubs (not yet in aihxp/mirror-box)

Three runbooks should land in Mirror Box's `docs/runbooks/`:

1. **`availability-burn.md`:** "Service returning 5xx. Steps: 1. Check Fly status page. 2. Check `fly logs`. 3. Roll back if a recent deploy is implicated. 4. File a Honeycomb incident note with the trace IDs of failing requests."
2. **`latency-burn.md`:** "p95 above 400ms. Steps: 1. Check `fly status` for machine-level issues. 2. Check Honeycomb for a recent traffic spike. 3. Check for an outlier trace (long tail). 4. If sustained, resize the Fly VM via `fly scale vm shared-cpu-1x --memory 512`."
3. **`exporter-broken.md`:** "Spans stopped arriving in Honeycomb. Steps: 1. Check `HONEYCOMB_API_KEY` is still valid (rotate if needed). 2. Check `fly logs | grep otel` for exporter warnings. 3. Verify Honeycomb is up via their status page. Service continues serving; this is not a user-facing incident."

These runbooks live in aihxp/mirror-box, not in observe-ready. observe-ready declares the shape; aihxp/mirror-box implements them.

## Post-incident hardening (none yet)

No incidents yet. When one happens:

- Write a post-mortem in `docs/incidents/YYYY-MM-DD.md` in aihxp/mirror-box.
- Update the relevant runbook with lessons learned.
- If an SLO error budget is exhausted, freeze deploys until a root cause is found.

observe-ready's Step 10 (incident review cadence) would normally schedule weekly incident review. At Mirror Box's traffic levels, that's overkill; review after each incident is sufficient.

## Tier verdict for observability

observe-ready's 4-tier model:

- **Tier 1 (telemetry wired):** yes. OTel spans, structured logs, trace IDs visible in responses. Confirmed at Mirror Box v1.0.0 ship.
- **Tier 2 (SLOs + alerts):** SLOs defined in this document; alerts wired in Honeycomb pending operator config. **Gap:** alert routing target (Slack vs. GitHub Issue vs. email) depends on operator setup. Mirror Box ships without this wired; this document is the spec.
- **Tier 3 (dashboards + runbooks):** dashboard shape specified (4 charts); runbooks stubbed; neither is built. **Gap:** dashboard URLs and three runbook files to land in aihxp/mirror-box.
- **Tier 4 (post-incident hardening + on-call):** post-incident hardening is post-incident; no incidents yet. On-call is explicitly not maintained (unattended dogfood).

Mirror Box clears Tier 1 today. Tier 2 and Tier 3 need Mirror Box's operator to do 1-2 hours of Honeycomb configuration and write three runbook stubs.

## Handoff (back to the suite)

This dogfood doesn't trigger downstream re-runs. The four charts, three runbooks, and the alert wiring are Mirror Box's responsibility, not observe-ready's. observe-ready declares the shape.

If Mirror Box's operator lands the Tier 2 and Tier 3 items, they become the audit trail for a future Mirror Box v1.1.0 observability posture.

Ready-suite version table and SUITE.md are unchanged by this commit.
