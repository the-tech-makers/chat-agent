# Configuration and Environment Reference

**Status:** Configuration contract. Exact variable names are finalized in implementation; names below are recommended canonical names.

## 1. Rules

- Parse and validate configuration once at process startup.
- Fail startup for missing required settings or invalid security-sensitive values.
- Never commit real secrets or provide production secret examples.
- Use separate configuration schemas for web/API, widget and worker.
- Prefer managed identity and secret-manager references over long-lived environment secrets.
- Redact secret values from startup diagnostics.
- Keep non-secret feature flags separate from credentials.

## 2. Configuration catalogue

| Variable | Required | Consumers | Description |
|---|---:|---|---|
| `NODE_ENV` | Yes | All | Runtime mode |
| `APP_ENV` | Yes | All | local/ci/staging/production |
| `APP_BASE_URL` | Yes | Web/API | Canonical public application URL |
| `API_BASE_URL` | Yes | Widget/SDK | Public API URL |
| `WIDGET_BASE_URL` | Yes | Web/Widget | Public widget origin |
| `DATABASE_URL` | Yes | API/Worker | PostgreSQL connection via TLS in hosted env |
| `DATABASE_POOL_MAX` | No | API/Worker | Connection pool cap |
| `DATABASE_SSL_MODE` | Hosted | API/Worker | Database TLS policy |
| `QUEUE_URL` | Yes | Worker/API | Durable queue endpoint |
| `OBJECT_STORAGE_ENDPOINT` | Yes | API/Worker | Private object storage endpoint |
| `OBJECT_STORAGE_BUCKET` | Yes | API/Worker | Private bucket name |
| `OBJECT_STORAGE_REGION` | Provider-dependent | API/Worker | Storage region |
| `OBJECT_STORAGE_ACCESS_KEY` | Avoid | API/Worker | Prefer workload identity |
| `OBJECT_STORAGE_SECRET_KEY` | Avoid | API/Worker | Prefer workload identity |
| `KMS_KEY_ID` | Hosted | API/Worker | Key encryption key reference |
| `SECRET_MANAGER_ENDPOINT` | Hosted | API/Worker | Secret manager endpoint |
| `AUTH_ISSUER_URL` | Yes | Web/API | SaaS auth issuer |
| `AUTH_AUDIENCE` | Yes | API | Expected token audience |
| `SESSION_SIGNING_KEY_REF` | Yes | API | Secret manager reference |
| `WIDGET_ALLOWED_ORIGINS_MODE` | Yes | API | Strict origin policy mode |
| `MODEL_GATEWAY_DEFAULT_PROVIDER` | Optional | API/Worker | Platform default provider identifier |
| `MODEL_GATEWAY_DEFAULT_MODEL` | Optional | API/Worker | Platform default model identifier |
| `OPENROUTER_API_KEY_REF` | If enabled | API/Worker | Secret reference, not raw key |
| `GEMINI_API_KEY_REF` | If enabled | API/Worker | Secret reference, not raw key |
| `EMBEDDING_PROVIDER` | If knowledge enabled | Worker | Approved embedding adapter |
| `EMBEDDING_MODEL` | If knowledge enabled | Worker | Embedding model ID |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Optional | All | Telemetry collector endpoint |
| `LOG_LEVEL` | No | All | Structured log level |
| `SENTRY_DSN` | Optional | Web/API/Worker | Error reporting endpoint |
| `PUBLIC_WIDGET_KEY` | Browser public | Widget | Publishable workspace identifier; not secret |
| `RATE_LIMIT_STORE_URL` | Optional | API | Shared rate-limit backend |

The presence of a platform provider secret reference does not automatically activate a route. An administrator must explicitly publish a platform default and policy.

## 3. Validation

- URLs must be HTTPS outside local development and match approved host policy.
- Database/queue/storage endpoints must not resolve to public untrusted hosts in production.
- Numeric limits must have safe minimum/maximum bounds.
- Secret references must resolve at runtime using service identity.
- Model/provider IDs must exist in the enabled catalog.
- Origin lists must be normalized exact origins; no wildcard suffix matching.
- `NODE_ENV` and `APP_ENV` must not be conflated.

## 4. Local development

Use a committed `.env.example` containing placeholders only. A local `.env` is ignored by Git. Provide Docker Compose for local PostgreSQL/pgvector and any required local queue/storage emulator, while keeping application inference hosted. CI uses fake providers and synthetic secrets. Do not require a developer to configure a live model key for unit tests.

## 5. Rotation and change management

Rotate database, auth, storage, provider and signing credentials according to a documented schedule and immediately after suspected exposure. Support overlapping signing keys for token verification during rotation. Record who changed a setting, when, why and affected environment. Configuration schema changes require migration notes and validation tests.

## 6. Acceptance criteria

- All runtime settings are typed and validated.
- Missing required settings fail fast with safe diagnostics.
- No committed file contains a real credential.
- Provider secrets are referenced from secret management.
- Platform defaults remain inactive until explicitly published.
