# Quality Assurance and Automated Test Strategy

## 1. Purpose

Define how Chat Agent is verified from the first commit through production. Tests must protect tenant isolation, identity, authorization, credential routing and tool execution as first-class requirements—not only test happy-path chat responses.

The machine-readable catalogue is [tests/cases/catalog.json](../tests/cases/catalog.json). Stable test IDs are referenced by the [traceability register](TRACEABILITY_AND_TASK_REGISTER.md).

## 2. Test principles

1. **Test boundaries before features:** tenancy, identity, authorization, secrets and outbound network controls are foundational.
2. **Deterministic by default:** use fake model providers, fake MCP servers, seeded databases and controlled clocks. CI must not require paid model API calls.
3. **Test both allow and deny paths:** every security rule needs positive and negative cases.
4. **Test isolation with at least two tenants:** single-tenant fixtures cannot prove isolation.
5. **Contract test external adapters:** provider and MCP behavior is validated against stable contracts.
6. **No secrets in fixtures:** use clearly fake tokens and keys; never commit real credentials or customer data.
7. **Test migrations from empty and previous schema:** schema changes are not complete until both paths work.
8. **Keep end-to-end tests few but meaningful:** cover complete critical user journeys.
9. **Treat flaky tests as defects:** quarantine only with an owner, issue and expiry date.
10. **A green build is necessary, not sufficient:** milestone acceptance also requires manual operational/security review where automation cannot establish safety.

## 3. Recommended test stack

| Layer | Recommended tool | Purpose |
|---|---|---|
| Unit | Vitest | Pure functions, policy logic, routing, validation |
| Component | Vitest + Testing Library | Dashboard and widget components |
| API integration | Vitest + Supertest or native fetch harness | Route handlers and service boundaries |
| Database integration | Testcontainers or disposable PostgreSQL service | Real constraints, transactions, RLS, migrations, pgvector |
| MCP integration | In-process fake MCP server + protocol contract tests | Discovery, auth, timeout and tool execution |
| Browser E2E | Playwright | Onboarding, dashboard, widget, guest/authenticated chat |
| Security | Vitest adversarial suites + OWASP ZAP baseline where applicable | SSRF, auth, injection, headers and abuse limits |
| Load | k6 | Throughput, latency, concurrency and rate limiting |
| Dependency/security | pnpm audit plus a maintained scanner such as OSV-Scanner/Trivy | Dependency and image vulnerabilities |

Exact package versions and tool compatibility should be pinned when M0 is implemented. Avoid requiring Docker for pure unit tests; database and browser suites may use CI services/containers.

## 4. Test levels and minimum expectations

### Unit tests
- Test one function/module in isolation.
- Cover valid, invalid, boundary and exceptional inputs.
- No network or real provider credentials.
- Examples: model route resolution, role policy, origin validation, token expiry, schema validation, prompt assembly limits.

### Integration tests
- Exercise multiple real modules and infrastructure boundaries.
- Use PostgreSQL for RLS, foreign keys, uniqueness and transactions.
- Use fake hosted-model and MCP servers.
- Verify retry, timeout, idempotency, audit and secret redaction behavior.

### End-to-end tests
- Exercise a real browser against a running test deployment.
- Cover owner onboarding, agent configuration, widget chat, guest access and authenticated identity.
- Assert user-visible outcomes and server-side state.
- Keep test data isolated per run and clean it up safely.

### Security tests
- Attempt unauthorized and cross-tenant access, not only expected use.
- Include SSRF with redirects/DNS rebinding defenses, forged identity, replay, prompt injection, tool escalation, secret leakage and abuse limits.
- Security tests must fail closed when identity or policy services are unavailable.

### Operational tests
- Health/readiness, graceful shutdown, queue retry/idempotency, backup/restore, rate limiting, telemetry redaction and provider outage behavior.
- Load tests should establish a baseline before public beta; targets must be set from product SLOs rather than invented prematurely.

## 5. CI pipeline and gates

Recommended pull request pipeline, in order:

1. Install from lockfile (pnpm install --frozen-lockfile).
2. Formatting and lint.
3. TypeScript typecheck.
4. Unit tests.
5. Build all affected apps/packages.
6. Database migration + integration tests against PostgreSQL with pgvector.
7. API/MCP/provider contract tests.
8. Playwright smoke tests for critical user flows.
9. Secret/dependency/container scans.
10. Upload test reports, traces and coverage artifacts.

**Required PR checks:** install, lint, typecheck, unit, build, database integration, security baseline. E2E smoke should be required once the M3 widget environment exists. Do not mark checks as passed if a suite was skipped due to missing configuration.

**Release gate:** all required checks green; no open release-blocking defect; migration and rollback reviewed; security and operational checklist signed off.

## 6. Coverage policy

Coverage is a signal, not the completion criterion. Initial targets after test harness setup:
- Critical security and authorization modules: 90%+ branch coverage target.
- Domain/service modules: 80%+ line coverage target.
- UI components: focus on behavior and accessibility rather than chasing line coverage.
- All P0 requirements: at least one automated test mapped to each acceptance criterion.
- Every authorization denial rule: explicit negative test.
- Every provider/MCP adapter: contract suite.

