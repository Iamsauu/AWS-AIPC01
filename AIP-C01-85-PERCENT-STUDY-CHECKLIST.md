# AWS Certified Generative AI Developer – Professional (AIP-C01)
## Knowledge checklist for an 85% practice-exam target

Last verified against the official AWS exam guide: 23 July 2026.

## First, calibrate the target

- The certification is **AWS Certified Generative AI Developer – Professional**, exam code **AIP-C01**.
- The exam has **75 questions**: 65 scored and 10 unscored.
- AWS reports a scaled score from 100–1,000; the passing score is **750**.
- Therefore, “85%” is a useful **practice-exam safety margin**, but it is not a direct conversion to an AWS scaled score.
- Multiple-response questions require every correct response; do not expect partial credit.
- There is no penalty for guessing.

Official domain weights:

| Domain | Weight |
|---|---:|
| 1. Foundation Model Integration, Data Management, and Compliance | 31% |
| 2. Implementation and Integration | 26% |
| 3. AI Safety, Security, and Governance | 20% |
| 4. Operational Efficiency and Optimization | 12% |
| 5. Testing, Validation, and Troubleshooting | 11% |

Domains 1–3 are 77% of the scored exam. They deserve most of the study time, but Domains 4–5 cannot be skipped if the goal is an 85% buffer.

## What “know” must mean at professional level

For every topic below, you should be able to:

1. Recognize the requirement and hidden constraint in a scenario.
2. Select the correct AWS service, feature, API, or architecture.
3. Explain why it is better than each distractor.
4. Identify security, reliability, latency, cost, and operational-overhead tradeoffs.
5. Diagnose a broken implementation from its symptoms.

Memorizing service definitions is not sufficient.

---

# Domain 1 — FM integration, data management, and compliance (31%)

## 1.1 Requirements and GenAI solution design

Know how to translate business requirements into:

- Use case: generation, summarization, extraction, classification, semantic search, tool use, multimodal processing, or agentic workflow.
- Interaction: synchronous, streaming, asynchronous, batch, or long-running human approval.
- Quality target: relevance, factual accuracy, groundedness, consistency, fluency, fairness, or task completion.
- Non-functional requirements: latency, throughput, availability, Region/data residency, privacy, auditability, cost, and operational overhead.
- Build-versus-managed choice: Amazon Bedrock managed capabilities versus SageMaker AI/custom containers.
- Proof of concept: representative dataset, quality metrics, latency, token consumption, cost, and a clear promotion threshold.
- Standard architecture: AWS Well-Architected Tool, Generative AI Lens, reusable AWS CDK/CloudFormation components, and cross-account controls.

Be able to design a PoC that measures business value and technical feasibility before production.

## 1.2 Foundation model selection and lifecycle

Build a model-selection matrix covering:

- Modality: text, image, audio, or multimodal.
- Context-window and output-token limits.
- Tool/function calling support.
- Converse/ConverseStream support versus provider-specific InvokeModel payloads.
- Streaming capability.
- Region availability and model lifecycle status.
- On-demand, provisioned throughput, batch inference, and latency-optimized inference support.
- Quality on a representative dataset.
- Latency, throughput, input/output token price, and price-to-quality ratio.
- Safety, language, and domain fit.

Know these patterns:

- **Dynamic selection without deployment:** model IDs, parameters, routing rules, and feature flags in AWS AppConfig; automatic rollback using CloudWatch alarms.
- **Multi-provider stable contract:** API Gateway plus a Lambda adapter that normalizes provider-specific requests and responses.
- **Regional resilience:** Bedrock cross-Region inference profiles, including geographic profiles when residency matters.
- **Predictable sustained demand:** Bedrock Provisioned Throughput.
- **Large offline workloads:** Bedrock batch inference or SageMaker Batch Transform.
- **Custom/open-weight model:** SageMaker AI endpoint with an appropriate serving container.
- **Customized model lifecycle:** LoRA/adapters, SageMaker Model Registry, approval status, canary/linear deployment, monitoring, and rollback.

