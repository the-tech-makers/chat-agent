# Implementation Roadmap

## 1. Purpose

This roadmap converts the approved product and architecture documents into an implementation sequence that can be tracked as GitHub issues or project-board items.

The roadmap prioritizes a secure, usable MVP for the project owner first, while keeping the architecture suitable for later multi-customer SaaS release.

Related documents:
- [Product Scope](PRODUCT_SCOPE.md)
- [System Architecture](ARCHITECTURE.md)
- [Database Design](DATABASE_DESIGN.md)
- [Integration Architecture](INTEGRATION_ARCHITECTURE.md)
- [Identity and Authorization](IDENTITY_AND_AUTHORIZATION.md)
- [Security Architecture and Threat Model](SECURITY_AND_THREAT_MODEL.md)
- [Technology Stack and Project Structure](TECH_STACK_AND_PROJECT_STRUCTURE.md)

## 2. Delivery principles

1. Build a working vertical slice early; avoid implementing every abstraction before the first end-to-end chat works.
2. Preserve tenant boundaries and authorization from the first migration. Do not defer RLS or identity design until after features exist.
3. Use hosted model APIs only. Tests and local development must work with fake providers and must not require a live inference key.
4. MCP tools are disabled until explicitly reviewed and enabled. Every execution passes through the Tool Execution Broker.
5. BYOK routing is deterministic. A BYOK failure is visible to the workspace; no implicit credential or provider substitution.
6. Keep the first release a modular monolith. Split services only when operational evidence justifies it.
7. Treat free tiers as development allowances, not production guarantees. Set spend limits before external beta.
8. Each milestone ends with demonstrable acceptance criteria and automated tests.

## 3. Priority definitions

- **P0 — MVP blocker:** Required for a safe, working MVP.
- **P1 — MVP important:** Needed for a complete and operable MVP, but can follow the first vertical slice.
- **P2 — Post-MVP:** Valuable after the core product is stable.
- **P3 — Later scale:** Only when usage or customer demand justifies it.

## 4. Milestone overview

| Milestone | Goal | Priority | Exit condition |
|---|---|---|---|
| M0 — Repository foundation | Reproducible monorepo and developer workflow | P0 | Clean checkout installs, builds, tests, and starts |
| M1 — Database and workspace identity | Tenant-safe persistence and SaaS identity | P0 | Workspace membership and RLS tests pass |
| M2 — Model Gateway | Hosted inference, BYOK and routing policy | P0 | Fake and live adapter contract tests pass; routing rules enforced |
| M3 — Agent runtime and chat widget | End-to-end streaming chat | P0 | Visitor can complete a bounded chat session |
| M4 — Customer identity handoff | Secure authenticated end-user context | P0 | Customer backend can exchange a signed assertion for a scoped session |
| M5 — MCP integrations and tool broker | Safe customer-system actions | P1 | Reviewed tool executes only after authorization and approval checks |
| M6 — Knowledge and retrieval | Upload, ingest, retrieve and delete knowledge | P1 | Tenant- and ACL-filtered retrieval works end to end |
| M7 — Operations and beta hardening | Usage controls, monitoring and release readiness | P0 before public beta | Security, recovery, abuse and spend controls verified |
| M8 — Expansion | More connectors, APIs and scale capabilities | P2/P3 | Driven by measured demand |

## 5. M0 — Repository foundation

### M0.1 Initialize the monorepo
**Priority:** P0

Tasks:
- [ ] Pin supported Node.js and pnpm versions.
- [ ] Add root package.json, pnpm-workspace.yaml, and turbo.json.
- [ ] Configure TypeScript base settings and path conventions.
- [ ] Configure formatting, linting, and commit hygiene.
- [ ] Add .gitignore and placeholder-only .env.example.
- [ ] Add a concise contributor/developer setup section to README.

Acceptance criteria:
- A clean clone can install dependencies using the pinned package manager.
- Workspace package discovery succeeds.
- Lint, typecheck, and build commands exist and run.
- No secrets or local environment files are committed.

