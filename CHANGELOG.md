# Changelog

## v1.0.3 (2026-04-23)

Documentation-only patch. Reflects the arrival of `prd-ready` v1.0.0 (https://github.com/aihxp/prd-ready) as a live sibling in the ready-suite. This release completes the top of the planning tier: prd-ready defines WHAT we are building, upstream of architecture-ready (HOW), roadmap-ready (WHEN), and stack-ready (WITH WHAT TOOLS). No behavioral changes to the skill.

### Changed

- **`SUITE.md`** updated to list prd-ready at 1.0.0 alongside production-ready 2.5.6, repo-ready 1.6.2, stack-ready 1.1.5, deploy-ready 1.0.4, observe-ready 1.0.3, and launch-ready 1.0.1. Copy remains byte-identical across every live sibling.
- **SKILL.md frontmatter version** bumped to 1.0.3. No content change beyond the version tag.

## v1.0.2 (2026-04-23)

Documentation-only patch. SUITE.md updated to reflect the v1.0.0 release of sibling skill `launch-ready`, the shipping-tier skill that owns "tell the world the product exists." The shipping tier is now complete. No behavioral changes to observe-ready itself. The coupling with launch-ready: a launch generates a traffic spike; launch-ready reads `.observe-ready/SLOs.md` to surface any at-risk SLO in the launch runbook, and reads `.observe-ready/INDEPENDENCE.md` to know whether the status page is already out-of-band. observe-ready's existing artifact contract is unchanged. See [launch-ready](https://github.com/aihxp/launch-ready) for the sibling.

## v1.0.1 (2026-04-23)

Documentation-only patch. Reflects the arrival of repo-ready v1.6.0 as a live sibling in the ready-suite with its suite-membership retrofit (frontmatter interop fields, SUITE.md, Unicode cleanup). No behavioral changes to the skill.

### Changed

- **SUITE.md known-good versions table** updated: repo-ready now shows version 1.6.0 and its repo URL instead of "See its CHANGELOG."
- **SKILL.md frontmatter version** bumped to 1.0.1. No content change beyond the version tag.

## v1.0.0 (2026-04-22)

First stable release of observe-ready, the shipping-tier skill that owns "keep the app healthy once it is live" in the [ready-suite](SUITE.md). Ships with the full SKILL.md contract, eleven reference files, a ~7,500-word research report backing every guardrail, and full interop-standard frontmatter. Walked against a realistic solo-dev Fly.io scenario (Node API with login and signup journeys) before cut; rough edges surfaced during the walk are addressed below.

### The skill's three named failure modes

observe-ready introduces three terms the ecosystem did not already have a clean name for. Each maps to a specific class of real-world incident (citations in [`references/RESEARCH-2026-04.md`](references/RESEARCH-2026-04.md)).

- **Paper SLO.** An SLO with no error-budget policy, no alert wired to it, no review cadence, and no stakeholder who knows its number. The cousin of deploy-ready's "paper canary" across shipping-tier siblings: both name the "artifact present, mechanism absent" pattern that dominates AI-generated systems-engineering output. Refused by the skill.
- **Blind dashboard.** A dashboard above the fold with metrics that have no SLO behind them and nobody watched during the last incident. Distinct from generic "dashboard sprawl" because it names the artifact's binding, not its count.
- **Paper runbook.** A runbook written once, attached to an alert, and never executed. Commands reference renamed log fields; URLs 404. Distinct from "runbook drift" because the term names the artifact, not the process.

Related corollary: **unreachable runbook** (hosted on the infrastructure it documents). Facebook 2021 and Roblox 2021 are the citations.

### What ships

- **SKILL.md** with the ready-suite interop standard: eleven frontmatter fields populated, six required sections present. Twelve-step workflow (Step 0 to Step 11), four completion tiers (Instrumented, Promised, Traceable, Rehearsed) totaling 20 requirements, a grep-testable have-nots list, a session state template, explicit consume/produce contracts with sibling skills.
- **Eleven reference files** under `references/`. Load-on-demand table annotates each with the step or tier that loads it.
  - `observe-research.md`. Step 0 mode detection (A greenfield, B instrumented-but-unbound, C mature-with-sprawl, D incident-reactive, E cost-driven), paper-SLO detection procedure, observability-surface reachability question.
  - `logging-patterns.md`. Structured JSON format, required fields (timestamp, level, service, env, trace_id, span_id, event), level discipline, correlation ID propagation across HTTP/gRPC/async/cron, PII scrubbing at the OTel Collector with working attribute-processor config, sampling, aligned retention.
  - `metrics-taxonomy.md`. RED / USE / golden signals; per-topology catalogs for request/response, worker, batch, edge, data pipeline, database, cache, broker; cardinality discipline with per-backend pricing-model awareness; deadman-switch patterns.
  - `slo-design.md`. One SLI per journey rule, target selection against dependency ceiling, window choice, error-budget math, error-budget policy template with trigger/action/stakeholder/exit/escalation, multi-burn-rate math, low-traffic strategies (extended windows, synthetic, aggregation, uptime-based), SLO composition across dependency chains, quarterly review cadence.
  - `alert-patterns.md`. Three-tier severity ladder (PAGE / TICKET / LOG-ONLY), symptom-based pages vs. cause-based diagnostics, four-tier multi-burn-rate matrix with PromQL examples, runbook-URL-in-payload contract, alert-routing independence, deadman switches, correlation and suppression, quarterly pruning, the incident.io 2025 false-positive baseline citations.
  - `dashboards.md`. Above-the-fold rule (seven or fewer), four canonical charts (error budget, burn rate, SLI trend, top diagnostic), chart semantics (p99 not avg, rate as events/sec, error rate as ratio, deploy annotations), per-service-type catalog, sprawl budget of three dashboards per service, ownership metadata, independence check.
  - `tracing.md`. OpenTelemetry at graduated CNCF status (2024-2025), W3C Trace Context and tracestate and Baggage, propagation traps (async queues, long-lived gRPC streams, serverless cold starts, fan-out), head vs. tail sampling with buffer math, retention alignment to SLO window, span hygiene (low-cardinality names, semantic-conventions attributes, no PII), span-to-metric pipeline.
  - `error-tracking.md`. Sentry / Rollbar / Bugsnag / Honeycomb differences, release tagging via the deploy pipeline, grouping tuning on top-20 signatures, metrics-vs-errors complementary roles, PII scrubbing, frontend-specific notes (source-map upload, session replay, browser-console noise), regression detection, error-tracker independence.
  - `incident-response.md`. Runbook template with alert context, diagnose (with executable commands), act (decision tree), communicate (cadences), escalation, post-incident; runbook execution cadence (quarterly tabletop or real incident); out-of-band hosting; three-tier severity ladder (SEV-1, SEV-2, SEV-3); incident commander role; war-room protocol; status-page discipline; customer communication templates; on-call ergonomics.
  - `post-mortem.md`. Blameless format, causal analysis beyond Five Whys (contributing-factors tree, Swiss cheese model, learning-from-incidents), action-item tracking with owner/due/verification, the observability-gap action-item loop, alert pruning tied to post-mortem, public post-mortem discipline with examples, the "no incidents this quarter" note.
  - `vendor-landscape.md`. Datadog / Grafana Cloud / Honeycomb / New Relic / Splunk / Dynatrace / open-source OTel / Sentry / PagerDuty / incident.io / FireHydrant / Rootly / Blameless / Better Stack / Checkly / Cribl / Chronosphere / SigNoz / ServiceNow Cloud Observability / Groundcover. Pricing traps (Datadog custom-metric cardinality, New Relic CCU opacity), SLO support gaps, retention alignment, independence considerations per vendor, portability via OpenTelemetry.
- **Research report** (`references/RESEARCH-2026-04.md`, ~7,500 words). Named incidents: Slack 2021, Roblox 2021, Facebook 2021, AWS us-east-1 2021, Cloudflare 2019 and 2022, Datadog 2023, GitHub 2018, CrowdStrike 2024, AWS DynamoDB 2025, Checkly 2024. Tool landscape across Datadog, Grafana Cloud, Honeycomb, New Relic, Splunk, Dynatrace, Chronosphere, Cribl, Prometheus/Jaeger/OTel Collector, Sentry/Rollbar/Bugsnag, PagerDuty/incident.io/FireHydrant/Rootly/Blameless, Better Stack/Checkly, Lightstep/Groundcover, ClickHouse-based stacks. OpenTelemetry adoption state including 2025 stability notes, semantic-conventions status, propagation pitfalls, sampling tradeoffs with collector buffer math. Alert-fatigue literature from Rob Ewaschuk, the SRE workbook multi-burn-rate chapter, Charity Majors and Liz Fong-Jones on actionable alerts, incident.io and Runframe 2025-2026 data. Honeycomb Observability-2.0 framing, three-pillars debate, cardinality economics. SLO design canon (Hidalgo, Nobl9, SRE book and workbook), error-budget policy template. Naming-lane analysis concluding on paper SLO / blind dashboard / paper runbook / unreachable runbook with search-evidence. Frequency data: 85% false-positive alerts, 67% alert ignoring, 73% outage correlation, 70% log-spend never queried, DORA 2024 7.2% stability drop for AI-assisted delivery.
- **SUITE.md** at repo root listing observe-ready at 1.0.0 alongside production-ready 2.5.3, stack-ready 1.1.2, and deploy-ready 1.0.1 (sibling copies bumped as patches on this release).
- **README.md** with install paths for Claude Code, Codex, Cursor, Windsurf; the "what this skill prevents" incident-to-enforcement mapping across 14 incidents and findings; reference-library index; named-terms section.

### Refinements from the dogfood walk

A pre-release paper walk against a realistic solo-dev Fly.io scenario (Node API with login and signup journeys, 1K-5K daily actives) surfaced the following, addressed in v1.0.0:

- **Low-traffic SLO strategy documented inline.** Services with too few events for burn-rate math are common (most early apps). `slo-design.md` section 8 covers extended windows, synthetic traffic, service aggregation, and uptime-based SLI as named alternatives rather than forcing a request-ratio SLI onto a service that cannot support it.
- **Mode E (cost-driven) recognized as distinct and often co-occurring.** Teams in Mode C (sprawl) often arrive because they are also in Mode E (bill shock). The mode matrix in `observe-research.md` acknowledges the overlap; Step 3 cardinality and Step 7 sampling get mode-specific emphasis under Mode E.
- **Paper-SLO watchlist surfaces in STATE.md.** Agents mid-configuration often need to write a partial SLO (the policy is not yet agreed). Rather than block, the skill writes the SLO with `POLICY: TBD` and records it in the paper-SLO watchlist in STATE.md, so the gap is visible and prioritized.
- **Runbook-last-executed cadence split into soft (90 days) and hard (180 days) floors.** A quarterly tabletop cadence is the target; 180 days is the point past which the runbook is treated as paper regardless of justification.
- **Three-tier severity ladder instead of five.** PagerDuty and incident.io default UIs offer P1/P2/P3/P4/P5. Three tiers (PAGE / TICKET / LOG-ONLY for alerts; SEV-1 / SEV-2 / SEV-3 for incidents) force behavioral clarity and reduce ambiguity under stress.
- **Rollout-and-observability coupling as a first-class rule.** The deploy-ready paper-canary rule and the observe-ready "uniform rollouts defeat symptom-based alerts" rule are the same coupling from two sides. Surfaced explicitly in the SKILL.md synthesis and in `alert-patterns.md` section 9 correlation discipline.

### Compatibility

- Claude Code (primary)
- Codex
- Cursor
- Windsurf (manual SKILL.md upload)
- Any agent with skill loading

### Suite siblings at release

- production-ready 2.5.3 (patch bumped for SUITE.md table update)
- stack-ready 1.1.2 (patch bumped for SUITE.md table update)
- deploy-ready 1.0.1 (patch bumped for SUITE.md table update)
- repo-ready (live, see its own CHANGELOG)

Planned siblings (not yet released): prd-ready, architecture-ready, roadmap-ready, launch-ready.
