# Technology Stack and Project Structure

## 1. Purpose

This document turns the product and architecture decisions into a practical implementation blueprint for the Universal AI Agent SaaS.

The goal is to start with a maintainable modular monolith, preserve clear module boundaries, and avoid introducing distributed-system complexity before scale requires it. All inference is through hosted model APIs; local model runtimes are not part of the architecture.

Technology choices below are recommended defaults, not irreversible commitments. Pin exact package versions in lockfiles when implementation begins and validate current compatibility before adoption.

## 2. Technology decisions

### 2.1 Recommended baseline

| Area | Recommended technology | Reason / notes |
|---|---|---|
| Language | TypeScript | Shared types across dashboard, API, widget, and workers |
| Monorepo | pnpm workspaces + Turborepo | Simple package management and task orchestration |
| Web application | Next.js (App Router) | Dashboard, marketing pages, server-rendered UI |
| UI | React + Tailwind CSS + shadcn/ui | Accessible, composable dashboard components |
| API | Next.js route handlers initially; dedicated Fastify service if needed | Avoid premature service split; isolate API module contracts |
| Validation | Zod | Runtime validation for API, config, and tool schemas |
| ORM / migrations | Drizzle ORM + drizzle-kit | Type-safe SQL-oriented data access and migrations |
| Primary database | Managed PostgreSQL | Relational data, transactions, tenant constraints |
| Vector search | pgvector | Keep knowledge retrieval close to tenant metadata |
| Cache / ephemeral coordination | Managed Redis-compatible service, optional for MVP | Rate limits, short-lived coordination, caching |
| Queue | Managed durable queue or PostgreSQL-backed job queue initially | Reliable ingestion and async work; choose based on hosting |
| Object storage | S3-compatible private bucket | Original files, exports, large artifacts |
| Model SDK | Vercel AI SDK or provider-native SDKs behind adapters | Streaming and tool calling; keep provider contract internal |
| Initial model gateway | OpenRouter adapter | Multi-provider hosted inference; workspace BYOK and platform routing |
| Additional model adapter | Google Gemini API adapter | Optional direct provider path, separate from OpenRouter |
| Integration protocol | MCP remote Streamable HTTP first | Standardized customer tool connections |
| Other connectors | OpenAPI adapter later | Broader API coverage after MCP baseline |
| Authentication | Managed auth provider or mature auth library | Avoid custom password/OAuth implementation |
| Secrets | Cloud secret manager + envelope encryption | Integration and provider credentials |
| Observability | OpenTelemetry + structured logs + error tracking | Correlated traces across runtime and workers |
| Testing | Vitest, Playwright, testcontainers where practical | Unit, integration, and browser coverage |
| Deployment | Cloud-hosted Next.js plus managed Postgres, storage, queue | Low-ops starting point |

### 2.2 Important architectural constraints

- The model gateway is an internal abstraction. Business logic must not call OpenRouter or Gemini SDKs directly.
- Provider selection, credential ownership, and fallback behavior are policy decisions, not model SDK defaults.
- A workspace BYOK credential failure must be surfaced. Never silently use another key/provider/model. Platform fallback is possible only when an administrator explicitly enables and configures it.
- Keep MCP transport and execution behind the Integration service and Tool Execution Broker.
- Keep customer-user authorization separate from workspace dashboard authorization.
- All tenant-owned database operations must be workspace-scoped and protected by RLS.
- Do not place secrets in client-side code or NEXT_PUBLIC environment variables.
- Hosted model APIs are the only supported inference execution mode for this product.

### 2.3 Model provider architecture

Define a provider-neutral interface around the capabilities the runtime actually needs:

- Model discovery/catalog metadata.
- Text generation and streaming.
- Tool/function calling.
- Structured output where supported.
- Usage and token accounting.
- Provider error normalization.
- Capability and context-window metadata.
- Credential validation where provider APIs support it.

Suggested internal components:

1. **Model Catalog** — providers, models, capabilities, availability metadata.
2. **Credential Resolver** — resolves platform-managed or workspace BYOK credential references.
3. **Routing Policy** — chooses the configured provider/model for the workspace and request.
4. **Provider Adapter** — translates the internal request into provider-specific SDK calls.
5. **Usage Recorder** — records provider usage and cost metadata.
6. **Fallback Evaluator** — checks only explicitly configured fallback routes and policy.

OpenRouter can be the first adapter to simplify access to multiple hosted models. Add direct Gemini or other provider adapters when they offer a concrete benefit, such as a required capability, pricing arrangement, or customer preference. Do not treat a model aggregator as the source of truth for authorization or workspace policy.

## 3. Monorepo layout

Recommended initial structure:

~~~text
chat-agent/
├── apps/
│   ├── web/                     # Next.js dashboard and public pages
│   ├── widget/                  # Embeddable chat widget
│   └── worker/                  # Background job process
├── packages/
│   ├── agent-runtime/           # Agent loop, prompt assembly, orchestration
│   ├── api-contracts/           # Shared request/response schemas and types
│   ├── auth/                    # Platform auth and customer identity exchange
│   ├── authorization/           # Policies, permissions, decision helpers
│   ├── config/                  # Environment and typed configuration
│   ├── database/                # Drizzle schema, migrations, repositories
│   ├── integrations/            # MCP and future connector adapters
│   ├── model-gateway/           # Provider routing and provider adapters
│   ├── observability/           # Logging, tracing, metrics and redaction
│   ├── security/                # Secret interfaces, crypto helpers, SSRF policy
│   ├── storage/                 # Object storage abstraction
│   ├── ui/                      # Shared dashboard UI components
│   └── utils/                   # Small generic utilities only
├── infra/
│   ├── docker/                  # Local development service definitions
│   ├── deployment/              # Deployment configuration and runbooks
│   └── scripts/                 # Operational scripts
├── docs/
├── tests/
│   ├── integration/
│   ├── security/
│   └── e2e/
├── .github/
│   └── workflows/
├── .env.example
├── .gitignore
├── package.json
├── pnpm-workspace.yaml
├── turbo.json
└── README.md
~~~

Do not create every package as an independent deployable service. Packages are code boundaries inside one application initially.

## 4. Application and module responsibilities

### 4.1 apps/web

Responsibilities:
- SaaS dashboard and workspace settings.
- Workspace/member/provider/integration/knowledge management screens.
- API route handlers and server actions for dashboard workflows.
- Public identity-exchange and chat endpoints where appropriate.
- Authentication callbacks and protected dashboard routing.

Rules:
- Route handlers should be thin: validate input, authenticate, authorize, call an application service, and map the result to HTTP.
- Put business logic in package-level application services, not React components.
- Keep privileged credentials server-side.
- Do not expose internal admin APIs through public widget routes.

### 4.2 apps/widget

Responsibilities:
- Embeddable chat UI.
- Anonymous session bootstrap.
- Authenticated session token handling after customer backend exchange.
- Streaming message display, citations, and permitted user confirmations.

Rules:
- Use an isolated iframe or equivalent strong embedding boundary for the initial widget.
- Public installation identifiers are not secrets.
- Never accept a browser-supplied customer role or trusted user ID as authentication.
- Do not store provider or integration credentials in the widget.
- Render model/tool output safely; never inject arbitrary HTML.

### 4.3 apps/worker

Responsibilities:
- Knowledge ingestion and chunking.
- Embedding generation using hosted embedding APIs.
- Async exports, cleanup, scheduled maintenance, and retryable jobs.
- Long-running integration tasks.

Rules:
- Revalidate tenant, actor, and policy context at job execution time.
- Use idempotency keys and bounded retries.
- Use dead-letter handling and observable job states.
- Do not assume a queue payload is trusted merely because it came from an internal producer.

### 4.4 packages/agent-runtime

Responsibilities:
- Agent turn lifecycle and bounded orchestration.
- Prompt/context assembly.
- Model/tool iteration.
- Conversation state management.
- Streaming event production.
- Tool proposal dispatch to the Tool Execution Broker.

