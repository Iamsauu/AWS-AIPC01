# Domain 1 — Foundation Model Integration, Data Management, and Compliance

Status: Verified  
Official weight: 31%  
Official tasks: 1.1–1.6  
Last verified: 2026-07-23

## Why this domain matters

Domain 1 is the largest AIP-C01 domain. It tests the front half of a production GenAI system:

```text
requirements
  → measurable proof of concept
  → model and inference choice
  → validated, model-ready data
  → embeddings and vector architecture
  → retrieval and grounding
  → governed prompts and workflows
```

At professional level, do not choose a service from one keyword. Convert the scenario into quality, latency, throughput, availability, residency, security, cost, and operational-ownership constraints. Then eliminate answers that solve only one dimension.

Detailed explanations live in:

- [Bedrock model selection and runtime APIs](../concepts/bedrock-model-selection-and-runtime-apis.md)
- [Data quality and multimodal processing](../concepts/data-quality-and-multimodal-processing.md)
- [RAG, Knowledge Bases, and vector search](../concepts/rag-knowledge-bases-vector-search.md)
- [Prompt engineering, Prompt Management, and Flows](../concepts/prompt-engineering-management-and-flows.md)
- [SageMaker custom-model deployment](../concepts/sagemaker-custom-model-deployment.md)
- [Cost, latency, throughput, and caching](../concepts/cost-latency-throughput-and-caching.md)
- [Evaluation, testing, and quality gates](../concepts/evaluation-testing-and-quality-gates.md)

## Official coverage map

The official Domain 1 blueprint defines 28 skills. This is the completeness checklist.

| Skill | What you must be able to do | Canonical coverage |
|---|---|---|
| 1.1.1 | Turn business needs and technical constraints into an architecture | This page: requirements contract |
| 1.1.2 | Build a PoC that proves feasibility, performance, and business value | This page: PoC scorecard |
| 1.1.3 | Standardize approved components and architecture review | This page; enterprise integration guide |
| 1.2.1 | Select FMs by capability, limitation, measured quality, and economics | Model selection guide |
| 1.2.2 | Switch models/providers without changing application code | Model selection guide: AppConfig and adapters |
| 1.2.3 | Design model resilience and graceful degradation | Model selection guide: inference profiles and fallbacks |
| 1.2.4 | Operate customized-model versions, deployment, rollback, and retirement | Model selection summary; SageMaker deployment guide |
| 1.3.1 | Validate data automatically and publish quality signals | Data processing guide |
| 1.3.2 | Process text, image, audio, video, and tabular inputs | Data processing guide |
| 1.3.3 | Format model-specific inference inputs correctly | Data processing and model API guides |
| 1.3.4 | Normalize, extract, clean, and enhance inputs | Data processing guide |
| 1.4.1 | Design vector and metadata architectures for augmentation | RAG guide |
| 1.4.2 | Design metadata for semantic relevance and filtering | RAG guide |
| 1.4.3 | Scale vector indexes, shards, and domain routing | RAG guide |
| 1.4.4 | Integrate document systems, repositories, and internal wikis | RAG guide |
| 1.4.5 | Keep the index current with incremental, scheduled, or event-driven sync | RAG guide |
| 1.5.1 | Select fixed, semantic, hierarchical, custom, or no chunking | RAG guide |
| 1.5.2 | Select embeddings by modality, language, dimension, quality, and cost | RAG guide |
| 1.5.3 | Deploy a compatible vector-search solution | RAG guide |
| 1.5.4 | Apply semantic, keyword, hybrid, filter, and reranking techniques | RAG guide |
| 1.5.5 | Handle complex queries with expansion, decomposition, and transformation | RAG guide |
| 1.5.6 | Expose retrieval through stable functions, APIs, or MCP tools | RAG and agents/MCP guides |
| 1.6.1 | Control behavior with roles, instructions, templates, schemas, and Guardrails | Prompt guide |
| 1.6.2 | Maintain context and deterministic clarification | Prompt guide |
| 1.6.3 | Govern prompt templates, variables, versions, releases, and audit | Prompt guide |
| 1.6.4 | Test edge cases and prevent prompt regression | Prompt and evaluation guides |
| 1.6.5 | Improve prompts with structured components and feedback | Prompt guide |
| 1.6.6 | Build chains, branches, reusable steps, and pre/post-processing | Prompt guide |

