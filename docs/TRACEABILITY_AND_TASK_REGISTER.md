# Requirements Traceability and Task Register

## 1. Purpose

This register connects each approved project document to implementation tasks, acceptance evidence, and automated tests. It is the project-level index; detailed test procedures live in [Quality and Test Strategy](QUALITY_AND_TEST_STRATEGY.md), and machine-readable test definitions live in [Automated Test Catalogue](../tests/cases/catalog.json).

**Rule:** a requirement is not complete until its implementation is merged, its acceptance criteria are met, and its required automated tests pass.

## 2. Status model

Use these exact task statuses in GitHub Issues and the GitHub Project:

| Status | Meaning | Entry / exit rule |
|---|---|---|
| Backlog | Accepted, not started | Default for approved work |
| Ready | Refined and unblocked | Scope, acceptance criteria, dependencies and tests are defined |
| In Progress | Actively being implemented | An assignee is actively working |
| In Review | Pull request open | Implementation and tests are ready for review |
| Blocked | Cannot proceed | Add a blocker issue/link and explanation |
| Done | Verified and merged | Acceptance criteria met, required CI checks green, PR merged |
| Won't Do | Explicitly deferred/rejected | Decision and rationale recorded |

Do not use "Done" for code that has not passed its required tests. A test failure reopens the associated task or creates a linked defect.

## 3. GitHub Project setup

Create one GitHub Project named **Chat Agent — Delivery** and configure:
- Status: Backlog, Ready, In Progress, In Review, Blocked, Done, Won't Do.
- Priority: P0, P1, P2, P3.
- Milestone: M0 through M8 (matching the implementation roadmap).
- Work type: Feature, Engineering, Security, Test, Documentation, Bug.
- Fields: Assignee, Iteration (optional), Target date (optional), Test status (Not run, Passing, Failing, Not applicable).

Recommended views:
1. **Delivery board** — grouped by Status.
2. **Current milestone** — filtered by active Milestone, grouped by Status.
3. **P0 / blockers** — Priority is P0 or Status is Blocked.
4. **Verification** — grouped by Test status.
5. **Roadmap** — grouped by Milestone.

GitHub Project is the live status source. This Markdown register is the durable scope and traceability index. Update the issue/project when work changes; update this file when scope, task decomposition, or acceptance mapping changes.

## 4. Issue and pull request conventions

Create one issue per independently verifiable task (normally 0.5–3 days). Avoid one issue per tiny code edit and avoid broad issues such as "build backend".

Issue title format: [M2][P0][Model Gateway] Enforce BYOK no-fallback behavior.

Every implementation issue must contain:
- **Requirement IDs:** e.g. REQ-MOD-014.
- **Scope:** what is and is not included.
- **Dependencies:** linked issues.
- **Acceptance criteria:** observable, binary statements.
- **Test IDs:** e.g. SEC-004, MOD-006.
- **Security/privacy considerations:** including "none identified" where appropriate.
- **Evidence:** CI run, screenshots, logs with secrets redacted, or test report.

Every pull request must link its issue and include:
- Summary and design notes.
- Requirement IDs and test IDs.
- Migrations / rollout / rollback notes where relevant.
- Test commands executed and results.
- Security and data migration impact.
- Confirmation that no secrets, personal data, or customer data were added to fixtures/logs.

Suggested labels:
- priority:P0, priority:P1, priority:P2, priority:P3
- type:feature, type:engineering, type:security, type:test, type:bug, type:docs
- area:platform, area:tenancy, area:identity, area:models, area:agent-runtime, area:mcp, area:knowledge, area:widget, area:operations

## 5. Document-to-task traceability

The task IDs below are stable references. Create GitHub Issues from these tasks as each milestone is refined. Status is initially **Backlog** unless explicitly changed in the Project.

### DOC-01 — Product scope ([PRODUCT_SCOPE.md](PRODUCT_SCOPE.md))