Understand that model development and advanced training theory are out of scope, but deploying and operating customized FMs is in scope.

## 1.3 Data validation and multimodal preprocessing

Know how to build automated, monitored pipelines:

- AWS Glue Data Quality and DQDL for completeness, uniqueness, validity, and row-level pass/fail.
- Quarantine invalid records; do not allow them into embedding or inference pipelines.
- Publish data-quality metrics to CloudWatch.
- Flatten and normalize nested JSON before validation where needed.
- SageMaker Data Wrangler for interactive preparation; SageMaker Processing for managed custom processing.
- Amazon Bedrock Data Automation for structured extraction from documents and images using blueprints.
- Amazon Transcribe for audio, custom vocabulary, and transcript PII redaction.
- Amazon Textract for document OCR/forms/tables when Bedrock Data Automation is not the required managed GenAI workflow.
- Amazon Comprehend for entities, intent, and PII in text.
- Lambda for lightweight normalization, assembly, schema validation, and orchestration.
- Correct model-ready JSON, conversation message structure, content types, and multimodal input packaging.

Be able to choose a service by data type:

| Input | Likely managed processing |
|---|---|
| JSON/tabular | Glue, Glue Data Quality, SageMaker Processing |
| PDF/image with layout | Bedrock Data Automation or Textract |
| Audio | Transcribe |
| Noisy text/entities/PII | Comprehend plus Lambda |
| Mixed modalities | Specialized processors, S3 staging, then Lambda assembly |

## 1.4 Vector stores and embedding architecture

Understand:

- What embeddings represent and why vector similarity enables semantic retrieval.
- Dimension, precision/binary representation, storage cost, latency, and relevance tradeoffs.
- Why dimension must match the vector-index mapping.
- Why a lower dimension is valid only after evaluation proves that relevance remains above the requirement.
- Batch generation of embeddings for ingestion.
- Multimodal embeddings when text and images must share a vector space.

Know the main vector-store patterns:

- Amazon OpenSearch Serverless: low operational overhead and managed Knowledge Bases integration.
- Amazon OpenSearch Service: control over domains, shards, indexes, neural/hybrid search, and scaling.
- Amazon Aurora PostgreSQL with `pgvector`: relational filtering plus vectors.
- Supported third-party/custom Knowledge Bases vector stores when required.
- Metadata in DynamoDB or relational stores when vectors and operational records have different access patterns.

Know index design:

- HNSW/FAISS concepts at the level needed to select a compatible index.
- One index versus per-domain indexes.
- Shard sizing and why excessive shards increase query fan-out and coordination.
- Hierarchical/top-level routing when many domains exist.
- Reindexing implications: vector dimension and primary shard choices are expensive to change later.
- Monitoring indexing/search failures, JVM or memory pressure, latency, and ingestion statistics.

## 1.5 RAG and retrieval

Master the complete pipeline:

`source → parse → chunk → embed → index → retrieve → rerank/filter → augment prompt → generate → cite/evaluate`

Know:

- Bedrock Knowledge Bases data sources, managed ingestion, vector-store configuration, synchronization, and ingestion-job status.
- S3, Confluence, SharePoint, and other supported connectors; use supported document-level permissions when per-user source authorization is required.
- Incremental/scheduled refresh and source-change-triggered ingestion.
- Do not mark refresh complete until the ingestion job reaches a successful terminal state.
- Fixed-size, semantic, and hierarchical chunking.
- Hierarchical chunking: retrieve a precise child chunk but return its larger parent for context.
- Metadata filters for business unit, owner, date, jurisdiction, document version, or tenant.
- S3 metadata sidecar files and `includeForEmbedding`: distinguish fields that affect semantic similarity from filter-only fields.
- Embedding-model selection by domain, language, dimension, quality, and cost.
- Semantic versus keyword search.
- **Hybrid search** for natural language plus exact identifiers such as error codes, CVEs, policy IDs, or product numbers.
- Reranking to improve ordering of the top candidate chunks.
- Query expansion, decomposition, and transformation for multi-intent questions.
- `Retrieve` when the application controls final generation.
- `RetrieveAndGenerate` when a managed retrieval-plus-generation path is appropriate.
- Citation spans, retrieved references, locations, metadata, and scores.
- A retrieval score is not guaranteed factual confidence.
- Contextual grounding checks and an “insufficient evidence” path.
- Stable retrieval interfaces through MCP or AgentCore Gateway when several clients/stores need one contract.

