# System Architecture

## Accepted decisions

- Cloud-hosted SaaS; hosted model APIs only, no local inference.
- One universal agent per workspace for the MVP.
- Modular monolith initially, with independently scalable workers and clear domain boundaries.
- MCP first for integrations; OpenAPI and safe read-only database adapters may follow.
- Hybrid model selection: workspace preference plus administrator-managed platform defaults and workspace overrides.
- Both platform-managed credentials and workspace-owner BYOK credentials are supported.
- BYOK failures are surfaced; they never silently switch to platform credentials or another model.
- Platform-managed automatic fallback is administrator-controlled and disabled until explicitly configured.
- PostgreSQL is the system of record; pgvector supports knowledge retrieval.
- The customer's application remains authoritative for end-user authorization.

## 1. High-level architecture

    Customer website / application
      - iframe chat widget
      - signed identity handoff for authenticated users
                |
                v
    Edge / API gateway
      - origin validation, rate limits, request validation
                |
                v
    SaaS application (modular monolith)
      - tenancy, membership and identity
      - workspace and agent configuration
      - integration management and MCP discovery
      - tool registry and policy engine
      - context and knowledge management
      - conversation API, usage, audit and administration
                |
                v
    Universal Agent Runtime
      - load published agent version
      - assemble scoped context and retrieved knowledge
      - bounded model/tool-call loop
      - invoke policy-controlled tools
          |                         |
          v                         v
    Model Gateway             Tool Execution Broker
      - routing                  - MCP adapters
      - capability checks        - schema validation
      - credential resolution    - authorization/policy
      - provider adapters        - timeout/output limits
      - optional fallback        - customer tools/APIs
          |                         |
          v                         v
    Hosted model APIs          Customer MCP servers / APIs

Shared infrastructure: PostgreSQL + pgvector, private object storage, durable queue/workers, managed secret encryption, telemetry.

The runtime must not call model providers directly. All inference requests pass through the Model Gateway. Tool execution must pass through the broker and server-side policy engine; the model never receives credentials.

## 2. Application boundaries

Start with a modular monolith deployed as stateless application processes. Keep modules separated by domain interfaces so high-load or high-risk modules can later become services.

| Module | Responsibilities |
|---|---|
| Identity and tenancy | Organizations, workspaces, members, roles, tenant resolution |
| Agent configuration | Settings, versioning, draft/publish/rollback |
| Integration manager | MCP endpoints, validation, discovery and health |
| Tool registry | Normalized schemas, risk classification, enablement |
| Policy engine | Actor/workspace/tool authorization, approvals and limits |
| Context engine | Curated operating context, review, version and publication |
| Knowledge | File lifecycle, extraction, chunking, embeddings and retrieval |
| Conversation | Sessions, messages, streaming and feedback |
| Agent runtime | Orchestrates context, model calls and bounded tool loop |
| Model Gateway | Provider-neutral inference, routing, credentials and usage |
| Administration | Provider catalog, platform defaults, workspace overrides and fallback |
| Metering and audit | Usage attribution, operational events and audit history |

## 3. Model Gateway

### 3.1 Provider abstraction

Define an internal provider-neutral interface. Provider-specific SDKs and HTTP details stay inside adapters.

Conceptual operations:
- Generate a response, including streaming.
- Submit tool/function definitions and process tool-call responses.
- Normalize provider errors, finish reasons and usage.
- Expose provider/model identifiers and request IDs for diagnostics.
- Validate supported capabilities.

Do not assume every provider/model supports the same features. Maintain a capability registry for streaming, tool calling, structured output, vision and context limits. Validate requirements before routing.

Initial adapters may include OpenRouter and direct hosted APIs such as Google Gemini, subject to API availability, terms and capability validation. New providers are added through adapters, not runtime-specific branches.

### 3.2 Configuration layers

Resolve configuration in this order:

1. Workspace BYOK binding, when explicitly configured for the selected provider/model and enabled for the request.
2. Administrator workspace override, when configured.
3. Platform global default, when configured and permitted.

