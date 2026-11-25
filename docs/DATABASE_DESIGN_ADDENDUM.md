# Database Design Addendum — Workflow Support Entities

**Status:** Proposed logical schema required to close gaps AUD-001–AUD-014 in [Contract Alignment Audit](CONTRACT_ALIGNMENT_AUDIT.md). This addendum extends, but does not replace, [Database Design](DATABASE_DESIGN.md). Exact Drizzle types, composite foreign keys and retention indexes are finalized in migrations.

## 1. Workspace invitations

### workspace_invitations
- id UUID PK
- workspace_id FK -> workspaces
- invited_email normalized
- role constrained to allowed workspace roles; owner assignment is not available through invitation
- token_hash unique (never store invitation token)
- invited_by_user_id FK -> users
- expires_at, accepted_at nullable, revoked_at nullable
- status (pending, accepted, expired, revoked)
- created_at, updated_at

Only one active pending invitation per normalized email/workspace. Acceptance revalidates expiry, role policy and workspace state.

## 2. Server API tokens

### workspace_api_tokens
- id UUID PK
- workspace_id FK -> workspaces
- token_hash unique; token_prefix for identification only
- name, scopes JSONB
- created_by_user_id FK -> users
- expires_at nullable, last_used_at nullable, revoked_at nullable
- created_at

Generate high-entropy tokens; display once; compare hashes in constant time. Scope every token to one workspace and an explicit permission set.

## 3. Identity provider registration and replay prevention

### customer_identity_providers
- id UUID PK
- workspace_id FK -> workspaces
- issuer, audience, algorithm allow-list
- verification_key_set_reference or public JWKS URL subject to SSRF-safe fixed-host policy
- status, created_at, updated_at

### identity_assertion_nonces
- id UUID PK
- workspace_id FK -> workspaces
- issuer, nonce_hash unique within issuer/workspace
- expires_at, consumed_at, created_at

Store only public verification material. Restrict key-fetch destinations and cache with bounded TTL. Nonce rows expire after the assertion replay window.

## 4. Widget and chat sessions

### chat_sessions
- id UUID PK
- workspace_id FK -> workspaces
- agent_id FK -> agents
- actor_type (guest, authenticated)
- actor_reference nullable
- origin
- token_jti_hash unique
- status (active, revoked, expired, closed)
- expires_at, last_seen_at nullable, revoked_at nullable
- created_at

Conversation rows reference chat_session_id. Store token identifier/hash metadata only, never bearer token bytes. Enforce workspace/agent consistency with composite foreign keys.

## 5. Tool approvals and execution attempts

### tool_approvals
- id UUID PK
- workspace_id FK -> workspaces
- conversation_id FK -> conversations
- tool_id FK -> integration_tools
- tool_schema_version
- arguments_hash
- requested_by_actor_type, requested_by_actor_reference
- approved_by_user_id nullable FK -> users
- status (pending, approved, denied, expired, consumed, revoked)
- expires_at, decided_at nullable, consumed_at nullable
- created_at

Approval authorizes exactly the stored tool/schema/argument hash and actor scope. Any changed argument requires a new approval.

### tool_execution_attempts
- id UUID PK
- workspace_id FK -> workspaces
- tool_execution_log_id FK -> tool_execution_logs
- attempt_number
- idempotency_key
- lease_owner nullable, lease_expires_at nullable
- status (queued, running, succeeded, failed, timed_out, cancelled)
- started_at, completed_at nullable
- result_reference nullable
- normalized_error_code nullable
- created_at, updated_at

Unique (tool_execution_log_id, attempt_number); enforce unique idempotency key in its workspace scope. Lease expiry enables safe recovery; never retry non-idempotent side effects without upstream idempotency.

## 6. Idempotency and durable run events

### idempotency_records
- id UUID PK
- workspace_id nullable for pre-workspace account operations
- caller_scope, key_hash, operation
- request_hash
- response_status, response_reference or bounded response JSONB
- status (in_progress, completed, failed)
- expires_at, created_at, updated_at