## 1.6 Prompt engineering and prompt governance

Know prompt construction:

- System/role instructions, task, context, constraints, examples, output schema, and failure behavior.
- Parameterized variables and reusable templates.
- Structured JSON outputs and schema validation.
- Few-shot examples and reusable prompt components.
- Prompt chaining and conditional branching.
- Clarification when required information is missing.

Know inference parameters:

- Lower temperature/narrower top-p or top-k generally improves determinism.
- Higher temperature generally increases diversity.
- `maxTokens` must match the task; an unnecessarily high value can consume throughput allocation.
- Always validate changes with evaluation data rather than relying on intuition.

Know AWS features:

- Bedrock Prompt Management: reusable variables, versions, stable prompt ARN, variants, and traceable releases.
- Bedrock Flows/Prompt Flows: prompt nodes, condition nodes, Knowledge Base nodes, Lambda nodes, and traceable per-node execution.
- Advanced Prompt Optimization: curated evaluation samples, target models, evaluation method, review, then version the approved result.
- Prompt regression testing, edge-case datasets, and release quality gates.
- CloudTrail for API activity; application logs for who used which prompt/model version.

Important mock correction: do **not** assume that Prompt Management itself provides a complete native approval workflow. Use a deployment pipeline/manual approval process for controlled promotion. Practice Exam 1 Question 6 overstates this; Practice Exam 2 Question 4 gives the safer architecture.

---

# Domain 2 — Implementation and integration (26%)

## 2.1 Agentic AI, tools, MCP, and human review

Understand agent-loop components:

- Goal/instructions, planning or ReAct loop, tool selection, observation, memory/state, stopping condition, final response.
- Deterministic orchestration must control high-risk actions; the FM should not be the only workflow controller.
- Maximum iterations, timeouts, retries, error branches, unsafe terminal states, and circuit breakers.
- Least-privilege IAM per tool.
- Human-in-the-loop approval for regulated or consequential actions.

Know Step Functions Standard workflows:

- Task, Choice, Parallel, Wait/callback, Succeed, and Fail states.
- Execution history for audit.
- Retry versus Catch behavior.
- Callback task token and `SendTaskSuccess`/`SendTaskFailure`.
- Long-running approvals lasting hours or days.
- Loop counter and risk flags.
- Circuit-breaker state in DynamoDB with TTL/cooldown.

Know tool integration:

- Strict JSON/tool schema, required parameters, types, validation, idempotency, structured errors, and safe retry.
- Lambda for stateless/lightweight tools.
- ECS/Fargate or AgentCore Runtime for native dependencies, long-running, CPU-heavy, stateful, or streaming MCP tools.
- MCP client/server contract and streamable HTTP.
- AgentCore Gateway targets and target-prefixed tool names.
- Strands Agents and AWS Agent Squad for specialized/multi-agent systems.
- AgentCore Memory: actor ID, per-session ID, short-term context, long-term preferences, batch writes, and explicit session close/flush.
- Agent tracing and AgentCore/Bedrock agent evaluations.

## 2.2 Model deployment choices

Be able to distinguish:

| Requirement | Preferred pattern |
|---|---|
| Bursty interactive requests | Bedrock on-demand |
| Predictable high synchronous demand | Bedrock Provisioned Throughput |
| Large asynchronous offline generation | Bedrock batch inference |
| Custom/open-weight interactive model | SageMaker real-time endpoint |
| Offline custom-model processing | SageMaker Batch Transform |
| Large LLM serving | SageMaker LMI container |