| Task ID | Priority | Task | Acceptance evidence | Test IDs |
|---|---|---|---|---|
| PROD-001 | P0 | Define workspace onboarding and lifecycle | Owner can create and configure an isolated workspace | TEN-001, API-001 |
| PROD-002 | P0 | Implement one versioned agent configuration per workspace | Publish, inspect and roll back configuration versions | AGT-001, AGT-002 |
| PROD-003 | P0 | Implement conversation/session/message lifecycle | Guest and signed-in chat flows persist correctly | CHAT-001, CHAT-002 |
| PROD-004 | P0 | Implement embeddable isolated chat widget | Widget loads only on allowed origins and can start a chat | WID-001, SEC-001 |
| PROD-005 | P0 | Implement tool review, enablement and policy controls | Newly discovered tools remain disabled until approved | MCP-003, SEC-006 |
| PROD-006 | P0 | Implement knowledge upload and retrieval MVP | Supported file is ingested, indexed and retrieved with source attribution | KNOW-001, KNOW-002 |
| PROD-007 | P1 | Implement context generation, review and publishing | Draft context cannot affect published agent until approved | CTX-001 |
| PROD-008 | P1 | Implement usage, feedback and operational logs | Usage and feedback are scoped and queryable | OBS-001, TEN-003 |

### DOC-02 — System architecture ([ARCHITECTURE.md](ARCHITECTURE.md))

| Task ID | Priority | Task | Acceptance evidence | Test IDs |
|---|---|---|---|---|
| ARC-001 | P0 | Establish modular monolith boundaries and dependency rules | Packages depend only on approved inward-facing contracts | ARC-001 |
| ARC-002 | P0 | Implement Universal Agent Runtime orchestration | Runtime loads published config, context, retrieval and tools in order | AGT-001, AGT-003 |
| ARC-003 | P0 | Implement hosted Model Gateway provider adapter contract | Fake provider can run all runtime tests without API credentials | MOD-001 |
| ARC-004 | P0 | Implement platform defaults, workspace preferences and explicit overrides | Resolution is deterministic and auditable | MOD-002, MOD-003 |
| ARC-005 | P0 | Enforce BYOK credential ownership and no silent fallback | BYOK errors are surfaced; no other credential is attempted | MOD-004, MOD-005 |
| ARC-006 | P1 | Implement worker and durable job boundaries | Retryable work is idempotent and observable | JOB-001 |
| ARC-007 | P1 | Implement audit, usage and tracing conventions | Correlation IDs link requests without logging secrets | OBS-001, SEC-010 |

### DOC-03 — Database design ([DATABASE_DESIGN.md](DATABASE_DESIGN.md))

| Task ID | Priority | Task | Acceptance evidence | Test IDs |
|---|---|---|---|---|
| DB-001 | P0 | Convert logical schema into Drizzle migrations | Fresh database migrates from zero and schema matches definitions | DB-001 |
| DB-002 | P0 | Add tenant foreign keys, uniqueness and lifecycle constraints | Invalid cross-tenant relationships are rejected | TEN-001, DB-002 |
| DB-003 | P0 | Implement PostgreSQL RLS policies and transaction-scoped tenant context | Cross-tenant reads/writes are denied, including missing tenant context | TEN-002, TEN-003 |
| DB-004 | P0 | Implement secret-reference metadata only | No raw provider/MCP secret is stored in ordinary tables | SEC-008 |
| DB-005 | P1 | Add indexes and query plans for tenant-scoped access | Critical queries use intended indexes at representative scale | DB-003 |
| DB-006 | P1 | Add migration backup, restore and rollback procedure | Restore drill is documented and successful | OPS-002 |

### DOC-04 — Integration architecture ([INTEGRATION_ARCHITECTURE.md](INTEGRATION_ARCHITECTURE.md))

