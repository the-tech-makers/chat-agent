# Database Design

## 1. Design goals

- PostgreSQL is the system of record; pgvector is used for knowledge embeddings.
- Every tenant-owned row is scoped to an organization and/or workspace.
- Enforce tenant boundaries in application services and with PostgreSQL row-level security (RLS) where practical.
- Store secrets outside ordinary configuration tables; database rows hold references and non-sensitive metadata.
- Keep provider configuration, credential ownership and fallback policy explicit and auditable.
- Use UUID primary keys, foreign keys, UTC timestamps, unique constraints and lifecycle/status fields.

This is a logical MVP schema, not a finalized migration. Validate types and indexes against the selected PostgreSQL version and Drizzle conventions.

## 2. Tenant and membership tables

### users
- id UUID PK
- auth_provider, auth_subject (unique pair)
- email nullable, display_name nullable
- status, created_at, updated_at

Use a managed identity provider where appropriate; do not store passwords when authentication is delegated.

### organizations
- id UUID PK
- name, slug (unique)
- status (active, suspended, deleted)
- created_at, updated_at

### organization_members
- id UUID PK
- organization_id FK -> organizations
- user_id FK -> users
- role (owner, admin, member, billing_viewer)
- status, invited_at, joined_at, created_at

Unique: (organization_id, user_id).

### workspaces
- id UUID PK
- organization_id FK -> organizations
- name, slug
- status, created_at, updated_at

Unique: (organization_id, slug).

### workspace_members
- id UUID PK
- workspace_id FK -> workspaces
- user_id FK -> users
- role (owner, admin, developer, analyst, viewer)
- status, created_at

Unique: (workspace_id, user_id).

## 3. Agent configuration

### agents
- id UUID PK
- workspace_id FK -> workspaces
- name, description
- status (draft, published, disabled)
- active_version_id nullable FK -> agent_versions
- created_at, updated_at

MVP invariant: one universal agent per workspace. Enforce a unique workspace constraint, or a unique partial index by agent kind if future agent types are anticipated.

### agent_versions
- id UUID PK
- agent_id FK -> agents
- version_number
- configuration JSONB (non-secret settings only)
- context_version_id nullable
- created_by_user_id
- published_at, created_at

Unique: (agent_id, version_number). Published versions are immutable; create a new version for changes and rollback by repointing active_version_id.

## 4. Model provider catalog and routing

Provider/model capability metadata is platform-maintained and may be refreshed as APIs change.

### model_providers
- id UUID PK
- provider_key unique (e.g. openrouter, google_gemini)
- display_name
- adapter_key (internal adapter identifier)
- status (enabled, disabled, maintenance)
- documentation_url nullable
- created_at, updated_at

Do not store API keys here.

### models
- id UUID PK
- provider_id FK -> model_providers
- model_key (provider-specific identifier)
- display_name
- status (enabled, disabled, deprecated)
- supports_streaming boolean
- supports_tool_calling boolean
- supports_structured_output boolean
- supports_vision boolean
- context_window_tokens nullable
- input_price_per_million nullable
- output_price_per_million nullable
- pricing_currency default USD
- availability_metadata JSONB
- last_verified_at nullable
- created_at, updated_at

Unique: (provider_id, model_key). Pricing/availability are estimates and must be timestamped; free-tier availability is not permanent.

### provider_credentials
Credential metadata only; secret bytes live in a managed secret store or encrypted vault.
- id UUID PK
- owner_type (platform, workspace)
- owner_workspace_id nullable FK -> workspaces
- provider_id FK -> model_providers
- secret_reference unique
- label
- status (pending_validation, active, invalid, revoked)
- created_by_user_id nullable
- last_validated_at nullable
- last_used_at nullable
- expires_at nullable
- created_at, updated_at

Constraints:
- owner_type=platform implies owner_workspace_id IS NULL.
- owner_type=workspace implies owner_workspace_id IS NOT NULL.
- Workspace-owned credential rows are inaccessible outside that workspace.
- Secret reference must not contain secret material.
- A workspace credential can be provider-wide or model-scoped using the binding table below.

### provider_credential_model_bindings
- id UUID PK
- credential_id FK -> provider_credentials
- model_id FK -> models
- created_at

Unique: (credential_id, model_id).

A provider-wide credential has no model bindings. The service must define scope resolution explicitly and avoid ambiguous matches.

### workspace_model_preferences
- id UUID PK
- workspace_id FK -> workspaces
- task_type (general_chat, tool_use, structured_output, other)
- preferred_model_id FK -> models
- credential_mode (byok, platform_managed)
- selected_credential_id nullable FK -> provider_credentials
- created_at, updated_at

Unique: (workspace_id, task_type).

When credential_mode=byok, selected_credential_id must reference a workspace-owned credential belonging to the same workspace and compatible with the selected model/provider. When platform_managed, selected_credential_id must be null; platform credential resolution uses admin configuration. Enforce this in services and, where practical, database constraints/triggers.

