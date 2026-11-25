# Deployment and Operations Runbook

**Status:** Production design baseline; provider-specific commands are selected during implementation.

## 1. Deployment topology

Deploy the dashboard/API, widget static assets, background workers, PostgreSQL, private object storage, secret manager/KMS and durable queue as managed cloud services where feasible. Separate public ingress from private data services. Do not expose PostgreSQL, Redis/queue brokers, worker control ports or secret manager endpoints to the public internet.

A modular monolith is the initial application architecture. API and worker processes may share packages but have separate runtime identities, scaling policies and network permissions.

## 2. Environments

| Environment | Purpose | Data |
|---|---|---|
| Local | Developer workflow | Synthetic/local data |
| CI | Deterministic validation | Ephemeral test data and fake providers |
| Preview/staging | Integration and release verification | Synthetic or approved anonymized data |
| Production | Customer workloads | Tenant data under production controls |

Never copy production secrets into lower environments. Production access is least-privilege and audited. Environment configuration is versioned as schema, while secret values live only in secret management.

## 3. Release process

1. Pull request passes formatting, lint, type checks, unit and integration tests, dependency/security scans and build.
2. Migration compatibility reviewed (expand/contract).
3. Deploy to staging/preview.
4. Run smoke, API contract, widget, identity, tenant-isolation and provider adapter tests.
5. Confirm backup freshness, queue health and rollback plan.
6. Deploy immutable artifact using gradual/canary rollout where supported.
7. Run production smoke checks and monitor error/latency/budget indicators.
8. Promote or roll back; record release and migration status.

Database migrations must be backward-compatible with currently running application versions. Destructive schema changes require a later release after old code is removed.

## 4. Health and readiness

- Liveness: process is alive; no dependency calls.
- Readiness: required configuration loaded and critical dependencies available.
- Dependency health: database, queue, object storage, KMS and provider adapter health.
- Worker readiness includes queue subscription and graceful shutdown capability.

Do not make liveness fail because an optional model provider is down. Readiness should reflect dependencies required for that service's function.

## 5. Backups and disaster recovery

Define and test:
- PostgreSQL point-in-time recovery and scheduled backups.
- Object storage versioning/replication where appropriate.
- Secret manager recovery and key-rotation procedures.
- Queue durability and replay policy.
- Infrastructure/configuration as code backup.
- Restore ordering and integrity verification.

Set explicit RPO/RTO targets before production launch based on customer commitments. Run restore drills at least quarterly and after material backup architecture changes. A backup is not proven until restored and validated.

## 6. Monitoring and alerting

Monitor availability, API latency/error rate, worker queue age/depth, job failure/retry counts, database saturation/connection pool, storage errors, KMS failures, provider latency/error/rate limits, token usage, cost, budget exhaustion, integration timeouts, ingestion backlog and security events.

Alerts need owner, severity, runbook, actionable threshold and suppression policy. Avoid high-cardinality labels and never include prompts, credentials, PII or raw tool payloads in metrics.

## 7. Incident response

Severity levels:
- SEV-1: tenant isolation breach, credential exposure, broad outage, destructive data loss.
- SEV-2: major customer-facing feature outage, widespread provider/integration failure.
- SEV-3: limited degradation or isolated tenant issue.
- SEV-4: minor defect or non-urgent operational issue.

Response sequence: detect → triage → contain → communicate → eradicate → recover → validate → post-incident review. Security incidents preserve evidence, rotate affected secrets, revoke sessions/tokens, assess tenant impact and meet applicable notification obligations. Postmortems record timeline, root cause, impact, contributing factors and tracked corrective actions.

## 8. Scaling and capacity

Scale stateless API and worker pools independently. Bound per-workspace concurrency and queue fairness to prevent noisy-neighbor behavior. Use database connection pooling, indexes, partitioning only when measured, and caching only for data with explicit tenant-aware keys and invalidation. Apply backpressure when queue or provider capacity is exhausted.

## 9. Cost controls

Track infrastructure and model cost by workspace with tenant-safe aggregation. Enforce platform and workspace budgets, queue retention, storage quotas, request caps and concurrency limits. Free provider tiers are optional development conveniences, not production SLOs. Do not promise zero operating cost.

## 10. Operational acceptance

- Restore drill demonstrates database and object recovery.
- Rollback is tested for application and migration compatibility.
- Alerts route to an owner and link to a runbook.
- Worker shutdown drains or safely requeues work.
- Tenant usage and cost are attributable without exposing content.
- Production secrets are isolated from CI, preview and developer machines.


## 11. Production policy alignment

Use [Production Policies and Service-Level Specification](PRODUCTION_POLICIES_AND_SLOS.md) as the authoritative policy decision register. In particular:

- Do not publish uptime, support response, RPO or RTO commitments until approved and measured.
- Configure per-data-class retention and make deletion jobs/tombstone replay part of restore procedures.
- Enforce upload, session, request, concurrency and budget limits in shared server-side infrastructure.
- Keep provider allowlists and regional-processing restrictions centrally configurable.
- Keep paid overages disabled until a price book, billing lifecycle and explicit customer consent/contract are approved.
- Store policy version and effective configuration with operational/audit evidence, but never log secret values or raw sensitive content.

Production readiness evidence must include the approved policy matrix, recovery drill report, provider review records, monitoring ownership, and tested emergency kill switches.
