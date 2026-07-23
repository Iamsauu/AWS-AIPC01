# RAG, Knowledge Bases, and Vector Search

Status: Verified  
Official tasks: 1.4.1–1.4.5, 1.5.1–1.5.6  
Last verified: 2026-07-23

## Why this matters

Retrieval-augmented generation (RAG) gives an FM current, private, and attributable evidence without teaching those facts through fine-tuning. AIP-C01 tests the entire retrieval system: source integration, parsing, chunking, metadata, embeddings, vector storage, search, reranking, complex-query handling, synchronization, stable interfaces, grounding, and evaluation.

## Core concepts

### RAG is a pipeline

```text
source
  → parse
  → chunk
  → enrich metadata
  → embed
  → index
  → retrieve + filter
  → rerank
  → augment prompt
  → generate
  → cite + validate + evaluate
```

Each stage has a distinct failure signal. “The answer is wrong” is not a diagnosis.

### RAG versus customization

| Need | Prefer |
|---|---|
| Frequently changing facts, private documents, citations | RAG |
| Stable tone, response behavior, task skill not fixed by prompts/RAG | Fine-tuning or other customization after evaluation |
| Exact transaction/account state | Authorized tool/API call, possibly combined with RAG |
| Both current evidence and specialized behavior | RAG plus evaluated customized model |

Do not fine-tune a model merely to memorize documents that will change.

### Current Bedrock Knowledge Base types

Current AWS documentation distinguishes:

| Capability | Managed Knowledge Base | Customer-managed Knowledge Base |
|---|---|---|
| Datastore/index ownership | Bedrock manages ingestion, storage, indexing, retrieval | Customer selects and manages supported vector store/configuration |
| Embedding and reranking | Managed by service | Configured by customer through supported options |
| Source connectors | S3, SharePoint, Confluence, Google Drive, OneDrive, Web Crawler, Custom | S3 and Custom in the current comparison |
| Document-level ACL filtering | Supported for eligible managed connectors; not Web Crawler | Implement according to source/application/vector design |
| Agentic multi-step retrieval | `AgenticRetrieveStream` | Query decomposition through supported RAG configuration |
| Native AgentCore Gateway target | Supported managed-KB connector | Use a custom integration when required |
| Index/mapping control | Limited | Greater control |
| Primary retrieval APIs | `Retrieve`, `AgenticRetrieveStream` | `Retrieve`, `RetrieveAndGenerate` where supported |

The name “Knowledge Bases for Amazon Bedrock” alone is not enough to select an API. `RetrieveAndGenerate` documentation explicitly says it cannot be used with **Managed Knowledge Bases**. Managed Knowledge Bases use `Retrieve` for direct retrieval or `AgenticRetrieveStream` for multi-step retrieval and optional citation-backed synthesis.

### Retrieval, generation, and evidence

`Retrieve` returns relevant content, metadata, location, and a relevance score. Use it to isolate search quality from generation.

`RetrieveAndGenerate` combines retrieval and FM generation for supported customer-managed/vector Knowledge Bases and returns citations/retrieved references.

`AgenticRetrieveStream` can plan/decompose a complex query, retrieve in multiple steps, and optionally produce a citation-backed response for a Managed Knowledge Base.

A similarity/relevance score is not calibrated factual confidence. A citation proves only that a chunk was retrieved unless the cited span actually supports the claim. Implement an insufficient-evidence response.

## How it works

### Requirements before choosing a store

Capture:

- corpus size, file count, and update rate;
- query rate, latency SLA, concurrency, and availability;
- exact-key versus semantic needs;
- modality and embedding dimension/type;
- metadata/filter and ACL requirements;
- source connectors and synchronization;
- Region, residency, encryption, and network boundaries;
- operational ownership and existing database skills;
- backup, restore, deletion, and cost constraints.

### Parsing and chunking

| Strategy | Best fit | Trade-off |
|---|---|---|
| Default | General documents; approximately 300-token chunks at sentence boundaries in current docs | Simple baseline, less control |
| Fixed size + overlap | Uniform prose and predictable index volume | Can split topics or duplicate context |
| Semantic | Topic boundaries vary | FM-assisted ingestion cost and latency |
| Hierarchical | Precise child matching but parent context needed | More storage/processing; retrieved child is replaced with parent context |
| No chunking | Pre-segmented records or special formats | Each source object/record must already be meaningful |
| Custom transformation | Domain-specific sections, code, tables, or legal clauses | Lambda/processing ownership and testing |

