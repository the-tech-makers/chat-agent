# Architecture Decision Records

This file is the index and current decision log. Detailed records should be added as numbered files under `docs/adr/` when implementation validates or changes a decision. A decision may be superseded; never silently rewrite history.

## ADR-001 — Cloud-hosted deployment

**Status:** Accepted  
**Decision:** Host the SaaS and agent runtime in cloud infrastructure.  
**Reason:** Centralized operations, managed services and multi-tenant delivery.  
**Consequences:** Cloud provider and service selection remain deployment decisions; data residency, cost, portability and vendor lock-in require review.

## ADR-002 — One universal agent per workspace for MVP

**Status:** Accepted  
**Decision:** Each workspace configures one universal agent rather than separate specialized agents.  
**Reason:** Simplifies configuration, identity, policy and customer onboarding.  
**Consequences:** Agent behavior is configured through tools, context, knowledge and workspace policy. Multiple agents are post-MVP.

## ADR-003 — Hybrid model credentials with strict BYOK boundaries

**Status:** Accepted  
**Decision:** Support workspace BYOK and platform-managed provider credentials. BYOK failures do not silently fall back to platform credentials or another model/provider. Platform fallback is disabled until explicitly configured by an administrator.  
**Reason:** Transparent customer control and predictable cost/security boundaries.  
**Consequences:** Route resolution, credential source and fallback policy must be auditable and visible.

## ADR-004 — Hosted model APIs only

**Status:** Accepted  
**Decision:** Use hosted model APIs; do not include local model hosting or Ollama in the product architecture.  
**Reason:** Focus on managed SaaS operation and provider abstraction.  
**Consequences:** Provider availability, pricing, quotas, data processing terms and network egress become operational dependencies.

## ADR-005 — MCP-first integration strategy

**Status:** Accepted  
**Decision:** MCP is the first integration protocol; OpenAPI and controlled database adapters may follow.  
**Reason:** Standardized tool discovery and invocation.  
**Consequences:** Tool policy, actor authorization, approval and SSRF defenses are mandatory.

## ADR-006 — Customer application remains authorization authority

**Status:** Accepted  
**Decision:** The SaaS verifies customer identity assertions but does not invent or replace customer application permissions.  
**Reason:** Customer application owns its user accounts and access model.  
**Consequences:** Sensitive actions require customer-authorized checks or scoped connector credentials; signed identity alone is not blanket permission.

## ADR-007 — Modular monolith first

**Status:** Accepted  
**Decision:** Begin with a modular monolith and separate worker process, maintaining package boundaries.  
**Reason:** Reduces operational complexity while preserving future extraction paths.  
**Consequences:** Enforce dependency direction and module contracts from the start.

## ADR-008 — PostgreSQL and pgvector

**Status:** Accepted baseline  
**Decision:** Use PostgreSQL as primary relational store and pgvector for initial vector retrieval.  
**Reason:** Consolidated transactional and vector storage.  
**Consequences:** Vector index strategy and capacity must be benchmarked; avoid premature separate vector infrastructure.

## ADR-009 — Iframe-isolated widget

**Status:** Accepted baseline  
**Decision:** Deliver an embeddable chat widget in a cross-origin iframe.  
**Reason:** Isolation from customer page styles and scripts.  
**Consequences:** Requires strict postMessage protocol, origin validation and accessible iframe UX.

## ADR-010 — OpenRouter-first hosted gateway adapter

**Status:** Preferred starting point, subject to provider and legal validation  
**Decision:** Implement a provider-neutral gateway with OpenRouter as a likely first adapter and optional direct Gemini adapter.  
**Reason:** Hosted multi-provider access behind a single integration surface.  
**Consequences:** Must verify model availability, data handling, pricing and tool/streaming compatibility. Never assume Google Antigravity is a production inference API.

## ADR-011 — Zero-cost-first development, not zero-cost production

**Status:** Accepted  
**Decision:** Prefer free tiers and local infrastructure for development where safe, with fake providers in CI.  
**Reason:** Keep early experimentation affordable.  
**Consequences:** Free quotas are variable and not production guarantees; production requires budget and capacity planning.