### M0.2 Create application shells
**Priority:** P0

Tasks:
- [ ] Create apps/web using Next.js App Router.
- [ ] Create apps/widget with the chosen isolated iframe build approach.
- [ ] Create apps/worker as a separately runnable process.
- [ ] Add minimal health/readiness endpoints where applicable.
- [ ] Add shared UI and API-contract package shells only when first needed.

Acceptance criteria:
- Web app starts and serves a placeholder page.
- Widget can be built independently and rendered in a local test host page.
- Worker starts, reports readiness, and shuts down cleanly.
- Apps import shared packages through workspace dependencies.

### M0.3 Local infrastructure and CI
**Priority:** P0

Tasks:
- [ ] Add Docker Compose for PostgreSQL with pgvector.
- [ ] Add optional Redis and S3-compatible local storage profiles only if needed.
- [ ] Add migration and database reset scripts.
- [ ] Add GitHub Actions for install, lint, typecheck, tests, and build.
- [ ] Add dependency and secret scanning suitable for the repository plan.

Acceptance criteria:
- A developer can start required local infrastructure with documented commands.
- CI passes on a clean checkout.
- Routine CI uses mocks and does not require live provider credentials.

## 6. M1 — Database and workspace identity

### M1.1 Database baseline
**Priority:** P0

Tasks:
- [ ] Add Drizzle ORM and migration tooling.
- [ ] Implement users, organizations, workspaces, memberships, roles, and invitations.
- [ ] Implement agents, installations, conversations, messages, and chat sessions at minimum viable scope.
- [ ] Add workspace ownership constraints and indexes.
- [ ] Add a migration identity separate from the application runtime identity.

Acceptance criteria:
- Migrations create the schema from an empty database.
- Foreign keys and unique constraints prevent invalid workspace relationships.
- Repository methods require explicit workspace scope for tenant-owned records.

### M1.2 Tenant isolation and RLS
**Priority:** P0 — security gate

Tasks:
- [ ] Enable RLS on tenant-owned tables.
- [ ] Create non-owner runtime database role without BYPASSRLS.
- [ ] Establish transaction-scoped workspace context.
- [ ] Add policies for workspace members and explicitly scoped service operations.
- [ ] Add negative tests for cross-workspace reads and writes.

Acceptance criteria:
- Tenant isolation tests run using the same privilege level as the application.
- Missing tenant context denies access rather than broadening access.
- Connection pooling cannot leak tenant context between requests.

### M1.3 SaaS dashboard authentication
**Priority:** P0

Tasks:
- [ ] Select and configure the authentication provider/library.
- [ ] Implement sign-in, sign-out, session renewal, and account linking policy.
- [ ] Implement protected dashboard routes.
- [ ] Implement workspace creation and membership invitations.
- [ ] Add platform-admin authorization as a separate privilege boundary.

Acceptance criteria:
- Unauthenticated users cannot access workspace dashboard data.
- Workspace roles are checked server-side for every protected operation.
- Platform-admin access cannot be gained by editing client-side profile data.

## 7. M2 — Model Gateway

### M2.1 Provider-neutral contracts
**Priority:** P0

Tasks:
- [ ] Define normalized generation, streaming, tool-call, usage, and error contracts.
- [ ] Define provider capability metadata.
- [ ] Define a fake provider for deterministic unit and integration tests.
- [ ] Add timeout, cancellation, token, and output limits.

Acceptance criteria:
- Runtime depends on internal contracts, not a provider SDK.
- Provider errors map to stable internal error categories.
- Fake provider supports streaming and tool-call test scenarios.

### M2.2 OpenRouter adapter
**Priority:** P0

Tasks:
- [ ] Implement OpenRouter generation and streaming adapter.
- [ ] Normalize tool calls and usage metadata.
- [ ] Handle provider rate limits, timeouts, unavailable models, and malformed responses.
- [ ] Add opt-in live smoke tests with strict spend limits.

