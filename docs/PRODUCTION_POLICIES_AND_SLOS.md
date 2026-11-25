# Production Policies and Service-Level Specification

**Status:** Draft — policy decisions require owner approval before public production commitments  
**Last reviewed:** 2026-11-26  
**Applies to:** Platform operations, workspace owners, end users, integrations, model providers, and hosted deployments

This document converts unresolved operational and commercial questions into explicit requirements. It distinguishes:
- **Requirement:** behavior the implementation must support.
- **Proposal:** a starting value for owner review; not an approved promise.
- **TBD / decision required:** must be approved and recorded before the affected capability is marketed or enabled in production.

No proposal in this document is a contractual SLA, legal conclusion, pricing commitment, or provider guarantee.

## 1. Policy governance

### 1.1 Authority and change control

- Platform owner is the policy authority for platform-wide retention, platform abuse controls, service availability statements, infrastructure recovery objectives, pricing, and provider allowlists.
- Workspace owners may configure workspace-level limits only within platform-enforced ceilings.
- Workspace owners cannot weaken platform security, legal holds, abuse controls, deletion safeguards, or provider restrictions.
- Policy changes must be versioned, dated, attributable, reviewed for migration impact, and recorded in the audit log.
- Runtime decisions must use a versioned effective policy snapshot. A request must not observe a partially updated combination of limits.
- Customer-facing terms, privacy notices, DPA, and published SLA must be reviewed independently before launch. This specification does not replace them.

### 1.2 Required policy model

Implement versioned policy records with:
- policy key and scope (platform, workspace, and where justified plan);
- typed value and validation schema;
- effective-from and optional effective-until timestamps;
- version, status (draft, active, retired);
- actor, approval metadata, and change reason;
- source/precedence and immutable audit history.

Effective precedence: mandatory platform safety/security ceiling → platform plan entitlement → workspace override where permitted → request/session-specific restriction. A workspace override can tighten but never exceed a platform maximum.

## 2. Conversation, event, and audit retention

### 2.1 Data classes

Retention must be configured independently for at least:
1. Conversation messages and run transcripts.
2. Durable run events and SSE replay records.
3. Tool invocation inputs/outputs and approval records.
4. Attachments and extracted text.
5. Knowledge source files, chunks, embeddings, and derived indexes.
6. Operational logs, traces, and metrics.
7. Security/audit events.
8. Billing, usage, and idempotency records.
9. Backups and disaster-recovery copies.

Do not apply one generic retention period to every data class.

### 2.2 Required behavior

- Retention is measured from a documented lifecycle timestamp (for example, conversation last activity, event creation, or object deletion request); each class must define its clock.
- Expired data is deleted or irreversibly anonymized by a durable, retryable retention worker.
- Deletion must cover primary rows, object storage, vector indexes, search indexes, caches, derived artifacts, and provider-side data where deletion is supported and contractually available.
- Cascading deletion must preserve minimal tombstones needed to prevent stale backup replay or accidental resurrection.
- Legal hold, abuse investigation, or billing obligations may pause deletion only when authorized, scoped, documented, and reviewed. Holds must not become indefinite by default.
- A user-facing deletion request must have a trackable state and completion semantics. Distinguish request accepted, deletion in progress, primary-store deletion complete, and downstream/backup expiry.
- Backups are not individually edited for ordinary deletion. Deleted records must remain suppressed by tombstones when backups are restored; backup expiry completes removal from retained backup media.
- Retention jobs must be idempotent, observable, rate-limited, and safe to resume after failure.
- Never put secrets, raw provider credentials, or unnecessary message content into audit records.

### 2.3 Proposed defaults for owner review

These are conservative planning values, not approved commitments:

