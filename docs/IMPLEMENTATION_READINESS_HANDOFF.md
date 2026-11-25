# Implementation-Readiness Review and Coding-Agent Handoff

**Review date:** 2026-11-26  
**Status:** Documentation review complete; implementation has not started  
**Audience:** The coding agent and the project owner  
**Scope:** Cross-document consistency, API/OpenAPI coverage, logical database coverage, test traceability, implementation order, and unresolved decisions

## 1. Executive verdict

**The repository is ready for implementation to begin at M0. It is not yet a fully closed production implementation contract.**

The product direction and major architectural boundaries are consistent:
- Cloud-hosted SaaS, modular monolith, separate worker process.
- One universal agent per workspace.
- Hosted model APIs only; no local inference/Ollama.
- Hybrid platform-managed and workspace BYOK credentials, with no silent BYOK fallback.
- MCP-first integrations; all tool calls pass through a deterministic Tool Execution Broker.
- Customer application remains authoritative for end-user authorization.
- PostgreSQL/pgvector, private object storage, durable jobs, managed secret storage, and an isolated iframe widget.
- Both guest visitors and authenticated application users are supported.

The main remaining gaps are not reasons to delay repository foundation work. They are contract-closure tasks that must be resolved before implementing the affected subsystem or exposing it to customers.

### Readiness by area

| Area | Readiness | Finding |
|---|---|---|
| Product and architecture | Ready | Core decisions and MVP boundaries are consistent |
| M0 repository foundation | Ready | Five detailed GitHub issues already exist |
| Tenant schema and RLS | Needs implementation design review | Logical schema is substantial; role/permission and composite-FK details need one canonical choice |
| Model Gateway/BYOK | Ready to implement against fake providers | Strict routing invariants are clear; live provider approval remains operational |
| Agent runtime | Ready to implement | Lifecycle, untrusted context, bounded execution and terminal states are specified |
| Widget and identity | Needs contract closure before M4 | Session model exists, but customer subject mapping and token lifecycle must be made explicit |
| MCP tool broker | Needs schema alignment before execution | Stable tool identity vs immutable tool-version records are not fully reconciled |
| Knowledge | Ready after schema migration design | Source versioning and deletion lifecycle are specified; worker/storage choices remain implementation decisions |
| Public API | Partial contract | OpenAPI is an initial slice, not the complete API catalogue |
| Production operations | Not production-ready yet | Policy values and provider/residency/legal decisions remain TBD |
| Tests | Good planned coverage, not executable evidence | 86 catalogue entries exist; all are planned, not passing tests |

## 2. Confirmed invariants — do not change without an ADR

1. A workspace is the tenant boundary for its agent, conversations, integrations, knowledge, usage and BYOK credentials.
2. The SaaS dashboard identity and customer application's end-user identity are separate identity domains.
3. A customer identity assertion proves identity only; it does not grant blanket permission to customer records.
4. Authorization defaults to deny and is checked server-side for every protected tool invocation.
5. Model-generated tool calls are proposals. The Tool Execution Broker owns validation, authorization, approval, execution and audit.
6. Newly discovered or materially changed MCP tools remain disabled until reviewed and approved.
7. BYOK failures never silently switch to platform credentials, another provider, another model or platform-paid billing.
8. Platform fallback remains disabled until a platform administrator explicitly enables a valid policy and route.
9. Retrieved knowledge, user content, MCP metadata and tool output are untrusted data, never policy instructions.
10. Widget public identifiers expose publishable configuration only. Short-lived scoped session tokens authorize chat operations.
11. RLS is defense in depth; application authorization and tenant-safe relationships are also required.
12. Free API tiers are development allowances, not production capacity or availability guarantees.
13. Retention, limits, residency, pricing, RPO/RTO and SLA values remain unapproved until explicitly decided.

## 3. Cross-document findings and required resolutions

These findings are the handoff review output. The coding agent should address them in the indicated phase and update this document's status as evidence is produced.

### HND-001 — Customer identity subject mapping is not fully modeled

**Severity:** P0 before authenticated identity handoff (M4)

The identity specification requires mapping the customer issuer plus stable subject to a workspace-scoped customer identity. The database addendum adds `customer_identity_providers` and replay nonces, but does not define the actual customer identity mapping table. A `chat_sessions.actor_reference` alone is not a durable issuer/subject mapping contract.

