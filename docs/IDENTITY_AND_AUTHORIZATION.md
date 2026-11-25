# Identity and Authorization Architecture

## 1. Purpose

This document defines how the Universal AI Agent SaaS identifies people and services, establishes workspace membership, propagates end-user identity, and authorizes every data or tool operation.

The central security rule is:

> Authentication establishes who a caller is. Authorization determines what that caller may do. A successful chat session, valid login, or workspace membership must never be treated as blanket permission to access customer data.

The customer application remains the authority for its own users, records, and business permissions. The SaaS platform is authoritative for platform accounts, organizations, workspaces, configuration, and platform operations.

## 2. Security objectives

- Isolate every organization and workspace.
- Support anonymous website visitors without granting them authenticated-user privileges.
- Support authenticated application users through a customer-controlled identity handoff.
- Keep platform administrators, workspace owners, workspace members, visitors, and service identities distinct.
- Authorize each protected tool invocation using current, server-verified identity and policy.
- Prevent clients and language models from asserting roles, tenant IDs, ownership, or permissions.
- Preserve an auditable record of identity, policy decisions, approvals, and tool execution.
- Minimize personal data received and retained by the platform.

## 3. Trust boundaries and authorities

| Domain | Authoritative system | Responsibility |
|---|---|---|
| Platform identity | Universal Agent SaaS | SaaS user accounts, sessions, MFA, platform roles |
| Organization/workspace access | Universal Agent SaaS | Membership, workspace roles, workspace configuration |
| Customer end-user identity | Customer application / identity provider | User identity, account status, application roles and entitlements |
| Customer business data | Customer application and connected systems | Record-level access, ownership, business rules |
| Tool execution | SaaS Tool Execution Broker plus target system | Enforce platform policy and target-system authorization |
| Model provider | Configured hosted provider | Inference only; not an authorization authority |

The model is untrusted with respect to authorization. Tool descriptions, retrieved documents, conversation messages, and model-generated arguments cannot grant permission.

## 4. Actor types

### 4.1 Platform actors

- **Platform administrator**: manages platform-wide provider configuration, operational controls, abuse response, and approved support workflows.
- **Platform operator/support**: limited operational access where explicitly granted; should not automatically receive customer conversation or business-data access.
- **Platform service identity**: workload identity used by internal services and workers. It is not a human user and must have narrowly scoped permissions.

Platform roles must be separate from workspace roles. A workspace owner is not a platform administrator.

### 4.2 Workspace actors

- **Workspace owner**: manages workspace settings, integrations, identity handoff configuration, members, usage, and approved credentials.
- **Workspace administrator**: performs delegated configuration and member management.
- **Workspace developer**: configures agents, prompts, knowledge sources, and integrations according to granted permissions.
- **Workspace analyst**: reads permitted usage, audit, and operational reports.
- **Workspace member**: uses workspace management features explicitly granted to them.

Use permissions as the enforcement primitive; roles are named bundles of permissions. Avoid scattered checks such as `if role === "owner"` throughout application code.

### 4.3 End users

- **Anonymous visitor**: a browser or client that has not presented a verified customer identity.
- **Authenticated customer user**: an end user whose identity has been verified through the customer application's trusted server-side handoff.
- **Customer service identity**: a backend identity used by the customer application to call the SaaS for server-to-server operations.

An anonymous visitor must never be upgraded to an authenticated customer user merely because the browser supplies an email address, user ID, role, JWT-like string, or custom metadata.

## 5. Identity flows

### 5.1 SaaS dashboard login

1. A workspace operator signs in through the SaaS authentication system.
2. The SaaS verifies credentials or federated identity and applies MFA where configured.
3. The session is associated with a platform user ID.
4. Workspace membership and permissions are loaded server-side for the requested workspace.
5. Every management API request checks the session, active membership, workspace scope, and required permission.
6. The client-provided workspace ID is treated only as a selector; it is never proof of membership.

Use a mature managed identity provider or a well-maintained authentication library. Do not implement password storage, token signing, or OAuth protocol handling from scratch.

### 5.2 Anonymous website visitor

