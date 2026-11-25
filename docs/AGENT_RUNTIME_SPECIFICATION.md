# Agent Runtime Specification

**Status:** Normative MVP behavior. This document defines the execution contract for the single universal agent in each workspace.

## 1. Purpose and boundaries

The runtime converts a verified user message into a safe, policy-governed response using configured model providers, authorized context, approved knowledge and permitted tools. It does not own customer application authorization, execute arbitrary code, or treat retrieved content as instructions.

Core components:
1. Request/session verifier
2. Conversation state manager
3. Context assembler
4. Policy and tool planner
5. Model gateway client
6. Tool Execution Broker
7. Streaming/event publisher
8. Persistence and usage recorder

## 2. Request lifecycle

1. Authenticate caller and validate session token.
2. Resolve workspace, agent, conversation, actor and allowed capabilities.
3. Apply request, abuse and quota limits.
4. Persist user message idempotently.
5. Load bounded conversation history and relevant summaries.
6. Retrieve authorized knowledge and context.
7. Resolve effective model route and credential source.
8. Build model request from trusted prompt layers and explicitly labeled untrusted data.
9. Execute model turn; validate proposed tool calls against current policy.
10. Execute approved tools through the broker, then append bounded results.
11. Continue model turn within iteration/time/token budgets.
12. Persist assistant result, citations, tool audit records and usage.
13. Emit terminal event and release resources.

Each accepted request must end in completed, failed, cancelled or expired state. A worker crash must be recoverable through durable job state and idempotent transitions.

## 3. Prompt and trust boundaries

Prompt layers, in precedence order:
- Platform safety and isolation rules (immutable)
- Workspace policy and agent instructions (validated, bounded)
- Runtime task instructions
- Conversation history
- Retrieved knowledge and tool outputs (untrusted data)
- Current user message (untrusted input)

Content from users, web pages, documents, MCP tools and customer systems is data, not policy. Delimit and label it. Never concatenate untrusted text into privileged instruction sections. The model's output is not authorization. Tool execution requires deterministic server-side checks regardless of model intent.

Never expose internal reasoning, hidden system prompts, provider credentials, signing material, other tenant content or internal risk scores.

## 4. Context assembly

Inputs:
- Conversation history within configured token and age limits
- Persisted summary, if current and provenance tracked
- Verified identity claims needed for the current task
- Workspace-approved profile/context records
- Knowledge retrieval results filtered by tenant, source ACL, status and user authorization
- Tool catalog descriptions for tools enabled in this session

Context assembler must enforce per-source and total budgets, redact sensitive fields, attach provenance and timestamps, deduplicate overlapping records and provide explicit "not found" behavior. Stale or inaccessible sources are omitted, not silently treated as authoritative. Context may not expand user permissions.

## 5. Tool planning and execution

The model may propose tool calls; the broker decides whether they can execute.

Required checks for every call:
- Tool is enabled for the workspace and current environment.
- Tool schema validates strictly; unknown arguments are rejected unless schema allows them.
- Caller identity and action scope are valid.
- Tool risk level and confirmation requirement are satisfied.
- Rate, timeout, concurrency and result-size budgets remain.
- Destination/network policy passes for remote calls.
- Idempotency and duplicate-execution rules are satisfied.
- Tool result is marked untrusted and sanitized before reuse.

MVP tools should be read-only by default. Writes, external messages, payments, account changes, destructive actions and actions with material external side effects require explicit approval unless a separately reviewed policy permits them. The model cannot grant itself approval.

Tool execution states: proposed → policy_checked → awaiting_approval (if required) → running → succeeded/failed/timed_out/cancelled. Audit every transition. Approval is bound to exact tool, arguments hash, actor, workspace and expiry; any argument change invalidates approval.

## 6. Loop and resource limits

Enforce configurable hard limits:
- Maximum model turns per user request
- Maximum tool calls per turn and total
- Maximum concurrent tool calls
- Maximum wall-clock execution time
- Maximum prompt, completion and tool-result tokens/bytes
- Maximum attachment count and size
- Maximum retries per dependency
- Maximum total estimated and actual cost

Defaults are deployment configuration, not hard-coded product promises. Budget exhaustion returns a safe, actionable error and preserves partial progress where appropriate. No infinite agent loops.

## 7. Model gateway interaction

Runtime requests a capability (for example, chat with tools and streaming), not a provider-specific API. The gateway resolves route, credential, model capabilities and limits. Runtime must not implement provider-specific fallback. A BYOK failure is surfaced to the workspace as a stable provider error; it must not silently switch to a platform key, another provider or another model. Platform fallback is only possible when an administrator explicitly enables a policy and the request is eligible under that policy.

## 8. Streaming and cancellation

Send user-visible text deltas as they become available. Tool events expose safe labels/status, not secrets or hidden arguments. Cancellation propagates to model request, queued work and cancellable tools; non-cancellable operations are marked as such and reconciled. Persist the terminal state even if the browser disconnects. Reconnection may replay events using event IDs where retention permits.

## 9. Errors and recovery

| Failure | Runtime behavior |
|---|---|
| Invalid session | Reject before persistence |
| Quota exceeded | Reject or stop before additional billable work |
| Provider authentication error | Surface BYOK/platform credential failure per route policy |
| Provider timeout | Bounded retry only if request is safe and policy allows |
| Tool timeout | Record timeout; model may continue with explicit unavailable result |
| Retrieval unavailable | Continue only if policy allows; disclose reduced context where relevant |
| Worker crash | Recover from durable state; avoid duplicate side effects |
| Client disconnect | Continue or cancel according to configured session policy |
| Moderation/policy block | Stop unsafe action; provide safe response |

Retry only transient failures, use jittered exponential backoff, and honor provider retry-after. Do not retry non-idempotent tools without a verified idempotency mechanism.

## 10. Persistence and observability

Persist message state, model route metadata, provider request IDs, usage/cost, citations, tool call IDs and redacted audit events. Avoid storing full prompts or tool payloads by default. If diagnostic capture is enabled, require explicit retention and access policy with secret/PII redaction.

Trace spans should correlate request → runtime → model call → retrieval → tool call. Metrics include latency, tokens, estimated cost, failures, tool approval/denial, cancellation and queue age. Never use conversation text as metric labels.

## 11. Acceptance criteria

- Cross-tenant context cannot enter a prompt.
- Model-generated tool calls cannot bypass broker policy.
- Tool arguments are validated and approval is bound to an immutable hash.
- Runtime enforces all configured loop and cost limits.
- BYOK failures never silently cross credential/provider boundaries.
- Cancellation and worker recovery do not duplicate side effects.
- Every accepted request reaches one terminal state with traceable usage.