**Required resolution:**
- Add a `customer_identities` (or equivalently named) table keyed by workspace and registered issuer/provider plus stable external subject.
- Define uniqueness, status/deprovisioning, creation/update timestamps, and whether the external subject is stored raw, encrypted, or as a keyed digest. Make the choice based on lookup and privacy requirements.
- Link authenticated chat sessions to the internal customer identity row; retain guest sessions as a distinct actor type.
- Define how subject deletion, issuer key rotation, and customer-user deprovisioning revoke active sessions.
- Add cross-workspace and issuer-collision tests.

Do not treat the general SaaS `users.auth_provider/auth_subject` columns as the customer application's identity map.

### HND-002 — MCP tool version schema needs one canonical model

**Severity:** P0 before enabling tool execution (M5)

The integration specification describes `integration_discovery_runs` and immutable `integration_tool_versions`, with a stable logical tool identity across schema changes. The base database design instead has a single mutable `integration_tools` schema row with `schema_version`. The addendum's `tool_approvals` references `tool_id` plus a version value, while the integration specification binds approvals to a specific immutable tool version.

**Required resolution:**
- Keep a stable `integration_tools` identity and add immutable `integration_tool_versions` records (or document an equivalent versioned representation).
- Add `integration_discovery_runs` for lifecycle and diagnostics.
- Bind approvals, execution attempts, permission decisions and audit events to the immutable tool-version ID, not only a mutable version string.
- Define whether permissions apply to a logical tool, a specific version, or both; changed schemas must require review as specified.
- Add workspace-consistent foreign keys for integration, tool, version, approval and execution rows.
- Align `tool_execution_logs` and `tool_execution_attempts` with integration ID, run ID, tool version, actor reference, idempotency key and result reference.

The coding agent must not implement two competing tool-version models.

### HND-003 — Workspace role enum and permission-catalog model need reconciliation

**Severity:** P0 during M1 authorization implementation

The base database design stores role names directly on `workspace_members`. The admin/identity specifications also describe roles, permissions, role-permission mappings and permission-based authorization. Both can work, but the source of truth is not explicit.

**Required resolution:**
- Choose either a normalized role/permission schema or a small fixed role enum mapped centrally to a permission catalogue.
- Preserve platform roles separately from workspace roles.
- Avoid scattered checks such as `role === "admin"`; services must call the canonical permission evaluator.
- Define organization-level membership inheritance into workspaces explicitly; organization membership must not imply customer-app permissions.
- Document migration and test strategy for role/permission changes.

### HND-004 — OpenAPI is intentionally incomplete relative to the API catalogue

**Severity:** P0 before implementing each uncovered public route

Static inspection found **11 path entries and 16 operations** in `docs/openapi/openapi.yaml`. This is explicitly the initial contract slice, not a complete mirror of `API_SPECIFICATION.md`.

Route families that still need contract coverage include, at minimum:
- Workspace members and invitations.
- Agent publish/version lifecycle.
- Chat session read/feedback and any message/run operations not already represented.
- Model routing preferences and workspace routing updates.
- Credential listing and revocation.
- MCP connection CRUD, validation, discovery, tool review and permissions.
- Knowledge source upload/list/reprocess/delete/search.
- Usage, audit, deletion and workspace lifecycle operations.
- Tool approval decision endpoints.
- Platform-admin provider catalog, defaults, overrides and fallback policy endpoints.

**Required resolution:**
- Keep OpenAPI 3.1 as the reviewed public contract during documentation-first development.
- Add each route family before implementing that family.
- Every operation must define caller authentication, permission, request/response schema, error responses, idempotency, rate-limit behavior and tenant scope.
- Validate OpenAPI syntax/schema and operation/path-parameter consistency in CI. The current review counted paths/operations but did not run a formal OpenAPI linter.

Do not claim API contract completeness until all in-scope MVP routes are represented or explicitly deferred.

### HND-005 — Generic policy versioning is specified conceptually but not fully represented in schema

**Severity:** P1 before policy-heavy M7 features; design before M1/M2 schema freeze

The production policy specification requires versioned policy snapshots and precedence across platform safety ceilings, plan entitlements, workspace overrides and request/session restrictions. Existing schema has some specialized policy tables, but no complete canonical representation for policy version, approval, effective time and immutable change history.