1. The customer's website initializes the chat widget with a public, non-secret installation identifier.
2. The SaaS validates that the installation is enabled for the requested workspace and allowed origin.
3. The SaaS issues a short-lived anonymous chat session with a random, unguessable identifier.
4. The session is scoped to that workspace, installation, and conversation; it carries no customer roles or private-data permissions.
5. Rate limits, abuse controls, and public-tool allowlists are applied.
6. The customer may optionally provide low-risk display context, but it must be treated as unverified metadata.

Do not embed workspace secrets, model provider keys, MCP credentials, or privileged API tokens in the widget.

### 5.3 Authenticated customer user handoff

Preferred pattern: a server-to-server identity exchange that produces a short-lived, signed assertion for the SaaS.

1. The customer application authenticates the end user using its own identity system.
2. The customer's backend—not browser JavaScript—calls the SaaS identity-exchange endpoint using a registered integration credential or mutually authenticated mechanism.
3. The backend submits a signed assertion containing a stable, opaque subject identifier, audience, issuer, workspace/install binding, issued-at time, expiry, and unique token ID.
4. The SaaS validates signature against the registered customer key/JWKS, issuer, audience, expiry, not-before, token ID/replay policy, and workspace binding.
5. The SaaS maps the issuer plus subject to a customer identity record scoped to the workspace.
6. The SaaS creates a short-lived chat session bound to that identity and returns a browser-safe session token.
7. The widget uses only the resulting session token. It never receives the customer's service credential or signing key.

Recommended assertion claims:

| Claim | Requirement |
|---|---|
| `iss` | Exact registered customer issuer |
| `sub` | Stable opaque user identifier; avoid email as primary key |
| `aud` | Exact SaaS identity-exchange audience |
| `iat` | Issued-at timestamp with bounded clock skew |
| `exp` | Short expiry, typically no more than 1–2 minutes |
| `jti` | Unique token ID for replay detection where practical |
| `workspace_id` / `integration_id` | Must match registered integration binding |
| `auth_time` | Optional; useful for freshness and step-up checks |
| `acr` / `amr` | Optional authentication assurance metadata |

The precise token format can be JWT or another signed format. Use established libraries and asymmetric signing where feasible. Support key rotation through registered key IDs and JWKS or an equivalent secure key-distribution mechanism.

Never accept authorization-critical claims such as `is_admin`, unrestricted `roles`, `can_access_all_records`, or arbitrary permission lists from a browser. If customer roles or attributes are included in the signed assertion, treat them as bounded claims from a trusted backend and map them through explicit workspace policy. For sensitive operations, ask the customer backend or target API to make the authoritative decision.

### 5.4 Logout, revocation, and identity changes

- Customer logout should revoke or invalidate the SaaS chat session where the integration supports a logout callback.
- Keep chat session lifetimes short and support server-side revocation.
- Re-check identity status and sensitive permissions during tool execution; do not rely only on claims captured when a long-lived conversation began.
- Define behavior for deleted, suspended, or deprovisioned customer users.
- Do not silently convert an expired authenticated session into an anonymous session while retaining conversation access to authenticated-only data.
- Account linking between anonymous and authenticated conversations must be an explicit, policy-controlled operation with audit history.

## 6. Authorization model

Use a combination of RBAC and attribute/resource-based checks:

- **RBAC** for platform and workspace management permissions.
- **ABAC** for contextual conditions such as actor type, authentication assurance, origin, integration, time, and risk.
- **Resource-level authorization** for records and actions owned by the customer application.

Every decision should evaluate:

1. Actor identity and type.
2. Organization and workspace scope.
3. Integration and tool identity.
4. Requested action and resource.
5. Current customer authorization evidence.
6. Tool risk level and confirmation requirements.
7. Session status, rate limits, and relevant policy version.

Default decision is deny. Missing, stale, malformed, or unavailable authorization evidence must not be interpreted as permission.

### 6.1 Permission examples

Example permission names (the exact catalog can evolve):

- `workspace.read`, `workspace.update`
- `workspace.members.read`, `workspace.members.manage`
- `agent.read`, `agent.configure`
- `integration.read`, `integration.configure`
- `integration.tools.review`, `integration.tools.enable`
- `credentials.create`, `credentials.rotate`, `credentials.revoke`
- `knowledge.read`, `knowledge.manage`
- `conversations.read`, `conversations.export`
- `usage.read`, `audit.read`
- `platform.providers.manage`, `platform.workspaces.manage`