Start from document semantics, not one universal token number. Chunk headers with their body, preserve page/section provenance, and avoid overlap so large that the top results repeat the same passage.

Hierarchical chunking is the exam pattern for “retrieve the precise clause but give the model enough surrounding context”: search child chunks, then return the parent.

Feature availability is Knowledge Base type and data-source specific. The current Managed-versus-Customer-managed comparison lists built-in/default or fixed-size chunking for Managed Knowledge Bases. For a customer-managed S3 source, follow the current Knowledge Base chunking and ingestion configuration documentation before assuming that semantic, hierarchical, no-chunking, or custom transformation is accepted.

Text chunking settings do not necessarily govern native multimodal embedding processing in the same way. Verify the selected parser/embedding path.

### Metadata

Useful metadata includes:

- source URI, source system, document ID, and content hash;
- title, author/owner, department, domain, product, and language;
- created/effective/expiry/revision timestamps;
- jurisdiction, confidentiality, tenant, and permissions;
- page, section, parent/child IDs, modality, and parser version.

Use metadata for filtering when a field constrains eligibility or precision. Include a field in embeddings only when its words should influence semantic similarity.

For an S3 source, the sidecar is named:

```text
fileName.extension.metadata.json
```

It resides beside the source and is currently limited to 10 KB. In the sidecar, `includeForEmbedding: true` concatenates that key/value into the chunk before embedding; `false` leaves it filter-only.

Never rely on metadata filters alone as the authorization boundary. Authenticate the caller, compute allowed scope, and then apply filters as defense in depth.

### Embeddings

Choose by:

1. supported modalities and languages;
2. domain retrieval quality;
3. vector dimension and data type;
4. maximum input and batching behavior;
5. vector-store compatibility;
6. storage, indexing, and query cost.

Titan Text Embeddings V2 currently supports 1,024 dimensions by default and 512 or 256 dimensions, with documented input limits of 8,192 tokens or 50,000 characters. Lower dimension can reduce storage and search work; only evaluation can show whether recall/relevance remains acceptable.

For text, image, document, video, or audio cross-modal retrieval, choose a currently supported multimodal embedding model that maps required modalities into a shared semantic space. Current AWS documentation describes Nova Multimodal Embeddings. Learn the capability, not a mock’s old brand label.

The index mapping dimension must exactly match the embedding output. Re-embedding is required when a model/dimension change is incompatible; version the index and migrate through evaluation rather than mutating production blindly.

### Vector stores

Current Bedrock Knowledge Base documentation lists supported stores including:

- Amazon OpenSearch Serverless;
- Amazon OpenSearch Service managed clusters;
- Amazon Aurora PostgreSQL-compatible with `pgvector`;
- Amazon Neptune Analytics;
- Amazon S3 Vectors;
- Pinecone;
- Redis Enterprise Cloud;
- MongoDB Atlas.

Support varies by Knowledge Base type, Region, vector data type, and feature. Current docs state that OpenSearch Serverless and OpenSearch managed clusters support binary vectors for Knowledge Base documents; S3 Vectors uses float vectors.

For an OpenSearch Serverless Knowledge Base index that uses metadata filtering, current guidance requires the `faiss` engine. An existing `nmslib` index must be replaced with a compatible index/Knowledge Base rather than assumed to support the same filtering behavior.

| Store | Prefer when | Main caution |
|---|---|---|
| Managed Knowledge Base datastore | Lowest vector operations and supported managed feature set | Less index-level control; different APIs |
| OpenSearch Serverless | Managed scaling, vector plus lexical/hybrid search | OCU cost and collection/index configuration |
| OpenSearch managed cluster | Detailed shard/node/plugin control and existing expertise | Capacity, shard, patching, and scaling ownership |
| Aurora PostgreSQL + `pgvector` | Relational joins/transactions plus vector search | Connection/capacity/index tuning; setup requirements |
| S3 Vectors | Cost-oriented durable vector storage at large scale | Feature/data-type differences; validate latency/search needs |
| Neptune Analytics | Graph plus vector relationships | Use only when graph semantics matter |
| External supported store | Existing enterprise platform/feature need | Network, credentials, support, and data-egress ownership |

Aurora integration requires the vector extension and supported Bedrock setup, including RDS Data API and Secrets Manager; the Knowledge Base and database must meet documented account/Region requirements.

