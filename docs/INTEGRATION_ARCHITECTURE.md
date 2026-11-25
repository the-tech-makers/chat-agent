# Integration Architecture

## 1. Purpose and decisions

This document defines how a workspace connects its application capabilities to Chat Agent, how tools are discovered and governed, and how the Universal Agent invokes those tools safely.

Accepted decisions:
- MCP is the first-class integration protocol for the MVP.
- One universal agent per workspace consumes that workspace's approved integration capabilities.
- Customer integrations are workspace-scoped and isolated.
- Integration credentials are stored server-side as secret references; they are never exposed to the browser or model.
- Discovered tools are disabled until reviewed and explicitly enabled.
- Tool authorization is deterministic server-side policy, never a model decision.
- The SaaS uses outbound connections to customer-hosted MCP endpoints. No customer database is directly exposed to the model.
- OpenAPI and narrowly scoped read-only database connectors are future extensions, not MVP requirements.

## 2. Integration model

A workspace can configure one or more integration connections. Each connection represents a remote MCP server and contains:
- workspace ownership
- display name and environment (development/staging/production)
- remote endpoint URL and supported transport
- authentication method and secret references
- connection status and health timestamps
- discovered tool catalog and schema versions
- sync/discovery history

The connection is a transport and trust boundary, not an authorization grant. Connecting a server does not automatically enable its tools.

Logical flow:

    Workspace owner configures MCP endpoint
                  |
                  v
    Connection validation and secure egress
                  |
                  v
    Authentication / OAuth handshake
                  |
                  v
    MCP initialize + capability negotiation
                  |
                  v
    tools/list discovery and schema normalization
                  |
                  v
    Risk classification + review queue
                  |
                  v
    Workspace owner approves selected tools
                  |
                  v
    Versioned tool registry + policy configuration
                  |
                  v
    Agent runtime proposes a tool call
                  |
                  v
    Tool Execution Broker re-authorizes and validates
                  |
                  v
    Remote MCP tools/call
                  |
                  v
    Bounded result normalization + audit
                  |
                  v
    Agent runtime receives untrusted tool result
                  
## 3. Supported connection modes

### 3.1 MVP: customer-hosted remote MCP

The customer supplies an HTTPS endpoint reachable from the SaaS outbound network. Prefer Streamable HTTP transport when supported by the server. Support legacy HTTP+SSE only if required by target MCP server compatibility and only through a maintained adapter.

Do not support arbitrary local stdio processes in the multi-tenant SaaS runtime. A SaaS service should not execute customer-provided commands or binaries on its hosts.

### 3.2 Network reachability

MVP connection requirements:
- Publicly reachable HTTPS MCP endpoint, or a future supported private connectivity mechanism.
- Valid TLS certificate and hostname.
- Explicit connection timeout and response-size limits.
- Customer confirms they own or are authorized to connect the endpoint.

Private networking options (VPN, private link, customer-side relay/agent, allowlisted static egress IPs) can be considered later. Do not claim private-network support until implemented and threat-modeled.

### 3.3 Future connection types

- OpenAPI: import an OpenAPI document, normalize selected operations into governed tools.
- Read-only database adapter: customer-configured, schema-restricted, parameterized queries with row/column limits; no arbitrary SQL or writes.
- Customer-side relay: outbound-initiated tunnel for private MCP servers, requiring a separate security and lifecycle design.

## 4. Authentication and credential management

Authentication is configured per integration connection. Support only explicit, well-defined schemes; do not allow arbitrary user-controlled authentication code.

### 4.1 MVP authentication options

- No authentication, only when the customer explicitly confirms the endpoint is intentionally public.
- Bearer token / API key in an authorization header.
- OAuth 2.1 where the MCP server supports a compatible flow. Store client secrets and refresh tokens securely. Use PKCE for authorization-code flows where applicable, validate state/redirect URI, and protect token refresh against concurrency.
- Additional schemes only through reviewed provider-specific adapters.

The UI should identify which party owns the credential and what access it grants. Prefer narrow scopes and read-only permissions. Do not ask users to paste long-lived personal tokens when a safer OAuth flow is available.

### 4.2 Secret handling

- Secret material is submitted over TLS and immediately stored through the managed secret subsystem using envelope encryption/KMS.
- Database records contain secret references and metadata only.
- Secret values are write-only after submission: never return them to the dashboard after saving.
- Decrypt only inside the connector worker immediately before a request.
- Redact authorization headers and token-bearing URLs from logs, traces and error messages.
- Support validation, rotation, revocation and deletion.
- Audit credential lifecycle events without recording secret values.
- Enforce workspace ownership at secret resolution. A connector must not be able to reference another workspace's secret.

Never put credentials in MCP tool descriptions, tool arguments, model prompts, browser storage or client-side widget configuration.

## 5. Connection lifecycle and discovery

Use an explicit state machine:

    draft -> validating -> connected -> discovering -> review_required
       |         |             |              |
       v         v             v              v
    disabled   error        degraded       approved
                                              |
                                              v
                                           active
                                              |
                                  schema/auth/health failure
                                              |
                                              v
                                           degraded