| Data class | Proposed default | Decision |
|---|---:|---|
| Conversation content | 90 days after last activity | TBD |
| Run/SSE replay events | 7 days | TBD |
| Tool inputs/outputs | 30 days, with sensitive payload minimization | TBD |
| Attachments | Follow owning conversation; orphan cleanup within 24 hours | TBD |
| Knowledge source and derived index | Until source deletion or workspace deletion | Confirm |
| Operational logs/traces | 30 days, content-minimized | TBD |
| Security/audit events | 365 days | Legal/security review |
| Usage/billing ledger | 7 years only if required by applicable accounting/tax law; otherwise define shorter period | Jurisdiction review |
| Idempotency records | At least the documented client retry window; proposal 24 hours | Confirm |
| Backups | See section 6 | TBD |

Do not ship these values as defaults until the owner has approved them and jurisdictional review is complete. The configuration system must support different values by environment and data class.

### 2.4 Deletion acceptance criteria

- A deletion request creates a durable deletion job and tombstone.
- Repeating the request is idempotent.
- Worker retries cannot restore deleted data.
- Restore procedures replay tombstones before making restored data accessible.
- Completion status and any backup-expiry caveat are visible to authorized workspace administrators.
- Tests cover workspace deletion, conversation deletion, knowledge deletion, partial provider failure, and backup restore replay.

## 3. Upload, attachment, and widget-session policies

### 3.1 Upload controls

All upload limits are server-enforced. Client-side validation is only a usability feature.

Required configurable limits:
- maximum bytes per file;
- maximum aggregate bytes per message;
- maximum aggregate bytes per workspace over a rolling period;
- maximum files per message;
- allowed MIME types and file extensions;
- maximum extracted text, pages, archive expansion ratio, and processing time;
- maximum concurrent ingestion jobs per workspace;
- quarantine and malware-scanning behavior;
- retention and deletion behavior.

Requirements:
- Treat file extension and client MIME type as untrusted; inspect content and reject mismatches.
- Reject executable content and unsupported archive formats by default.
- Protect against decompression bombs, parser exploits, path traversal, oversized page counts, and malicious embedded content.
- Store uploads in private object storage with short-lived, scoped download URLs.
- Do not expose object storage credentials or permanent public URLs.
- Apply malware scanning/quarantine before downstream parsing or retrieval.
- Enforce tenant ownership and authorization on every attachment read, download, parse, and delete.
- Never treat uploaded content as trusted instructions. Apply the prompt-injection and data-exfiltration controls in the security threat model.
- Keep original file, normalized metadata, extracted text, and derived index lifecycle linked to one source/attachment record.

### 3.2 Proposed initial limits

For MVP load testing only; owner approval required before enforcement:

| Control | Proposed starting value | Decision |
|---|---:|---|
| Maximum single file | 10 MiB | TBD |
| Maximum files per message | 5 | TBD |
| Maximum aggregate per message | 25 MiB | TBD |
| Maximum extracted text | 5 MiB UTF-8 equivalent | TBD |
| Supported types | PDF, TXT, Markdown, DOCX initially | Confirm |
| Archive uploads | Disabled initially | Confirm |
| Per-workspace ingestion concurrency | 2 jobs | TBD |

Limits must be plan-aware only after a pricing/entitlement model is approved. A limit increase must not bypass malware scanning or parser safety limits.

### 3.3 Widget session lifecycle

- Widget session tokens are short-lived, audience-bound, workspace-bound, origin-bound, and scoped to the widget capabilities granted at issuance.
- Session expiry and refresh are distinct: expired tokens are rejected; a permitted refresh flow issues a new token after re-validating origin and identity.
- Tokens must not be stored in URLs, browser local storage, analytics, referrers, or logs. Prefer in-memory handling and secure transport.
- Revocation must invalidate a session immediately or within a documented bounded propagation interval.
- A conversation may outlive a widget session; session expiry must not silently grant continued access. A fresh session must re-authorize access to an existing conversation.
- Anonymous/guest sessions must have stricter abuse controls and must not inherit authenticated-user privileges.
- Cross-origin messages must validate exact allowed origin, message schema, source window, nonce, and replay protection.
- Session creation and refresh must be rate-limited.