DynamoDB is strong for source/job/conversation/operational metadata, idempotency keys, and routing state. The AIP blueprint phrase “DynamoDB with vector databases for metadata management” does not make DynamoDB the semantic vector engine.

### OpenSearch architecture

Use domain-specific indexes when different corpora require different analyzers, embedding models, lifecycle policies, or access boundaries. Route the query to relevant domains, search them in parallel when justified, normalize/merge candidates, and rerank globally.

More shards are not automatically faster. Too many small shards consume heap/CPU and add fanout; too few oversized shards limit parallelism and recovery. Current OpenSearch guidance is roughly 10–30 GiB per shard for search-latency-focused workloads and 30–50 GiB for write-heavy workloads, followed by workload testing. Primary shard count is costly to change, so benchmark before production.

### Search and reranking

| Technique | Best fit |
|---|---|
| Semantic/vector | Paraphrases and conceptual similarity |
| Keyword/lexical | Exact IDs, codes, names, rare terms |
| Hybrid | Both conceptual language and exact terms |
| Metadata filter | Tenant, jurisdiction, product, date, language, document status |
| Reranking | Candidate set contains the answer but order is weak |

For supported customer-managed stores, hybrid retrieval requires the appropriate text field/index capability. Current documentation identifies Aurora RDS, OpenSearch Serverless, and MongoDB as supported hybrid options under the applicable Knowledge Base configuration. Managed Knowledge Bases run their documented managed hybrid behavior.

Reranking improves the order of candidates; it cannot recover a document excluded by a bad filter or absent from the candidate set. Retrieve a measured candidate count, rerank, then pass only justified top evidence to generation.

### Complex queries

- **Query expansion:** add synonyms/domain forms when recall is low.
- **Query transformation:** normalize spelling, units, identifiers, or generate a search-oriented form.
- **Decomposition:** split multi-part/multi-hop questions, retrieve per subquery, merge, deduplicate, rerank, then synthesize.

Customer-managed `RetrieveAndGenerate` supports query decomposition through the documented `QUERY_DECOMPOSITION` orchestration option. Managed Knowledge Bases use agentic retrieval for multi-step planning. Add latency/cost budgets and cap subqueries.

### Source integration and maintenance

Managed Knowledge Bases currently support seven connector categories: S3, SharePoint, Confluence, Google Drive, OneDrive, Web Crawler, and Custom. Verify connector-specific permissions, limits, ACL support, and supported change behavior.

For CRM or unsupported sources, a common pattern is a managed extraction/integration path into S3 plus a Knowledge Base source, or a custom source/direct ingestion path.

Synchronization is incremental for added, modified, and deleted content under the documented connector behavior. Calling `StartIngestionJob` only starts work. Poll `GetIngestionJob` until terminal `COMPLETE`, inspect statistics/failures, and alarm on stale data.

Event-driven S3 updates can invoke a coalescing workflow that starts ingestion. Avoid launching one full sync per bursty object event. Scheduled sync is appropriate when source APIs or change rates do not justify event handling.

Direct ingestion supports documented S3/custom-source workflows. Direct changes are not necessarily written back to the original connector/source, so define the source of truth and do not mix sync paths accidentally.

### ACL-aware retrieval

For supported Managed Knowledge Base connectors, document permission information can filter retrieval. The application must authenticate the user and provide verified `userContext`; Bedrock does not authenticate that identity. Current docs indicate that omitting context for an ACL-enabled source yields no results rather than unrestricted results.

ACL filtering is not a complete authorization system. Protect the Knowledge Base/API itself and verify that the user is entitled to the supplied identity. Web Crawler does not provide the same source-document ACL behavior.

### Stable retrieval interface

Expose retrieval through:

- a versioned function contract;
- an authenticated API Gateway/Lambda service;
- an MCP server/tool with strict input schema;
- the native AgentCore Gateway connector for a Managed Knowledge Base when it fits.

A stable tool should accept query, permitted filters/scope, result limit, and correlation context; return content/snippets, source location, metadata, score, and retrieval version. It must not let the caller inject arbitrary tenant/ACL filters.

The Managed Knowledge Base AgentCore Gateway target binds the permitted Knowledge Base/data-source identifiers and exposes documented retrieval operations such as `Retrieve` or `AgenticRetrieveStream`.

## AWS services and APIs