A workspace owner may add a credential for a particular provider and bind it to a model or model family. Ordinary configuration records store only a secret reference. Encrypted secret material belongs in the secret-storage subsystem.

A workspace model preference is a routing preference, not permission to bypass provider capability checks, platform safety limits or administrator restrictions.

### 3.3 BYOK failure behavior

When using a customer-owned credential:
- Use only that credential and its configured provider/model route.
- On authentication, quota, rate-limit, availability, model or request failure, return a normalized actionable error to the workspace owner/admin and a safe generic error to an end user.
- Do not silently retry with a platform key, another workspace credential, another provider or another model.
- Never expose the raw key, full provider response, internal endpoint or sensitive prompt/tool payload in UI or logs.
- A retry against the same route may be offered when safe, but must not change credential ownership or model selection.

### 3.4 Platform-managed fallback

Fallback is **disabled by default**. Only an authorized platform administrator can explicitly enable and configure it.

When disabled, a platform-managed provider/model failure is reported and logged; no alternate provider/model is attempted.

When enabled, the administrator configures:
- eligible source routes and ordered fallback routes
- permitted platform credential references for each route
- retryable error classes, such as temporary outage or rate limit
- maximum attempts and total time budget
- capability requirements and model restrictions
- optional cost ceiling

Fallback is limited to platform-managed credentials. It must never cross credential ownership boundaries. BYOK requests are excluded even if platform fallback is enabled globally. Do not retry non-retryable errors such as invalid request/schema, policy denial or unsupported capability.

### 3.5 Routing algorithm

1. Resolve workspace and published agent version server-side.
2. Determine task requirements: tool calling, structured output, streaming, etc.
3. Resolve workspace preference or effective administrator/platform default.
4. Resolve credential ownership and secret reference.
5. Check provider/model status, capability compatibility, workspace policy and usage limits.
6. Call the selected provider adapter.
7. Normalize and record response metadata, latency, token usage and cost estimate.
8. On failure, stop for BYOK; for platform-managed requests, consult the effective administrator fallback policy and attempt only configured routes when enabled.
9. Return the response/stream to the runtime.

Do not select a model based only on a free/paid label. Availability, quotas, terms and capabilities change. Keep pricing and availability metadata refreshable; free tiers are best-effort, not guaranteed.

### 3.6 Credential security

- Encrypt provider keys using envelope encryption backed by managed KMS/key-encryption keys.
- Keep ciphertext and key metadata separate from ordinary configuration; use secret references.
- Decrypt only in the server-side Model Gateway immediately before use.
- Never return secret material to the browser, model context, logs, traces, analytics or support exports.
- Enforce tenant ownership in every secret lookup; platform credentials are accessible only to authorized server roles.
- Support rotation, revocation, validation status and deletion.
- Audit credential lifecycle and assignment events without recording secret values.
- Prefer per-workspace credentials and least-privilege service identities.

## 4. Integration and tool execution

MCP is the initial integration boundary. Each connection has a workspace owner, endpoint, transport/auth configuration, health state and secret references. Validate destinations to prevent SSRF, including DNS resolution and private/link-local address checks at connection and reconnect time.

Discovery pipeline:
1. Validate endpoint and authentication.
2. Connect through controlled outbound networking.
3. Discover tools and schemas.
4. Normalize schemas and assign risk classification.
5. Keep discovered tools disabled until reviewed/approved.
6. Version the approved registry and permissions.
7. Revalidate periodically and record health/schema changes.

Tool execution:
- Runtime submits a proposed call to the broker.
- Broker resolves tenant, actor, tool version and policy.
- Validate arguments against the stored schema.
- Enforce enablement, actor authorization, rate/size/time limits and approval requirements.
- Invoke the connector using server-side credentials.
- Bound and sanitize outputs before returning them to the model.
- Record execution metadata and audit outcomes.

Treat tool descriptions, tool results, remote MCP metadata and retrieved content as untrusted data. Model output is never an authorization decision.

## 5. Identity and tenant isolation