Domain 1 coverage: **28/28 official skills explained or linked to a canonical explanation**.

## Task 1.1 — Analyze requirements and design GenAI solutions

### Requirements contract

Capture these before selecting a model or service:

| Dimension | Questions that change the architecture |
|---|---|
| Business outcome | Is success faster handling, higher conversion, reduced errors, or a regulated decision? |
| Task | Generation, summarization, extraction, classification, semantic search, multimodal analysis, or tool use? |
| Interaction | Synchronous, streaming, queued asynchronous, batch, or long-running approval? |
| Evidence | Must an answer cite current enterprise sources or can the model use general knowledge? |
| Quality | Correctness, groundedness, retrieval relevance, format compliance, consistency, fairness, or task completion? |
| Scale | Requests/minute, tokens/minute, concurrency, corpus size, update rate, and peak pattern? |
| Availability | Single-Region tolerance, geographic failover, recovery target, and degraded mode? |
| Data | Modalities, sensitivity, retention, ownership, residency, source permissions, and update frequency? |
| Integration | Browser, mobile, internal APIs, webhook, event bus, data lake, CRM, or on-premises system? |
| Governance | Approved model/prompt/data versions, audit evidence, human review, and rollback? |
| Economics | Cost per successful task, monthly ceiling, idle capacity, and operational staffing? |

### Bedrock versus SageMaker AI

| Requirement | Prefer | Reason |
|---|---|---|
| Managed access to supported FMs and GenAI features | Amazon Bedrock | Lowest model-serving ownership; native Knowledge Bases, Guardrails, evaluations, prompts, agents, and runtime APIs |
| Arbitrary open-weight/custom container, GPU topology, or serving stack | SageMaker AI | Control over image, instance, storage, scaling, and model artifacts |
| Frequently changing enterprise documents | RAG/Knowledge Bases | Update retrieval data without retraining a model |
| Stable behavior or style not solved by prompt/RAG, with suitable training data | Customization | Higher lifecycle and evaluation burden; prove the simpler approaches failed first |

The closest distractor is often technically possible but unnecessarily custom. “Least operational overhead” favors a native managed capability only when it still satisfies security, control, and feature requirements.

### PoC scorecard

A credible PoC uses representative—not toy—data and defines promotion thresholds before testing.

| Area | Minimum evidence |
|---|---|
| Quality | Task-specific benchmark, reference or rubric, error taxonomy, cohort results |
| Retrieval | Recall/relevance at k, citation coverage, wrong-source rate, stale-source tests |
| Performance | End-to-end latency, time to first token if streaming, throughput, throttles |
| Cost | Input/output/cache tokens, embedding and ingestion cost, vector cost, cost per successful task |
| Safety | Prompt-attack suite, PII tests, denied-topic tests, unsupported-answer behavior |
| Operations | Logs, metrics, traceability, retry/timeout behavior, deployment and rollback path |
| Business value | Baseline versus candidate outcome and an explicit go/no-go threshold |

Do not select a model from public benchmarks alone. Evaluate the exact prompt, data distribution, output contract, Region, and runtime configuration that the application will use.

### Standardized implementation

Publish reusable CDK or CloudFormation components for approved:

- model invocation and mandatory Guardrail attachment;
- private networking, IAM boundaries, KMS keys, and logging;
- Knowledge Base ingestion and retrieval;
- token, latency, error, and cost metrics;
- evaluation gates and rollback;
- source/prompt/model-version attribution.

Review deployments with AWS Well-Architected Tool and the Generative AI Lens. A reusable construct prevents drift; it does not remove each workload owner’s responsibility to validate quality and risk.