Separate permission to configure an integration from permission to execute its tools. Separate permission to view audit metadata from permission to inspect sensitive conversation content.

### 6.2 Tool authorization pipeline

The Tool Execution Broker must enforce this sequence for every invocation:

1. Resolve the chat session from a server-verified token.
2. Load the actor, workspace, installation, and integration from trusted database records.
3. Verify that the session is active and correctly scoped.
4. Resolve the tool by an immutable internal tool ID and current approved version; do not trust a model-supplied server URL or tool name alone.
5. Check that the tool is enabled and allowed for this workspace and actor type.
6. Validate arguments against the approved schema; reject unknown fields and unsafe values.
7. Evaluate workspace policy, actor permissions, risk classification, and any required human confirmation.
8. Obtain or propagate target-system credentials using the configured integration identity strategy.
9. Where the operation concerns customer-user data, enforce the customer's user-level authorization at the target system or through a trusted customer authorization endpoint.
10. Execute with timeouts, egress controls, and least-privilege credentials.
11. Record the decision and outcome, redacting secrets and sensitive payloads.
12. Return only the minimum result allowed by policy.

The LLM may propose a tool and arguments. It cannot skip, reorder, or override this pipeline.

### 6.3 Customer authorization strategies

Support explicit strategies per integration:

**A. Delegated end-user token (preferred for user-specific access)**

The customer provides a token scoped to the authenticated user and target API. The target system enforces that user's permissions. Tokens must be short-lived where possible and stored or passed securely.

**B. Customer backend authorization proxy**

The SaaS calls a customer-owned backend endpoint that evaluates whether a given subject may perform a specified action on a resource. Define a stable request/response contract, timeout, decision expiry, and fail-closed behavior.

**C. Service credential with explicit policy mapping**

The integration uses a service identity. The SaaS must enforce an explicit customer-approved policy that constrains which resources/actions each actor can access. This mode is higher risk because the service credential may have broad access. Do not assume that an authenticated chat user is entitled to everything the service account can read.

**D. Public/anonymous integration**

Allow only explicitly public, low-risk operations and public data. Apply origin restrictions, abuse controls, quotas, and response filtering. Never expose service credentials or unrestricted private search through a public visitor session.

For MVP, implement A or B for authenticated access where practical, plus a tightly restricted D mode for public website chat. If C is offered, make its limitations visible during setup and require explicit customer acknowledgement.

## 7. Conversation and session boundaries

- Every conversation belongs to exactly one workspace and one installation.
- A conversation has one actor context at a time: anonymous or a specific authenticated customer identity.
- Conversation IDs are opaque identifiers, not authorization credentials.
- Every read or mutation checks workspace and actor scope server-side.
- Prevent insecure direct object reference (IDOR) by never fetching a conversation, message, file, or tool result solely by its public ID.
- Define whether authenticated users can see prior conversations across devices; default to private per-user conversations.
- Anonymous conversations should be bound to a browser session and should not be discoverable by guessing IDs.
- Conversation sharing, transfer, and account linking require explicit product policy and audit events.

## 8. Data model additions

These are logical additions to the database design; names can be adapted to the final ORM schema.

### 8.1 Identity and access tables

| Table | Important fields | Purpose |
|---|---|---|
| `platform_users` | `id`, `auth_subject`, `status`, timestamps | SaaS dashboard identities |
| `workspace_memberships` | `workspace_id`, `platform_user_id`, `status` | Membership and lifecycle |
| `roles` | `id`, `scope`, `name` | Platform/workspace role definitions |
| `permissions` | `id`, `key`, `description` | Atomic permissions |
| `role_permissions` | `role_id`, `permission_id` | Role-to-permission mapping |
| `membership_roles` | `membership_id`, `role_id` | Assigned workspace roles |
| `customer_identity_issuers` | `workspace_id`, `issuer`, `audience`, `jwks_uri`, `status` | Trusted customer identity issuers |
| `customer_identities` | `workspace_id`, `issuer_id`, `subject`, `status` | External customer-user mapping |
| `identity_exchange_events` | `workspace_id`, `issuer_id`, `jti_hash`, `result`, timestamp | Replay detection and exchange audit |
| `chat_sessions` | `workspace_id`, `installation_id`, `actor_type`, `customer_identity_id`, `expires_at`, `revoked_at` | Anonymous/authenticated chat sessions |
| `authorization_policies` | `workspace_id`, `integration_id`, `policy_json`, `version`, `status` | Explicit integration/tool policy |
| `tool_authorization_decisions` | `workspace_id`, `session_id`, `tool_id`, `decision`, `reason_code`, `policy_version` | Decision audit |
| `tool_approvals` | `workspace_id`, `session_id`, `tool_call_id`, `approved_by`, `expires_at`, `status` | Human approval records |

