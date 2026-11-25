# Cross-Document Contract Alignment Audit

**Audit date:** 2026-11-26  
**Scope:** Product scope, architecture, database design, API, identity, integrations, security, runtime, model gateway, widget, knowledge, administration, operations and test catalogue.  
**Disposition:** Documentation-level audit only; no code or migrations were changed.

## 1. Executive result

The architecture is coherent at the major-boundary level: cloud-hosted modular monolith, one universal agent per workspace, hosted model APIs, hybrid credentials with strict BYOK boundaries, MCP-first tools, customer-owned authorization, PostgreSQL/pgvector and an isolated iframe widget.

The documents are not yet a fully closed implementation contract. Several API resources and workflows are not represented as explicit database entities; a few policy details need to be made normative; and the original test catalogue did not yet cover all recently added specifications. The gaps below are assigned a proposed resolution so an implementation agent does not have to guess.

## 2. Contract gaps and resolutions

| ID | Area | Gap / ambiguity | Required resolution |
|---|---|---|---|
| AUD-001 | Workspace lifecycle | API/admin docs describe invitations and workspace deletion, but the logical schema has no invitation or deletion-job entity. | Add invitation records with hashed token, expiry, inviter, role, accepted/revoked timestamps; add durable lifecycle job/tombstone records. |
| AUD-002 | Widget sessions | API/widget docs issue scoped widget session tokens, but schema does not model session/token revocation or origin binding. | Add session records with workspace, agent, origin, actor, token hash/jti, expiry, revocation and last-seen metadata. Never store raw bearer tokens. |
| AUD-003 | Customer identity | Identity exchange requires registered issuers/keys, nonce replay protection and external subject mapping; schema only has general user auth subject. | Add customer identity provider configuration and nonce/replay records. Store public verification keys and issuer/audience metadata only. |
| AUD-004 | Tool approvals | Runtime/integration docs define approval bound to argument hash, actor and expiry, but schema only references approval_id from execution logs. | Add tool_approvals table with immutable arguments hash, tool/version, actor, workspace, approver, status, expiry and decision timestamps. |
| AUD-005 | API idempotency | API requires idempotency for eligible writes, but no persistence model or retention rule is defined. | Add idempotency_records keyed by tenant/client scope + key + operation; persist request hash and replayable result metadata with bounded TTL. |
| AUD-006 | SSE replay | API describes Last-Event-ID replay but database design has no durable event/outbox record. | Define durable run_events/outbox with monotonic per-run sequence, event type, redacted payload and retention; otherwise explicitly disable replay in MVP. |
| AUD-007 | Usage budgets | Gateway describes atomic budget reservations and reconciliation; schema has usage_events but no reservation/limit ledger. | Add workspace_budget_policies and usage_reservations with idempotent reserve/settle/release transitions. |
| AUD-008 | Knowledge versions | Knowledge spec describes source versions and atomic publish, but schema only has sources/documents/chunks without explicit source version or active-version pointer. | Add knowledge_source_versions and bind documents/chunks to version; publish by atomically switching active version. |
| AUD-009 | Deletion lifecycle | Privacy spec requires deletion tombstones and backup replay, but schema has only deleted_at on documents. | Add deletion_jobs and deletion_tombstones with resource scope, status, completed stores and backup-expiry metadata. |
| AUD-010 | Workspace API keys | API supports workspace server credentials, but no API-key/token table is defined. | Add workspace_api_tokens with token hash, prefix, scopes, creator, expiry, last-used and revocation metadata. |
| AUD-011 | Audit model | Admin spec expects before/after diffs, reason and request correlation; audit_logs metadata is generic. | Standardize audit columns: request_id, reason, outcome, resource version, redacted before/after JSON and actor role snapshot. |
| AUD-012 | Conversation identity | Widget/runtime distinguish session from conversation; database has conversations but no explicit session table or actor subject issuer. | Add chat_sessions and bind conversations/runs to session; preserve external issuer + subject mapping without trusting client identifiers. |
| AUD-013 | Attachments | API and widget reference uploaded attachments and scoped grants, but storage schema is not explicit. | Add uploads/attachments with object key, content hash, detected MIME, scan state, size, workspace/session ownership, expiry and deletion status. |
| AUD-014 | Tool execution recovery | Runtime requires idempotent recovery and approval state, but execution logs alone do not model durable execution attempts/leases. | Add tool_execution_attempts with idempotency key, lease, attempt number, state transition timestamps and result reference. |
| AUD-015 | API schema source | API document asks for runtime validation and OpenAPI generated from one source, but does not choose contract-first vs code-first. | For documentation phase, maintain OpenAPI 3.1 as reviewed public contract; implementation should generate/validate runtime schemas from the same typed definitions or add drift checks. |
| AUD-016 | Retention | Privacy document deliberately defers exact periods; production runbook also lacks numeric SLO/RPO/RTO. | Keep these as launch-blocking policy decisions. Do not invent defaults in code; require explicit deployment/product configuration and approved public policy. |

