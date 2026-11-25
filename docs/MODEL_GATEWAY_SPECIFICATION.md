# Model Gateway Specification

**Status:** Normative routing and credential behavior for MVP.

## 1. Responsibility

The gateway provides a provider-neutral interface for hosted model APIs. It owns provider adapters, capability metadata, credential resolution, routing policy, usage normalization, timeout/retry behavior and provider error translation. It does not decide whether a user is authorized to access workspace data or execute tools.

Initial integration preference: OpenRouter as a hosted multi-provider gateway, with a direct Gemini adapter as an optional provider. All providers must be implemented behind the same adapter contract. Availability, pricing, quotas and model IDs are dynamic and must be refreshed from provider sources; do not encode promotional free tiers as guaranteed capacity.

## 2. Adapter contract

Each adapter implements:
- Provider identity and supported authentication modes
- Model/capability discovery or a maintained catalog import
- Request translation for text, streaming, tools, structured output and supported modalities
- Response and streaming event normalization
- Usage normalization (input/output/cached tokens where supplied)
- Error classification and retry hints
- Provider request ID extraction
- Health/circuit-breaker signals

Capabilities must be explicit: context window, max output, tool calling, streaming, JSON/structured output, image/audio support, and known constraints. Unknown capability means unsupported, not assumed supported.

## 3. Credential source and resolution

Credential source values: `workspace_byok`, `platform_managed`. Credential records hold secret-manager references; database stores metadata and encrypted envelope/reference, not plaintext keys.

Resolution precedence:
1. An explicit workspace provider/model binding, if valid and enabled.
2. A workspace default binding for the selected provider, if valid and enabled.
3. A platform default route only if workspace policy permits platform-managed usage and administrator has published an active route.
4. Otherwise return `credential_not_configured`.

A workspace's explicit BYOK binding is authoritative for that provider/model. If it fails authentication, quota, or availability checks, surface the corresponding error; do not fall through to platform credentials, a different provider or a different model. The workspace owner must make any routing change. A platform administrator may define a separately named fallback policy, but it is disabled until explicitly configured and must be visible in effective routing diagnostics.

## 4. Route selection

Inputs:
- Requested capability and optional model preference
- Workspace routing preferences
- Workspace provider/model bindings
- Platform defaults and administrator overrides
- Explicit fallback policy, if active
- Model capability catalog
- Workspace budget and provider availability

The route resolver produces an immutable decision record: provider, model, credential source and credential reference, policy version, reason code, capability match and budget reservation. It must not expose credential values.

Workspace owner may select only models exposed and enabled in the platform catalog. Platform admin may disable a provider/model globally, configure platform credentials, set global defaults, and publish workspace-specific overrides. Workspace overrides are versioned and auditable. Workspace owners can inspect the effective route and reason without seeing platform secret material.

## 5. Failure classification

| Class | Example | Retry | Route switch |
|---|---|---|---|
| Authentication | Invalid/revoked key | No | No silent switch |
| Permission/model access | Model unavailable to key | No | No silent switch |
| Quota/billing | Provider quota exceeded | Only per explicit policy | No silent switch |
| Rate limit | 429 | Bounded, honor retry-after | No silent switch |
| Transient server | 5xx | Bounded retry | Only explicit fallback policy |
| Timeout/network | Connection timeout | Bounded if safe | Only explicit fallback policy |
| Invalid request | Unsupported parameter | No | No |
| Content/policy | Provider refusal | No | No |

For BYOK, the default is strict no-switch behavior for every failure class. If an administrator enables a narrowly scoped platform fallback, the user must be informed and the event audited; fallback must never be used to conceal a broken BYOK key. No automatic fallback across workspace BYOK keys.

## 6. Usage, budgets and billing records

Record provider/model, credential source, request ID, input/output/cached token counts, provider-reported cost when available, normalized estimated cost, currency, latency, outcome and pricing catalog version. Mark estimates distinctly from provider-reported charges. Never invent cost when usage/pricing is unavailable.

Reserve estimated workspace budget before billable execution and reconcile actual usage after response. Handle concurrent reservations atomically. Define policies for unknown pricing, budget exhaustion, refunds/failed requests, retries and provider charges that arrive late. Workspace owners configure alerts and limits; platform operators set global safety caps.

## 7. Security

- Never log or return API keys.
- Restrict credential access to gateway service identity.
- Use KMS/secret manager envelope encryption and rotation.
- Redact provider headers and error bodies.
- Validate provider/model identifiers against catalog to prevent arbitrary endpoint selection.
- Egress allow-list provider hosts; block user-controlled base URLs.
- Separate credential metadata from secret material.
- Audit create, rotate, test, bind, unbind and revoke events.
- Prevent cross-workspace credential reference lookup.

## 8. Operational behavior

Use per-provider connection and request timeouts, bounded concurrency, circuit breakers and provider-specific rate limits. A circuit breaker reports unhealthy status; it does not itself authorize a route switch. Catalog updates are versioned and can be rolled back. Adapter contract tests use deterministic fakes; provider smoke tests are optional and must not run in ordinary CI.

## 9. Acceptance criteria

- Effective route is deterministic and explainable.
- BYOK binding failures do not silently use platform keys or another model.
- Platform fallback is off until explicitly configured and auditable.
- Provider secrets never appear in API responses, logs or traces.
- Usage and cost distinguish provider-reported from estimated values.
- Unsupported model capabilities are rejected before provider invocation.