- Bedrock Knowledge Bases: managed and customer-managed ingestion/retrieval.
- `Retrieve`, `RetrieveAndGenerate`, `AgenticRetrieveStream`.
- `StartIngestionJob` and `GetIngestionJob`.
- Bedrock reranking models and query-decomposition settings.
- Amazon OpenSearch Service and OpenSearch Serverless.
- Amazon Aurora PostgreSQL-Compatible Edition with `pgvector`.
- Amazon S3 Vectors and other supported vector stores.
- Titan Text Embeddings V2 and supported multimodal embedding models.
- S3, EventBridge, Lambda, Step Functions, and scheduled rules for ingestion.
- AgentCore Gateway or MCP/API contracts for retrieval tools.
- CloudWatch and CloudTrail for operational and control-plane evidence.

## Architecture patterns

### Customer-managed RAG

```text
sources → ingestion workflow → parser/chunker → embedding model → controlled vector store
                                                            ↑
client → authz → query transform → filter + retrieve → rerank → FM → citations/validation
```

### Managed Knowledge Base with ACLs

```text
enterprise connector + source ACLs
  → Managed KB sync
user → application authentication → verified userContext
  → Retrieve or AgenticRetrieveStream
  → permission-filtered evidence
```

### Multi-domain retrieval

```text
query → domain router
          ├─ product index
          ├─ policy index
          └─ support index
       → merge/deduplicate → global rerank → evidence budget → generation
```

## Decision table

| Requirement | Prefer | Avoid |
|---|---|---|
| Lowest ownership with supported connectors/features | Managed Knowledge Base | Building a vector platform by default |
| Exact index/mapping/vector-store control | Customer-managed Knowledge Base | Managed datastore with no required controls |
| Inspect retrieval separately from generation | `Retrieve` | Debugging only the final answer |
| Customer-managed one-call grounded response | `RetrieveAndGenerate` | Using it with Managed Knowledge Bases |
| Managed multi-hop retrieval | `AgenticRetrieveStream` | Pretending one vector query solves every multi-part question |
| Narrow clause plus context | Hierarchical chunking | One huge document chunk |
| Exact ID plus semantic question | Hybrid search | Vector-only exact lookup |
| Correct candidates in weak order | Larger measured candidate set + reranker | Reranking documents never retrieved |
| Wrong jurisdiction/version | Metadata pre-filter | Asking the FM to ignore ineligible chunks |
| Cross-modal search | Shared-space multimodal embeddings | Separate incomparable vectors without fusion |
| Existing relational system plus vectors | Aurora `pgvector` | Adding OpenSearch automatically |
| Low vector operational ownership | Managed KB or compatible serverless store | Self-managed cluster without control requirement |
| Stable agent integration | Strict MCP/API/Gateway contract | Giving agents raw database credentials |

## Security and governance

- Authenticate callers before retrieval and authorize permitted corpora, tenants, and filters.
- Treat ACL `userContext` as verified identity input; never accept it directly from an untrusted client.
- Separate tenants physically or logically according to risk; metadata filtering is defense in depth, not the only boundary.
- Grant Knowledge Base roles only required source, embedding, vector-store, KMS, and secret access.
- Use private connectivity and resource policies where required.
- Encrypt source objects, vector storage, secrets, and logs.
- Preserve deletion/tombstone behavior so removed documents disappear from both index and caches.
- Treat retrieved content as untrusted prompt data; defend against instructions embedded in documents.
- Log source IDs, retrieval configuration, filters, chunk/version IDs, and prompt/model version without leaking document contents broadly.
- Restrict direct ingestion and sync permissions; otherwise an attacker can poison the knowledge corpus.

## Cost, latency, and reliability

| Driver | Control |
|---|---|
| Vector count | Better segmentation, less duplicate overlap, retention/deletion |
| Vector width | Evaluate lower dimensions or compatible binary vectors |
| Semantic parsing/chunking | Use only where measured retrieval gain pays for ingestion |
| Query fanout | Domain routing and bounded parallel searches |
| Reranking | Rerank a measured candidate set, not the full corpus |
| Oversized generation context | Deduplicate, filter, cap top evidence, compress safely |
| Repeated full sync | Incremental/event-coalesced/scheduled sync |
| OpenSearch overhead | Right shard count/size and connection reuse |
| Aurora connection pressure | Pool/reuse connections and tune capacity/index |
| Agentic/decomposed query | Cap steps/subqueries and set latency/cost budget |