Acceptance criteria:
- Adapter passes the shared provider contract suite.
- API keys never appear in logs, traces, errors, or browser bundles.
- Provider-specific fields do not leak into runtime domain types.

### M2.3 Credential ownership and routing
**Priority:** P0 — product invariant

Tasks:
- [ ] Implement credential references and encrypted secret storage integration.
- [ ] Implement platform-managed credential records.
- [ ] Implement workspace BYOK credential bindings by provider/model.
- [ ] Implement workspace model preferences and administrator overrides.
- [ ] Implement deterministic credential resolution and auditable routing decisions.
- [ ] Implement explicit administrator-configured fallback routes, disabled by default.
- [ ] Surface BYOK failures without silent key/provider/model substitution.

Acceptance criteria:
- Tests prove that the selected credential matches the configured owner and binding.
- BYOK failure returns a clear safe error unless an explicit eligible fallback is configured.
- Fallback never occurs merely because another key is available.
- Secret plaintext is not persisted in application tables or emitted to telemetry.

### M2.4 Usage and budgets
**Priority:** P0 before external traffic

Tasks:
- [ ] Record model, provider, credential owner, token usage, and estimated cost where available.
- [ ] Add workspace and platform usage limits.
- [ ] Add configurable warning thresholds and hard stops.
- [ ] Add administrator kill switch for a provider/model or workspace.

Acceptance criteria:
- Usage is attributable to the correct workspace and credential owner.
- Requests exceeding hard limits are rejected before provider invocation.
- Cost estimates are labeled as estimates when provider pricing/usage is incomplete.

## 8. M3 — Agent runtime and chat widget

### M3.1 Conversation service
**Priority:** P0

Tasks:
- [ ] Implement conversation and message repositories.
- [ ] Implement session ownership and expiration.
- [ ] Implement bounded conversation history selection.
- [ ] Add retention and deletion primitives.

Acceptance criteria:
- A session can access only its authorized conversation.
- Concurrent message handling has defined ordering/idempotency behavior.
- Deletion removes or schedules removal of associated content according to retention policy.

### M3.2 Bounded agent runtime
**Priority:** P0

Tasks:
- [ ] Implement the agent turn lifecycle.
- [ ] Implement trusted system instruction and context assembly.
- [ ] Enforce step, token, wall-clock, and tool-call limits.
- [ ] Treat retrieved documents and tool output as untrusted data.
- [ ] Stream typed events to the API layer.
- [ ] Add cancellation and safe failure behavior.

Acceptance criteria:
- A turn completes with a fake provider.
- Runaway tool/model loops are bounded.
- Prompt and tool output cannot alter authorization policy.

### M3.3 Embeddable widget
**Priority:** P0

Tasks:
- [ ] Build the iframe-based chat interface.
- [ ] Implement installation bootstrap and anonymous session creation.
- [ ] Implement message streaming, retry, cancel, and reconnect behavior.
- [ ] Add safe rendering for markdown and citations.
- [ ] Add origin allowlisting and content security policy.
- [ ] Add basic accessibility and responsive behavior.

Acceptance criteria:
- A sample customer page can embed the widget using documented instructions.
- Public installation identifiers grant no privileged access.
- The widget cannot read or expose server-side credentials.
- Invalid origins and expired sessions are rejected.

## 9. M4 — Customer identity handoff

### M4.1 Signed identity exchange
**Priority:** P0 for authenticated application users

Tasks:
- [ ] Define a documented server-to-server identity assertion contract.
- [ ] Validate issuer, audience, signature, expiry, issued-at, nonce, and replay policy.
- [ ] Bind assertions to the correct workspace and installation.
- [ ] Map customer subject IDs to opaque internal actor references.
- [ ] Issue short-lived, scoped chat sessions.
- [ ] Implement revocation and session invalidation.

Acceptance criteria:
- The browser cannot mint or elevate a customer identity.
- Expired, replayed, wrong-audience, and wrong-workspace assertions are rejected.
- Tool authorization uses verified actor context and current policy, not model claims.
- Anonymous visitors remain restricted to public capabilities.

