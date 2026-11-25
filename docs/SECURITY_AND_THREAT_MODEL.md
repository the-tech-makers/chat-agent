# Security Architecture and Threat Model

## 1. Purpose and security posture

This document defines the security boundaries, threat model, baseline controls, and verification strategy for the Universal AI Agent SaaS.

The platform connects customer applications, end users, model providers, MCP servers, APIs, databases, files, and knowledge sources. It therefore acts as a security-sensitive intermediary. A compromise can expose data across tenants, misuse customer credentials, or cause actions in connected systems.

Security principles:

- **Deny by default:** access is granted only by explicit policy.
- **Tenant isolation:** every request, job, query, cache entry, vector search, and object reference is tenant-scoped.
- **Least privilege:** each actor, integration, credential, worker, and service receives only the access it needs.
- **Customer authority:** the customer application remains authoritative for customer-user permissions and business data.
- **Model distrust:** model output, retrieved content, tool descriptions, and user messages are untrusted input.
- **Explicit action boundaries:** reading, writing, destructive, and administrative tools have separate policies.
- **Secure failure:** authorization, identity, and policy uncertainty fails closed for protected operations.
- **Auditability:** security-relevant decisions are traceable without logging secrets or unnecessary personal data.

This is an engineering baseline, not a certification or legal compliance statement. Applicable privacy, data residency, and regulatory requirements must be assessed for the launch markets and customer types.

## 2. System assets

| Asset | Examples | Security impact if compromised |
|---|---|---|
| Tenant data | Conversations, files, embeddings, customer context | Confidentiality and privacy breach |
| Customer credentials | MCP tokens, API keys, OAuth refresh tokens | Unauthorized access to customer systems |
| Model credentials | Platform provider keys and workspace BYOK keys | Financial abuse and data exposure |
| Identity material | Sessions, signed assertions, issuer keys | Account takeover and privilege escalation |
| Tool definitions | Schemas, server URLs, enablement policies | Unauthorized execution or SSRF |
| Agent configuration | System instructions, routing, policies | Behavior manipulation and data leakage |
| Audit records | Access decisions, tool execution events | Loss of forensic evidence |
| Platform control plane | Admin roles, provider defaults, tenant configuration | System-wide compromise |
| Build/deployment supply chain | Dependencies, CI secrets, artifacts | Persistent application compromise |

## 3. Trust boundaries

1. **Browser/widget to SaaS edge:** public and potentially hostile. Treat all client data as untrusted.
2. **SaaS API to application services:** authenticated internal boundary. Validate service identity and request scope.
3. **Tenant application services to database/storage/queue:** enforce tenant context and least-privilege service credentials.
4. **Agent runtime to model provider:** outbound data boundary. Minimize and classify context sent to providers.
5. **Tool broker to MCP/API/database targets:** high-risk outbound boundary. Enforce target allowlists, authorization, and egress policy.
6. **Customer application to identity exchange:** trust is established only through registered backend credentials and verified signed assertions.
7. **Platform administration boundary:** highly privileged control plane. Require strong authentication, MFA, approval for sensitive changes, and immutable audit events.
8. **Background workers:** process queued untrusted or tenant-originated work. Revalidate tenant, actor, and policy context when executing.

No trust boundary is removed merely because services share a VPC, repository, or cloud account.

## 4. Threat actors and assumptions

Threat actors include:

- Anonymous internet users probing the public widget and API.
- Malicious authenticated customer users attempting to exceed their permissions.
- Compromised customer accounts, API keys, MCP servers, or integration endpoints.
- Malicious or compromised third-party MCP servers and API providers.
- Attackers exploiting prompt injection in user messages, web pages, documents, or tool results.
- Malicious insiders or compromised platform/workspace administrator accounts.
- Attackers exploiting vulnerable dependencies, CI/CD credentials, or cloud configuration.
- Other tenants attempting cross-tenant access through identifiers, caches, vector retrieval, or asynchronous jobs.

Assume that all customer-provided content, external tool output, remote server metadata, and model output can be malicious. Assume credentials may eventually leak and design for rotation, revocation, and containment.

## 5. Threat model and mitigations

### 5.1 Cross-tenant data exposure

**Attack paths**
- Missing workspace filter in SQL or ORM query.
- IDOR through guessed conversation, file, integration, or tool IDs.
- Incorrect tenant context in background jobs.
- Shared cache keys that omit workspace identity.
- Vector similarity search that retrieves embeddings from other workspaces.
- Object-storage paths or signed URLs that are not tenant-bound.

