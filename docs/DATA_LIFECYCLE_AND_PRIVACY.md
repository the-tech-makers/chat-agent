# Data Lifecycle, Privacy and Tenant Offboarding

**Status:** Product and engineering requirements; legal policy wording requires jurisdiction-specific review before public launch.

## 1. Principles

Collect the minimum data needed to provide the service. The customer controls its application data and remains responsible for determining what it may lawfully send. The SaaS acts according to contractual role and applicable law; this document is not a legal determination. Separate operational metadata from customer content, and make retention and deletion behavior explicit.

## 2. Data classes

| Class | Examples | Default handling |
|---|---|---|
| Account data | Email, name, authentication subject | Minimum required; access controlled |
| Workspace metadata | Settings, roles, integration configuration | Tenant isolated |
| Secrets | Provider keys, MCP tokens, signing keys | Secret manager/KMS; never plaintext logs |
| Conversation content | Messages, attachments, citations | Workspace-configurable retention |
| Knowledge content | Originals, extracted text, chunks, vectors | Tenant ACL and source lifecycle |
| Operational data | Request IDs, latency, token usage | Minimized, redacted, bounded retention |
| Audit records | Actor, action, outcome, resource | Append-only and retention controlled |
| Security telemetry | Abuse signals, IP/device metadata | Minimized, purpose-limited |

## 3. Retention

Each data category must have a documented retention period and deletion trigger. MVP defaults should be conservative and configurable within platform minimum/maximum constraints. Do not claim a universal retention period until product policy and legal review are complete. Expired data is purged from primary stores, caches, search indexes and derived stores; backup copies age out according to the published backup schedule.

## 4. User and workspace rights

Provide authenticated workflows, subject to customer contract and legal obligations, for:
- Access/export of workspace-owned data.
- Correction of account/profile information.
- Deletion of conversations or knowledge sources where authorized.
- Workspace offboarding and deletion.
- Revocation of identity sessions, API tokens and integration credentials.
- Provider credential removal and key rotation.

The SaaS must support customer requests to locate and delete data using stable tenant/resource identifiers. Avoid using email or message text as the only deletion key.

## 5. Workspace offboarding workflow

1. Verify requester authority and require step-up authentication.
2. Explain affected data, integrations and irreversible consequences.
3. Mark workspace `deletion_pending` and block new sessions/writes according to policy.
4. Revoke API keys, widget sessions, OAuth grants and connector credentials.
5. Stop scheduled syncs and queued ingestion; cancel or reconcile running jobs.
6. Delete primary conversation, attachment, knowledge, vector, cache and integration data according to retention/legal hold.
7. Remove secret-manager entries and object storage objects.
8. Retain only minimal billing, fraud-prevention or legally required records under separate access/retention rules.
9. Record completion and remaining backup expiry date.
10. Verify no deleted content is reintroduced by backup restore; replay deletion tombstones.

## 6. Provider and connector data

Data sent to model and embedding providers is limited to the request context required for the task. Provider retention, training, region and subprocessors vary; disclose the selected provider and applicable terms to workspace owners. Do not promise zero retention unless contractually and technically verified. Customer connector access is limited to approved tools and fields.

## 7. Privacy by design

- Tenant separation and least privilege.
- Data minimization and purpose limitation.
- Encryption in transit and at rest.
- Secret isolation and access auditing.
- Configurable retention and deletion.
- No training use of customer data unless explicitly contracted and legally permitted.
- No sale of customer conversation content.
- Avoid sensitive personal data in telemetry and analytics.
- Support data export and deletion verification.

## 8. Legal and compliance readiness

Before public launch, obtain review of privacy notice, terms, DPA, subprocessor list, international transfer mechanism, cookie/consent requirements, breach notification process, data residency claims and applicable Indian and other target-market privacy laws. Do not advertise a certification or compliance status before independent validation.

## 9. Acceptance criteria

- Data inventory identifies storage location, purpose, owner and retention.
- Workspace deletion revokes credentials and removes primary and derived data.
- Backup expiry and tombstone replay are documented and tested.
- Provider data handling is disclosed accurately.
- Export and deletion actions are authorized and auditable.