## 3. Cross-document rules confirmed

1. A workspace is the tenant boundary for agent, conversations, integrations, knowledge, usage and BYOK credentials.
2. Organization membership may govern workspace access but does not grant customer application permissions.
3. Platform-admin privileges are separate from workspace roles.
4. BYOK credentials are never silently replaced by platform credentials or another model/provider.
5. Platform fallback remains disabled until an administrator explicitly publishes a valid policy and route.
6. Model-generated tool calls are proposals; the Tool Execution Broker makes the authorization decision.
7. Discovered MCP tools remain disabled until reviewed and enabled.
8. Retrieved knowledge and tool output are untrusted data, not instructions.
9. Widget public keys identify configuration only; short-lived session tokens authorize narrowly scoped chat operations.
10. Customer backend identity assertions establish identity but do not grant blanket application permissions.
11. RLS is defense-in-depth; service authorization and tenant-aware foreign keys remain required.
12. Free provider quotas are not production capacity commitments.

## 4. API-to-database coverage

The API route catalogue is a logical route plan, not proof that every supporting table already exists. The AUD items above are prerequisites for implementing their respective workflows. In particular, do not implement widget sessions, idempotency, approval, replay, budget reservation or deletion orchestration as ephemeral in-memory-only behavior.

## 5. OpenAPI contract status

The companion `docs/openapi/openapi.yaml` is the initial machine-readable public contract for core workspace, agent, widget-session, chat, identity and provider-credential operations. It intentionally marks the first contract slice and does not claim that every endpoint in the API catalogue has been fully schema-modeled. Extend it before implementing additional route families.

## 6. Decisions required before production

The following are product/operational choices, not safe assumptions:
- Numeric conversation, audit, telemetry, attachment and knowledge retention periods.
- Backup retention, deletion completion SLA, RPO and RTO.
- Public API rate limits, widget token lifetime and maximum upload sizes.
- Supported regions, data residency, subprocessors and provider data-handling commitments.
- Pricing, workspace budget defaults, overage behavior and billing policy.
- Support-access approval policy and customer notification commitments.

Record each approved choice in the relevant specification and an ADR if it changes architecture.

## 7. Audit completion criteria

- Every AUD item is either implemented in the logical schema, explicitly deferred with a documented MVP limitation, or rejected by an approved ADR.
- OpenAPI and API prose agree on path, authentication, status and payload semantics.
- Every public route has authorization and tenant-isolation test coverage.
- Every durable workflow has retry, idempotency and terminal-state semantics.
- No production numeric policy is silently inferred from a development default.


## 8. Policy follow-through

The previously open policy decisions are now specified in [Production Policies and Service-Level Specification](PRODUCTION_POLICIES_AND_SLOS.md), including ten decision IDs POL-001–POL-010. This closes the documentation gap at the requirements level, but does **not** mean the business decisions are approved or that implementation is complete.

AUD-016 remains an explicit production gate: numeric retention, upload/session limits, rate limits, backup retention, RPO/RTO, residency, provider commitments and pricing/overage settings require owner approval. Proposed example values must remain configuration proposals and must not be copied into hardcoded constants or public promises.