For SageMaker LLM deployments, understand:

- GPU memory and model-loading constraints.
- Uncompressed model artifacts using `ModelDataSource` where appropriate.
- EBS volume sizing.
- Model artifact download timeout.
- Container startup health-check timeout.
- LMI containers and large-model serving.
- Batch Transform `MultiRecord`, concurrency, and payload-size tuning.
- Endpoint autoscaling, utilization, deployment guardrails, and rollback.

## 2.3 Enterprise integration architecture

Know:

- API-based integration for legacy and enterprise systems.
- Event-driven loose coupling with EventBridge.
- Queue-based buffering and backpressure with SQS.
- SNS for fanout/notification.
- Immediate webhook acknowledgment followed by asynchronous processing.
- Idempotency keys for at-least-once event delivery.
- DynamoDB for job state, results, sessions, and TTL.
- Hybrid/on-premises access through Outposts and private routing.
- Geographic inference profiles for residency-constrained inference.
- A centralized GenAI gateway for validation, throttling, normalization, guardrail enforcement, audit logging, and routing.

Know CI/CD:

- CodePipeline stages and approvals.
- CodeBuild security, prompt regression, guardrail integration tests, and test reports.
- CodeDeploy Lambda canary/linear traffic shifting and CloudWatch-triggered rollback.
- CloudFormation/CDK deployment.
- Last-known-good configuration and supported pipeline rollback behavior.

## 2.4 Bedrock API contracts

Know these precisely:

- Use the **regional `bedrock-runtime` endpoint** for inference.
- `Converse`: common message-style synchronous API across supported models.
- `ConverseStream`: common streaming chat API across supported models.
- `InvokeModel`: model-specific JSON request body; build model-aware serialization.
- `InvokeModelWithResponseStream`: model-specific streaming where supported.
- `additionalModelRequestFields`: provider/model-specific options with Converse.
- `GetFoundationModel`: verify capabilities such as streaming support.
- `CountTokens`: estimate input with the actual model and request shape before invocation.
- `ApplyGuardrail`: independently evaluate INPUT or OUTPUT content.
- Knowledge Bases `Retrieve` versus `RetrieveAndGenerate`.
- Agent `InvokeAgent` with trace enabled.
- Correct `Content-Type`, model ID/profile ARN, message roles, and content blocks.
- Customized models can require different native request schemas.

Know resilient API behavior:

- Request validation with API Gateway JSON schema.
- AWS SDK exponential backoff and jitter for retryable errors.
- Rate limiting, quotas, token-based throttling, fallback, and graceful degradation.
- Correlation IDs and X-Ray across API Gateway, Lambda, and downstream calls.

## 2.5 Application patterns and developer tools

Know:

- API Gateway HTTP API versus REST API versus WebSocket.
- REST/Lambda response streaming and server-sent events when the existing browser contract must remain REST.
- WebSockets for bidirectional/long-lived streaming use cases.
- Synchronous short requests versus SQS-backed job IDs for long requests.
- Parallel model calls with Step Functions Parallel state.
- Bedrock Flows for managed prompt/workflow configuration.
- AWS Amplify authentication, UI components, conversation routes, owner-based authorization, and rapid accessible UI delivery.
- OpenAPI for API-first contracts.
- Amazon Q Developer for IDE chat, AWS SDK help, code suggestions, refactoring, project guidance, and security scanning.

---

# Domain 3 — AI safety, security, and governance (20%)

## 3.1 Input/output safety and hallucination control

Know Bedrock Guardrails:

- Harmful content filters.
- Denied topics and word/profanity filters.
- Sensitive-information filters, PII masking, and custom regex.
- Prompt-attack/jailbreak detection.
- Contextual grounding/relevance checks.
- Automated reasoning policy checks where applicable.
- Guardrail tiers, thresholds, `BLOCK`, `DETECT`, and `MASK` behavior.
- Guardrail identifier and version on runtime calls.
- Guard content input tags.
- Applying a guardrail directly during inference versus calling `ApplyGuardrail`.
- IAM condition controls that require an approved guardrail and prevent bypass.

Use defense in depth:

`API validation/throttling → input sanitization/PII redaction → guardrail INPUT → model/tool restrictions → guardrail OUTPUT → deterministic post-validation → safe response`

Know:

- Prompt injection, jailbreak, instruction override, prompt leakage, data exfiltration, unsafe tool parameters, and adversarial testing.
- Ground with RAG; verify citations and retrieved evidence.
- Enforce output JSON schema.
- Fail closed for regulated/high-risk decisions.
- Parameterized read-only SQL/templates instead of executing arbitrary FM-generated SQL.
- Human review only for ambiguous or high-risk cases to preserve normal-path latency.

## 3.2 Data security and privacy

Know:

- IAM least privilege, resource scoping, condition keys, and separate roles per component/tool.
- IAM Identity Center federation and temporary credentials; never use long-term developer access keys when SSO is available.
- VPC interface endpoints/PrivateLink for Bedrock Runtime, Athena, Glue, and other supported services.
- S3 gateway endpoint and endpoint/bucket policies.
- Private subnets without public internet routes when required.
- Lake Formation table, row, and column controls.
- KMS/encryption, Secrets Manager, and secure credential handling.
- Comprehend for real-time text PII detection/redaction.
- Macie for sensitive-data discovery in S3, not synchronous prompt redaction.
- Consistent placeholders when redaction must preserve context.
- S3 Lifecycle for retention/deletion.
- Disable or sanitize invocation logging when raw sensitive content must not be retained.
- Never place raw PII/PHI in logs, traces, request metadata, or guardrail audit fields.

## 3.3 Governance, audit, and compliance evidence

Know:

- CloudTrail: control-plane/API audit activity.
- CloudWatch structured application logs: runtime evidence, prompt version, model ID, validation result, sanitized metadata, latency, and correlation ID.
- Bedrock model invocation logging: supported prompt/response invocation evidence; configure per account and Region.
- Glue Data Catalog/lineage and source metadata for data attribution.
- SageMaker Model Cards: intended use, limitations, risk, model version, and linked evaluation evidence.
- AWS Audit Manager Generative AI Best Practices Framework: automated plus manually collected evidence.
- Audit Manager results and model cards are evidence artifacts, not legal compliance certifications.
- Organizational policies, approvals, versioning, retention, audit trail, monitoring, and remediation.

## 3.4 Responsible AI

Know how to operationalize:

- Fairness: stratified/balanced cohort datasets and cohort-level comparisons.
- Explainability/transparency: user citations and orchestration traces.
- Privacy: minimization, masking, retention, and access control.
- Safety: adversarial testing, guardrails, human escalation, and fail-closed paths.
- Robustness: regression tests, drift monitoring, fallback, and rollback.
- Accountability: owners, model cards, approval gates, logs, and reviewer feedback.

Critical distinction:

- **Citation/retrieved references** explain which sources support an answer.
- **Agent trace** explains orchestration steps, tool calls, and failures.
- Neither is provider-internal chain-of-thought.
- Do not invent a “model confidence score.” Use measured grounding, relevance, task, or evaluation scores and label them accurately.

---

# Domain 4 — Operational efficiency and optimization (12%)

## 4.1 Cost and token efficiency

Know:

- Input and output tokens both affect cost and throughput.
- `CountTokens` before invocation; include planned output limit in admission control.
- Per-tenant token buckets are more accurate than request-count limits for variable prompt sizes.
- Trim redundant instructions and retrieval context.
- Summarize older conversation turns; store a running summary plus recent turns.
- Reduce retrieved chunk count and apply metadata filters.
- Set a task-specific `maxTokens`.
- Prompt compression and context pruning.
- Prompt caching: stable prefix, cache checkpoint, model-specific minimum size, changing content after the checkpoint.
- Semantic/application caching for equivalent deterministic requests.
- Edge caching only when responses are safe to share and request identity is deterministic.
- Smaller models for routine tasks; larger models for complex requests.
- Model cascading or Bedrock Intelligent Prompt Routing.
- Compare cost-to-quality, not cost alone.

## 4.2 Performance and throughput

Know:

- Streaming reduces perceived latency, not necessarily total generation time.
- Measure time to first token (TTFT) separately from end-to-end latency.
- Latency-optimized inference where supported.
- Parallelize independent model/tool calls.
- Batch inference for asynchronous volume.
- Provisioned Throughput for predictable synchronous capacity.
- Cross-Region inference for resilience/capacity when residency permits.
- Reduce oversized output limits that consume token-based capacity.
- Reuse SDK/HTTP clients outside Lambda handlers; keep-alive and connection pooling.
- Optimize vector indexes, filters, chunk counts, shard count/size, and reranking.
- Capacity planning by requests, input/output tokens, concurrency, and model latency.

## 4.3 Monitoring and observability

Build dashboards/alarms for:

- Invocation count, input/output tokens, errors, throttles, invocation latency, and TTFT.
- Cost and cost anomalies.
- Quality, grounding, hallucination, response drift, and prompt regression.
- Retrieval latency, failures, relevance, indexing, document/chunk success/failure.
- Agent task completion, repeated tool calls, tool latency/error, and loop count.
- Business outcomes and user ratings.

Know the evidence sources:

- CloudWatch service metrics and custom metrics.
- Structured CloudWatch Logs plus Logs Insights.
- Bedrock model invocation logging and `requestMetadata` for product/tenant/experiment attribution.
- X-Ray for cross-service request-path latency.
- Agent/Flow traces for orchestration.
- CloudWatch Synthetics for deployed user journeys.
- CloudTrail for API audit.

---

# Domain 5 — Testing, validation, and troubleshooting (11%)

## 5.1 Evaluation systems

Prepare representative, versioned test datasets. Know JSONL dataset requirements at a practical level.

Know evaluation dimensions:

- Relevance/helpfulness.
- Factual accuracy/correctness.
- Faithfulness/groundedness.
- Context relevance and context coverage for retrieval.
- Citation quality.
- Consistency and semantic drift.
- Fluency and coherence.
- Safety, toxicity, bias, and fairness.
- Agent task completion, tool-use effectiveness, and reasoning/orchestration quality.
- Latency, token efficiency, cost-to-quality, and business outcomes.

Know evaluation types:

- Bedrock model evaluation.
- LLM-as-a-judge with explanations.
- Human-based evaluation and comparison ratings.
- Bring-your-own/precomputed external-model responses.
- RAG retrieve-only evaluation.
- RAG retrieve-and-generate evaluation.
- Agent and AgentCore evaluation.
- A/B and cohort testing.
- Regression, synthetic-user, canary, and continuous evaluation.

Know the release pattern:

`fixed benchmark → candidate evaluation → quality/cost/latency gate → canary cohort → live feedback/monitoring → promote or rollback`

Store results in S3; analyze with Glue/Athena; report trends with QuickSight when stakeholder dashboards are required.

Readiness requires human feedback as well as automated evaluation for subjective or high-risk use cases.

## 5.2 Troubleshooting matrix

