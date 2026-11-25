# Admin and Workspace Operations Specification

**Status:** Normative role and control-plane design.

## 1. Control-plane separation

The platform control plane manages SaaS-wide configuration, provider catalog, platform credentials, abuse controls and operational status. Workspace control plane manages tenant membership, agent settings, BYOK credentials, integrations, knowledge and usage. Platform operators must not routinely inspect customer conversations or content.

## 2. Roles

| Role | Scope | Core permissions |
|---|---|---|
| Platform owner | Entire platform | All platform configuration and emergency operations |
| Platform administrator | Entire platform | Provider catalog, global policy, workspace support within audited grants |
| Support operator | Limited platform scope | Metadata-only diagnostics; time-bound elevated access if approved |
| Workspace owner | One workspace | Full workspace configuration, membership and billing/usage settings |
| Workspace admin | One workspace | Agent, integrations, knowledge and member administration except ownership transfer |
| Workspace developer | One workspace | API keys, integration configuration and test tools within granted scopes |
| Workspace analyst | One workspace | Read analytics and usage; no secret or content access by default |
| End user | Session/application scope | Chat and explicitly permitted actions |
| Guest | Single widget session | Limited chat; no account-level privileged actions |

A user may hold different roles in different workspaces. Platform roles never imply customer application privileges.

## 3. Permission model

Use deny-by-default permissions rather than role-name checks scattered through code. Permissions are resource/action pairs, for example `workspace.settings.read`, `workspace.settings.write`, `provider_credential.manage`, `integration.tool.enable`, `knowledge.source.delete`, `usage.read`, `platform.provider.manage`.

Enforce at API/service layer and database boundary. UI hiding is not authorization. Ownership transfer requires reauthentication, explicit confirmation and audit. Prevent removal/demotion of the final active owner unless ownership is transferred.

## 4. Platform administration

Platform administrators may:
- Publish/disable providers and models with capability metadata.
- Configure platform-managed credentials and global default routes.
- Set workspace-specific overrides and explicit fallback policies.
- Configure platform safety caps, rate limits and abuse controls.
- Suspend/reinstate workspaces with reason and audit record.
- Inspect operational metadata and aggregate usage.
- Grant time-bound support access where permitted.

Any support access to customer content must be explicitly scoped, time-bound, reason-coded, approved according to policy, visible in audit logs and revocable. Emergency access must be reviewed after the incident.

## 5. Workspace administration

Workspace owners/admins may:
- Manage members, invitations and role grants.
- Configure agent behavior and widget appearance.
- Configure model preference and workspace BYOK credentials.
- Register MCP servers and approve tools.
- Upload/manage knowledge sources.
- Set usage alerts, budgets and retention settings within platform bounds.
- Review audit events and revoke tokens/sessions.

Workspace owners cannot read platform credentials or override mandatory platform safety controls.

## 6. Provider credential workflow

1. Owner chooses provider/model from enabled catalog.
2. Secret is submitted over TLS to a dedicated credential service.
3. Service validates shape and may perform a minimal provider test, with explicit consent and safe limits.
4. Secret is encrypted and stored in secret manager/KMS-backed vault.
5. Database stores metadata, reference, status, owner and rotation timestamps.
6. Owner binds credential to provider/model and selects it in routing settings.
7. Effective route preview shows source and policy reason, never key material.
8. Revocation disables new use immediately and invalidates cached credential material.

Never log submitted values or return them after save. Rotation should support overlap only where provider semantics require it.

## 7. Workspace lifecycle

States: `trial`, `active`, `restricted`, `suspended`, `deletion_pending`, `deleted`. State transitions are reasoned, authorized and audited. Suspension behavior must be explicit for API, widget, running jobs, integrations and data access. Deletion requires confirmation, retention/legal-hold evaluation and a staged purge workflow.

## 8. Audit requirements

Audit actor, role snapshot, workspace/resource, action, outcome, timestamp, request ID, source IP/device metadata where lawful, reason and before/after diff with secrets and sensitive content redacted. Audit records are append-only to application roles and have defined retention and export controls.

## 9. Acceptance criteria

- Every admin operation has an explicit permission and scope.
- Workspace role cannot grant platform permissions.
- Support access is limited, approved, time-bound and audited.
- Secret values are never readable after creation.
- Last-owner safeguards and ownership transfer work.
- Suspension and deletion transitions have defined side effects and recovery behavior.