**Required resolution:**
- Decide whether to use typed policy tables per domain or a generic versioned policy registry plus domain-specific typed records.
- Define atomic publication and the effective-policy snapshot recorded on each run/decision.
- Ensure workspace policies can tighten, but never weaken, mandatory platform controls.
- Audit actor, approver, reason, effective interval and previous version.
- Do not turn proposed numeric values in the policy document into hardcoded constants.

### HND-006 — Usage metering needs price-book version linkage

**Severity:** P1 before paid billing; budget enforcement still required before external beta

The database has append-only `usage_events` and budget reservations, while the production policy requires historical monetary calculations to use an immutable price-book version. The current logical schema does not define a dedicated price-book/model-price version entity or a clear reference from each priced usage event.

**Required resolution:**
- Add a versioned price book or immutable provider/model price snapshots.
- Record the price-book version and currency used for every monetary calculation.
- Distinguish provider-reported usage/cost from platform estimates.
- Preserve original usage facts even when current model pricing changes.
- BYOK provider charges remain provider-billed unless a separately approved commercial arrangement says otherwise.

### HND-007 — Test strategy contains a stale catalogue count

**Severity:** P2 documentation correction

The quality strategy still says the catalogue contains 72 planned cases. The current JSON catalogue contains **86 entries with 86 unique IDs**. The catalogue is still planned coverage, not evidence that tests have been implemented or passed.

**Required resolution:** keep the count synchronized or remove the hard-coded count; update it whenever the catalogue changes.

### HND-008 — Installation and ownership relationships need explicit schema mapping

**Severity:** P1 before widget/MCP lifecycle is finalized

Product, identity and integration specifications refer to an installation/widget deployment binding. The database addendum models `chat_sessions.origin` and workspace/agent scope but does not clearly define a first-class installation entity, while some API paths identify a workspace and public key.

**Required resolution:**
- Decide whether one workspace can have multiple widget installations/environments with distinct origins, public keys and lifecycle.
- If yes, add a workspace-scoped installation record and bind public keys, sessions, customer identity issuers and integration environment to it where required.
- If no for MVP, document the single-installation constraint and make the schema/API enforce it.
- Do not rely on origin alone as the identity of an installation.

### HND-009 — Formal contract validation remains pending

**Severity:** P0 CI requirement in M0/M1

The OpenAPI file has been structurally inspected for path and operation counts, but this review did not run a YAML parser, OpenAPI validator, generated-client check or runtime contract test. The test catalogue is valid JSON with unique IDs, but its entries are plans, not executable tests.

**Required resolution:**
- Add YAML parse and OpenAPI 3.1 validation to CI.
- Lint duplicate operation IDs, undeclared path parameters, unresolved references and security overrides.
- Validate JSON catalogue schema, required fields, unique IDs and traceability references in CI.
- Add drift checks between runtime validators, OpenAPI and API prose once code exists.

## 4. Exact implementation order

The coding agent should follow this sequence. It may split work into parallel branches only where dependencies are explicitly independent.

### Stage 0 — Repository foundation (M0)