Unique (caller_scope, key_hash, operation). Same key with a different request hash returns conflict. Store only the minimum replayable response; redact sensitive values.

### run_events
- id UUID PK
- workspace_id FK -> workspaces
- run_id FK -> agent_runs
- event_sequence
- event_type
- payload JSONB, redacted before persistence
- created_at

Unique (run_id, event_sequence). Define retention and replay authorization. If durable replay is deferred, API must not promise Last-Event-ID support in MVP.

## 7. Budgets and reservations

### workspace_budget_policies
- id UUID PK
- workspace_id FK -> workspaces
- period (daily, monthly)
- limit_amount, currency
- action_on_limit (block, warn)
- enabled, effective_from, created_at, updated_at

### usage_reservations
- id UUID PK
- workspace_id FK -> workspaces
- run_id FK -> agent_runs
- idempotency_key unique within workspace
- estimated_amount, currency
- actual_amount nullable
- status (reserved, settled, released, expired)
- expires_at, settled_at nullable, created_at, updated_at

Reserve atomically against a locked or otherwise concurrency-safe budget aggregate. Reconcile actual provider usage exactly once.

## 8. Knowledge source versioning

### knowledge_source_versions
- id UUID PK
- workspace_id FK -> workspaces
- source_id FK -> knowledge_sources
- version_number
- content_hash
- status (processing, ready, active, failed, archived, deleting)
- created_at, published_at nullable

Unique (source_id, version_number). Add active_version_id to knowledge_sources. Bind documents and chunks to a source version and ensure retrieval sees only active, authorized versions.

## 9. Uploads and attachments

### attachments
- id UUID PK
- workspace_id FK -> workspaces
- chat_session_id nullable FK -> chat_sessions
- uploaded_by_actor_type, uploaded_by_actor_reference
- object_key unique, original_filename, detected_mime_type
- size_bytes, checksum_sha256
- scan_status (pending, clean, rejected, failed)
- status (uploading, available, expired, deleting, deleted)
- expires_at, created_at, deleted_at nullable

Object keys are opaque and tenant-scoped. Download authorization is checked on every access; signed URLs are short-lived.

## 10. Deletion orchestration and tombstones

### deletion_jobs
- id UUID PK
- workspace_id nullable FK -> workspaces
- resource_type, resource_id
- requested_by_actor_type, requested_by_actor_id nullable
- status (queued, running, blocked_legal_hold, completed, failed)
- requested_at, started_at nullable, completed_at nullable
- error_code nullable

### deletion_tombstones
- id UUID PK
- workspace_id nullable
- resource_type, resource_id
- deletion_job_id FK -> deletion_jobs
- primary_store_deleted_at nullable
- object_store_deleted_at nullable
- vector_store_deleted_at nullable
- cache_deleted_at nullable
- backup_expiry_at nullable
- created_at

Tombstones are retained long enough to prevent restored backups from resurrecting deleted resources. Restrict tombstone access and avoid retaining deleted content in tombstone metadata.

## 11. Recommended changes to existing entities

- Add `chat_session_id` to conversations.
- Add `source_version_id` to knowledge_documents and knowledge_chunks.
- Add `request_id`, `reason`, `actor_role_snapshot`, `before_state_redacted` and `after_state_redacted` to audit_logs.
- Add an explicit active source version pointer to knowledge_sources.
- Add composite unique keys (workspace_id, id) to tenant-owned parent tables where required for composite foreign keys.
- Add explicit constraints preventing cross-workspace references among sessions, conversations, agents, approvals, tools, sources and attachments.

## 12. Migration ordering

1. API token and invitation tables.
2. Customer identity provider and nonce tables.
3. Chat session table and conversation relation.
4. Tool approval and execution-attempt tables.
5. Idempotency and run-event tables.
6. Budget policy and reservation tables.
7. Knowledge source versions and active pointer.
8. Attachments and upload lifecycle.
9. Deletion jobs/tombstones and audit metadata extension.

Every new table must have tenant isolation/RLS, indexes, retention behavior, migration rollback notes and tests. Use service roles only for narrowly scoped cross-tenant platform operations.