Use composite foreign keys or equivalent constraints to ensure that referenced identities, sessions, memberships, integrations, and policies belong to the same workspace. Store a hash of replay identifiers rather than raw assertions. Never store raw customer identity tokens in audit logs.

### 8.2 Session token storage

Prefer opaque, random session tokens with only a hash stored server-side. If self-contained tokens are used, keep them short-lived and pair them with server-side revocation/session state for sensitive operations.

Browser tokens should be scoped to the chat API, have short expirations, and be protected against XSS/CSRF according to the delivery model. For first-party embedded widgets, evaluate secure, HttpOnly cookies where feasible; if bearer tokens are used in JavaScript, keep them short-lived, memory-resident where possible, and rotate them.

## 9. Tenant isolation and database enforcement

- Every tenant-owned table must carry `workspace_id` directly or be reachable through a constrained parent relationship.
- Enable PostgreSQL Row-Level Security for tenant-owned data as defense in depth.
- Set tenant context from verified server-side session state inside a transaction; never from an untrusted request header or body.
- Use separate database roles for migrations, application runtime, and background workers where practical.
- Runtime database roles must not own tenant tables or have `BYPASSRLS`.
- Use composite foreign keys such as `(workspace_id, integration_id)` to prevent cross-tenant references.
- Test cross-tenant reads, updates, deletes, joins, background jobs, exports, and vector retrieval.
- Apply the same workspace filter to pgvector similarity searches; vector similarity is not an authorization mechanism.
- Object-storage keys and signed download URLs must be tenant-scoped and short-lived.

RLS is not a substitute for application authorization. Both layers are required.

## 10. API and protocol surface

Suggested endpoints:

| Endpoint | Caller | Purpose |
|---|---|---|
| `POST /v1/identity/exchange` | Customer backend | Exchange trusted customer assertion for chat session |
| `POST /v1/chat/sessions/anonymous` | Public widget | Create restricted anonymous session |
| `POST /v1/chat/sessions/refresh` | Chat client | Rotate/refresh eligible session |
| `POST /v1/chat/sessions/revoke` | Trusted backend / authorized client | Revoke session |
| `GET /v1/chat/conversations/:id` | Chat session | Read only actor-authorized conversation |
| `POST /v1/chat/conversations/:id/messages` | Chat session | Submit message |
| `POST /v1/integrations/:id/authorization-check` | Internal broker | Internal policy evaluation; not public |
| `POST /v1/tool-approvals/:id/decision` | Authorized approver | Approve/reject a gated action |

The public widget API must not expose internal authorization-check endpoints or allow callers to specify arbitrary actor IDs.

## 11. Security controls

### Token and key security

- TLS for all network traffic.
- Validate exact issuer and audience; reject algorithm confusion and unsigned tokens.
- Maintain key rotation and emergency key revocation.
- Keep signing keys, integration credentials, and provider keys in a managed secret store or envelope-encrypted secret service.
- Never place secrets in widget bundles, URLs, logs, analytics events, or model prompts.
- Limit assertion lifetime and enforce clock-skew bounds.
- Use replay detection for identity exchange assertions.

### Browser and widget security

- Validate configured origins, but do not treat CORS as authentication.
- Use a strict Content Security Policy on the customer integration guidance.
- Protect cookie-based flows against CSRF.
- Apply output encoding and safe rendering to model responses and tool results.
- Rate-limit by workspace, installation, session, IP/network signal, and risk.
- Avoid collecting more personal information than required.

### Authorization and tool security

- Deny by default.
- Require explicit allowlisting for tools and actions.
- Separate read, write, administrative, and destructive capabilities.
- Require user confirmation for sensitive or irreversible actions; bind approval to the exact tool, arguments hash, actor, and expiry.
- Re-authorize at execution time; do not treat a previous model decision as approval.
- Fail closed if customer authorization checks time out or return malformed results.
- Use idempotency keys for retried mutations.
- Apply outbound network and SSRF controls independently of authorization.