## Task 1.2 — Select and configure foundation models

### Selection matrix

Shortlist only models that satisfy hard constraints, then benchmark the survivors.

1. Input/output modality and language.
2. Context and output limits.
3. Tool use, structured output, citations, and streaming support.
4. Compatible API and endpoint.
5. Region, inference-profile routing, and lifecycle state.
6. Supported inference type: on-demand, provisioned, batch, or latency-optimized.
7. Representative quality and safety results.
8. Latency, throughput, token economics, and migration risk.

Use `ListFoundationModels` and `GetFoundationModel` on the Bedrock control plane to discover modalities, streaming support, and lifecycle metadata. Use Bedrock Runtime for inference.

### Deployment and resilience decisions

| Requirement | Default pattern |
|---|---|
| Unpredictable interactive demand | Bedrock on-demand |
| Predictable sustained synchronous demand | Bedrock Provisioned Throughput |
| Large offline managed-FM job | Bedrock batch inference with S3 JSONL input/output |
| Same model, transient Regional capacity pressure | Cross-Region inference profile |
| Residency within a geography | Geographic profile whose documented destination Regions are all approved |
| Arbitrary provider or model switching | Stable API + provider-aware adapter + AppConfig |
| Routine versus complex requests in one supported family | Bedrock intelligent prompt routing after evaluation |
| Custom/open-weight model | SageMaker real-time, asynchronous, or batch endpoint as the workload requires |
| Customized-model release lifecycle | Model Registry + approval + canary/linear rollout + alarms + rollback |

For a model change without application deployment, store model IDs, inference settings, routing rules, and failover flags in AppConfig. Validate the configuration, deploy gradually, and attach CloudWatch alarms for automatic rollback.

Cross-Region inference is capacity routing, not a promise that processing remains in the source Region. A geography profile keeps routing within its documented geography; a global profile can route more broadly. IAM and service control policies must permit every destination in the selected profile.

## Task 1.3 — Implement data validation and processing

### Model-ready data pipeline

```text
land immutable raw input
  → malware/type/size checks
  → parse and normalize
  → validate schema and quality
  → split valid and quarantine records
  → redact or tokenize sensitive values
  → create modality-specific artifacts
  → assemble the exact model request
  → record lineage, version, and quality metrics
```

### Service decision

| Data and requirement | Preferred service |
|---|---|
| Recurring tabular/JSON rule checks | AWS Glue Data Quality with DQDL |
| Existing Python/custom container for batch transforms | SageMaker Processing |
| Documents/images/audio/video to structured insights | Bedrock Data Automation |
| OCR, forms, tables, expenses, or IDs | Amazon Textract |
| Speech recognition, custom vocabulary, transcript PII controls | Amazon Transcribe |
| Text entities, intent, classification, or PII | Amazon Comprehend |
| Lightweight schema validation and assembly | Lambda |

Flatten nested structs/lists before Glue Data Quality rules that do not support them. Publish rule pass/fail results to CloudWatch, route row-level failures to quarantine, and stop downstream embedding/inference when the required threshold fails.

Formatting is an API contract:

- `Converse`/`ConverseStream`: `messages` with `role` and typed content blocks.
- `InvokeModel`: provider/model-specific JSON for the selected `modelId`.
- SageMaker `InvokeEndpoint`: `ContentType` and body matching the deployed container’s schema.
- Multimodal inputs: supported format, size, source, and model capability; do not assume every text model accepts images or documents.

## Tasks 1.4 and 1.5 — Vector stores, embeddings, and retrieval

### End-to-end RAG

```text
source
  → parse
  → chunk
  → enrich metadata
  → embed
  → index
  → retrieve and filter
  → rerank
  → augment
  → generate
  → cite, validate, and evaluate
```

### Current Knowledge Base choice