**Controls**
- Derive tenant scope from verified server-side session and membership state.
- Require workspace predicates in repository/service methods.
- Enforce PostgreSQL RLS as defense in depth; runtime roles must not own tables or bypass RLS.
- Use composite foreign keys to prevent cross-tenant relationships.
- Include workspace and authorization scope in cache keys and queue payloads; revalidate on consumption.
- Filter vector search by workspace and access-control metadata before returning results.
- Use opaque IDs as defense in depth, never as authorization.
- Scope object storage keys and signed URLs to tenant and resource; keep URLs short-lived.
- Add automated cross-tenant negative tests for every tenant-owned resource.

### 5.2 Prompt injection and instruction hijacking

**Attack paths**
- User asks the agent to ignore its policies.
- Retrieved web pages, files, emails, or database rows contain hostile instructions.
- MCP tool descriptions or tool results attempt to influence the model.
- Hidden text, metadata, code comments, or image OCR contain instructions.
- A malicious document asks the model to reveal system prompts, secrets, or other tenants' data.

**Controls**
- Treat all user, retrieved, and tool-provided content as data, not trusted instructions.
- Keep system policy and tool authorization outside model-controlled context.
- Do not place credentials, signing keys, or raw secrets in prompts.
- Apply authorization in deterministic server code after model tool selection and before execution.
- Limit retrieval to actor-authorized sources before content reaches the model.
- Delimit and label untrusted content in prompts; this is defense in depth, not a complete solution.
- Restrict tools by actor type, workspace policy, risk class, and task scope.
- Require confirmation for sensitive or irreversible operations.
- Validate tool arguments against strict schemas and business rules.
- Test indirect prompt injection through documents and tool results.

### 5.3 MCP and third-party tool abuse

**Attack paths**
- A malicious MCP server returns deceptive tool descriptions or schemas.
- A tool name collides with a trusted tool name.
- Tool arguments target unintended resources or invoke destructive behavior.
- A server changes its tool behavior after approval.
- A remote endpoint exfiltrates data or uses credentials beyond intended scope.

**Controls**
- Register integrations with immutable internal IDs and verified endpoint configuration.
- Treat remote tool metadata and output as untrusted.
- Require workspace review and explicit enablement for each tool.
- Bind approval to server identity, tool ID, schema/version hash, and risk classification.
- Re-review tools when schemas or server identity change; disable changed tools pending review for high-risk integrations.
- Separate read-only, write, destructive, and administrative permissions.
- Use per-integration credentials with minimum scopes and rotation support.
- Enforce execution-time authorization and argument validation.
- Limit response size, execution time, concurrency, and output exposure.
- Maintain an emergency integration/tool disable switch.
- Never let the model provide an arbitrary MCP endpoint or credential reference.

### 5.4 SSRF and outbound network abuse

**Attack paths**
- User or model supplies a URL pointing to cloud metadata, localhost, private network, or internal control-plane services.
- Redirects bypass initial URL checks.
- DNS rebinding changes a previously safe hostname to a private address.
- A compromised MCP server uses the SaaS network as a proxy.

**Controls**
- Use an outbound egress proxy or isolated connector network with destination policy.
- Allow only approved schemes (normally HTTPS) and explicitly configured destinations.
- Reject loopback, link-local, private, multicast, reserved, and cloud metadata IP ranges unless a documented private-connectivity feature explicitly permits them.
- Resolve and validate DNS at connection time; defend against rebinding and validate every redirect target.
- Block redirects by default for sensitive connector requests or revalidate each hop.
- Restrict ports and protocols; do not allow arbitrary TCP tunnels.
- Apply connection, response, and total request timeouts plus response-size limits.
- Do not expose internal service names, metadata endpoints, or private IPs to connector runtimes.
- Log destination metadata safely without recording authorization headers or query secrets.

### 5.5 Credential theft and misuse

**Attack paths**
- Secrets accidentally included in browser bundles, logs, prompts, traces, or error reports.
- Weak encryption or overly broad secret-store permissions.
- Workspace users access credentials belonging to another workspace.
- A compromised integration uses a long-lived, overprivileged token.
- BYOK provider credentials are silently used in an unintended fallback route.