| Task ID | Priority | Task | Acceptance evidence | Test IDs |
|---|---|---|---|---|
| INT-001 | P0 | Implement MCP connection lifecycle and transport adapter | Valid server connects; invalid/unreachable server fails safely | MCP-001 |
| INT-002 | P0 | Implement secure endpoint validation and outbound egress policy | Private, loopback, link-local and metadata destinations are blocked | SEC-002, SEC-003 |
| INT-003 | P0 | Implement tool discovery and schema normalization | Tool metadata/schema is versioned and normalized | MCP-002 |
| INT-004 | P0 | Default discovered tools to disabled and require review | Discovery never grants execution permission | MCP-003 |
| INT-005 | P0 | Implement Tool Execution Broker with server-side policy | All tool calls pass authorization and policy evaluation | MCP-004, SEC-006 |
| INT-006 | P1 | Implement user confirmation and approval lifecycle for risky actions | Unapproved write/destructive actions never execute | MCP-005 |
| INT-007 | P1 | Implement connection health, timeout, retry and audit | Failures are bounded, visible and correlated | MCP-006 |

### DOC-05 — Identity and authorization ([IDENTITY_AND_AUTHORIZATION.md](IDENTITY_AND_AUTHORIZATION.md))

| Task ID | Priority | Task | Acceptance evidence | Test IDs |
|---|---|---|---|---|
| IAM-001 | P0 | Implement platform user, organization and workspace membership authorization | Roles are enforced server-side on every management endpoint | IAM-001, IAM-002 |
| IAM-002 | P0 | Implement anonymous visitor identity with least privilege | Guest identity cannot assert authenticated claims or access private tools | IAM-003 |
| IAM-003 | P0 | Implement signed customer identity handoff with replay protection | Valid short-lived token verifies; tampered, expired and replayed tokens fail | IAM-004 |
| IAM-004 | P0 | Bind chat sessions to verified workspace and actor | Client-supplied tenant/user/role values are ignored | IAM-005 |
| IAM-005 | P0 | Re-authorize protected tool calls at execution time | Revoked access takes effect without trusting stale model context | IAM-006 |
| IAM-006 | P1 | Implement audit records for authentication and authorization decisions | Decisions are attributable without exposing sensitive claims | SEC-010 |

### DOC-06 — Security and threat model ([SECURITY_AND_THREAT_MODEL.md](SECURITY_AND_THREAT_MODEL.md))

| Task ID | Priority | Task | Acceptance evidence | Test IDs |
|---|---|---|---|---|
| SEC-001 | P0 | Enforce widget origin allowlist and safe browser messaging | Untrusted origins cannot initialize privileged sessions | SEC-001 |
| SEC-002 | P0 | Implement SSRF-safe outbound HTTP and DNS/IP validation | Private/reserved addresses and redirects to them are denied | SEC-002, SEC-003 |
| SEC-003 | P0 | Implement rate limits, payload limits and abuse controls | Limits apply per workspace, origin and actor as appropriate | SEC-004 |
| SEC-004 | P0 | Implement secret vault references, encryption and redaction | Secrets are not returned to clients or emitted in logs | SEC-008, SEC-009 |
| SEC-005 | P0 | Add prompt-injection and untrusted-content boundaries | Retrieved/tool content cannot override system policy or authorization | SEC-005 |
| SEC-006 | P0 | Add tool risk classification and deny-by-default policy | Read/write/destructive tools require their appropriate policy/approval | SEC-006, MCP-005 |
| SEC-007 | P1 | Add dependency, container and secret scanning to CI | Critical findings block release or have documented exception | OPS-003 |
| SEC-008 | P1 | Add incident response, backup and restore runbooks | Restore and credential-revocation drills pass | OPS-002, OPS-004 |

### DOC-07 — Technology stack and project structure ([TECH_STACK_AND_PROJECT_STRUCTURE.md](TECH_STACK_AND_PROJECT_STRUCTURE.md))

