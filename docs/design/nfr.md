# NFR: Agentic AI Migration Assistant

Companion to [tdd.md](tdd.md). Targets are the **design assumptions in the source document, to be verified**; none has been measured. Items the source does not state are `TBD`.

## Performance

| Operation | Target Latency | Target Throughput |
| --- | --- | --- |
| Code model build, 500k LOC, cold | < 10 min | — |
| Code model build, incremental | < 2 min | — |
| Boundary proposal after code model ready | < 5 min (p95) | — |
| Concurrent analyses | — | 50 per region steady, burst 200 |
| API request latency (non-run endpoints) | TBD | TBD |
| Verifier / spec / plan / scaffold stages | TBD | TBD |

Repo size supported: up to 1M lines of Java, 5k classes.

## Scalability

- API stateless, 2–20 replicas; PgBouncer to protect Postgres connections.
- Analyzer workers 0–50 (KEDA on queue depth), watch memory on very large graphs. LLM workers 0–100, watch provider tokens-per-minute limits. Compile sandbox scales on queue depth.
- Postgres vertical + read replica; `audit_event` partitioned monthly (write throughput watch). Redis vertical then clustered.
- Large repos: incremental analysis (diff vs last SHA), sharded parsing by top-level package, Louvain pre-clustering with LLM refining clusters, summaries rather than raw code in the Proposer's context.
- Capacity planning figure: 200 runs/day on a 500k-line repo ≈ 4–6M tokens per cold run (~1,500 modules × ~2k tokens plus proposal/verify rounds); incremental runs should cut 70–90%. **Planning numbers, not measurements.**

## Availability

- API 99.9% monthly. Single region, zone-redundant in v1.
- Degraded modes: LLM fallback model (flagged); Redis loss (rebuild from Postgres/Blob); worker loss (resume from checkpoint).
- Error budget policy and maintenance windows: TBD.

## Durability & Recovery

- Postgres geo-backup and Blob geo-replication: RPO 1 h, RTO 4 h. In-flight runs restart from checkpoints.
- Retention: snapshots 30 days; artifacts immutable 1 year; code-model tables 12 months; checkpoints 90 days; audit, approvals and feedback 7 years; DLQ 14 days.
- Idempotent stages on `(run_id, stage)`; Blob reference written to Postgres only after upload succeeds.

## Reproducibility & Quality

- Same SHA + same prompt/model versions → identical code model; proposal agreement ≥ 90% across reruns (temperature 0, pinned models).
- 100% of cited evidence ids exist; ≥ 90% of sampled claims supported.
- Other quality targets (boundary F1/MoJoFM, planted-coupling recall, scaffold compile rate, reviewer edit rate): thresholds TBD, set after baseline measurement. CI blocks deploys on a drop beyond a set threshold (value TBD).

## Security & Compliance

- Source never leaves approved regions (which regions/providers: TBD per bank policy).
- Secrets and PII masked (gitleaks, Presidio) before every LLM call; providers with no-training and residency terms.
- Encryption at rest for snapshots; TLS in transit (specifics TBD).
- Tenant isolation via RLS, tenant-prefixed Blob and Redis keys; verified by zero-row tests across API, memory retrieval and direct SQL.
- Tamper-evident audit: hash-chained, INSERT-only app role.
- Model risk: treated as a model under SR 11-7 — inventory entry per LLM and prompt version, documented intended use (advisory only), pre-release validation on ground truth, ongoing drift monitoring, reproducibility of any past run.
- Four-eyes for PRs on protected repos; no agent can merge or alter repo settings.

## Resource Constraints

- Hard per-run token budget (Redis check before every LLM call) and per-tenant monthly caps; run stops and requests review at budget. Dollar values: TBD, configurable per tenant.
- Cost per run is a tracked metric; no target set (TBD).

## Monitoring & Alerting Expectations

- One `run_id` across API, workers, LangSmith traces and audit rows. OpenTelemetry → Azure Monitor; structured JSON logs with `tenant_id`, `run_id`, `stage`, `prompt_version`; no source code or secrets in logs.
- Dashboards: throughput and stage durations; queue depth; LLM latency, error and fallback rates; tokens and cost per run/tenant; approval and edit rates.
- Alerts: DLQ message (after 5 attempts), circuit breaker open, fallback rate, budget exhaustion, audit chain-check failure. Thresholds TBD.

## Verification

Each NFR is checked per the claims table in the source design (load test with k6 + queue injection at 200 concurrent runs, kill-worker / flush-Redis / throttle-LLM failure tests, rerun-5× reproducibility, tenant-isolation and audit-tamper tests, low-budget stop test). Pass conditions are those in the source's "Verification plan"; scheduling: TBD.

## Open Questions

- API latency and per-stage targets beyond those listed — TBD
- Quality metric thresholds and CI gate drop tolerance — TBD, after baseline
- Approved LLM providers/regions — TBD, pending bank data policy
- Cost-per-run ceiling in dollars — TBD
- Multi-region / DR beyond v1 — out of scope for v1, TBD