**Controls**
- Store secret values only in a managed secret manager or envelope-encrypted credential service.
- Keep encryption keys separate from ciphertext; use KMS/HSM-backed key management where available.
- Store only secret references and non-sensitive metadata in ordinary application tables.
- Enforce workspace ownership checks on secret references and credential operations.
- Never return stored secret values after creation; support rotate and revoke flows.
- Redact secrets from logs, traces, exceptions, analytics, and model context.
- Use short-lived OAuth tokens and narrow scopes where supported.
- Apply provider-specific routing rules: BYOK failures must be visible and must not silently switch to another key/provider/model. Platform fallback is allowed only when explicitly configured by platform policy and disclosed.
- Rotate platform keys and support emergency revocation.
- Audit credential creation, use metadata, rotation, and revocation without logging values.

### 5.6 Identity, session, and authorization attacks

**Attack paths**
- Forged customer identity assertion or incorrect audience/issuer validation.
- Assertion replay, token theft, session fixation, or privilege escalation.
- Browser submits a fabricated user ID or role.
- Stale permissions remain active after customer account suspension.
- Confused-deputy behavior: a broad service credential acts on behalf of a user without checking the user's rights.

**Controls**
- Use backend-only identity exchange with registered issuers and verified signatures.
- Validate issuer, audience, expiry, not-before, signing algorithm, key ID, workspace binding, and replay policy.
- Use short-lived opaque chat sessions with server-side revocation.
- Keep platform roles separate from customer roles and workspace roles.
- Re-authorize sensitive tool calls at execution time.
- Prefer delegated end-user tokens or a customer authorization proxy for private customer data.
- Never infer user-level permissions from a service credential's access.
- Fail closed when customer authorization is unavailable or indeterminate.
- Bind approvals to actor, tool, arguments hash, policy version, and expiry.

### 5.7 API abuse, denial of service, and cost exhaustion

**Attack paths**
- High-volume anonymous requests, oversized prompts, recursive tool loops, expensive retrieval, or model calls.
- Repeated retries amplify provider or connector load.
- Attackers exploit streaming connections or queue backlogs.
- A tenant consumes shared capacity and degrades service for others.

**Controls**
- Rate-limit at edge and application layers by IP/network signal, workspace, installation, actor, and session.
- Enforce request body, message, attachment, context, tool output, and token limits.
- Set per-tenant quotas, concurrency limits, and spend budgets.
- Bound tool-call count, agent steps, recursion, retries, and total execution time.
- Use queue backpressure, per-tenant fairness, circuit breakers, and dead-letter queues.
- Apply idempotency keys to retryable mutations.
- Detect abnormal usage and provide operator kill switches.
- Protect streaming endpoints against connection exhaustion and enforce maximum stream duration.

### 5.8 Data leakage to model providers

**Attack paths**
- Full conversation or entire knowledge base is sent when only a small excerpt is needed.
- Private data from one tenant enters another tenant's prompt through shared state or cache.
- Secrets or sensitive tool results are included in model context.
- Provider retention or processing terms do not match customer expectations.

**Controls**
- Minimize and filter context before provider submission.
- Enforce tenant and actor authorization before retrieval and prompt assembly.
- Redact credentials and unnecessary personal or sensitive data.
- Provide configurable provider/model routing and document provider data handling.
- Keep provider credentials out of prompts and tool results.
- Avoid cross-tenant prompt caches; key caches by tenant, model, policy, and authorization scope.
- Define retention and deletion behavior for prompts, completions, traces, and provider logs.
- Do not claim zero retention or training exclusion unless verified contractually for the selected provider and account configuration.

### 5.9 File, document, and knowledge ingestion

**Attack paths**
- Malicious files exploit parsers or archive extraction.
- Oversized or recursive archives exhaust resources.
- Documents contain prompt injection or hidden instructions.
- Unauthorized documents become searchable through embeddings.
- Poisoned content contaminates retrieval or agent behavior.

**Controls**
- Validate file type by content, not extension alone.
- Enforce size, page, archive-depth, decompression, and processing-time limits.
- Scan uploads for malware where supported; isolate parsers in sandboxed workers.
- Store originals in private object storage with tenant-scoped access.
- Track source, owner, visibility, and ACL metadata through chunking and embedding.
- Enforce authorization filters during retrieval, not only ingestion.
- Treat extracted text and metadata as untrusted.
- Support source deletion and propagate deletion to chunks, embeddings, caches, and derived artifacts.
- Record ingestion status and parser version for reproducibility.

### 5.10 Supply-chain and deployment compromise

**Attack paths**
- Vulnerable or malicious dependency.
- Compromised CI token, deployment secret, build artifact, or container base image.
- Unreviewed code reaches production.
- Development credentials or test data leak into production.