**Proposed session defaults:** 15-minute access-token lifetime and 30-minute idle timeout; hard maximum session age 8 hours. These are proposals only. Decide refresh semantics and revocation propagation before release.

## 4. API rate limits, concurrency, and usage budgets

### 4.1 Enforcement dimensions

Rate limits must be independently configurable by:
- IP/network reputation for anonymous entry points;
- workspace;
- authenticated customer identity;
- widget session;
- API token/client;
- endpoint class;
- provider/model;
- tool/integration;
- upload and ingestion workload.

Use token-bucket or sliding-window algorithms with atomic shared state across application instances. Do not rely on process-local counters in production.

### 4.2 Required behavior

- Enforce request rate, concurrent in-flight runs, token/input size, output token ceiling, tool-call count, and expensive workload limits.
- Reserve budget before expensive model/provider work. Reconcile reservation against actual usage after completion, failure, cancellation, or timeout.
- Reservations must expire safely and be idempotent; a failed request must not permanently consume budget.
- Budget accounting must distinguish estimated, reserved, actual, and adjusted usage.
- A single request cannot exceed the workspace's effective per-run ceiling.
- Anonymous traffic receives conservative platform defaults and abuse controls.
- Return 429 with a stable machine-readable error code and Retry-After where meaningful. Do not disclose another tenant's quota or usage.
- Provider rate limits and platform limits are separate. Provider throttling must not be represented as a platform quota exhaustion.
- Limits and usage must be visible to authorized workspace owners. Do not expose internal provider credentials or other workspaces' utilization.
- Provide emergency platform kill switches for model calls, tools, uploads, and public widget traffic.

### 4.3 Proposed MVP limits

Illustrative initial test profile only; not production entitlements:

| Dimension | Proposed test profile | Decision |
|---|---:|---|
| Anonymous chat starts | 10/minute/IP | TBD |
| Authenticated chat starts | 30/minute/user | TBD |
| Workspace chat starts | 120/minute | TBD |
| Concurrent runs | 5/workspace | TBD |
| Maximum input | 32k tokens/request | Model capability + plan TBD |
| Maximum output | 2k tokens/request | Model capability + plan TBD |
| Tool calls | 10/run | Risk policy may lower |
| Knowledge ingestion | 2 concurrent jobs/workspace | TBD |

These values must be validated by load testing, provider constraints, and abuse analysis. Do not hardcode them in application logic.

### 4.4 Budget exhaustion and BYOK

- BYOK credentials are used only for the explicitly configured provider/model binding.
- If a BYOK request fails, report a sanitized provider-specific failure to the workspace owner and follow the strict no-silent-fallback rule in the Model Gateway Specification.
- Platform-managed credits, if introduced, are a separate billing source and must not silently pay for a BYOK failure.
- When a hard budget is exhausted, reject new billable work before provider invocation. In-flight work follows a documented reservation policy.
- Soft thresholds may notify owners but cannot create unbounded spend.
- Admin overrides require reason, actor, scope, expiry, and audit event.

## 5. Service availability and support commitments

### 5.1 No public SLA by implication

Until the owner approves a formal SLA and has production measurements, the service is **best effort / preview**. Do not publish uptime percentages, support response promises, or service credits based on this draft.

### 5.2 Operational requirements

- Define service health separately for dashboard/API, widget/chat, model gateway, integration/tool broker, knowledge ingestion, and administration.
- Publish a status page only when monitoring and incident communications are operational.
- Monitor request success, latency percentiles, stream interruption, provider errors, queue age, ingestion failures, database saturation, and security events.
- Define incident severity, on-call ownership, escalation, customer communications, and post-incident review.
- A provider outage is a dependency incident, not necessarily a platform outage; status reporting must identify affected capabilities without exposing secrets.
- Deployments must support rollback and schema compatibility.
- Rate limits and circuit breakers must protect dependencies during degraded operation.