| Need | Choose |
|---|---|
| Bedrock owns ingestion, datastore, indexing, embedding, reranking, connectors, and agentic retrieval | **Bedrock Managed Knowledge Base** |
| You must select/control the vector store, mapping, dimension, index, parser, or advanced retrieval settings | **Customer-managed Knowledge Base** |
| You need an application-specific retrieval pipeline not supported by Knowledge Bases | Custom RAG |

Current AWS behavior matters:

- `RetrieveAndGenerate` is for customer-managed/vector and other supported non-managed Knowledge Bases; the API documentation says it **cannot** be used with Managed Knowledge Bases.
- Managed Knowledge Bases use `Retrieve` for one-pass retrieval and `AgenticRetrieveStream` for multi-step retrieval and optional citation-backed synthesis.
- Third-party connectors, document-level permission filtering, and native AgentCore Gateway integration are current Managed Knowledge Base advantages.

### Retrieval decision rules

| Symptom/requirement | Change |
|---|---|
| Narrow fact buried in long document | Smaller child chunks or hierarchical chunking |
| Need precise match plus surrounding context | Hierarchical: match child, return parent |
| Topic boundaries vary substantially | Semantic chunking; accept additional ingestion cost |
| Exact IDs and natural language | Hybrid keyword + vector search |
| Right candidates, wrong order | Retrieve more candidates, then rerank |
| Wrong jurisdiction/tenant/version | Correct metadata and apply filters before generation |
| Multi-intent question | Decompose into subqueries, retrieve, merge, then generate |
| Many domains with distinct terminology/models | Domain indexes and routing; measure fanout |
| Stale answers | Incremental sync and terminal ingestion-job verification |

The embedding dimension must match the index mapping. A smaller dimension can reduce storage and search cost, but only an evaluation can show whether domain relevance remains acceptable. Exact identifiers often need lexical fields; embeddings do not reliably preserve exact matching.

For S3 sources, a sidecar is named `fileName.extension.metadata.json`, stored beside the source, and currently limited to 10 KB. Set `includeForEmbedding: true` only for fields whose value should affect semantic similarity. Keep ACLs, revision dates, owners, and other filter-only attributes out of embeddings unless the use case proves otherwise.

Never treat a retrieval similarity score as guaranteed factual confidence. Return sources, verify claim support, and implement an insufficient-evidence path.

## Task 1.6 — Prompt engineering and governance

### Prompt contract

A production prompt should state:

1. role and scope;
2. task and success criteria;
3. trusted context and delimiter rules;
4. constraints and forbidden behavior;
5. representative examples when useful;
6. output schema;
7. behavior when information is missing or conflicting.

Use lower temperature and a narrower candidate distribution when the task needs repeatability, but validate on a benchmark. Set a task-sized `maxTokens`; excessive output allocation can waste capacity and increase truncation or throttling risk elsewhere.

When supported, Bedrock structured outputs constrain the response to a JSON schema and are stronger than the instruction “return JSON.” Still validate semantic and business constraints after parsing.

### Governance and workflow

| Need | Feature |
|---|---|
| Reusable variables, draft, variants, numbered snapshots | Prompt Management |
| Sequential prompts, conditions, Knowledge Base/Lambda steps | Bedrock Flows |
| Long-running business process, durable approval, broad AWS orchestration | Step Functions Standard |
| Fast heuristic rewrite of one short prompt | Simple prompt optimization |
| Evaluation-driven optimization across samples/models | Advanced Prompt Optimization |
| Production approval | CI/CD or manual approval gate around a versioned prompt |
| API activity | CloudTrail |
| Per-request prompt/model/user attribution | Sanitized application logs and/or invocation metadata |

Prompt Management creates a mutable `DRAFT` and numbered version snapshots. Its current feature documentation does **not** document a native approval status or reviewer workflow. Treat “approved” as a release-process state enforced by a pipeline or manual approval step, then deploy the exact numbered prompt ARN.

Flows support version/alias deployment and per-node traces. They are not a substitute for Step Functions when the requirement is a durable human approval lasting hours or days.