**Controls**
- Pin and audit dependencies; use lockfiles and automated vulnerability alerts.
- Generate SBOMs and scan dependencies/container images in CI.
- Use minimal base images and non-root containers.
- Protect main branch with review and required checks before public launch.
- Use short-lived CI credentials and scoped deployment permissions.
- Separate development, staging, and production accounts/secrets/data.
- Sign or attest build artifacts where supported.
- Keep production secrets out of build-time client bundles.
- Maintain rollback and incident-response procedures.

### 5.11 Privileged insider and control-plane abuse

**Attack paths**
- Platform admin changes global model routing or fallback behavior.
- Support user inspects customer content without authorization.
- Workspace owner grants excessive permissions or exports sensitive data.
- Attacker compromises an administrator account.

**Controls**
- Require MFA for privileged users; use step-up authentication for sensitive actions.
- Separate operational support from content access.
- Require time-bound, reason-coded, approved break-glass access.
- Log all privileged configuration and data-access events.
- Apply least privilege and periodic access reviews.
- Require confirmation or dual control for high-impact global changes where practical.
- Alert on unusual admin activity and bulk exports.
- Make support access visible and auditable to customers where appropriate.

## 6. Security architecture controls by layer

### Edge and widget
- TLS, secure headers, origin allowlists, request validation, rate limiting, bot/abuse controls.
- Public widget configuration contains identifiers only, never secrets.
- Strict output encoding and safe markdown/HTML rendering.
- Content Security Policy guidance for customer embedding.

### API and application
- Centralized authentication and authorization middleware.
- Schema validation at every external boundary.
- CSRF protections for cookie-authenticated state-changing requests.
- Safe error responses without stack traces or secret-bearing details.
- Request IDs and security event correlation.
- Explicit CORS policy; CORS is not an access-control substitute.

### Agent runtime and model gateway
- Provider adapters with explicit credential binding and routing policy.
- No implicit provider/key fallback.
- Prompt construction separates trusted policy from untrusted data.
- Bounded agent steps, context size, and tool-call budgets.
- Structured tool-call validation and broker-only execution.
- No model access to raw integration secrets.

### Tool execution and connectors
- Dedicated broker/worker boundary.
- Egress proxy, destination allowlists, DNS/IP validation, and SSRF defenses.
- Per-integration credentials, isolated execution context, and scoped timeouts.
- Explicit tool approvals and immutable audit events.
- Response sanitization and size limits.

### Data and infrastructure
- PostgreSQL RLS and composite tenant constraints.
- Encryption in transit and at rest.
- Managed secret storage and key rotation.
- Private object storage and tenant-scoped signed URLs.
- Queue payload integrity, tenant context, and reauthorization at consumption.
- Backups encrypted and restoration tested.
- Separate service identities and least-privilege cloud IAM.

## 7. Security event logging and monitoring

Log security-relevant events such as:

- Authentication success/failure and session creation/revocation.
- Identity exchange rejection and replay detection.
- Workspace membership and role changes.
- Credential create/rotate/revoke events.
- Integration and tool enablement, schema changes, and disablement.
- Authorization allow/deny decisions and approval events.
- Cross-tenant access denials and RLS violations where observable.
- Rate-limit triggers, suspicious traffic, and abuse controls.
- Platform-admin changes, break-glass access, and exports.
- Security configuration and key rotation events.

Each event should include timestamp, workspace (when applicable), actor type and stable ID, action, target ID, outcome, reason code, correlation ID, and policy/version metadata.

Never log raw tokens, passwords, private keys, provider keys, full signed assertions, or unredacted sensitive payloads. Restrict access to security logs, protect integrity, and define retention.

## 8. Incident response

Prepare runbooks for:

1. **Credential exposure:** revoke/rotate affected key, identify usage window, disable integration if needed, notify affected customer according to policy.
2. **Cross-tenant exposure:** contain affected endpoints/jobs, preserve evidence, identify impacted tenants and records, remediate root cause, assess notification obligations.
3. **Malicious MCP integration:** disable integration/tool globally or per workspace, revoke connector credentials, inspect execution history.
4. **Account compromise:** revoke sessions, reset authentication factors, review privilege changes and exports.
5. **Provider/model abuse:** disable affected provider route or key, enforce budgets, preserve request metadata with sensitive content redacted.
6. **Supply-chain incident:** stop deployments, rotate CI/deployment secrets, rebuild from trusted sources, assess deployed artifacts.