### M4.2 Customer authorization strategies
**Priority:** P0 before private tools

Tasks:
- [ ] Define integration authorization modes: service-level, delegated user token, and customer authorization callback where applicable.
- [ ] Define policy behavior when user authorization cannot be verified.
- [ ] Re-check authorization immediately before protected tool execution.
- [ ] Add audit records for allow/deny decisions without recording sensitive payloads.

Acceptance criteria:
- Login alone does not grant access to customer private data.
- Protected actions fail closed if customer authorization cannot be established.
- Authorization is evaluated for the current actor, resource, and action.

## 10. M5 — MCP integrations and Tool Execution Broker

### M5.1 MCP connection lifecycle
**Priority:** P1

Tasks:
- [ ] Implement remote Streamable HTTP connection support.
- [ ] Implement connection authentication and secure credential references.
- [ ] Discover and normalize tools and JSON schemas.
- [ ] Store tool metadata, schema version, and review status.
- [ ] Add health checks, timeouts, and connection lifecycle management.

Acceptance criteria:
- A test MCP server can be connected and its tools discovered.
- Tool discovery does not automatically enable execution.
- Connection secrets are encrypted and redacted.

### M5.2 Tool Execution Broker
**Priority:** P0 security gate before enabling tools

Tasks:
- [ ] Require an explicit allowlist and enabled state for each tool.
- [ ] Validate model-proposed arguments against the approved schema.
- [ ] Resolve verified actor and workspace context.
- [ ] Enforce workspace policy and customer authorization.
- [ ] Require human confirmation for configured high-risk actions.
- [ ] Enforce SSRF-safe egress, timeouts, response-size limits, and output sanitization.
- [ ] Add idempotency and duplicate-action protections where supported.
- [ ] Record immutable audit events for tool decisions and executions.

Acceptance criteria:
- No model response can directly invoke a connector.
- Disabled, unreviewed, unauthorized, or unapproved tools cannot execute.
- Tool results are treated as untrusted content by the agent runtime.
- Security tests cover private-network access and redirect/DNS edge cases.

## 11. M6 — Knowledge and retrieval

### M6.1 Source and file management
**Priority:** P1

Tasks:
- [ ] Implement source records and private object storage.
- [ ] Add upload validation, file size limits, and content-type checks.
- [ ] Add malware scanning integration point and safe quarantine state.
- [ ] Implement source status and deletion lifecycle.

Acceptance criteria:
- Uploaded files are private and workspace-scoped.
- Untrusted files are not parsed in the web request process.
- Deletion is traceable and removes source access immediately.

### M6.2 Ingestion worker
**Priority:** P1

Tasks:
- [ ] Add durable job processing, retries, idempotency, and dead-letter state.
- [ ] Implement isolated parsing with resource limits.
- [ ] Implement text normalization and chunking.
- [ ] Implement hosted embedding provider adapter.
- [ ] Persist chunks and vectors with workspace and ACL metadata.

Acceptance criteria:
- Reprocessing a job does not duplicate active chunks.
- Oversized, malformed, or hostile files fail safely.
- Worker reloads and validates source state rather than trusting queue payloads.

### M6.3 Retrieval and deletion
**Priority:** P1

Tasks:
- [ ] Implement vector and keyword retrieval.
- [ ] Apply workspace, source, and actor ACL filters before context assembly.
- [ ] Add citations linking answers to permitted source passages.
- [ ] Implement deletion of originals, chunks, vectors, caches, and derived records.

Acceptance criteria:
- Cross-workspace and unauthorized source passages never enter model context.
- Retrieval results include stable source references.
- Deletion tests verify all derived content is removed or rendered inaccessible.

## 12. M7 — Operations and beta hardening

**Priority:** P0 before public beta