High availability covers the whole path: sources, ingestion state, embedding service, vector store, retrieval API, FM, and application. A cross-Region FM does not protect a single-Region vector store.

## Failure modes and troubleshooting

| Symptom | Likely stage/cause | Evidence | Corrective action |
|---|---|---|---|
| New document absent | ingestion incomplete/failed | ingestion status/statistics and source hash | wait for `COMPLETE`, fix failure, resync |
| Deleted document still cited | deletion not synchronized/cache stale | source/index IDs and cache | verify tombstone/sync and invalidate cache |
| No results in ACL-enabled KB | missing/incorrect `userContext` | authenticated identity and request | supply verified supported context |
| Cross-tenant result | authorization/filter bug | effective filter, ACL, document metadata | fail closed; repair authz and reindex metadata |
| Exact code not found | semantic-only search | query and lexical field | use hybrid/keyword |
| Relevant result below cutoff | candidate count/ranking weak | pre/post-rerank lists | increase candidates, rerank, evaluate |
| Reranker cannot find answer | document absent before reranking | candidate set | fix query/filter/chunking/index |
| Duplicate context dominates | overlap or duplicate sources | hashes and chunk IDs | deduplicate and reduce overlap |
| Wrong surrounding context | chunks too small | parent/section links | hierarchical chunking |
| Broad irrelevant chunks | chunks too large | retrieval samples | smaller/semantic chunks |
| Dimension error | embedding/index mismatch | model output and mapping | rebuild compatible versioned index |
| OpenSearch latency/heap pressure | too many shards/fanout | shard size/count, CPU, heap | consolidate/reindex after testing |
| Aurora timeouts | connections/capacity/index | DB metrics/query plan | pool, scale, tune vector index |
| Citation does not support claim | generation/evidence alignment | cited span and answer clause | claim-level verification or refuse |
| `RetrieveAndGenerate` rejected | Managed KB used | KB type and API error | use `Retrieve`/`AgenticRetrieveStream` |
| Source says one thing, answer another | prompt injection or hallucination | raw chunk, prompt, response | isolate instructions, Guardrails, validate evidence |

## Common exam traps

1. RAG is for changing knowledge; fine-tuning is not a document-sync mechanism.
2. “Knowledge Base” does not identify whether it is Managed or customer-managed.
3. `RetrieveAndGenerate` cannot be used with current Managed Knowledge Bases.
4. Starting an ingestion job is not proof of completion.
5. A relevance score is not factual confidence.
6. A citation is useful only if its span supports the claim.
7. Metadata filters do not replace authentication/authorization.
8. ACL-aware retrieval still requires the application to verify the user.
9. Smaller dimensions reduce storage only if evaluation proves acceptable recall.
10. More shards are not automatically faster.
11. Reranking cannot restore a document excluded by a filter.
12. Embeddings do not guarantee exact code/ID matching; use lexical/hybrid search.
13. DynamoDB can manage metadata/state but is not the semantic vector engine in the blueprint pattern.
14. Direct ingestion and connector synchronization can have different sources of truth.
15. Agentic retrieval adds reasoning steps, latency, and cost; it is not required for every query.
16. Current Managed Knowledge Bases may have lower ownership than an older mock’s OpenSearch Serverless design.
17. An older OpenSearch Serverless `nmslib` index is not interchangeable with the `faiss` index required for current Knowledge Base metadata filtering.

## Local mock references

Mocks are practice material. Where a mock and current documentation differ, follow the documented product boundary.

| Focus | Questions |
|---|---|
| Hierarchical chunking | PE1-Q17 |
| Stable MCP retrieval interface | PE1-Q20 |
| Grounding, citations, score caveat | PE1-Q22, Q46; PE2-Q57 |
| Ingestion and terminal-state verification | PE1-Q31; PE2-Q58 |
| Connectors and source integration | PE1-Q33, Q35; PE2-Q75 |
| Vector-store choice/operations | PE1-Q40, Q58; PE2-Q22, Q33, Q44 |
| Shards and multi-index routing | PE1-Q42, Q70; PE2-Q36 |
| Evaluation and debugging | PE1-Q43; PE2-Q47, Q52 |
| Token/context reduction | PE1-Q44; PE2-Q28 |
| Embeddings and metadata | PE1-Q49, Q65; PE2-Q26 |
| Hybrid and reranking | PE1-Q51, Q61; PE2-Q73 |
| Managed KB APIs/Gateway | PE2-Q57, Q70 |

