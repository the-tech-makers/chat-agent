# Glossary

| Term | Definition |
|---|---|
| Agent runtime | Orchestration service that assembles context, calls models and brokers tool execution. |
| BYOK | Bring Your Own Key; a workspace supplies and controls its provider credential. |
| Context | Live or structured application data assembled for one request. |
| Conversation | Persistent sequence of user and assistant messages associated with a session. |
| Credential source | Origin of the key used for a model call: workspace BYOK or platform-managed. |
| Customer identity assertion | Server-signed statement from a customer application used to establish an end-user identity. |
| Embedding | Numeric vector representation used for semantic retrieval. |
| Integration | Configured connection to an external service or customer system. |
| Knowledge source | Managed document or content source that is ingested and indexed for retrieval. |
| MCP | Model Context Protocol; protocol for discovering and invoking tools/resources exposed by an MCP server. |
| Model gateway | Provider-neutral layer that resolves model routes, credentials, capabilities and usage. |
| Platform fallback | Explicit administrator-configured alternative route; disabled until published and eligible. |
| RLS | PostgreSQL Row-Level Security used as defense-in-depth for tenant isolation. |
| Session | Scoped interaction context for a guest or authenticated actor. |
| Tenant / workspace | Isolated customer boundary for configuration, users, data, tools and usage. |
| Tool Execution Broker | Server-side policy enforcement and execution boundary for model-proposed tool calls. |
| Universal agent | The single configurable agent instance associated with a workspace in the MVP. |
| Widget session token | Short-lived credential restricted to a workspace, agent, origin, session and operation set. |