## Cross-domain decision table

| Requirement phrase | First answer to consider | Main distractor |
|---|---|---|
| Current documents, no retraining | Knowledge Base/RAG | Fine-tuning to memorize changing content |
| Lowest ownership, supported connectors, document ACLs | Managed Knowledge Base | Self-managed crawler and vector cluster |
| Exact vector mapping or database control | Customer-managed Knowledge Base | Fully managed datastore with no index access |
| Natural language plus codes/IDs | Hybrid search | Semantic-only retrieval |
| Precision and larger context | Hierarchical chunking | One very large chunk |
| Reusable versioned prompt | Prompt Management | Prompt copied into each repository |
| Editable prompt workflow with branches | Bedrock Flows | Hard-coded Lambda orchestration |
| Long-lived approval/audit workflow | Step Functions Standard | A prompt or Flow pretending to be an approval system |
| Switch routing without deployment | AppConfig + adapter | Hard-coded `modelId` |
| Predictable steady inference | Provisioned Throughput | More Lambda concurrency |
| Large offline inference | Bedrock batch inference | One synchronous invocation per record |
| Structured extraction from evolving documents | Bedrock Data Automation blueprint | Hand-built OCR/parser stack |

## Security and compliance

- Use least-privilege IAM for models, prompts, Knowledge Bases, source buckets, embedding models, and vector stores.
- Encrypt data at rest with supported KMS controls and in transit; use private endpoints where required.
- Treat a source connector’s permissions separately from application authorization.
- For Managed Knowledge Bases, ACL-aware filtering requires authenticated, verified `userContext`; Bedrock does not authenticate the end user.
- Keep tenants/jurisdictions separated through authorization plus metadata filters; filters alone are not the complete security boundary.
- Sanitize telemetry. Do not log raw prompts, retrieved chunks, or PII merely because auditability is required.
- Record model, prompt, Knowledge Base/data-source, ingestion, and evaluation versions so a response can be reconstructed.
- Confirm every Region in an inference profile and every model/provider data-use term before deployment.

## Cost, latency, and reliability

| Cost/latency driver | Control |
|---|---|
| Repeated long prompt prefix | Prompt caching on supported models |
| Excess conversation history | Summarize/prune old turns; keep recent turns |
| Too many overlapping RAG chunks | Deduplicate sources, improve chunking/filtering, cap results |
| High vector storage | Evaluate smaller dimensions or binary/on-disk options with compatible stores |
| Semantic chunking/parser FM cost | Use only when quality gain exceeds ingestion cost |
| Repeated full reingestion | Incremental sync; coalesce source events |
| Predictable throttling | Capacity plan and consider Provisioned Throughput |
| Offline volume | Batch inference/processing |
| Perceived chat delay | Streaming; separately measure full response latency |
| Regional capacity event | Compatible cross-Region inference profile and tested degradation |

## Failure modes and troubleshooting

| Symptom | Evidence to inspect | Likely cause | Corrective action |
|---|---|---|---|
| Model validation error | API, endpoint, model ID, request body | Converse/native payload mixed up | Use the documented request contract |
| Model switch silently changes behavior | Capability matrix and evaluation | Unsupported fields or different defaults | Provider-aware config plus regression gate |
| Ingestion started but answers are stale | `GetIngestionJob` status/statistics | Workflow marked success too early | Wait for terminal `COMPLETE`; alarm on failures |
| Retrieval returns wrong tenant/version | Metadata and filter trace | Missing/incorrect metadata or user context | Repair metadata, authorization, and pre-filter |
| Exact code is missed | Retrieved chunks and query analysis | Semantic-only search | Hybrid keyword/vector retrieval |
| Right chunk ranks low | Candidate and reranked order | Candidate set too small or poor ranking | Increase candidates, rerank, re-evaluate |
| Broad irrelevant chunks | Chunk boundaries | Chunk too large | Fixed/semantic/hierarchical redesign |
| Context-window failure | CountTokens and chunk/history sizes | Too much history or retrieval context | Prune, summarize, filter, reduce result count |
| JSON parsing failures | Raw response and schema support | Prompt-only formatting or unsupported schema | Use structured outputs when supported; validate |
| Prompt release regresses | Versioned benchmark | No quality gate or mixed versions | Evaluate candidate, approve exact version, rollback |
| Glue rules fail on nested data | Input schema and Glue output | Struct/list not flattened | Flatten before DQ evaluation |
| Transcription still exposes PII | Redacted and sampled output | ML redaction miss or unsupported type/language | Validate, layer controls, fail closed where required |