A connection may also be disabled or deleted from any non-terminal state. Deletion revokes/forgets credentials and schedules associated metadata cleanup according to retention policy.

### 5.1 Create and validate

1. Workspace owner/admin submits endpoint, transport and authentication configuration.
2. API validates URL syntax, scheme, port policy and workspace authorization.
3. Connector service performs SSRF-safe DNS and destination checks.
4. Establish TLS and connect using strict timeouts and response limits.
5. Authenticate and perform MCP initialization/capability negotiation.
6. Store server identity/capabilities and a sanitized validation result.
7. Mark connected or return a safe actionable error.

### 5.2 Discover tools

1. Call MCP tools/list using pagination where supported.
2. Validate response size and tool count.
3. Normalize tool name, description, input schema and annotations.
4. Validate JSON Schema against supported schema constraints.
5. Assign a stable internal tool ID and retain the remote tool name.
6. Compute a schema fingerprint/version.
7. Classify risk and mark newly discovered or changed tools as disabled and requiring review.
8. Show a human-readable diff when tools or schemas change.

Do not treat server-provided annotations or descriptions as proof that a tool is safe. They are hints for review only.

### 5.3 Discovery refresh

Support manual refresh first. Scheduled refresh can be added with bounded frequency and backoff.

On refresh:
- unchanged schemas retain their approved version and policy
- changed schemas create a new version and require review before execution
- removed tools are marked unavailable; do not silently remap calls
- newly discovered tools remain disabled
- auth/transport failures update health state without deleting approved configuration

## 6. Tool catalog, review and approval

For every discovered tool, show:
- integration and remote server name
- tool name and description
- normalized input schema and required fields
- risk classification and why it was assigned
- whether it reads, writes, sends messages, changes permissions or triggers external effects (best-effort classification requiring owner review)
- current enablement and actor access
- schema version and last discovery time

Suggested risk levels:
- **Low:** read-only lookup with bounded output and no sensitive data.
- **Medium:** accesses personal/account data or performs a limited reversible action.
- **High:** changes records, sends communications, triggers payments, changes access or performs irreversible/external actions.

Risk classification is a starting signal, not an authorization policy. Workspace owners must explicitly enable tools and configure actor scopes. High-risk actions should default to disabled and may require end-user confirmation or a customer-side approval workflow.

Approval applies to the exact tool/schema version and scope. A material schema change invalidates prior approval until reviewed again.

## 7. Runtime tool execution

All tool calls go through the Tool Execution Broker.

### 7.1 Execution sequence

1. Model returns a proposed tool name and arguments.
2. Runtime maps it to an internal tool ID in the active published agent version; never accept an arbitrary endpoint or remote tool name from model output.
3. Broker resolves workspace, integration, actor, conversation and published policy server-side.
4. Check integration health and tool enablement.
5. Check actor authorization (guest/authenticated/role/user scope) and any customer authorization requirement.
6. Validate arguments against the approved schema version; reject unknown or additional properties unless explicitly permitted.
7. Apply approval requirements, rate limits, concurrency limits and per-tool budgets.
8. Resolve connector credential and call the remote server.
9. Enforce connection, response, execution-time and output-size limits.
10. Normalize and sanitize the result; redact likely secrets and avoid returning unnecessary fields.
11. Persist execution metadata and audit outcome.
12. Return the bounded result to the model as untrusted data.

### 7.2 Authorization rules

- Guest users receive only explicitly guest-enabled tools and public scopes.
- Authenticated users are identified through verified identity handoff.
- Workspace tool permissions are necessary but may not be sufficient: customer-specific object-level authorization may require a customer authorization endpoint or narrowly scoped identity claims.
- Re-check authorization for every call, including calls in the same conversation.
- Never infer permission from prior assistant messages, user claims or tool descriptions.
- Deny by default when identity, policy, integration status or authorization service is unavailable for a protected action.

### 7.3 Approval patterns

MVP should support tool enablement and policy-level approval requirements. For high-risk actions, prefer explicit confirmation before execution. Confirmation must be bound to:
- the exact tool and schema version
- canonicalized arguments or an argument hash
- workspace, actor and conversation
- expiry and one-time nonce

If arguments change, approval is invalid. For workflows requiring business approval, delegate to the customer's own application rather than treating a chat confirmation as sufficient.

## 8. SSRF and outbound network security

This is a critical security boundary because customers supply remote URLs.

- Permit HTTPS by default; allow HTTP only for local development in a non-production environment with explicit safeguards.
- Reject loopback, private, link-local, multicast, reserved and cloud metadata IP ranges for public SaaS connections.
- Resolve DNS and validate every A/AAAA address before connecting.
- Defend against DNS rebinding by pinning/validating the resolved destination for the connection and rechecking redirects/reconnects.
- Disable redirects by default; if enabled for a protocol requirement, revalidate every redirect target and cap redirect count.
- Restrict ports to an allowlist or explicit supported set.
- Use an outbound egress proxy/broker with network-level deny rules as defense in depth.
- Prevent access to cloud metadata endpoints and internal control planes.
- Apply per-workspace connection quotas, request rate limits, concurrency limits and bandwidth limits.
- Never allow tool output to cause the connector to fetch arbitrary URLs without a separate validated fetch capability.