- Resolve tenant/workspace from server-validated widget configuration or authenticated identity token; never trust a client-supplied workspace ID alone.
- Guest sessions receive narrowly scoped random identifiers and public permissions only.
- Authenticated handoff uses short-lived signed tokens with issuer, audience, expiry, subject and workspace binding.
- The customer application remains authoritative for end-user permissions. Use customer authorization endpoints or short-lived, signed, explicitly scoped claims where required.
- Re-check authorization at every sensitive tool execution.
- Enforce tenant ownership in application queries and PostgreSQL row-level security as defense in depth.

## 6. Context and knowledge

Keep operating context separate from knowledge:
- **Context** is concise, curated instruction/configuration describing the application, domain, terminology and behavioral constraints. It is reviewed, versioned and published with the agent.
- **Knowledge** is source material indexed for retrieval. It has document/source lifecycle, chunk metadata, access scope and deletion handling.

At runtime, assemble published context, conversation state, actor scope and permission-filtered retrieved passages. Retrieved content is evidence/data, not instructions. Documents and tool results cannot override system policy.

## 7. Data and infrastructure

| Component | Initial choice / role |
|---|---|
| Primary database | PostgreSQL |
| Vector retrieval | pgvector |
| ORM/migrations | Drizzle proposal |
| Object storage | S3-compatible private buckets |
| Background work | Durable queue with retry/dead-letter handling |
| Cache | Optional managed Redis-compatible cache; never authorization source of truth |
| Secrets | Managed KMS/envelope encryption |
| Runtime | Stateless Node.js/TypeScript |
| Widget | Isolated iframe with origin allowlist and strict postMessage validation |
| Telemetry | OpenTelemetry-compatible traces, metrics and structured logs |

Keep provider calls, MCP calls, ingestion and embeddings outside long-running web request handlers where appropriate. Use durable jobs for ingestion, discovery refresh and retryable background work.

## 8. Reliability, limits and observability

- Apply per-workspace, per-user and per-IP rate limits as appropriate.
- Bound model turns, tool calls, wall-clock duration, tokens, output size, concurrency and retries.
- Use idempotency keys for retried state-changing operations.
- Apply circuit breakers/backoff to provider and integration calls; circuit-breaker behavior must not become unconfigured model fallback.
- Record request/run IDs, workspace, provider/model, credential ownership class, latency, status, token counts, estimated cost and fallback attempt metadata.
- Redact secrets and sensitive payloads. Keep content logging opt-in and retention-limited.
- Alert on error rates, provider latency, quota exhaustion, queue lag, integration health and unusual usage.

## 9. Security baseline

- Strict tenant isolation and RLS defense in depth.
- SSRF-safe outbound connector broker and egress controls.
- Encryption in transit and at rest; managed key rotation.
- No secrets in prompts, browser payloads, logs or error strings.
- Server-side authorization and tool policy enforcement.
- Prompt-injection defenses through instruction/data separation and deterministic policies.
- Secure widget origin validation and signed identity handoff.
- Audit trails for credential, policy, integration and agent publication changes.
- Data retention, export and deletion workflows.
- Dependency scanning, security testing, backups and recovery drills.

## 10. Deployment topology and evolution

MVP deployment:
- Web/API application
- Runtime process or isolated runtime worker
- Background worker(s)
- PostgreSQL with pgvector
- Private object storage
- Durable queue
- Managed secrets/KMS
- Telemetry collector/backend

Deploy stateless compute horizontally. Keep database migrations backward-compatible and use expand/migrate/contract changes. Separate worker concurrency pools for model calls, MCP calls and document ingestion.

Do not split into microservices prematurely. Extract a module only when scale, security isolation, independent release cadence or operational ownership justifies it.

## 11. Open implementation decisions

- Hosting/cloud provider and regions
- Authentication vendor
- Queue product
- Secret manager/KMS product
- Exact provider adapter SDKs and initial model allowlist
- Pricing/usage policy and customer-facing quotas
- Data residency, retention periods and compliance targets
