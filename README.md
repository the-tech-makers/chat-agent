# Chat Agent — Product Scope & MVP

## Product vision
Chat Agent is a cloud SaaS that lets any website or application connect its context, tools, knowledge, and data sources to a universal AI agent. The SaaS performs onboarding and discovery, generates a governed agent configuration, and provides a chat experience for visitors and authenticated application users.

**Product principle:** one shared, tenant-aware agent runtime; customer-specific behavior is configuration, context, integration capabilities, and permissions—not bespoke code.

## Target users
- SaaS/application owners who want an agent embedded in their product.
- Website visitors who need public information or assistance.
- Authenticated application users who need permission-aware, account-specific assistance.

## MVP goals
1. A customer can create an organization and workspace.
2. A customer can connect one MCP server and validate the connection.
3. The platform can discover MCP tools and normalize their metadata.
4. Customers can review, enable/disable and permission tools.
5. Customers can add knowledge documents and retrieve relevant passages.
6. Customer can configure/publish an agent using generated context and explicit policy.
7. A visitor can use an embeddable chat widget.
8. An application can hand off a verified authenticated identity using a signed short-lived token.
9. The agent can call approved tools with policy checks and bounded execution.
10. Customer can view conversations, usage and tool execution logs.

## Scope: MVP

### Included
- **SaaS tenancy:** organizations, workspaces, workspace members and roles; tenant-aware authorization.
- **Agent:** one universal agent per workspace; structured configuration, prompt/context settings, versioning and publish/rollback.
- **Chat:** session creation, message persistence, streaming response, conversation history, feedback.
- **MCP integration (first connector):** connect, validate, discover tools, normalized schemas, health checks and connection status. Start with a limited, tested set of supported connection modes.
- **Tool registry:** workspace-scoped tool metadata, schemas, enabled state, risk classification and permissions.
- **Policy enforcement:** independent server-side policy checks; read-only automatic actions by default; approval or explicit controls for writes and sensitive actions.
- **Knowledge:** upload supported text/PDF documents, extraction, chunking, embeddings, retrieval, source management and refresh/delete.
- **Context engine v1:** normalize integration metadata and knowledge; generate editable descriptions and context; version and publish; customer review of suggested mappings.
- **Widget:** iframe-isolated embed, branding, welcome text, streaming, mobile responsive UI, guest sessions and configurable origins.
- **Authenticated handoff:** customer backend issues short-lived signed tokens; verify issuer, audience, signature, expiry and workspace mapping; enforce authorization at tool execution.
- **Operations:** usage metering, audit events, rate limits, run traces, basic analytics and integration health.
- **Evaluation:** small workspace-owned regression suite and security tests before publishing.

### Explicitly out of scope for MVP
- Broad catalog of native integrations.
- Arbitrary database connectivity to every vendor/type. Pilot one safe read-only database adapter only after the MCP flow is reliable.
- Unrestricted SQL execution or autonomous database writes.
- Agent-to-agent orchestration or multiple specialized agents.
- Autonomous long-running workflows, scheduled proactive agents and complex event triggers.
- Fine-tuning/custom model training.
- Advanced enterprise SSO/SCIM, dedicated deployments and custom SLAs.
- Voice, multimodal live video, advanced mobile SDKs.
- Marketplace and third-party agent templates.
- Complex workflow builder.
- Long-term user memory by default; memory is session-scoped for MVP, with retention controls.
- Human-agent inbox beyond a basic handoff/escalation mechanism if included.

## MVP user journeys

### 1. Customer onboarding
1. Sign up and create organization.
2. Create workspace and agent.
3. Configure branding and domain/origin allowlist.
4. Connect MCP source.
5. Test connection and discover capabilities.
6. Review tools, descriptions and risk levels; disable or restrict tools.
7. Add knowledge documents and wait for indexing.
8. Review generated context and run test questions.
9. Publish agent.
10. Install widget or integrate chat API.

### 2. Visitor chat
1. Widget initializes using a public widget identifier.
2. Backend validates origin and resolves workspace.
3. Backend creates/loads a scoped guest session.
4. Visitor sends message.
5. Runtime loads agent version, session history, relevant knowledge and public tools.
6. Model responds or requests tool calls.
7. Policy engine validates each tool request; executor applies limits.
8. Response streams to widget; logs and usage are recorded.

### 3. Authenticated chat
1. Customer application backend issues a short-lived signed identity token.
2. Agent API validates token and claims.
3. Server maps verified identity to workspace and actor.
4. Tools enforce actor-level authorization and tenant/data scopes.
5. Agent response streams; audit records are created.

### 4. Integration discovery
1. Customer configures connection and credentials.
2. Connection manager validates destination, auth and connectivity.
3. Discover tool definitions and input schemas.
4. Normalize metadata, classify risk and validate schemas.
5. Customer reviews and enables approved tools.
6. Store integration/configuration version and health status.

## Functional requirements

### Tenant and identity
- All tenant-owned records are workspace/org scoped.
- Tenant resolution is server-side; never trust a tenant ID supplied by a client.
- Enforce membership/roles in application code and database policy where appropriate.
- Anonymous visitor identity is limited to a signed/scoped guest session.
- Authenticated identity is based on verified token claims—not arbitrary widget payload fields.