### 5.3 Proposed internal SLO candidates

For a future paid production tier, validate these with measured baseline and capacity tests before adoption:
- API availability: 99.5% monthly.
- Chat request acceptance latency: p95 under 1 second, excluding provider generation.
- First model token latency: measured separately by provider/model; no universal guarantee.
- Control-plane API latency: p95 under 500 ms for non-streaming reads under normal load.
- Queue processing: 95% of normal-priority ingestion jobs begin within 5 minutes.

These are candidate engineering objectives, not current commitments. Define exclusions, measurement windows, maintenance treatment, and service-credit rules in a separate approved SLA.

## 6. Backup, disaster recovery, RPO, and RTO

### 6.1 Required backup design

- Automated encrypted PostgreSQL backups and point-in-time recovery where supported.
- Separate backup access controls and credentials from application runtime credentials.
- Private, encrypted object-storage versioning/replication or equivalent recovery strategy for customer files.
- Documented backup scope, exclusions, retention, encryption, key recovery, and deletion/tombstone handling.
- Backups must be isolated from ordinary application write permissions and protected against accidental or malicious deletion.
- Restore must replay deletion tombstones and verify tenant boundaries before service is reopened.
- Backup success is not proven until restore tests succeed.
- Test full and partial restore procedures on a defined cadence and retain evidence.
- Document recovery dependencies: KMS/secret recovery, DNS, identity provider, provider credentials, queues, object storage, database, and application deployment.

### 6.2 Proposed recovery targets

Initial planning proposals; owner and infrastructure-provider approval required:

| Service/data | Proposed RPO | Proposed RTO | Decision |
|---|---:|---:|---|
| Core PostgreSQL metadata and conversations | 15 minutes | 4 hours | TBD |
| Attachments/object storage | 24 hours | 8 hours | TBD |
| Knowledge indexes | 24 hours or rebuild from source | 24 hours | TBD |
| Audit and billing ledger | 15 minutes | 8 hours | TBD |
| Queue/run state | 15 minutes; recover/reconcile in-flight work | 4 hours | TBD |

RPO is the maximum targeted data-loss window. RTO is the target time to restore service. Actual achievable values depend on selected managed services, region, tier, backup configuration, and tested recovery procedures. Never claim these targets until tested.

### 6.3 Proposed backup retention

Planning proposal:
- Daily recovery points: 30 days.
- Weekly recovery points: 12 weeks.
- Monthly recovery points: 12 months where lawful and cost-justified.

All periods are TBD. Align with data-retention policy, customer contracts, cost, and applicable law. Backup expiry must be observable and deletion tombstones must prevent restored data from reappearing.

### 6.4 Recovery drills

Before production:
- Perform a clean database restore into an isolated environment.
- Restore object data and verify checksums.
- Rebuild or restore vector/search indexes.
- Replay deletion tombstones.
- Reconcile queue leases and in-flight model/tool runs.
- Verify KMS/secret recovery without exposing key material.
- Run tenant-isolation and authorization smoke tests.
- Record actual RPO/RTO and remediate gaps.

Repeat at least quarterly after production launch (proposed cadence; owner approval required).

## 7. Data residency, subprocessors, and model-provider processing

### 7.1 Data inventory and classification

Maintain a current inventory for each data class with:
- data controller/processor roles as determined by counsel and contract;
- storage region and backup region;
- transit destinations;
- provider/subprocessor and service purpose;
- data categories transmitted;
- retention and deletion behavior;
- encryption and access controls;
- applicable transfer mechanism and contractual terms;
- customer-facing disclosure location.

Do not infer data residency from a provider's marketing statement or from the region selected for the SaaS database.

### 7.2 Provider gateway requirements