1. Complete GitHub issue [#1 — Initialize pnpm/Turborepo monorepo](https://github.com/the-tech-makers/chat-agent/issues/1).
2. Then work in parallel on:
   - [#2 — Create web, widget and worker app shells](https://github.com/the-tech-makers/chat-agent/issues/2)
   - [#3 — Add PostgreSQL and pgvector development environment](https://github.com/the-tech-makers/chat-agent/issues/3)
   - [#4 — Add CI and deterministic test harness](https://github.com/the-tech-makers/chat-agent/issues/4)
3. Complete [#5 — Add health checks and local setup documentation](https://github.com/the-tech-makers/chat-agent/issues/5).
4. Add automated JSON-catalogue validation and OpenAPI syntax/schema validation to CI (HND-009).

**Exit gate:** a clean clone installs, typechecks, lints, builds, starts the local dependencies and passes deterministic CI without live provider keys.

### Stage 1 — Tenant and persistence foundation (M1)

1. Select the SaaS dashboard authentication provider/library.
2. Resolve HND-003: canonical roles/permissions and organization/workspace membership behavior.
3. Resolve tenant ownership, composite foreign keys, runtime DB role, transaction-scoped RLS context and background-job tenant context.
4. Implement users, organizations, workspaces, memberships, invitations, platform-admin boundary and audit foundation.
5. Implement one-agent-per-workspace constraints and agent versions.
6. Implement sessions, conversations, messages, runs, idempotency and durable event state according to the chosen MVP replay contract.
7. Add two-tenant RLS/IDOR tests and migration-from-empty tests.

**Exit gate:** runtime-role tests prove cross-tenant reads/writes fail; no missing tenant context broadens access; schema constraints prevent cross-workspace references.

### Stage 2 — Model Gateway and usage controls (M2)

1. Define provider-neutral request, streaming, tool-call, error and usage contracts.
2. Implement fake provider and contract suite first.
3. Integrate the secret manager/envelope-encryption abstraction.
4. Implement the first hosted provider adapter (OpenRouter is the preferred starting adapter, subject to provider due diligence).
5. Implement deterministic platform/BYOK route resolution, workspace bindings and admin overrides.
6. Implement strict BYOK no-substitution behavior and disabled-by-default platform fallback.
7. Implement usage events, reservations, budget settlement/release, per-run ceilings and kill switches.
8. Resolve HND-005/HND-006 before policy publication or paid metering.

**Exit gate:** fake-provider tests prove deterministic routing, correct credential ownership, redaction, budget reservation settlement and no silent fallback. Live provider smoke tests are opt-in and spend-capped.

### Stage 3 — Guest chat vertical slice (M3)

1. Implement conversation/message service and agent-run lifecycle.
2. Implement bounded context assembly and prompt trust boundaries.
3. Implement typed streaming, cancellation, terminal-state persistence and reconnect semantics.
4. Implement guest widget session bootstrap, exact origin checks and iframe protocol.
5. Implement basic dashboard agent configuration and publish/version behavior.
6. Use the fake provider in E2E tests before enabling live provider traffic.
7. Resolve the installation model in HND-008 before multiple customer installs/environments are supported.

**Exit gate:** a sample customer page can embed the widget; a guest can complete a bounded conversation; invalid origin/session and cross-tenant attempts fail.

### Stage 4 — Authenticated customer identity (M4)

1. Resolve HND-001 and HND-008.
2. Register customer identity issuer/audience/key metadata.
3. Implement server-to-server assertion validation, replay prevention and key rotation.
4. Map issuer + subject to a workspace-scoped customer identity.
5. Issue, refresh and revoke short-lived chat sessions.
6. Re-authorize access to existing conversations after session refresh.
7. Add fail-closed customer authorization strategy (delegated user token and/or customer authorization proxy).

**Exit gate:** forged, expired, replayed, wrong-issuer, wrong-audience, wrong-workspace and deprovisioned identities are rejected; authenticated identity cannot access records merely because a service credential can.

### Stage 5 — MCP integrations and Tool Execution Broker (M5)

1. Resolve HND-002 and add discovery/version persistence.
2. Implement remote Streamable HTTP MCP connection lifecycle and SSRF-safe egress.
3. Implement authentication/secret lifecycle and discovery history.
4. Implement normalized immutable tool versions and review workflow.
5. Implement disabled-by-default tools and actor-specific permissions.
6. Implement durable approvals bound to exact tool version, canonical arguments hash, actor and expiry.
7. Implement execution attempts, leases, idempotency and bounded retries.
8. Implement result sanitization, audit and customer authorization checks.

**Exit gate:** fake MCP server tests prove no unreviewed, changed, unauthorized, expired-approval or cross-tenant tool can execute. Non-idempotent side effects are not retried without upstream idempotency.

### Stage 6 — Knowledge and retrieval (M6)

1. Implement private object storage and upload grants.
2. Resolve and enforce upload file/type/count/extraction/concurrency limits from approved configuration.
3. Add quarantine/malware-scanning integration and isolated parsing.
4. Implement durable ingestion jobs, source versions and atomic publication.
5. Implement hosted embeddings through an approved provider adapter.
6. Implement workspace/source/actor ACL filtering before model context assembly.
7. Implement citations, re-ingestion, rollback, deletion jobs and tombstone behavior.

**Exit gate:** hostile/oversized files are rejected or quarantined; only active authorized chunks are retrieved; failed re-ingestion preserves the previous active version; deletion covers originals and derived data.

### Stage 7 — Operations and private/public beta gates (M7)

1. Configure retention and deletion policy (POL-001).
2. Approve upload/session/rate/budget settings (POL-002–POL-004).
3. Complete monitoring, alert ownership, incident response and rollback.
4. Approve backup retention, RPO/RTO and pass a restore drill (POL-006).
5. Approve residency and provider data-processing allowlist (POL-007/POL-008).
6. Complete security assessment, abuse/load tests, tenant-isolation regression and emergency kill-switch drills.
7. Keep service preview/best-effort until an SLA is formally approved (POL-005).
8. Approve pricing, entitlements and overage behavior before any paid launch (POL-009/POL-010).

**Exit gate:** every applicable production policy release gate has recorded evidence; no unresolved critical/high security issue lacks an explicit risk owner and mitigation.

### Stage 8 — Post-MVP (M8)

Only after measuring customer demand: OpenAPI connector, read-only database adapters, additional direct providers, advanced knowledge connectors, analytics/evaluations, enterprise SSO/SCIM, additional SDKs and scale-oriented service extraction.

## 5. Dependency map

```text
M0 Repository + CI
        |
        v
M1 Tenant DB + Auth + RLS
        |
        +-----------------------+
        |                       |
        v                       v
M2 Model Gateway           M3 Guest Chat / Widget
        |                       |
        +-----------+-----------+
                    |
                    v
             M4 Customer Identity
                    |
                    v
             M5 MCP Tool Broker

M1 + object storage + durable queue + embedding adapter
                    |
                    v
             M6 Knowledge

M2 + M3 + M4 + M5 + M6
                    |
                    v
             M7 Production gates
                    |
                    v
             M8 Expansion
```

M2 and the guest-chat portion of M3 can be developed in parallel after M1 contracts are stable. Private/customer-specific tools in M5 must not be enabled until M4 authorization is complete. M6 can be developed in parallel with M4/M5 once tenant-safe storage, queue and policy foundations exist.

## 6. Decisions required and when they block

| Decision | Needed by | Blocks | Recommendation / guardrail |
|---|---|---|---|
| SaaS dashboard auth provider/library | M1 | Dashboard identity and membership implementation | Choose a maintained managed-auth option; keep provider behind an internal auth boundary |
| Canonical role/permission model (HND-003) | M1 | Authorization middleware and RLS policy design | Central permission evaluator; platform roles separate |
| Customer identity subject mapping (HND-001) | M4 design, before M4 coding | Authenticated session exchange | Explicit workspace + issuer + subject mapping |
| Widget installation cardinality (HND-008) | M3/M4 | Public key, origin and session binding | Decide multi-installation vs one-installation MVP and enforce it |
| MCP tool-version model (HND-002) | M5 | Discovery, approvals and execution persistence | Stable tool identity + immutable version records |
| API/OpenAPI contract strategy (AUD-015/HND-004) | Before each route family | Public API implementation | Maintain reviewed OpenAPI and validate runtime schemas against it |
| Generic policy versioning (HND-005) | Before policy engine / M7 | Runtime policy snapshots | Versioned, atomic, auditable effective policy |
| Immutable price-book model (HND-006) | Before paid billing | Invoice-grade monetary calculations | Store price-book version with every priced usage event |
| Exact retention periods (POL-001) | Before public production | Production retention behavior and legal commitments | Approve by data class after legal review |
| Upload/session/rate/budget numeric limits (POL-002–POL-004) | Before public beta | Public traffic and upload exposure | Configurable, tested, server-enforced values; no hardcoded proposals |
| SLA/support commitments (POL-005) | Before publishing SLA | Public contractual claims | Remain preview/best-effort until measured and approved |
| Backup retention/RPO/RTO (POL-006) | Before production | Production launch | Select managed tiers and demonstrate restore |
| Data regions/provider processing (POL-007/POL-008) | Before customer data production | Provider enablement and privacy claims | Verify every storage, backup, telemetry and inference path |
| Plans/prices/overages (POL-009/POL-010) | Before paid launch | Billing and paid entitlements | Prefer hard-stop beta; no automatic paid overage without explicit approval |

**Do not block M0 on unresolved pricing, SLA, residency or retention decisions.** Do not enable the affected production capability until its decision is approved.

## 7. API and schema coverage rules for the coding agent

For every endpoint implemented:
- Add or update its OpenAPI path and operation first.
- Define authentication class, permission, tenant scope and non-disclosing cross-tenant behavior.
- Define request/response/error schemas and idempotency semantics.
- Map every durable resource to a table/entity and define unique/composite foreign-key constraints.
- Add positive, negative, cross-tenant, retry and failure-path tests.
- Ensure widget/browser contracts never expose workspace, provider or connector secrets.

For every durable workflow:
- Define state machine and legal transitions.
- Define idempotency key scope and retry behavior.
- Define lease/timeout/recovery behavior.
- Define audit events and redaction.
- Define retention and deletion behavior.
- Ensure background jobs re-load authoritative database state and re-establish tenant scope.

## 8. Test catalogue and evidence status

The current `tests/cases/catalog.json` contains **86 test cases with 86 unique IDs**. They are planned acceptance/security/operational test definitions, not implemented or passing automated tests.

The coding agent must:
- Preserve stable IDs and update traceability when requirements change.
- Implement tests at the milestone where the capability is introduced.
- Keep deterministic fake provider and fake MCP tests in ordinary CI.
- Use at least two tenant fixtures for isolation tests.
- Add OpenAPI validation, JSON catalogue schema/uniqueness checks and contract drift checks to CI.
- Record manual production evidence for legal review, provider terms, real restore drills and incident readiness.

## 9. Documentation authority and change rules

When documents conflict, use this resolution order:
1. Explicit accepted ADRs for settled architectural decisions.
2. Product Scope for MVP inclusion/exclusion.
3. Domain-specific normative specifications for subsystem behavior.
4. Database Design + approved Addenda for persistence model.
5. API Specification + OpenAPI for external API contract (OpenAPI must be extended before implementing a missing route).
6. Production Policies for operational/commercial policy; TBD means not approved.
7. Implementation Roadmap for delivery order.
8. Traceability register and test catalogue for verification mapping.

This is not permission to ignore contradictions. If two normative documents conflict, stop work on that subsystem, propose one canonical contract, update affected documents and add an ADR if the decision changes architecture.

Do not silently change the accepted hosted-model-only, one-agent-per-workspace, MCP-first, customer-owned-authorization or strict-BYOK decisions.

## 10. Final handoff checklist

- [ ] M0 issues #1–#5 completed and CI evidence linked.
- [ ] HND-001 through HND-009 resolved or explicitly deferred with approved rationale.
- [ ] OpenAPI contains every route before its implementation begins.
- [ ] Database migrations include tenant-safe composite relationships and RLS.
- [ ] All 86 catalogue cases are either implemented, explicitly deferred or retired with rationale; catalogue count is not a pass metric.
- [ ] M7 production policy decisions are approved and evidence-backed before public production.
- [ ] README and roadmap remain aligned with actual implementation status.

## 11. Related documents

- [Product Scope](PRODUCT_SCOPE.md)
- [System Architecture](ARCHITECTURE.md)
- [Database Design](DATABASE_DESIGN.md) and [Database Addendum](DATABASE_DESIGN_ADDENDUM.md)
- [API Specification](API_SPECIFICATION.md) and [OpenAPI 3.1](openapi/openapi.yaml)
- [Identity and Authorization](IDENTITY_AND_AUTHORIZATION.md)
- [Security and Threat Model](SECURITY_AND_THREAT_MODEL.md)
- [Agent Runtime](AGENT_RUNTIME_SPECIFICATION.md)
- [Model Gateway](MODEL_GATEWAY_SPECIFICATION.md)
- [Integration Architecture](INTEGRATION_ARCHITECTURE.md)
- [Widget and SDK](WIDGET_AND_SDK_SPECIFICATION.md)
- [Knowledge and Retrieval](KNOWLEDGE_AND_RETRIEVAL_SPECIFICATION.md)
- [Production Policies and SLOs](PRODUCTION_POLICIES_AND_SLOS.md)
- [Implementation Roadmap](IMPLEMENTATION_ROADMAP.md)
- [Quality and Test Strategy](QUALITY_AND_TEST_STRATEGY.md)
- [Requirements Traceability](TRACEABILITY_AND_TASK_REGISTER.md)
- [Automated Test Catalogue](../tests/cases/catalog.json)