## Common exam traps

1. RAG updates knowledge; fine-tuning changes model behavior. Do not retrain for frequently changing documents.
2. Starting `StartIngestionJob` is not successful synchronization. Check the terminal status and statistics.
3. A vector score is relevance evidence, not calibrated factual confidence.
4. Metadata filtering improves precision but does not replace authentication and authorization.
5. Smaller embeddings save storage only if retrieval quality remains above threshold.
6. More shards are not automatically faster. Excess shards increase memory and coordination.
7. Streaming improves time to first token, not necessarily total generation time.
8. Provisioned Throughput solves predictable model capacity, not offline batch economics.
9. Prompt instructions are not a safety control. Apply Guardrails and deterministic validation separately.
10. Versioning is not approval. Prompt Management snapshots a prompt; the release workflow approves it.
11. A Flow is not the default for durable human approval; Step Functions Standard is.
12. Current Managed Knowledge Bases and customer-managed/vector Knowledge Bases do not have identical APIs or features.

## Verified documentation versus mock assumptions

| Mock claim | Current documentation-backed interpretation |
|---|---|
| PE1-Q6: Prompt Management supplies an approval workflow | Overstated. Prompt Management supplies drafts, variants, and versions; enforce approval in deployment governance. PE2-Q4 uses the safer pattern. |
| PE1-Q10/33/40: use `RetrieveAndGenerate` with a generally described “managed” KB | Valid for customer-managed/vector Knowledge Bases. The current `RetrieveAndGenerate` API explicitly excludes **Managed Knowledge Bases**. |
| PE1-Q35: Confluence/SharePoint connectors on a general Knowledge Base | Current documentation places third-party connectors and document ACLs with Managed Knowledge Bases. Check the KB type, not just the product name. |
| PE1-Q49: Lambda batch-generates millions of embeddings | Possible, but not automatically least-operations at large scale. Managed KB ingestion or Bedrock batch-capable embedding patterns can be better; benchmark quotas and orchestration. |
| PE1-Q65: choose a Titan multimodal embedding by brand | Learn the capability requirement: shared cross-modal vector space. Select a currently supported multimodal embedding model; current docs also describe Nova Multimodal Embeddings. |
| PE2-Q36: 30–50 GiB shards as a universal search target | Not universal. Current OpenSearch guidance cites 10–30 GiB when search latency is the priority and 30–50 GiB for write-heavy workloads. Measure the actual workload. |
| PE2-Q44: OpenSearch Serverless is lowest ownership when no vector service exists | It remains a low-operations customer-managed option. Current Managed Knowledge Bases remove even more datastore ownership if custom index control is not required. |
| PE2-Q57/70/75: Managed KB uses `Retrieve`/`AgenticRetrieveStream`, Gateway connector, and ACL context | Confirmed by current AWS documentation. |

## Local mock references

These questions are learning evidence, not authoritative product documentation.