For customer private networks, introduce a separate reviewed private-connectivity product rather than weakening public endpoint protections.

## 9. Reliability and operations

- Use bounded connect, initialize, discovery and tool-call timeouts.
- Retry only transient network/provider errors and only within strict limits; do not retry non-idempotent actions unless the remote operation supports idempotency.
- Use exponential backoff with jitter for health checks and scheduled discovery.
- Apply per-connection circuit breakers and concurrency pools.
- Distinguish authentication failure, authorization denial, server unavailable, protocol error, schema error, timeout and rate limit.
- Do not automatically enable tools after recovery.
- Maintain health states such as connected, degraded, authentication_required, unavailable and disabled.
- Provide a connection test and last-success/last-error metadata without leaking sensitive response bodies.

## 10. Data model additions

The logical database design should include or refine these records:

### integrations
- id, workspace_id, name, integration_type, endpoint_url
- transport, auth_mode, credential_reference
- status, server_info JSONB, capabilities JSONB
- last_validated_at, last_health_check_at
- created_by_user_id, created_at, updated_at, deleted_at

### integration_discovery_runs
- id, workspace_id, integration_id
- status, started_at, completed_at
- server_version, discovered_tool_count
- error_code, sanitized_error_summary
- initiated_by_user_id

### integration_tools
- id, workspace_id, integration_id
- remote_tool_name, display_name, description
- input_schema JSONB, schema_fingerprint, schema_version
- risk_level, risk_reasons JSONB
- status (review_required, approved, disabled, removed)
- discovered_at, approved_at, approved_by_user_id

Unique: (integration_id, remote_tool_name, schema_version) as appropriate for retained versions. Keep a stable logical tool identity across versions but immutable schema-version records.

### integration_tool_versions
- id, integration_tool_id, version_number
- input_schema JSONB, description, annotations JSONB
- schema_fingerprint, discovered_at
- review_status, reviewed_by_user_id, reviewed_at

### tool_permissions
- id, workspace_id, integration_tool_id or tool_version_id
- actor_scope, actor_reference nullable
- permission (deny, allow, require_approval)
- created_by_user_id, created_at, updated_at

### tool_execution_logs
- id, workspace_id, integration_id, integration_tool_id, tool_version_id
- conversation_id, agent_run_id, actor_reference
- status, duration_ms, result_size_bytes
- normalized_error_code, remote_request_id nullable
- approval_id nullable, created_at

### tool_approvals (if durable confirmation is included in MVP)
- id, workspace_id, integration_tool_version_id
- conversation_id, actor_reference
- arguments_hash, status, expires_at, consumed_at
- created_at

All records must be tenant-scoped. Use composite foreign keys or service-level invariants to ensure integration, tool, version, permission and execution records belong to the same workspace. Secret bytes remain outside these tables.

## 11. API surface (conceptual)

Workspace dashboard APIs:
- create/update integration connection
- validate/test connection
- initiate OAuth and complete callback
- trigger discovery and view discovery run
- list tools and schema diffs
- approve/disable tool versions
- configure actor permissions and approval requirements
- rotate/revoke integration credentials
- view health and execution history

Runtime/internal APIs:
- resolve tool by internal ID and published version
- authorize proposed tool call
- execute approved tool call
- record result and audit event

The browser must never call a customer's MCP server directly for privileged operations. Runtime endpoints must authenticate service-to-service calls and derive workspace scope from trusted server context.

## 12. Implementation phases

1. **MCP connection foundation:** remote Streamable HTTP client, connection CRUD, endpoint validation, SSRF-safe egress, secret references.
2. **Discovery:** initialize/capabilities, tools/list, pagination, schema validation, schema fingerprints and discovery history.
3. **Review and registry:** dashboard review, risk hints, disabled-by-default tools, approval and versioning.
4. **Execution broker:** schema validation, actor policy, bounded calls, result sanitization, audit and logs.
5. **Identity-aware permissions:** guest/authenticated scopes and customer authorization integration contract.
6. **Hardening:** rate/concurrency limits, health checks, backoff, circuit breakers, abuse testing and security review.

## 13. MVP acceptance criteria

- A workspace owner can configure and validate a supported remote MCP endpoint.
- Production connections use HTTPS and pass SSRF/destination validation.
- Credentials are write-only in the UI, encrypted and never logged or sent to the model.
- Discovery returns normalized, versioned tools and safe schema diffs.
- Newly discovered or materially changed tools are disabled pending review.
- The owner can approve/disable tools and configure actor scopes.
- Every runtime tool call is mapped to an approved internal tool/version.
- Every call is schema-validated and authorized server-side at execution time.
- Guest and authenticated permissions are distinct and tested.
- Calls have bounded time, concurrency, response size and rate.
- Results are treated as untrusted data and sanitized before model use.
- Cross-workspace access tests prove connections, tools, credentials and logs cannot leak.
- SSRF tests cover private IPs, IPv6, redirects, DNS rebinding and metadata endpoints.
- Audit logs capture connection, credential, discovery, approval and execution changes without secret values.