Tasks:
- [ ] Add structured logs, traces, metrics, and correlation IDs.
- [ ] Redact secrets, personal data, and sensitive tool arguments from telemetry.
- [ ] Add rate limits by workspace, installation, actor, and IP risk signals.
- [ ] Add quotas for messages, tokens, tools, files, and concurrent runs.
- [ ] Add abuse detection and provider/model kill switches.
- [ ] Add backup, restore, retention, and tenant deletion runbooks.
- [ ] Add incident response and credential rotation procedures.
- [ ] Add security regression tests for RLS, IDOR, SSRF, prompt injection, and secret leakage.
- [ ] Conduct a pre-beta threat-model review and independent security assessment appropriate to the exposure.

Acceptance criteria:
- Operators can identify a failing workspace/provider without exposing secret material.
- Rate and spend limits are enforced server-side.
- A backup restore is demonstrated in a non-production environment.
- Tenant deletion has a documented, testable lifecycle.
- No unresolved critical/high severity security issue is accepted without an explicit risk owner and mitigation.

## 13. M8 — Post-MVP expansion

Prioritize these based on actual customer demand:

- [ ] OpenAPI connector and customer-specific API adapters.
- [ ] Additional direct hosted model providers, including Gemini where justified.
- [ ] OAuth 2.1 connection flows and delegated user credentials for more MCP servers.
- [ ] Advanced knowledge sources: URLs, cloud drives, repositories, and scheduled sync.
- [ ] Conversation analytics and workspace-level evaluation.
- [ ] Agent configuration/versioning and controlled rollout.
- [ ] More granular policy engine and approval workflows.
- [ ] Additional languages and widget customization.
- [ ] Enterprise SSO, SCIM, and advanced audit export.
- [ ] Dedicated deployment/network isolation options for larger customers.

## 14. Dependency map

The critical dependency sequence is:

~~~text
M0 Repository foundation
  |
  v
M1 Database, RLS, workspace identity
  |
  +--------------------+
  v                    v
M2 Model Gateway       M4 Identity handoff design
  |                    |
  +---------+----------+
            v
      M3 Agent runtime and widget
            |
            v
      M4 Authenticated customer context
            |
            v
      M5 MCP broker and tools
            |
            v
      M6 Knowledge and retrieval
            |
            v
      M7 Beta hardening
            |
            v
      M8 Expansion
~~~

Some work can proceed in parallel after contracts are agreed. For example, widget UI work can use mocked chat APIs while the Model Gateway is being implemented. Private customer tools must not be enabled until identity handoff and authorization checks are complete.

## 15. Suggested GitHub issue labels

Create these labels in the repository:

- priority:P0, priority:P1, priority:P2, priority:P3
- type:feature, type:security, type:infrastructure, type:documentation, type:bug
- area:web, area:widget, area:worker, area:database, area:agent-runtime, area:model-gateway, area:integrations, area:auth
- status:blocked, status:needs-design, status:ready

Issue templates should request:
- Problem and expected outcome.
- Scope and explicit non-goals.
- Dependencies.
- Security and tenant-isolation impact.
- Acceptance criteria.
- Test plan.
- Operational/observability impact.

## 16. Definition of done

A task is complete only when:
- The acceptance criteria are met.
- Tests cover success, failure, and relevant authorization boundaries.
- Tenant scope is explicit for tenant-owned data.
- Secrets and sensitive payloads are not exposed in logs or client responses.
- Documentation and environment examples are updated.
- Migrations are safe and reviewed.
- CI passes.
- Any known limitation or operational requirement is documented.

## 17. First implementation batch

Start with this small, reviewable batch rather than generating the whole application at once:

1. [ ] Pin Node.js and pnpm versions and initialize the workspace.
2. [ ] Add TypeScript, lint, formatting, and test configuration.
3. [ ] Create the Next.js web shell.
4. [ ] Add PostgreSQL/pgvector Docker Compose configuration.
5. [ ] Add a minimal CI workflow.
6. [ ] Add a health endpoint and local setup instructions.

Do not implement provider keys, live MCP execution, or customer private-data access in this first batch. Establish the build and test foundation first.