| Topic | Local questions |
|---|---|
| PoC and standardization | PE1-Q10; PE2-Q37; PE2-Q67 |
| Model selection, API fit, and dynamic routing | PE1-Q21, Q59, Q71; PE2-Q17, Q19, Q31, Q38, Q55, Q56, Q61 |
| Resilience and capacity | PE1-Q18, Q45, Q60; PE2-Q9, Q35 |
| Customized-model lifecycle | PE1-Q54, Q62; PE2-Q25 |
| Data quality and multimodal processing | PE1-Q1, Q37, Q38, Q47, Q72; PE2-Q51, Q72 |
| Chunking and embeddings | PE1-Q17, Q49, Q65; PE2-Q22, Q26, Q36 |
| Retrieval, vector stores, and ingestion | PE1-Q20, Q31, Q33, Q35, Q40, Q42, Q51, Q58, Q61; PE2-Q33, Q44, Q47, Q58, Q70, Q73, Q75 |
| Grounding and citations | PE1-Q22, Q43, Q46; PE2-Q45, Q52, Q57 |
| Prompt construction and governance | PE1-Q6, Q9, Q24, Q29, Q30, Q34; PE2-Q4, Q8, Q23, Q24, Q27, Q30, Q43, Q49, Q54, Q71 |

## Hands-on validation

Complete these before marking Domain 1 learned:

1. Build a model matrix from `ListFoundationModels`/`GetFoundationModel`; reject one model for a concrete capability or lifecycle reason.
2. Invoke one message model with `Converse`, one specialized model with `InvokeModel`, and preflight both with `CountTokens` where supported.
3. Create a DQDL ruleset, flatten a nested field, split valid/quarantine records, and alarm on a failed rule.
4. Process a document/image with a Bedrock Data Automation blueprint and inspect the asynchronous structured output.
5. Create a small Knowledge Base, test two chunking strategies, attach metadata, sync an update, and prove stale content disappears.
6. Run `Retrieve` separately from generation, compare semantic versus hybrid retrieval, then add reranking.
7. Create a Prompt Management draft and numbered version; invoke the exact version with variables.
8. Build a Flow with prompt, condition, Knowledge Base, and Lambda nodes; enable the trace and deploy through an alias.
9. Run an evaluation-driven prompt change and reject it if a held-out quality or format threshold regresses.

## Recall questions

1. Which five hard constraints should eliminate an FM before quality benchmarking?
2. Why does an AppConfig model ID not by itself create a multi-provider abstraction?
3. When does a geographic inference profile fail even though the source Region is allowed?
4. Why does more Lambda concurrency not solve Bedrock model-capacity throttling?
5. Which pipeline step must occur before Glue Data Quality evaluates nested lists?
6. When is Bedrock Data Automation preferable to Textract, and when is Textract preferable?
7. What breaks if an embedding outputs 512 dimensions but the vector mapping expects 1,024?
8. What is the current API difference between Managed and customer-managed Knowledge Bases?
9. Which metadata belongs in an embedding, and which should remain filter-only?
10. Why is hierarchical chunking useful for a narrow clause requiring surrounding context?
11. When is hybrid search preferable to semantic search?
12. Why is a retrieval score not a confidence score?
13. What evidence proves an ingestion refresh completed?
14. Which prompt features are governed by Prompt Management, and which release control is external?
15. Why is a JSON instruction weaker than Bedrock structured outputs?
16. When should a Flow be replaced by Step Functions Standard?

## Official sources

- [AIP-C01 Domain 1](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain1.html)
- [Generative AI Lens lifecycle](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/generative-ai-lifecycle.html)
- [Bedrock model availability and compatibility](https://docs.aws.amazon.com/bedrock/latest/userguide/models.html)
- [Cross-Region inference profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-support.html)
- [AWS Glue Data Quality](https://docs.aws.amazon.com/glue/latest/dg/glue-data-quality.html)
- [Bedrock Data Automation](https://docs.aws.amazon.com/bedrock/latest/userguide/bda-how-it-works.html)
- [Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)
- [Managed versus customer-managed Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-build-managed.html)
- [Knowledge Base retrieval APIs](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-how-retrieval.html)
- [Prompt Management](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-management.html)
- [Bedrock Flows](https://docs.aws.amazon.com/bedrock/latest/userguide/flows.html)
- [Bedrock structured outputs](https://docs.aws.amazon.com/bedrock/latest/userguide/structured-output.html)