- Provider adapters must declare supported regions, data-processing terms, retention/training settings where documented, and relevant capabilities.
- Workspace owners must be able to see which provider/model will process a request and whether the credential is workspace-supplied or platform-managed, subject to safe disclosure.
- Platform administrators maintain an approved provider/model catalogue and can disable a provider globally or for selected workspaces.
- Route only the minimum context necessary for the request. Apply redaction and data minimization before provider submission.
- Do not send secrets, unrelated tenant data, hidden system configuration, or unauthorized retrieved documents to providers.
- Respect provider-specific retention, training, logging, and regional-processing controls. If required controls cannot be verified, mark the provider/model as restricted or unavailable for sensitive workloads.
- BYOK does not mean the SaaS platform has no processing responsibility: prompts, outputs, logs, traces, and tool context may still pass through platform infrastructure.
- No silent provider fallback. Any explicitly configured fallback must be visible, approved, auditable, and compatible with the workspace's data policy.
- Provider policy changes must be versioned and communicated when they materially affect processing.

### 7.3 Residency policy decision

**TBD before public production:** supported customer data regions, database region strategy, object-storage region, backup region, telemetry region, support-access geography, provider processing regions, cross-border transfer approach, and whether region pinning is a paid or standard capability.

For MVP, choose and document one deployment region only after reviewing target customers and provider availability. Do not advertise “data stays in your region” unless every relevant storage, backup, telemetry, support, and model-processing path satisfies that statement.

### 7.4 Provider due diligence gate

Before enabling a provider for production:
- review current terms, DPA, subprocessor list, retention controls, training use, incident notice, deletion process, and regional availability;
- record source URL, reviewer, review date, and approval;
- verify settings in the actual account/plan rather than assuming defaults;
- define which data classes may be sent;
- establish a disable/incident procedure;
- re-review on material provider policy changes and at least annually (proposed cadence).

Legal review is required for customer-facing privacy and data-processing commitments.

## 8. Pricing, quotas, and overage behavior

### 8.1 Separate technical capability from commercial entitlement

The product must function without requiring a public pricing model during internal development. Implement entitlement and metering interfaces so plans can be introduced without rewriting runtime behavior.

Potential billable dimensions:
- model input/output tokens or provider-reported cost;
- platform-managed model spend;
- storage and attachment bytes;
- knowledge ingestion/embedding usage;
- tool execution volume;
- monthly active users or chat sessions;
- premium features, retention, or regional hosting.

Do not assume all dimensions should be billed. Pricing design is a commercial decision.

### 8.2 Required metering

- Store immutable usage events with workspace, actor/session where available, request/run, provider/model, usage dimensions, timestamps, source, and reconciliation status.
- Distinguish provider-reported usage from estimates.
- Store currency and price-book version for monetary calculations.
- Never derive historical invoices from mutable current model prices.
- Deduplicate retries and webhook deliveries.
- Keep usage ledger access tenant-scoped and audited.
- Define treatment of failed, cancelled, retried, partially streamed, and provider-timeout requests.
- BYOK provider charges are billed by the provider to the key owner unless a separate platform agreement explicitly states otherwise. The platform may still meter usage for limits and analytics.
- Platform-paid usage must have hard spending caps and alert thresholds.

### 8.3 Overage modes

Support these modes as configuration, but do not enable a mode commercially until approved:
1. **Hard stop:** reject billable work at quota exhaustion.
2. **Prepaid balance:** deduct from an available balance; stop when exhausted.
3. **Postpaid overage:** allow usage and invoice later, only with explicit contract and spend controls.
4. **Admin grant:** temporary, audited credit/limit override with expiry.

Default recommendation for initial public beta: hard stop, no automatic paid overage, and no surprise charges. This is a proposal requiring approval.

### 8.4 Pricing decision record

**TBD before public paid launch:**
- plans and included quotas;
- price per billable unit;
- currency and tax handling;
- billing provider and invoice lifecycle;
- BYOK feature entitlement;
- free-tier abuse limits;
- overage mode and spend ceiling;
- refunds/credits and failed-request treatment;
- grace period, suspension, and reactivation;
- usage export and dispute process.