| Task ID | Priority | Task | Acceptance evidence | Test IDs |
|---|---|---|---|---|
| TECH-001 | P0 | Create pnpm/Turborepo workspace and package boundaries | Clean install, lint, typecheck and build work in CI | ARC-001, CI-001 |
| TECH-002 | P0 | Add environment validation and safe configuration loading | Missing/invalid required config fails at startup with safe errors | CFG-001 |
| TECH-003 | P0 | Add PostgreSQL/pgvector development environment and migrations | New developer can start DB and migrate from documented steps | DB-001 |
| TECH-004 | P0 | Add unit/integration/e2e test harness and fake provider | Tests run without live provider keys | CI-001, MOD-001 |
| TECH-005 | P1 | Add worker, queue, object storage and observability adapters | Adapters have contract tests and health reporting | JOB-001, OBS-001 |
| TECH-006 | P1 | Add CI security and release checks | Required checks are branch-protected and reproducible | OPS-003 |

### DOC-08 — Implementation roadmap ([IMPLEMENTATION_ROADMAP.md](IMPLEMENTATION_ROADMAP.md))

| Task ID | Priority | Task | Acceptance evidence | Test IDs |
|---|---|---|---|---|
| ROAD-001 | P0 | Create M0–M8 milestone issues and dependency links | Each milestone has ordered, independently verifiable issues | CI-001 |
| ROAD-002 | P0 | Enforce milestone exit gates | Milestone cannot close with failing required tests or open P0 blockers | CI-002 |
| ROAD-003 | P1 | Publish test and delivery reports per milestone | Report includes test totals, failures, skips and linked defects | OBS-001 |

## 6. Definition of Ready

An issue may move from Backlog to Ready only when:
- Requirement and scope are clear.
- Acceptance criteria are testable.
- Dependencies are linked and available.
- Relevant test IDs exist or a test-design task is linked.
- Security, privacy and tenant-isolation impact is considered.
- The issue is small enough to review and verify independently.

## 7. Definition of Done

An issue may move to Done only when:
- Code and documentation are merged through a reviewed pull request.
- Acceptance criteria are met.
- Required unit, integration, security and/or end-to-end tests pass.
- No unresolved P0/P1 defect remains in the issue's scope.
- Migrations, observability, error handling and rollback notes are addressed where applicable.
- Test evidence and deployed environment (if applicable) are recorded.


## 9. Created GitHub issues — M0

The first five M0 implementation issues have been created. They are open and therefore currently in **Backlog**. Update their status through issue activity and the GitHub Project when available.