### platform_model_defaults
- id UUID PK
- task_type
- model_id FK -> models
- platform_credential_id FK -> provider_credentials
- enabled boolean
- updated_by_admin_id
- created_at, updated_at

Unique: task_type. The credential must be platform-owned. Do not expose platform credential references to workspace clients.

### workspace_model_overrides
- id UUID PK
- workspace_id FK -> workspaces
- task_type
- model_id FK -> models
- platform_credential_id FK -> provider_credentials
- enabled boolean
- updated_by_admin_id
- reason nullable
- created_at, updated_at

Unique: (workspace_id, task_type). The referenced credential must be platform-owned. Validate ownership in service logic and database constraints/triggers where appropriate.

### platform_fallback_policies
- id UUID PK
- scope_type (global, workspace)
- workspace_id nullable FK -> workspaces
- enabled boolean NOT NULL DEFAULT false
- max_attempts integer NOT NULL DEFAULT 0
- total_timeout_ms nullable
- retryable_error_codes JSONB
- updated_by_admin_id
- created_at, updated_at

Constraints:
- Global scope requires workspace_id IS NULL.
- Workspace scope requires workspace_id IS NOT NULL.
- enabled defaults to false.
- Enabling requires at least one valid fallback route and an authorized platform administrator action.

### platform_fallback_routes
- id UUID PK
- fallback_policy_id FK -> platform_fallback_policies
- source_model_id FK -> models
- fallback_model_id FK -> models
- platform_credential_id FK -> provider_credentials
- priority integer
- enabled boolean
- created_at, updated_at

Unique: (fallback_policy_id, source_model_id, priority). Credential must be platform-owned; routes must satisfy capability requirements. Routes are only eligible for platform-managed requests, never BYOK.

### model_request_logs
- id UUID PK
- workspace_id FK -> workspaces
- agent_id nullable FK -> agents
- conversation_id nullable
- run_id nullable
- provider_id FK -> model_providers
- model_id FK -> models
- credential_owner_type (platform, workspace)
- credential_id nullable FK -> provider_credentials
- outcome (success, error, fallback_success, fallback_error)
- normalized_error_code nullable
- provider_request_id nullable
- input_tokens, output_tokens nullable
- estimated_cost nullable
- latency_ms nullable
- fallback_attempt_number nullable
- created_at

Do not store API keys, authorization headers or sensitive prompt/tool content. Apply retention and access controls.

## 5. Integrations and tools

### integrations
- id UUID PK
- workspace_id FK -> workspaces
- integration_type (mcp, future_openapi, future_database)
- name, endpoint_url
- auth_mode
- credential_reference nullable
- status, last_health_check_at
- configuration JSONB (non-secret only)
- created_at, updated_at

### integration_tools
- id UUID PK
- integration_id FK -> integrations
- external_tool_key
- display_name, description
- input_schema JSONB
- risk_level
- enabled boolean NOT NULL DEFAULT false
- schema_version
- discovered_at, approved_at
- created_at, updated_at

Unique: (integration_id, external_tool_key).

### tool_permissions
- id UUID PK
- workspace_id FK -> workspaces
- integration_tool_id FK -> integration_tools
- actor_scope (guest, authenticated, role, user)
- actor_reference nullable
- permission (deny, allow, require_approval)
- created_at, updated_at

### tool_execution_logs
- id UUID PK
- workspace_id FK -> workspaces
- integration_tool_id FK -> integration_tools
- conversation_id nullable
- actor_id nullable
- status, duration_ms
- normalized_error_code nullable
- approval_id nullable
- created_at

## 6. Context, knowledge and conversations

### context_versions
- id UUID PK
- workspace_id FK -> workspaces
- version_number
- content (reviewed operating context)
- status (draft, published, archived)
- created_by_user_id
- published_at, created_at

### knowledge_sources
- id UUID PK
- workspace_id FK -> workspaces
- name, source_type
- status, created_at, updated_at

### knowledge_documents
- id UUID PK
- workspace_id FK -> workspaces
- source_id FK -> knowledge_sources
- object_key
- original_filename, mime_type, size_bytes, checksum
- ingestion_status
- access_scope JSONB
- created_at, updated_at, deleted_at

### knowledge_chunks
- id UUID PK
- workspace_id FK -> workspaces
- document_id FK -> knowledge_documents
- chunk_index
- content
- embedding vector (dimension selected to match the embedding model)
- metadata JSONB
- created_at

Unique: (document_id, chunk_index). Use a pgvector index and always filter by workspace/access scope before returning results.

### conversations
- id UUID PK
- workspace_id FK -> workspaces
- agent_id FK -> agents
- agent_version_id FK -> agent_versions
- actor_type (guest, authenticated)
- actor_reference nullable
- status, created_at, updated_at