Every incident should have an owner, timeline, impact assessment, evidence-preservation plan, customer communication decision, remediation items, and post-incident review.

## 9. Security verification strategy

### Automated tests required in CI

- Cross-tenant CRUD and query isolation for every tenant-owned resource.
- RLS behavior using the actual runtime database role.
- IDOR tests for conversations, files, knowledge sources, integrations, and tool results.
- Identity assertion signature, issuer, audience, expiry, key rotation, and replay tests.
- Session revocation and privilege-change propagation tests.
- Authorization tests for anonymous, authenticated, workspace, and platform actors.
- Tool schema validation, approval binding, and policy-version tests.
- SSRF tests for localhost, private IPv4/IPv6, link-local, metadata IPs, redirects, and DNS rebinding.
- Prompt injection tests through user content, retrieved files, MCP metadata, and tool output.
- Secret redaction tests for logs, traces, exceptions, and model requests.
- Rate-limit, quota, retry, and resource-exhaustion tests.
- File parser size/depth and malware-handling tests.

### Pre-release review

- Threat-model review for every new integration type and privileged feature.
- Dependency and container scanning.
- SAST and secret scanning.
- DAST against staging.
- Manual penetration test focused on tenant boundaries, tool broker, identity exchange, and SSRF before public launch.
- Backup restoration and incident-response exercise.
- Review provider data handling and customer-facing privacy disclosures.

### Release gates

A release must not ship if:
- A critical/high-severity vulnerability is unresolved without documented risk acceptance.
- Tenant isolation or authorization tests fail.
- Secrets are exposed in browser bundles, logs, or source control.
- Tool execution can bypass the broker or policy enforcement.
- A protected action proceeds when authorization is missing or indeterminate.
- An integration can reach prohibited internal network destinations.

## 10. Risk register

| Risk | Initial severity | MVP mitigation / release gate |
|---|---|---|
| Cross-tenant data leakage | Critical | RLS, tenant-aware queries, composite constraints, negative tests |
| Broad service credential used for end-user data | Critical | Delegated tokens or customer authorization proxy; explicit restricted policy |
| Prompt injection triggers unauthorized tool action | Critical | Deterministic broker authorization, tool allowlists, confirmation |
| SSRF through customer-configured MCP endpoint | High | Egress proxy, strict destination validation, DNS/redirect checks |
| Provider or integration secret leakage | Critical | Managed secret store, redaction, least privilege, rotation |
| Identity assertion replay or forgery | High | Signature/claim validation, short TTL, replay protection |
| Cost exhaustion | High | Quotas, budgets, bounded agent loops, rate limits |
| Malicious document ingestion | High | Sandboxed parsers, limits, malware scanning, ACL-aware retrieval |
| Compromised admin account | Critical | MFA, least privilege, step-up, audit, break-glass controls |
| Vulnerable dependency or CI compromise | High | Lockfiles, scanning, scoped CI credentials, protected releases |

Severity is a starting qualitative assessment and should be revised after architecture, deployment, and penetration-test evidence.

## 11. MVP security implementation order

### Before the first end-to-end prototype
- Tenant-scoped data model and authorization middleware.
- Secret storage abstraction and log redaction.
- Tool broker as the only execution path.
- Explicit model credential routing and no silent BYOK fallback.
- Anonymous session restrictions and rate limits.

### Before connecting real customer systems
- SSRF-safe outbound connector infrastructure.
- Integration/tool review and explicit enablement.
- Identity exchange for authenticated customer users.
- Customer-side authorization strategy for private data.
- Audit events and credential rotation/revocation.

### Before public beta
- RLS and cross-tenant negative test suite.
- Prompt-injection and malicious tool-output tests.
- Quotas, budget controls, and abuse monitoring.
- File-ingestion sandboxing if uploads are enabled.
- Incident runbooks, backup restore test, and external security review.

## 12. Open decisions

1. Which cloud provider, managed secret store, and KMS will be used?
2. Will outbound MCP/API traffic run through a dedicated egress proxy from the first customer integration?
3. Which SAST, DAST, dependency, container, and secret-scanning tools fit the initial budget?
4. What are the retention periods for conversations, tool traces, and security events?
5. Which customer data classes may be sent to each supported model provider?
6. Which actions require explicit end-user confirmation versus workspace-owner approval?

Resolve these decisions before public beta. Record accepted risks and compensating controls rather than leaving them implicit.