It must not:
- Execute MCP tools directly.
- Make authorization decisions based on model text.
- Read secrets directly from environment variables or database rows.
- Select a credential or fallback route outside Model Gateway policy.

### 4.5 packages/model-gateway

Responsibilities:
- Model catalog and capabilities.
- Workspace routing and credential resolution.
- OpenRouter and direct provider adapters.
- Streaming and tool-call normalization.
- Provider error handling and usage metadata.
- Explicit fallback policy evaluation.

Recommended provider interface (illustrative):

~~~ts
export interface ModelProvider {
  readonly providerId: string;

  generate(
    request: NormalizedModelRequest,
    credential: ResolvedCredential,
    options: ProviderRequestOptions
  ): Promise<NormalizedModelResponse>;

  stream(
    request: NormalizedModelRequest,
    credential: ResolvedCredential,
    options: ProviderRequestOptions
  ): AsyncIterable<NormalizedModelEvent>;

  validateCredential?(
    credential: ResolvedCredential
  ): Promise<CredentialValidationResult>;
}
~~~

The credential object should be short-lived in memory and must never be serialized into logs, traces, job payloads, or client responses.

### 4.6 packages/integrations

Responsibilities:
- MCP connection lifecycle and transport.
- Tool discovery and schema normalization.
- Integration credential references.
- Tool versioning and review state.
- Connector health checks.

The Tool Execution Broker should be a distinct module boundary, even if implemented in this package initially. It owns:
- Tool allowlist and enablement checks.
- Argument validation.
- Actor and workspace authorization.
- Customer authorization strategy.
- Approval requirements.
- Egress/SSRF enforcement.
- Execution timeout and output limits.
- Audit events.

### 4.7 packages/auth and packages/authorization

Keep these separate:

- auth: verifies sessions, identity assertions, issuers, and authentication state.
- authorization: evaluates permissions and policy for a verified actor and resource.

Authentication middleware should produce a trusted actor context. Authorization functions should consume that context and return explicit allow/deny decisions with reason codes.

### 4.8 packages/database

Responsibilities:
- Drizzle schema and migrations.
- Repositories and transaction helpers.
- Tenant context setup for RLS.
- Query-level constraints and indexes.
- Database health checks.

Rules:
- Avoid importing raw database clients throughout the application.
- Prefer repository methods that require workspace scope as an explicit argument.
- Set RLS tenant context within the same transaction as tenant queries.
- Runtime roles must not own tables or bypass RLS.
- Use composite constraints for workspace-owned relationships.

### 4.9 packages/security

Responsibilities:
- Secret manager interface and envelope-encryption integration.
- Redaction helpers.
- SSRF-safe URL and IP validation.
- Constant-time comparisons where appropriate.
- Cryptographic key and token helper abstractions.

Use platform-maintained cryptographic libraries. Do not invent encryption formats or token protocols.

## 5. Dependency direction

Use a one-way dependency graph:

- Apps may depend on packages.
- Agent Runtime may depend on API contracts, Model Gateway interfaces, authorization interfaces, and conversation repositories.
- Model Gateway may depend on provider SDK adapters, credential resolver interfaces, and usage interfaces.
- Integrations may depend on authorization, security, contracts, and persistence interfaces.
- Database should not import application services or UI code.
- Shared utilities must remain small and must not become a dumping ground for business logic.

Avoid circular dependencies. Where two modules need to communicate, define an interface at the boundary and inject the implementation.

## 6. Core request flow

### 6.1 Chat request

1. Widget sends a message with its scoped chat session.
2. Web/API validates the session and resolves trusted actor/workspace context.
3. Conversation service verifies conversation ownership and workspace scope.
4. Agent Runtime loads only authorized conversation and knowledge context.
5. Model Gateway resolves the workspace routing policy and the exact credential binding.
6. Provider adapter streams model output.
7. If the model proposes a tool call, Runtime passes the proposal to the Tool Execution Broker.
8. Broker resolves the approved tool, validates arguments, and performs authorization and approval checks.
9. Connector executes with scoped credentials and safe egress.
10. Sanitized tool result returns to Runtime as untrusted data.
11. Runtime continues within configured step/time/token limits.
12. Response stream is sent to the widget; usage, audit, and trace metadata are recorded.

