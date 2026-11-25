# Product Scope & MVP

## Product vision
Chat Agent is a cloud SaaS that lets any website or application connect its context, tools, knowledge, and data sources to a universal AI agent. The SaaS performs onboarding and discovery, generates a governed agent configuration, and provides a chat experience for visitors and authenticated application users.

**Product principle:** one shared, tenant-aware agent runtime; customer-specific behavior is configuration, context, integration capabilities, and permissions—not bespoke code.

## Target users
- SaaS/application owners who want an agent embedded in their product.
- Website visitors who need public information or assistance.
- Authenticated application users who need permission-aware, account-specific assistance.

## MVP goals
1. Organizations and workspaces.
2. Connect one MCP server and validate the connection.
3. Discover and normalize MCP tools.
4. Review, enable/disable and permission tools.
5. Upload and retrieve knowledge documents.
6. Configure, test and publish an agent.
7. Embeddable chat widget.
8. Verified authenticated identity handoff.
9. Policy-controlled tool execution.
10. Conversations, usage and logs.

## Included
- Tenant management: organizations, workspaces, members and roles.
- One universal agent per workspace, with configuration versioning and rollback.
- Chat sessions, messages, streaming, history and feedback.
- MCP connector: connection validation, discovery, schema normalization and health checks.
- Workspace tool registry, risk classification and permissions.
- Server-side policy enforcement; read-only defaults; explicit controls/approval for writes.
- Knowledge upload, extraction, chunking, embeddings, retrieval, source management.
- Context generation, review and publishing.
- iframe widget, branding, origin allowlist, guest sessions and responsive UI.
- Authenticated handoff with short-lived signed tokens and authorization at execution.
- Usage, audit, rate limits, traces, analytics and integration health.
- Basic workspace-owned regression and security tests.

## Out of scope
- Broad native integration catalog.
- General-purpose support for all database vendors/types (consider a safe read-only adapter later).
- Arbitrary/unrestricted SQL or autonomous database writes.
- Multi-agent orchestration and complex workflows.
- Scheduled autonomous workflows and event-triggered proactive agents.
- Fine tuning/custom model training.
- Enterprise SSO/SCIM, dedicated deployments and custom SLAs.
- Voice/live video and advanced mobile SDKs.
- Marketplace, complex workflow builder and default long-term memory.
- Full human-agent inbox.

## MVP user journeys
### Customer onboarding
1. Sign up and create organization/workspace.
2. Create agent and configure branding/origins.
3. Connect MCP source and validate.
4. Discover, review and permission tools.
5. Upload and index knowledge.
6. Review context, run tests and publish.
7. Install widget or use chat API.

### Visitor chat
Widget initializes with public identifier; server validates origin and resolves workspace; scoped guest session is created; runtime loads agent configuration, session, knowledge and permitted tools; policy checks every tool call; response streams and usage is logged.

### Authenticated chat
Customer backend issues a short-lived signed identity token; platform validates claims; actor and workspace are mapped; tool-level authorization is enforced; responses and audit events are recorded.

### MCP discovery
Customer configures connection and credentials; connection is validated; tool definitions/schemas are discovered, normalized and risk classified; customer approves tools; configuration and health are recorded.

## Functional requirements
### Tenant and identity
- Scope all tenant-owned records.
- Resolve tenants server-side, never trust client-supplied tenant IDs.
- Enforce membership and roles in application logic and database policies.
- Guest identity is limited to scoped sessions.
- Authenticated identity is derived from verified claims.

### Runtime
- Load published agent version, context, tools, policies and session state.
- Streaming and bounded tool-call loop.
- Validate schemas; enforce time, steps, calls, output size, tokens, retries and concurrency.
- Persist run/step traces and handle errors without exposing secrets.

### Integration security
- Treat tools, MCP metadata, API results and documents as untrusted.
- SSRF protection and outbound destination/IP validation.
- Encrypted secret storage; secrets never enter model context, browser or logs.
- Tools disabled by default until reviewed; policy checks outside the model.
- Log metadata and outcomes, not sensitive payloads by default.

### Knowledge
- Workspace-scoped source and chunks.
- Extraction, chunking, indexing status and supported formats.
- Permission-filtered retrieval and deletion lifecycle.
- Retrieved content is data, not instructions.

### Widget
- iframe isolation, secure messaging and allowed-origin checks.
- Public widget IDs do not grant privileged access.
- No secrets in widget; basic branding and responsive UI.

## Architecture
Modular monolith initially: dashboard/widget, API/backend, runtime, MCP connection manager and tool registry, policy executor, knowledge ingestion/retrieval worker, PostgreSQL + pgvector, object storage, durable job queue, managed secrets and OpenTelemetry-compatible telemetry.

## Core data entities
Organizations, organization members, workspaces, workspace members, agents, agent versions/policies, integrations, secret references, capabilities, health/sync runs, tools, permissions, execution logs/approvals, end users, sessions, messages, runs/steps, knowledge sources/documents/chunks, usage, API keys, audit logs and (later) webhooks.

## Acceptance criteria
- Supported onboarding path works without engineering intervention.
- Cross-workspace isolation tests fail closed.
- Tool schemas are validated; customers can disable tools.
- No tool executes without server-side authorization.
- Writes/sensitive actions disabled or require explicit configured approval.
- Tool calls bounded by time, count, output and rate limits.
- Widget origin validated; privileged data requires verified identity.
- Tokens validated for signature, expiry, issuer, audience and workspace.
- Knowledge retrieval isolated by workspace.
- Secrets encrypted and never exposed to model/widget/logs.
- Config versioning and rollback; traces and usage events for success/failure.
- Security and regression tests pass before production release.

## Milestones
1. Foundation: repo, CI, auth, tenancy, migrations and isolation.
2. Chat core: sessions, messages, streaming, model abstraction, limits and traces.
3. MCP: lifecycle, discovery, registry, execution and policy.
4. Context & knowledge: ingestion, retrieval, context generation, publishing and test console.
5. Widget & identity: iframe, guest sessions, origins and signed handoff.
6. Hardening & launch: audits, observability, limits, evaluations and runbooks.

## Risks and guardrails
Prompt injection: deterministic policy; tenant leakage: isolation checks and RLS; SSRF: outbound broker and destination validation; sensitive actions: disabled by default; cost/abuse: quotas and metering; context drift: versioning and tests; privacy: retention and deletion lifecycle.

## Definition of done
A customer can connect a supported MCP server, discover and approve capabilities, add knowledge, publish a context-aware agent, embed it on an allowed website, and support guest and verified authenticated conversations with isolation, permissions, limits, auditability and evaluation gates.
