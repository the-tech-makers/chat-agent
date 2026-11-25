# Knowledge and Retrieval Specification

**Status:** Normative MVP design; source connectors may be phased.

## 1. Purpose and separation

Knowledge is curated, indexed content intended for retrieval. Context is live or structured application state assembled for a specific request. Knowledge retrieval must not be used as an authorization system; source ACLs and actor permissions are checked before content is exposed.

MVP supports controlled document uploads and selected text/URL ingestion only after SSRF-safe fetch design. Database synchronization and broad crawling are later phases.

## 2. Source lifecycle

States: `registered` → `queued` → `processing` → `indexed`; terminal or exception states: `failed`, `paused`, `deleting`, `deleted`.

Each source has tenant/workspace ownership, source type, external reference, content hash, version, ingestion status, metadata, ACL, timestamps and retention classification. Re-ingestion creates a new version; publish new chunks atomically only after processing succeeds. Preserve the previous active version until the new version is ready.

## 3. Ingestion pipeline

1. Authenticate uploader and authorize source management.
2. Validate file type, extension, size and magic bytes.
3. Malware scan where supported; quarantine unknown/risky files.
4. Store original privately with encryption and tenant-scoped object key.
5. Extract text in an isolated worker with CPU, memory, page and time limits.
6. Normalize text while preserving source locations and headings.
7. Apply PII/sensitive-data handling policy.
8. Chunk using configurable token/character limits and overlap; retain document structure.
9. Generate embeddings using an approved hosted embedding provider.
10. Store chunks, vectors, provenance and ACL metadata.
11. Run indexing validation and atomically publish source version.

Never execute macros, scripts, embedded code or active content from documents. Treat extracted text as untrusted prompt data.

## 4. Chunk and metadata contract

Each chunk stores:
- Workspace ID and source ID/version
- Stable chunk ID and ordinal
- Text and content hash
- Embedding model/version and vector
- Heading/section path and page/line/anchor locator where available
- Source title and canonical reference
- ACL/classification metadata
- Created/updated timestamps

Chunk size, overlap and embedding dimensions are configuration tied to the selected embedding model. Do not mix incompatible vector dimensions in one index without explicit partitioning/migration.

## 5. Retrieval flow

1. Resolve workspace, actor and request purpose.
2. Determine allowed source set from tenant and ACL policy.
3. Apply metadata filters before or during vector search.
4. Retrieve candidate chunks using vector similarity and optional lexical search.
5. Rerank only when configured and within budget.
6. Deduplicate adjacent/overlapping chunks.
7. Apply relevance threshold and result count/token budget.
8. Return content with citations/provenance and retrieval diagnostics.
9. Record source/chunk references for audit and deletion.

Retrieval must filter by workspace and ACL at query time. Never rely on post-retrieval filtering alone if unauthorized text could be returned to an intermediate model or log. If authorization cannot be evaluated, fail closed.

## 6. Citations and answer grounding

Citation contains source ID, source title, locator, canonical URL or internal reference, and chunk/source version. The assistant may cite only retrieved sources supplied to that response. Do not fabricate page numbers or URLs. If evidence is insufficient, state uncertainty or ask for clarification rather than inventing a citation.

## 7. Freshness and deletion

Support re-ingestion, source pause, version rollback and deletion. Deletion removes source records, chunks, vectors, object storage objects and derived caches within the documented deletion SLA. Backups age out according to backup retention; deleted data must not reappear after restore without replaying deletion tombstones. Maintain a deletion ledger until all replicas and backups expire.

## 8. Security and privacy

- Strict workspace and source ACL filters.
- Private object storage; short-lived signed URLs.
- SSRF protection for URL ingestion: allow HTTP(S), block private/loopback/link-local/metadata IPs, resolve and revalidate redirects/DNS, restrict ports and egress, enforce response limits.
- Content extraction sandbox and parser patching.
- Encryption in transit and at rest.
- PII minimization and configurable retention.
- Prompt injection detection/containment: retrieved instructions never override system policy.
- Audit source creation, access, reprocessing, permission change and deletion.

## 9. Quality and observability

Track ingestion success/failure, extraction duration, chunk count, embedding latency/cost, index freshness, retrieval latency, empty-result rate, citation coverage and relevance evaluation. Do not log full source text by default. Build an evaluation corpus with expected relevant documents and unauthorized-document canaries.

## 10. Acceptance criteria

- A source is not searchable until a complete version is published.
- Retrieval never returns cross-workspace or unauthorized chunks.
- Every returned chunk has verifiable provenance.
- Failed re-ingestion does not destroy the previous active version.
- Deletion covers original, derived, cached and indexed content.
- Prompt injection in documents cannot change tool policy or authorization.