### Audit and privacy

Record: actor type and stable internal ID, workspace, integration, tool/version, action, decision, reason code, policy version, approval reference, correlation ID, timestamp, and outcome.

Do not log: raw bearer tokens, private keys, provider API keys, full signed assertions, or unnecessary sensitive tool arguments/results. Redact or hash identifiers where appropriate. Define retention and deletion policies for conversation content and identity mappings.

## 12. Failure behavior

| Condition | Required behavior |
|---|---|
| Missing or invalid identity assertion | Reject exchange; do not create authenticated session |
| Unknown issuer or key ID | Reject; do not trust supplied claims |
| Expired or replayed assertion | Reject and record safe audit event |
| Customer identity disabled | Reject or revoke associated sessions |
| Workspace/integration disabled | Reject request |
| Authorization service unavailable | Fail closed for protected operations |
| Target API returns forbidden | Do not retry with broader credentials |
| Session expired/revoked | Require a new trusted exchange |
| Tool approval missing/expired | Do not execute |
| Tenant context missing | Abort database operation |

Never silently fall back from a failed authenticated identity check to anonymous access for the same protected conversation or tool request.

## 13. Implementation plan

### Phase 1 — Foundation

- Define platform user, workspace membership, roles, and permission catalog.
- Integrate managed SaaS dashboard authentication.
- Add workspace-scoped authorization middleware and policy helpers.
- Add tenant context transaction handling and RLS tests.
- Add audit event schema and correlation IDs.

### Phase 2 — Anonymous chat

- Register widget installations and allowed origins.
- Implement anonymous session issuance, expiry, revocation, and rate limiting.
- Restrict anonymous sessions to explicitly public tools and knowledge.
- Add conversation ownership and IDOR tests.

### Phase 3 — Authenticated identity handoff

- Register customer issuer, audience, and public signing keys/JWKS.
- Implement backend-only identity exchange and replay defense.
- Map issuer + subject to workspace-scoped customer identities.
- Implement short-lived chat sessions and revocation.
- Provide customer integration examples for common server frameworks.

### Phase 4 — Tool authorization

- Implement the Tool Execution Broker authorization pipeline.
- Add delegated user-token and/or customer authorization-proxy strategies.
- Add policy versioning, approval workflow, and fail-closed behavior.
- Add integration-level security tests and audit coverage.

### Phase 5 — Hardening

- Add key rotation and emergency revocation workflows.
- Perform tenant-isolation and penetration testing.
- Test session fixation, token replay, IDOR, privilege escalation, confused-deputy, prompt-injection-to-tool, and cross-tenant vector retrieval scenarios.
- Document data retention, deletion, incident response, and customer responsibilities.

## 14. MVP acceptance criteria

The identity and authorization implementation is not complete until:

- Anonymous visitors can chat without receiving authenticated-user privileges.
- A browser cannot create an authenticated identity by submitting arbitrary user IDs or roles.
- A customer backend can exchange a short-lived signed assertion for a scoped chat session.
- Invalid, expired, wrong-audience, wrong-issuer, and replayed assertions are rejected.
- Customer identity is scoped by workspace and issuer.
- Every conversation and tool operation is authorized server-side.
- A valid chat session does not grant access to all records available to an integration service credential.
- Anonymous tools are explicitly allowlisted and cannot access private customer data.
- Customer authorization failures and timeouts fail closed.
- Workspace isolation is enforced by both application checks and database RLS.
- Sensitive actions require explicit approval where configured.
- Audit logs capture decisions without storing secrets or raw assertions.
- Automated tests cover cross-tenant access, IDOR, privilege escalation, session revocation, and tool authorization.

## 15. Open decisions before implementation

1. Which managed identity provider will power SaaS dashboard login?
2. Which assertion format and key-distribution mechanism will be supported first (JWT + JWKS is the recommended default)?
3. Will the first authenticated integration use delegated end-user API tokens, a customer authorization proxy, or both?
4. What is the default conversation visibility policy for authenticated users across devices?
5. Which tool risk classes require confirmation, and who may approve them?

These choices should be captured as configuration and explicit contracts, not implicit assumptions in application code.