### Agent runtime
- Load published agent version, context, tools, policies and conversation state.
- Support streaming responses and bounded tool-calling loop.
- Validate tool inputs and outputs against schemas.
- Enforce limits: time, steps, tool calls, output bytes, tokens, retries and concurrency.
- Persist run status and step trace.
- Handle errors without leaking secrets or internal stack traces.

### Integration/tool security
- Treat MCP server output, tool descriptions, API results and documents as untrusted.
- Protect against SSRF; validate destinations and resolved IPs; block metadata/private-network access unless approved and safely brokered.
- Secrets stored in managed encrypted secret storage; never exposed to the model or browser.
- Default tools to read-only and disabled until reviewed.
- Enforce permission and approval checks outside the model.
- Log execution metadata and outcomes; avoid logging sensitive payloads by default.

### Knowledge
- Workspace-scoped source and chunk records.
- Supported formats clearly documented.
- Extraction, chunking and indexing status.
- Retrieval filtered by workspace and source permissions.
- Deletion removes source, chunks and associated embeddings according to retention policy.
- Retrieved content is data, never trusted as agent instructions.

### Widget
- iframe-based UI with host-page isolation.
- Allowed-origin controls and secure messaging.
- Public widget identifiers do not grant privileged access.
- No secrets or database credentials in widget.
- Basic theming, mobile responsiveness and streaming.

## Architecture (MVP)
- Frontend/dashboard and widget.
- API gateway/backend with authentication, tenant resolution, rate limits and streaming.
- Modular agent runtime and context loader.
- MCP connection manager and normalized tool registry.
- Policy/authorization service and bounded tool executor.
- Knowledge ingestion worker and retrieval module.
- PostgreSQL for SaaS, conversation and metadata; pgvector for embeddings.
- Object storage for documents.
- Queue/durable job runner for ingestion and discovery.
- Managed secret storage.
- OpenTelemetry-compatible traces, logs and metrics.

Start as a modular monolith with workers; extract services only when scaling or isolation needs justify it.

## Core data entities
- organizations, organization_members
- workspaces, workspace_members
- agents, agent_versions, agent_policies
- integrations, integration_credentials (secret references only), integration_capabilities, integration_health, integration_sync_runs
- tools, tool_permissions, tool_execution_logs, tool_approvals
- end_users, sessions, messages, agent_runs, agent_run_steps
- knowledge_sources, documents, document_chunks
- usage_events, subscriptions/plans (billing can be deferred), api_keys
- audit_logs, webhooks (webhooks can be deferred)

Every tenant-owned entity must be scoped and protected by application authorization and appropriate database policies.

## MVP acceptance criteria
- A new customer can onboard without engineering assistance for the supported MCP path.
- Cross-workspace access tests fail closed for sessions, messages, knowledge, tools and logs.
- MCP discovery returns validated schemas and customer can disable tools.
- Agent never executes a tool without server-side authorization.
- Read-only tools are the default; write/sensitive actions are disabled or require explicit configured approval.
- Tool execution is bounded by time, call count, output size and rate limits.
- Widget only operates on approved origins and cannot access privileged data without verified identity.
- Authenticated token validation rejects invalid signatures, expired tokens, wrong audience/issuer and workspace mismatches.
- Knowledge retrieval cannot retrieve content across workspace boundaries.
- Secrets are encrypted and never sent to the model, widget or logs.
- Published configuration is versioned and rollbackable.
- Run traces and usage events exist for successful and failed requests.
- Security and regression evaluation suite passes before production release.

## Proposed MVP milestones
1. **Foundation:** monorepo, CI, auth, organizations/workspaces, database migrations and tenant isolation.
2. **Chat core:** sessions/messages, streaming, model provider abstraction, runtime limits and traces.
3. **MCP:** connection lifecycle, discovery, normalized tool registry, execution and policy.
4. **Context & knowledge:** document ingestion, retrieval, generated context, versioning and test console.
5. **Widget & identity:** embed loader/iframe, guest sessions, origin validation and signed identity handoff.
6. **Hardening & launch:** audits, observability, rate limits, regression/security evaluations, operational runbooks.

## Risks and guardrails
- **Prompt injection:** content is untrusted; policy is deterministic and independent of model instructions.
- **Tenant leakage:** tenant-aware access checks everywhere, RLS as defense-in-depth and isolation tests.
- **Integration SSRF:** outbound connection broker and destination/IP validation.
- **Sensitive actions:** disabled by default, explicit permission and approval workflows.
- **Cost/abuse:** quotas, rate limits, execution budgets, model routing and metering.
- **Context drift:** versioned discovery/context, tests and customer-reviewed publishing.
- **Data retention:** customer-configurable retention, deletion lifecycle and privacy-aware logs.

## Definition of done for MVP
A customer can connect a supported MCP server, discover and approve capabilities, add knowledge, publish a context-aware agent, embed it on an allowed website, and support guest plus verified authenticated conversations—while tenant isolation, permissions, execution limits, auditability and evaluation gates are enforced.