| Symptom | Check first | Likely remediation |
|---|---|---|
| Bedrock validation error | API, model capability, body schema, content type, message roles | Correct Converse or provider-native serialization |
| Generation parameter ignored | Whether the model/API supports it | Use documented model-specific fields |
| SageMaker JSON parse error | Endpoint input schema and content type | Send the exact expected JSON |
| Prompt exceeds context | CountTokens, history, retrieved chunks, output limit | Summarize, prune, filter, reduce chunks/maxTokens |
| Hallucinated answer | Retrieved evidence, grounding, citations | Improve retrieval, suppress unsupported output |
| Exact code not found | Vector-only search | Hybrid keyword + vector search, metadata filter, rerank |
| Wrong jurisdiction/version | Metadata and retrieval results | Fix metadata, filters, synchronization, then prompts |
| Broad irrelevant chunks | Chunking strategy | Smaller/hierarchical chunks and reranking |
| Retrieval latency | Shards, fan-out, filters, index pressure | Reindex/tune shards, segment domains, reduce candidates |
| Throttling | Token rate, output limit, concurrency | Lower maxTokens, batch, provisioned throughput, backoff |
| Slow first token | Static prefix, context size, non-streaming path | Prompt cache, trim context, ConverseStream, optimized latency |
| Repeated tool calls | Trace, tool errors, missing stop rule | Structured tool errors, max cycles, circuit breaker |
| Lambda downstream latency | Connection setup | Reuse SDK clients and enable keep-alive |
| SageMaker endpoint fails startup | Artifact size/download/load time/storage | LMI, uncompressed source, volume and timeout settings |
| Prompt/model release regresses | Versioned benchmark and baseline | Automated evaluation gate and rollback |
| Cannot isolate RAG fault | Retrieval result versus final answer | Test `Retrieve` separately before changing generation |

---

# AWS service decision knowledge

You do not need equal depth in every in-scope service. You do need to recognize the following decision boundaries:

| Need | Know the difference |
|---|---|
| Foundation-model APIs | Bedrock Runtime versus Bedrock control plane |
| Managed FM versus custom FM | Bedrock versus SageMaker AI |
| RAG | Knowledge Bases, OpenSearch, Aurora pgvector, Kendra/Q Business |
| Enterprise assistant with source ACLs | Q Business versus custom Bedrock application |
| Workflow | Step Functions versus Bedrock Flows versus FM-controlled agent |
| Async buffering | SQS versus EventBridge fanout versus SNS notification |
| Serverless compute | Lambda versus ECS/Fargate/AgentCore for heavier runtimes |
| Data quality | Glue Data Quality versus interactive Data Wrangler |
| PII | Comprehend for text in request path; Macie for data stored in S3 |
| OCR/document extraction | Bedrock Data Automation versus Textract |
| Audio | Transcribe |
| API | HTTP/REST/WebSocket API Gateway and streaming limitations |
| Audit/operations | CloudTrail versus CloudWatch Logs/metrics versus X-Ray |
| Data authorization | IAM and Lake Formation |
| Private access | VPC endpoints/PrivateLink and S3 gateway endpoints |
| Configuration | AppConfig versus code deployment |
| Deployment | CodePipeline/CodeBuild/CodeDeploy and CloudFormation/CDK |
| Cost | CountTokens, caching, routing, batch, provisioned throughput |

Also recognize the purpose—not every configuration detail—of the remaining services in the official in-scope list: Athena, EMR, Kinesis, MSK, AppFlow, App Runner, EC2, EKS, ECR, Connect, DocumentDB, Neptune, ElastiCache, DynamoDB Streams, Amplify, CodeArtifact, Lex, Rekognition, SageMaker Clarify/Ground Truth/JumpStart/Model Monitor/Neo/Unified Studio, QuickSight, Managed Grafana, Systems Manager, Service Catalog, DataSync, Transfer Family, CloudFront, ELB, Global Accelerator, Route 53, Cognito, IAM Access Analyzer, Encryption SDK, KMS, WAF, EBS, EFS, and S3 storage classes/replication.

Do not spend equal study time on this tail. Learn each service’s primary purpose and when it is clearly better or worse than the common alternatives.

---

# What the two local mock exams emphasize

Across 150 questions, overlapping keyword/theme counts are:

| Theme | Questions containing the theme |
|---|---:|
| Bedrock API integration/streaming | 41 |
| Knowledge Bases/RAG | 40 |
| CloudWatch/observability | 30 |
| Guardrails/safety | 26 |
| SageMaker/deployment | 26 |
| Vector search/embeddings | 25 |
| Evaluation | 17 |
| Data processing | 16 |
| Step Functions | 14 |
| Agentic AI/MCP | 14 |
| Cost/tokens/caching | 13 |
| Prompt Management/Flows | 12 |
| Security/private access | 10 |
| CI/CD/IaC | 8 |

Counts overlap because professional questions combine several requirements.

Recurring question-solving rules:

- “Least operational overhead” usually favors managed services, not custom EC2/EKS.
- “Immediate acknowledgement” plus bursty work usually means queue/event decoupling.
- “Exact identifier plus natural language” means hybrid search.
- “Long human approval” means Step Functions Standard callback, not polling.
- “Same model, predictable peak” points to Provisioned Throughput.
- “Same model, regional capacity issue” points to cross-Region inference.
- “Perceived latency” points to streaming and TTFT.
- “Stable prefix repeated” points to prompt caching.
- “Sensitive S3 data” points to Macie; “redact before invocation” points to Comprehend/Guardrails.
- “Audit who changed resources” points to CloudTrail; “debug runtime request” points to structured CloudWatch logs/X-Ray.
- “Quality before release” means a representative evaluation dataset and automated gate, not manual prompt tweaking.

Treat mock answers as study material, not authority. When a mock conflicts with current AWS documentation or supported behavior, follow the official documentation.

---

# Practical mastery checklist

Before booking the exam, be able to build or explain these without notes:

- [ ] A secure Bedrock Converse/ConverseStream API with request validation, guardrails, structured logs, and retries.
- [ ] A Knowledge Base over S3 with chunking, embeddings, metadata filtering, hybrid search, reranking, citations, and refresh monitoring.
- [ ] A retrieve-only test that separates retrieval quality from generation quality.
- [ ] An agent/tool loop with schema validation, maximum cycles, circuit breaker, least privilege, and human callback.
- [ ] A Prompt Management/Flows release with versioning, evaluation, approval gate, canary, and rollback.
- [ ] A privacy pipeline using Macie, Comprehend, Guardrails, sanitized logs, private endpoints, and retention controls.
- [ ] A model-selection comparison covering quality, API support, Region, lifecycle, latency, throughput, and cost.
- [ ] An on-demand versus provisioned versus batch deployment decision.
- [ ] A SageMaker LMI/custom-model deployment and its startup/storage failure modes.
- [ ] A CloudWatch/X-Ray dashboard and trace plan for tokens, latency, TTFT, errors, throttles, retrieval, and tool calls.
- [ ] A model/RAG/agent evaluation plan with automated and human metrics.
- [ ] A CI/CD quality gate with security tests, regression evaluation, canary rollout, and automatic rollback.

## 85% readiness gate

Do not use a single repeated mock score as proof of readiness. Memorization inflates it.

Use this gate:

- At least **85% on two unseen, timed, full-length exams**.
- At least **80% in every official domain**.
- At least **90% on Domains 1–3 combined**.
- For every missed question, explain why the correct option meets every constraint and why each distractor fails.
- Repeat missed concepts with changed scenarios, not the same wording.
- Complete at least one hands-on exercise for Bedrock APIs, RAG, guardrails, evaluation, Step Functions agent orchestration, and observability.

## Primary sources

- [Official AIP-C01 exam guide](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01.html)
- [Domain 1](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain1.html)
- [Domain 2](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain2.html)
- [Domain 3](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain3.html)
- [Domain 4](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain4.html)
- [Domain 5](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain5.html)
- [Official in-scope AWS services](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/aip-01-in-scope-services.html)

Local evidence:

- `udemy-practice-exam-1-questions-answers.json`
- `udemy-practice-exam-2-questions-answers.json`
- `udemy-practice-exams-explanations.json`
