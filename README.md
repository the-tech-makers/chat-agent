# Chat Agent

A universal, tenant-aware AI agent platform that connects application tools, context, knowledge and data, and serves website visitors and authenticated users.

## Project scope
The product vision, MVP requirements, user journeys, architecture, acceptance criteria and milestones are documented in **[Product Scope & MVP](docs/PRODUCT_SCOPE.md)**.

## Proposed technology stack
> Initial proposal; subject to validation as implementation begins.

| Layer | Proposed technology |
|---|---|
| Application/dashboard | Next.js, React, TypeScript |
| API and runtime | TypeScript, Node.js |
| Agent/model integration | Provider abstraction; tool-calling and streaming |
| Integration protocol | MCP first; OpenAPI connectors later |
| Database | PostgreSQL |
| Vector search | pgvector |
| ORM/migrations | Drizzle ORM (proposal) |
| Knowledge/document storage | S3-compatible object storage |
| Background jobs | Durable queue/worker system (selection pending) |
| Widget | Isolated iframe, TypeScript |
| Auth | Secure SaaS authentication; signed identity handoff |
| Deployment | Cloud hosting (provider selection pending) |
| Observability | OpenTelemetry-compatible telemetry |

## Hardware and system requirements
### Development machine (suggested)
- 4+ CPU cores
- 8 GB RAM minimum; 16 GB recommended
- 10+ GB free disk space
- Modern OS: macOS, Linux, or Windows with WSL2
- Node.js LTS, Git and Docker installed

### Production
The MVP is intended for cloud deployment. Actual compute, storage, scaling and database sizing will depend on traffic, model/provider choice, ingestion workload and retention. No fixed production sizing is established yet.

## Installation
> Setup instructions will be finalized once the application skeleton and dependencies are committed.

1. Install Git, Node.js LTS and Docker.
2. Clone this repository:
   ```bash
   git clone https://github.com/the-tech-makers/chat-agent.git
   cd chat-agent
   ```
3. Install dependencies (command depends on the package manager selected for the repo).
4. Configure environment variables using the forthcoming `.env.example`.
5. Start local services and the application using the forthcoming developer scripts.

## Documentation
- [Product Scope & MVP](docs/PRODUCT_SCOPE.md)

## Credits
Built by [The Tech Makers](https://github.com/the-tech-makers) and contributors.

## License
License has not yet been selected. See repository license status before reuse or redistribution.