### 6.2 Knowledge ingestion

1. API authenticates and authorizes the workspace actor.
2. File is stored in private object storage with tenant-scoped metadata.
3. An ingestion job is enqueued with an opaque file ID and workspace reference.
4. Worker reloads the file record and verifies current state and policy.
5. Sandboxed parser extracts text under size/time/depth limits.
6. Content is chunked and embedded through a configured hosted embedding provider.
7. Chunks and vectors are persisted with workspace, source, and ACL metadata.
8. Retrieval filters by workspace and actor authorization before returning context.
9. Deletion propagates to originals, chunks, vectors, caches, and derived artifacts.

## 7. API design conventions

- Version public APIs under /v1.
- Use Zod schemas at every external boundary.
- Use stable error codes and safe messages; do not expose stack traces or secrets.
- Include request/correlation IDs.
- Use cursor pagination for potentially large collections.
- Use idempotency keys for retryable mutations and tool actions.
- Keep public widget endpoints separate from dashboard/admin endpoints.
- Keep internal worker/broker endpoints inaccessible from the public internet.
- Use explicit DTOs; do not return ORM models or secret metadata wholesale.
- Document APIs with OpenAPI where it improves customer integration and SDK generation.

Suggested route groups:

~~~text
/v1/dashboard/*             # SaaS users; workspace permissions
/v1/widget/*                # public installation and chat session APIs
/v1/identity/exchange       # customer backend identity handoff
/v1/integrations/*          # workspace integration management
/v1/knowledge/*             # knowledge source management
/v1/platform/*              # platform-admin-only operations
/internal/*                 # private service and worker operations
~~~

## 8. Data and infrastructure choices

### PostgreSQL and pgvector

Use managed PostgreSQL with backups, point-in-time recovery where available, TLS, and private networking where supported. Enable pgvector for embeddings. Keep relational authorization metadata alongside vectors and filter before returning results.

### Redis-compatible cache

Redis is optional at the beginning. Add it when required for distributed rate limiting, short-lived coordination, or cache workloads. Never use cache as the sole source of truth for authorization, membership, or credential ownership.

### Durable jobs

Choose one queue mechanism that supports retries, visibility/timeouts, dead-letter handling, and operational inspection. For a small MVP, a PostgreSQL-backed queue can reduce infrastructure cost. Move to a managed queue when throughput, latency, or database contention justifies it.

### Object storage

Use a private S3-compatible bucket with tenant-scoped object keys, encryption, lifecycle policies, and short-lived signed URLs. Do not make uploaded customer documents public.

### Secrets and encryption

Use the cloud provider's managed secret store and KMS where available. For local development, use a local-only secrets file excluded from Git or a developer secret manager. Do not commit real credentials or production exports.

## 9. Environment configuration

Maintain a validated server-side configuration schema. Fail startup when required variables are missing or invalid.

Illustrative .env.example:

~~~dotenv
NODE_ENV=development
APP_BASE_URL=http://localhost:3000
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/chat_agent
REDIS_URL=
QUEUE_DRIVER=database
OBJECT_STORAGE_ENDPOINT=http://localhost:9000
OBJECT_STORAGE_BUCKET=chat-agent-dev
OBJECT_STORAGE_REGION=us-east-1
OBJECT_STORAGE_ACCESS_KEY_ID=
OBJECT_STORAGE_SECRET_ACCESS_KEY=
AUTH_SECRET=
SECRET_MANAGER_DRIVER=local
OPENROUTER_API_KEY=
GEMINI_API_KEY=
OTEL_EXPORTER_OTLP_ENDPOINT=
~~~

This file contains placeholders only. Real provider keys should normally be entered through the secure admin/BYOK credential workflow and stored in the secret system, not copied into browser-exposed configuration.

Separate configuration by environment:
- Local development.
- Automated test.
- Preview/staging.
- Production.

Never reuse production credentials or customer data in local development.

## 10. Local development environment

Use Docker Compose for infrastructure dependencies only; run the Next.js app and worker with the selected package manager for fast reload.

Suggested local services:
- PostgreSQL with pgvector extension.
- Optional Redis.
- S3-compatible local object storage such as MinIO, if needed.
- Optional mail catcher for authentication emails.

Do not require local model serving. Model calls use hosted provider APIs, with mocked adapters for deterministic tests and development without API spend.

Suggested developer commands (final names depend on package scripts):

~~~bash
pnpm install
docker compose -f infra/docker/compose.yml up -d
pnpm db:migrate
pnpm dev
pnpm test
pnpm lint
pnpm typecheck
~~~

A test mode should provide fake model and fake MCP adapters so most development and CI tests do not require paid API calls or live customer systems.

## 11. Zero-cost-first development strategy

The project should minimize fixed infrastructure cost while preserving production security boundaries.

| Capability | Cost-conscious starting point | Important limitation |
|---|---|---|
| Source control | GitHub free tier | Team and advanced security features may have plan limits |
| Web hosting | Free developer tier where suitable | Usage caps, sleeping/limits, and commercial terms can change |
| Database | Local Docker for development; free managed tier for experiments | Free tiers may have storage, compute, backup, or sleep limits |
| Model inference | Provider free quotas where currently offered; otherwise small BYOK budgets | Quotas and model availability are not guaranteed |
| Model aggregation | OpenRouter with a selected free/low-cost model when available | Free model routing can change and is not a production SLA |
| Object storage | Local MinIO in development; limited free cloud allowance for experiments | Egress and request charges may apply |
| Queue | PostgreSQL-backed queue initially | Adds load to primary database |
| Observability | OpenTelemetry with local/dev collector and limited free hosted plans | Retention and event caps vary |
| CI | GitHub Actions free allowance | Minutes and concurrency depend on plan |
| Email | Development mail catcher; limited provider sandbox for testing | Production sending and deliverability may incur cost |

Do not promise that the production service can operate indefinitely at zero cost. Hosted inference, database backups, storage, bandwidth, logs, and abuse protection can all incur charges. Add budget alerts, per-tenant quotas, hard usage limits, and a kill switch before inviting public traffic.

## 12. Testing strategy

### Unit tests
- Routing policy and credential selection.
- Provider adapter normalization.
- Permission evaluation and policy conditions.
- Tool schema validation and approval binding.
- Secret redaction and SSRF validation.
- Context assembly and output filtering.

### Integration tests
- PostgreSQL migrations and RLS using the runtime database role.
- Tenant-aware repositories and composite constraints.
- Identity exchange and session revocation.
- Model gateway with mocked provider adapters.
- MCP lifecycle and Tool Execution Broker.
- Queue retry, idempotency, and dead-letter behavior.
- Object storage upload, access, and deletion.

### End-to-end tests
- Workspace onboarding and configuration.
- Anonymous visitor chat with public-only tools.
- Authenticated identity handoff and user-scoped access.
- BYOK configuration, failure behavior, and explicit platform fallback.
- Knowledge ingestion, retrieval, and deletion.
- Tool confirmation and audit history.

### Security tests
- Cross-tenant access and IDOR.
- Prompt injection through files and tool output.
- SSRF, redirects, DNS rebinding, and private IP blocks.
- Secret exposure in logs, traces, browser bundles, and errors.
- Session replay, privilege escalation, and stale authorization.
- Rate-limit and budget enforcement.

CI should use fake model and connector adapters by default. Live-provider smoke tests should be optional, separately credentialed, budget-limited, and never required for every pull request.

## 13. CI/CD and deployment

### CI pipeline

For every pull request:
1. Install from lockfile.
2. Format/lint and typecheck.
3. Run unit tests.
4. Run database migration validation and integration tests.
5. Run security checks for dependencies and secrets.
6. Build all deployable apps.
7. Run selected browser tests.

Protect the production branch with review and required CI checks before public release.

### Deployment shape

Initial cloud deployment can use:
- One Next.js web deployment.
- One worker deployment/process.
- Managed PostgreSQL.
- Private object storage.
- Managed secret manager/KMS.
- Managed queue or PostgreSQL-backed jobs.
- Optional Redis.
- Centralized logs, metrics, traces, and error reporting.

Keep the worker independently scalable even if it shares packages and database schema with the web app. Keep connector egress isolated and controlled.

### Release process

- Deploy migrations using a dedicated migration identity.
- Prefer backward-compatible expand/migrate/contract schema changes.
- Deploy application and worker versions in a coordinated sequence.
- Verify health checks, queue lag, error rates, provider failures, and spend.
- Roll back application releases when safe; database rollback must be planned rather than assumed.
- Maintain tested backups and a restoration runbook.

## 14. Implementation sequence

### Milestone 1 — Repository foundation
- Initialize pnpm workspace and Turborepo.
- Create Next.js web app, widget app, and worker shell.
- Add shared TypeScript, lint, formatting, test, and build configuration.
- Add CI workflow and .env.example.
- Add Docker Compose for PostgreSQL/pgvector and optional Redis/object storage.

### Milestone 2 — Data and identity foundation
- Implement Drizzle schema and migrations.
- Implement workspace context and RLS.
- Add SaaS dashboard authentication.
- Add workspace membership and permission enforcement.
- Add anonymous chat sessions.

### Milestone 3 — Hosted model gateway
- Implement normalized provider contract.
- Add OpenRouter adapter.
- Add credential resolver and workspace BYOK binding.
- Add provider/model catalog and usage recording.
- Add explicit routing and fallback policy.
- Add fake provider for tests.

### Milestone 4 — Agent runtime and widget
- Implement conversation lifecycle and streaming.
- Add bounded agent loop and prompt assembly.
- Build iframe widget.
- Add public-only chat path and authenticated session support.

### Milestone 5 — MCP and tool execution
- Implement remote MCP connection and discovery.
- Add review/enablement lifecycle.
- Implement Tool Execution Broker and authorization pipeline.
- Add egress/SSRF controls and tool approvals.
- Add mocked MCP server for tests.

### Milestone 6 — Knowledge and background processing
- Implement private file storage.
- Add sandboxed extraction and chunking.
- Add hosted embedding adapter and pgvector retrieval.
- Add ACL-aware retrieval, deletion, and worker retries.

### Milestone 7 — Hardening and beta readiness
- Complete security regression suite.
- Add quotas, budgets, observability, and incident runbooks.
- Verify backup restoration and tenant deletion.
- Perform external security review before public beta.

## 15. Architecture decision records to create during implementation

Create concise ADRs for decisions that may be revisited:

- ADR-001: Monorepo and modular monolith.
- ADR-002: Next.js API boundary and when to split a dedicated API service.
- ADR-003: Authentication provider and customer identity assertion format.
- ADR-004: Model Gateway provider interface and OpenRouter adapter.
- ADR-005: BYOK credential storage and explicit fallback policy.
- ADR-006: MCP transport, egress model, and tool approval lifecycle.
- ADR-007: Queue implementation and worker deployment.
- ADR-008: Object storage and file-ingestion isolation.
- ADR-009: Tenant RLS and repository conventions.
- ADR-010: Observability, retention, and redaction.

## 16. Exit criteria before coding the full MVP

- The repository structure and package ownership are agreed.
- The exact runtime and package manager versions are pinned.
- Database, auth, queue, object storage, and deployment providers are selected.
- Model Gateway contract and credential-routing rules are tested independently.
- Tenant context and RLS are demonstrated with automated tests.
- Tool execution can occur only through the broker.
- Local development works without local inference and without requiring live paid model calls for routine tests.
- CI builds and tests the monorepo from a clean checkout.