### messages
- id UUID PK
- conversation_id FK -> conversations
- role (system, developer, user, assistant, tool)
- content JSONB or normalized text/content parts
- sequence_number
- created_at

Unique: (conversation_id, sequence_number). Avoid persisting secrets or unnecessary sensitive tool payloads.

### agent_runs
- id UUID PK
- workspace_id FK -> workspaces
- conversation_id FK -> conversations
- status, started_at, completed_at
- model_request_count
- tool_call_count
- error_code nullable

### usage_events
- id UUID PK
- workspace_id FK -> workspaces
- event_type
- quantity, unit
- provider_id nullable FK -> model_providers
- model_id nullable FK -> models
- credential_owner_type nullable
- estimated_cost nullable
- idempotency_key nullable
- occurred_at

Use append-only usage events for metering; aggregate separately for dashboards and billing.

## 7. Audit and platform administration

### audit_logs
- id UUID PK
- actor_type (user, platform_admin, system)
- actor_id nullable
- organization_id nullable FK -> organizations
- workspace_id nullable FK -> workspaces
- action
- resource_type, resource_id
- outcome
- metadata JSONB (redacted, non-secret)
- created_at

Audit provider credential creation, rotation, revocation and assignment; changes to model defaults, workspace overrides and fallback policies; model/provider enablement; agent publishing; integration and tool permission changes.

Platform admin roles and access must be managed separately from workspace roles. Require elevated authorization and preferably MFA for credential and routing administration.

## 8. Indexes and constraints

Recommended initial indexes:
- Every workspace-owned table: index workspace_id.
- organization_members: unique (organization_id, user_id).
- workspace_members: unique (workspace_id, user_id).
- agents: unique workspace_id for the one-agent MVP.
- agent_versions: unique (agent_id, version_number).
- models: unique (provider_id, model_key).
- provider_credentials: (owner_workspace_id, provider_id, status); platform credential lookup by (provider_id, status).
- workspace_model_preferences: unique (workspace_id, task_type).
- workspace_model_overrides: unique (workspace_id, task_type).
- platform_fallback_routes: (fallback_policy_id, source_model_id, priority).
- model_request_logs: (workspace_id, created_at), (provider_id, model_id, created_at).
- conversations: (workspace_id, created_at).
- messages: unique (conversation_id, sequence_number).
- knowledge_chunks: vector index plus workspace/document indexes.
- audit_logs: (workspace_id, created_at), (actor_id, created_at).

Use composite foreign keys where useful to ensure referenced agents, integrations, documents and credentials belong to the same workspace. Add CHECK constraints for enum-like values and nonnegative counts/costs. Use partial indexes for active records where appropriate.

## 9. Row-level security and service rules

- Enable RLS on tenant-owned tables and set tenant/workspace context transaction-locally after server-side authentication.
- Never accept tenant context directly from untrusted request parameters.
- Platform admin operations use a separate privileged role with audited access; do not reuse ordinary workspace database credentials.
- Background jobs carry verified workspace scope and re-establish it for each transaction.
- Secret resolution verifies owner_type and owner_workspace_id before decrypting.
- A workspace override may reference only a platform-owned credential.
- A BYOK preference may reference only a credential owned by that same workspace.
- A fallback route may reference only a platform-owned credential and runs only when the effective fallback policy is enabled.
- Enforce provider/model compatibility and capability checks in the Model Gateway, not only in UI validation.

## 10. Credential resolution examples

### Workspace uses its own key
1. Workspace has a preference for Provider A / Model X with BYOK selected.
2. Gateway verifies the credential belongs to the workspace and supports Provider A.
3. Gateway calls Provider A using that credential.
4. If it fails, gateway records a normalized error and stops. It does not use the platform default or fallback routes.

### Workspace uses platform-managed configuration
1. No BYOK route is selected.
2. Gateway resolves the workspace admin override, otherwise the global platform default.
3. Gateway uses only the assigned platform credential.
4. If the request fails and fallback is disabled (the default), gateway stops and reports the error.
5. If an administrator explicitly enabled and configured fallback, gateway may try only permitted ordered platform fallback routes within configured limits.

## 11. Migration and implementation sequence

1. Create tenant, user, membership and workspace tables; add RLS and isolation tests.
2. Add provider/model catalog and encrypted credential reference lifecycle.
3. Add workspace preferences, platform defaults and admin workspace overrides.
4. Add fallback policy and route tables with disabled-by-default constraints and authorization tests.
5. Add model request logs and usage events.
6. Add agent, conversation, integration, tool, context and knowledge tables.
7. Add indexes, retention jobs, audit events and operational dashboards.
8. Add integration tests proving BYOK never crosses into platform credentials and fallback cannot run unless an administrator enabled a valid policy.

Before production, verify cross-tenant foreign-key relationships, secret-store behavior, backup/restore, key rotation, deletion semantics and RLS bypass protections.