| Issue | Task | Tests | Status |
|---|---|---|---|
| [#1 Initialize pnpm/Turborepo monorepo](https://github.com/the-tech-makers/chat-agent/issues/1) | M0.1 / TECH-001, TECH-002 | CI-001, CFG-001, ARC-001 | Backlog |
| [#2 Create web, widget and worker shells](https://github.com/the-tech-makers/chat-agent/issues/2) | M0.2 / TECH-001, PROD-004 | CI-001, WID-001 | Backlog |
| [#3 Add PostgreSQL + pgvector dev environment](https://github.com/the-tech-makers/chat-agent/issues/3) | M0.3 / TECH-003 | DB-001, CI-001 | Backlog |
| [#4 Add CI and deterministic test harness](https://github.com/the-tech-makers/chat-agent/issues/4) | M0.3 / TECH-004, TECH-006, ROAD-002 | CI-001, CI-002, MOD-001 | Backlog |
| [#5 Add health checks and setup documentation](https://github.com/the-tech-makers/chat-agent/issues/5) | M0.2–M0.3 / TECH-002, ROAD-001 | CFG-001, CI-001 | Backlog |

The GitHub Project board itself has not been created in this pass because the connected GitHub actions available here support repository issues but do not expose GitHub Projects creation/configuration. The issues are live and can be added to a Project board later.

## 10. Reporting

At each milestone review, report:
- Tasks: Done / In Progress / Ready / Backlog / Blocked.
- Tests: Passed / Failed / Skipped / Not run.
- Open defects by severity.
- Requirements with no implementation task or no test mapping.
- Security exceptions and accepted risks.
- Milestone exit criteria and decision.

A task marked Done with missing test evidence is a process defect; correct the status rather than treating the missing evidence as optional.


## 11. Additional detailed specification coverage

The following documents refine the original scope and architecture. Their implementation work is allocated to the milestones below; these rows are requirement traceability, not claims of completed code.

| Document | Task ID | Priority | Implementation task | Acceptance evidence | Test coverage |
|---|---|---:|---|---|---|
| [API specification](API_SPECIFICATION.md) | API-001 | P0 | Implement versioned API contracts, auth scopes, error envelopes and pagination | Contract tests cover every public route class and cross-tenant denial | API-001, AUTH-001, TEN-001 |
| [Agent runtime specification](AGENT_RUNTIME_SPECIFICATION.md) | AGT-001 | P0 | Implement durable request lifecycle, context assembly, bounded orchestration and terminal states | Failure injection proves exactly one terminal state and no unauthorized context | AGT-001, AGT-002, TEN-001 |
| [Model gateway specification](MODEL_GATEWAY_SPECIFICATION.md) | MOD-002 | P0 | Implement provider adapter contract, deterministic route resolution and credential boundaries | BYOK failure tests prove no silent provider/key/model switch | MOD-002, MOD-003, SEC-002 |
| [Widget and SDK specification](WIDGET_AND_SDK_SPECIFICATION.md) | WID-002 | P1 | Implement iframe bootstrap, origin checks, session exchange and versioned bridge | Browser tests reject wrong origin, expired token and privileged message | WID-002, AUTH-002 |
| [Knowledge and retrieval specification](KNW-001) | KNW-001 | P1 | Implement versioned ingestion, ACL-filtered retrieval, citations and deletion | Retrieval and deletion tests cover tenant/source ACL boundaries | KNW-001, TEN-002 |
| [Admin and workspace operations](ADM-001) | ADM-001 | P1 | Implement permission catalog, owner safeguards, support access and workspace lifecycle | Authorization matrix and audit tests pass | ADM-001, SEC-003 |
| [Deployment and operations runbook](OPS-004) | OPS-004 | P1 | Implement release, backup/restore, incident and operational readiness procedures | Restore drill and rollback evidence recorded | OPS-004, OBS-001 |
| [Configuration reference](CFG-002) | CFG-002 | P0 | Implement typed per-process configuration schemas and secret references | Startup validation and redaction tests pass | CFG-002, SEC-001 |
| [Data lifecycle and privacy](PRV-001) | PRV-001 | P1 | Implement retention, export, deletion tombstones and workspace offboarding | Deletion verification covers primary, derived and backup replay paths | PRV-001, SEC-004 |
| [ADR and glossary](DOC-001) | DOC-001 | P1 | Maintain decision records and consistent domain terminology | Material architecture changes include reviewed ADR and doc updates | DOC-001 |

### Milestone ownership

- M1: API auth conventions, configuration validation, tenant boundaries and core permission schema.
- M2: model gateway and credential-source enforcement.
- M3–M4: runtime lifecycle, streaming contract, widget protocol and identity exchange.
- M5: tool broker contracts and integration authorization.
- M6: knowledge ingestion, retrieval ACLs and deletion.
- M7: admin operations, privacy/offboarding, production runbooks and operational evidence.

Stable test IDs above are planning identifiers; add corresponding cases to `tests/cases/catalog.json` before implementation begins. Do not mark these tasks Done until the contract and test evidence are implemented and reviewed.


## 12. Contract audit and schema closure tasks

| Document | Task ID | Priority | Required work | Acceptance evidence | Test IDs |
|---|---|---:|---|---|---|
| [Contract alignment audit](CONTRACT_ALIGNMENT_AUDIT.md) | AUD-001 | P0 | Resolve or explicitly defer every AUD-001–AUD-016 gap | Each gap has a schema/API/test implementation or approved deferral | AUD-001 |
| [Database design addendum](DATABASE_DESIGN_ADDENDUM.md) | DB-007 | P0 | Add durable entities for sessions, approvals, idempotency, budgets, versions and deletion | Migrations, RLS, constraints and lifecycle tests cover all new entities | API-002, WID-002, KNOW-005, PRV-001 |
| [OpenAPI 3.1 contract](openapi/openapi.yaml) | API-002 | P0 | Keep machine-readable route and schema contract synchronized with API prose | OpenAPI validation and contract tests pass; no undocumented breaking drift | API-002, API-003, API-004 |
| [Expanded test catalogue](../tests/cases/catalog.json) | QA-002 | P0 | Implement new cases for runtime, gateway, widget, retrieval, admin and privacy | Every case has an automation/manual target and evidence before status changes from planned | AGT-004, MOD-008, WID-002, KNOW-003, PRV-001 |

### New test catalogue coverage

The catalogue now contains 72 planned cases. Added IDs: API-002–API-004, AGT-004–AGT-005, MOD-007–MOD-009, WID-002–WID-003, KNOW-003–KNOW-005, ADM-001–ADM-002, PRV-001–PRV-002, OPS-005, CFG-002 and AUD-001.

These entries are planned test specifications, not implemented automated tests. The coding agent must add executable tests and attach run evidence as each milestone is completed.


## 8. DOC-25 — Production policies and service-level specification

Source: [Production Policies and Service-Level Specification](PRODUCTION_POLICIES_AND_SLOS.md). All values marked TBD or proposed remain unapproved until recorded in a policy decision/ADR.

| Task ID | Priority | Task | Acceptance evidence | Test IDs |
|---|---|---|---|---|
| POL-001 | P0 before production | Implement per-data-class retention and deletion policy | Expiry worker, deletion status, tombstone replay and restore tests pass | PRIV-001, PRIV-002, PRIV-003 |
| POL-002 | P0 before uploads | Enforce upload type/size/count/extraction/concurrency limits | Malicious, oversized, MIME mismatch and parser-abuse tests pass | KNOW-006, SEC-011 |
| POL-003 | P0 before public widget | Enforce widget session expiry, refresh, origin binding and revocation | Expired/replayed/cross-origin tokens are rejected; reauthorization is required | WID-004, SEC-012 |
| POL-004 | P0 before public beta | Enforce shared API limits, concurrency and usage reservations | Atomic limits, Retry-After, budget settlement and failure release are verified | API-008, MOD-009, OPS-003 |
| POL-005 | P0 before SLA claims | Define service SLO/SLA and support model | Public claims match measured service and approved contract; otherwise preview notice is explicit | OPS-004 |
| POL-006 | P0 before production | Approve backup retention, RPO/RTO and recovery procedures | Restore drill records achieved RPO/RTO and validates tombstones/tenant isolation | OPS-005, PRIV-004 |
| POL-007 | P0 before production | Approve residency and transfer policy | Data-flow inventory covers storage, backups, telemetry, support and provider processing | PRIV-005 |
| POL-008 | P0 before enabling provider | Complete provider due diligence and data policy approval | Provider has reviewed terms, DPA, retention/training settings and approved data classes | MOD-010, PRIV-006 |
| POL-009 | P0 before paid launch | Define plans, entitlements, price book and usage metering | Immutable usage ledger and historical price versioning reconcile correctly | BILL-001, BILL-002 |
| POL-010 | P0 before paid launch | Define overage, hard-stop and suspension behavior | Exhaustion blocks billable work without surprise spend; overrides are audited | BILL-003, MOD-011 |

### Cross-document policy invariants

- Proposed numeric values are not implementation constants. Resolve them through versioned, validated configuration.
- Platform safety ceilings override plan and workspace settings; workspace owners may tighten but not weaken them.
- A BYOK failure never triggers implicit platform billing or credential/provider substitution.
- Backup restoration must replay deletion tombstones before tenant data is exposed.
- A widget session expiring does not itself authorize continued access to a conversation.
- Public SLA, residency, compliance, retention, and pricing claims require explicit approval and supporting evidence.