## 9. Customer communication and policy transparency

- Publish a plain-language policy page for retention, uploads, limits, subprocessors, regions, and billing.
- Show workspace owners effective limits and the source of each limit (platform, plan, or workspace).
- Notify workspace owners before material policy changes, except urgent security changes where prior notice would increase risk.
- Surface quota exhaustion, provider failures, upload rejection, and session expiry with actionable but non-sensitive error codes.
- Never expose other tenants' limits, usage, identifiers, or billing data.
- Provide export and deletion status to authorized users.
- Keep public documentation consistent with actual runtime configuration and contractual terms.

## 10. Release gates

The following are production gates, not optional documentation tasks:

| Gate | Evidence required |
|---|---|
| Retention and deletion | Approved retention matrix; deletion worker tests; tombstone restore test |
| Upload safety | Enforced limits; malware/quarantine path; parser limits; malicious-file tests |
| Widget sessions | Expiry, refresh, origin binding, revocation, replay tests |
| Rate limits and budgets | Shared atomic enforcement; reservation reconciliation; load/abuse tests |
| Service commitments | Approved SLA or explicit no-SLA preview statement; monitoring and incident ownership |
| Backup and DR | Successful restore drill; measured RPO/RTO; recovery runbook |
| Residency and providers | Data-flow inventory; approved provider due diligence; accurate privacy disclosures |
| Pricing and overages | Approved plan/price book; metering reconciliation; no-surprise-spend tests |

A capability must remain disabled, private beta, or explicitly best-effort when its required gate is not met.

## 11. Open decision register

The following decisions require named owner, decision date, and evidence before production:

| ID | Decision | Recommended next action |
|---|---|---|
| POL-001 | Conversation, run, attachment, log, audit, usage, and idempotency retention | Approve a per-data-class retention matrix after legal review |
| POL-002 | Upload size, type, count, extraction and concurrency limits | Approve MVP limits after parser/security tests |
| POL-003 | Widget access/idle/hard expiry and refresh/revocation semantics | Approve session lifecycle with identity and widget specs |
| POL-004 | API rate limits and per-run/workspace budgets | Approve load-tested defaults and anonymous abuse policy |
| POL-005 | Public availability SLA and support commitments | Keep preview/best-effort until measured and staffed |
| POL-006 | Backup retention, RPO, and RTO | Select managed-service tiers and pass recovery drills |
| POL-007 | Data residency and cross-border processing | Select launch region and complete legal/provider review |
| POL-008 | Provider data-processing allowlist | Approve providers individually after due diligence |
| POL-009 | Plans, prices, included quotas, and BYOK entitlement | Decide before paid launch; keep metering architecture-neutral |
| POL-010 | Overage, suspension, and credit policy | Prefer hard stop for initial beta; obtain approval |

Each decision must be recorded in ARCHITECTURE_DECISIONS.md or a dedicated policy decision record, with links to affected specs and tests.

## 12. Related specifications

- [Data lifecycle and privacy](DATA_LIFECYCLE_AND_PRIVACY.md)
- [Deployment and operations runbook](DEPLOYMENT_AND_OPERATIONS_RUNBOOK.md)
- [Security and threat model](SECURITY_AND_THREAT_MODEL.md)
- [Database design](DATABASE_DESIGN.md) and [database addendum](DATABASE_DESIGN_ADDENDUM.md)
- [Model Gateway specification](MODEL_GATEWAY_SPECIFICATION.md)
- [Widget and SDK specification](WIDGET_AND_SDK_SPECIFICATION.md)
- [Knowledge and retrieval specification](KNOWLEDGE_AND_RETRIEVAL_SPECIFICATION.md)
- [Quality and test strategy](QUALITY_AND_TEST_STRATEGY.md)
- [Contract alignment audit](CONTRACT_ALIGNMENT_AUDIT.md)