Coverage exclusions must be explicit and reviewed. Do not inflate coverage with tests that only execute code without asserting behavior.

## 7. Test data and environments

- Unit: deterministic in-memory fixtures.
- Integration: ephemeral PostgreSQL with pgvector and transaction cleanup or isolated schemas.
- E2E: dedicated test workspace and seeded fake MCP/model endpoints.
- Staging: synthetic tenant data only unless separately approved.
- Production: no destructive test traffic; use health checks and safe synthetic probes.

Seed at least:
- Tenant Alpha with owner, member and guest identities.
- Tenant Beta with a separate owner and private resources.
- One workspace with a published agent and one with no published agent.
- Fake provider keys and fake MCP credentials.
- Read-only, write and destructive tool definitions.
- Public and private knowledge documents with non-sensitive synthetic content.

## 8. Test case lifecycle

Each test case in tests/cases/catalog.json has:
- Stable id.
- Requirement/task references.
- Priority and test level.
- Preconditions, steps and expected results.
- Automation target and tags.

Lifecycle:
- Draft → Ready → Automated → Active.
- A test can be Blocked only with a linked issue and explanation.
- Retired tests retain their ID and rationale; do not silently delete tests.
- Test IDs must remain stable across refactors.
- When a requirement changes, update its tests in the same pull request.

## 9. Defect severity

| Severity | Definition | Release handling |
|---|---|---|
| S0 Critical | Cross-tenant data exposure, credential compromise, unauthorized destructive action, systemic auth bypass | Stop release; immediate incident process |
| S1 High | Major security control failure or core journey unavailable | Block release |
| S2 Medium | Feature materially degraded with workaround | Fix or formally accept before release |
| S3 Low | Minor UX, cosmetic or low-impact issue | Prioritize in normal backlog |

## 10. Manual verification still required

Automation cannot fully prove:
- Privacy/legal compliance or data processing agreements.
- Provider contractual behavior and quota guarantees.
- Human usability and accessibility quality across all assistive technologies.
- Real-world incident response readiness.
- Backup restoration in the target production account.
- Customer-specific authorization correctness without customer integration review.

Record these as release checklist evidence, not as assumed test passes.

## 11. Milestone test exit gates

| Milestone | Minimum verification |
|---|---|
| M0 Foundation | Clean install, lint, typecheck, build, health endpoint, CI runs |
| M1 Tenancy/database | Migrations, constraints, RLS, cross-tenant denial tests |
| M2 Model gateway | Provider contract, deterministic routing, BYOK no-fallback, key redaction |
| M3 Agent/widget | Runtime unit/integration, guest chat, widget origin and E2E smoke |
| M4 Identity | Signature, expiry, replay, actor binding and authorization tests |
| M5 MCP | Fake-server contract, disabled-by-default, broker authorization, SSRF suite |
| M6 Knowledge | Upload validation, isolation, retrieval attribution, prompt-injection tests |
| M7 Beta hardening | Load, restore, incident runbook, security scans and full E2E regression |

## 12. Running tests

The exact scripts are added in M0. Intended developer interface:

~~~bash
pnpm test
pnpm test:integration
pnpm test:e2e
pnpm test:security
pnpm test:coverage
pnpm lint
pnpm typecheck
pnpm build
~~~

If an individual package exposes a narrower test command, document it in that package's README. CI and local commands should use the same scripts.


## 12. Expanded catalogue and contract validation

The machine-readable catalogue at `tests/cases/catalog.json` contains 72 planned test cases. New coverage includes:
- API idempotency, tenant concealment and error redaction.
- Runtime terminal-state recovery and untrusted retrieval boundaries.
- Model credential resolution, strict BYOK behavior and capability validation.
- Widget session/origin enforcement and postMessage validation.
- Knowledge ACL filtering, atomic re-ingestion and deletion.
- Workspace permission matrix and final-owner safeguards.
- Workspace offboarding, backup tombstone replay and restore drills.
- Typed configuration validation and cross-document contract review.

All entries remain `planned` until executable tests or a documented manual release check exist and evidence is recorded. Do not interpret a catalogue entry as a passing test.

### OpenAPI validation gate

The initial OpenAPI 3.1 file is `docs/openapi/openapi.yaml`. CI should validate its syntax and OpenAPI schema, lint operation IDs and path parameters, and run request/response contract tests against the API implementation. The OpenAPI file currently covers the core workspace, agent, widget session, identity exchange, chat, model catalogue and BYOK credential operations; remaining API catalogue route families must be added before their implementation is considered contract-complete.

### Contract drift gate

For each public operation, verify:
1. Method/path and caller authentication match `API_SPECIFICATION.md`.
2. Permission and tenant scope match identity/admin specifications.
3. Required persistence and constraints exist in the database design or an approved deferral.
4. Error/status semantics match OpenAPI.
5. Positive, negative, cross-tenant and retry/idempotency tests exist.