Documentation corrections:

- PE1-Q10/Q33/Q40 use “managed” loosely with `RetrieveAndGenerate`; the current API excludes the product named **Managed Knowledge Base**.
- PE1-Q35’s third-party connector pattern now belongs naturally to Managed Knowledge Bases, with documented connector/ACL behavior.
- PE1-Q49’s large Lambda embedding fanout is possible but not automatically least operations; compare managed ingestion and batch patterns.
- PE1-Q65 should be read as a shared cross-modal vector-space requirement, not a permanent model-brand answer.
- PE2-Q36’s 30–50 GiB shard target is not universal; current guidance gives 10–30 GiB for search-latency workloads and 30–50 GiB for write-heavy workloads.
- PE2-Q44’s OpenSearch Serverless answer remains a low-operations customer-managed choice, but current Managed Knowledge Bases can remove more datastore ownership when index control is unnecessary.

## Hands-on validation

1. Index a small corpus with fixed and hierarchical chunking; compare retrieved chunks and returned context.
2. Create S3 sidecars with filter-only and `includeForEmbedding` fields; prove the difference.
3. Build a dimension-compatible index, then deliberately test a mismatch.
4. Compare semantic, keyword, and hybrid search on IDs plus paraphrased questions.
5. Retrieve a larger candidate set, rerank it, and measure recall/relevance at k.
6. Call `Retrieve` without generation and classify one failure as source, ingestion, chunking, filter, or ranking.
7. Update and delete a source, start synchronization, poll terminal status, and prove the index changed.
8. Test a multi-intent query with decomposition and enforce a subquery/latency cap.
9. Exercise ACL-aware retrieval with valid, missing, and forged user context; verify the application rejects the forged identity.
10. Wrap retrieval in a strict MCP/API function that does not accept arbitrary tenant scope.
11. Compare Managed and customer-managed KB API calls, ownership, and observability.
12. Inspect citations at claim level and trigger an insufficient-evidence response.

## Recall questions

1. Which RAG stage would you inspect first when the right document never appears?
2. When should frequently changing knowledge use RAG rather than fine-tuning?
3. What distinguishes a Managed from a customer-managed Knowledge Base?
4. Which retrieval APIs apply to each current Knowledge Base type?
5. Why does hierarchical chunking improve precise retrieval with broad context?
6. Which metadata should influence embedding similarity, and which should be filter-only?
7. What is the exact S3 sidecar filename pattern and current size limit?
8. What happens when an embedding dimension does not match the index?
9. When is hybrid search preferable to semantic search?
10. Why can reranking not fix a bad metadata filter?
11. What does an ingestion start response fail to prove?
12. Why is a vector score not factual confidence?
13. What must an application verify before sending ACL `userContext`?
14. Why is DynamoDB not the vector-search answer in a metadata architecture?
15. When should one corpus be split into domain indexes?
16. Why can too many OpenSearch shards increase latency?
17. Which limits must constrain query decomposition?
18. What should a safe agent retrieval tool return and prohibit?

## Official sources

- [Knowledge Bases for Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)
- [Managed versus customer-managed Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-build-managed.html)
- [Create a Managed Knowledge Base](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-create.html)
- [Build a customer-managed Knowledge Base](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-build.html)
- [How Knowledge Base data works](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-how-data.html)
- [Supported vector stores and setup](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-setup.html)
- [Chunking options](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html)
- [S3 data-source metadata](https://docs.aws.amazon.com/bedrock/latest/userguide/s3-data-source-connector.html)
- [Synchronize a Managed Knowledge Base](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-sync.html)
- [Direct ingestion](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-direct-ingestion.html)
- [Knowledge Base retrieval](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-how-retrieval.html)
- [Configure and test retrieval](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-config.html)
- [`Retrieve` API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_Retrieve.html)
- [`RetrieveAndGenerate` API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_RetrieveAndGenerate.html)
- [Managed KB AgentCore Gateway connector](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-connector-managed-kb.html)
- [ACL-aware retrieval](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-retrieve-acl.html)
- [Titan Text Embeddings](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html)
- [Nova Multimodal Embeddings](https://docs.aws.amazon.com/nova/latest/nova2-userguide/embeddings.html)
- [OpenSearch shard strategy](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/bp-sharding.html)
- [Aurora PostgreSQL vector search](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.VectorDB.html)
