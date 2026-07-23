# Domain 4: Operational Efficiency and Optimization for GenAI Applications

Status: Verified against the current blueprint  
Exam weight: 12%  
Official tasks: 4.1, 4.2, 4.3  
Last verified: 2026-07-23

## Domain objective

Optimize GenAI cost, token use, latency, throughput, retrieval, resource allocation, and observability without silently reducing quality or safety.

Canonical deep dives:

- [Cost, latency, throughput, and caching](../concepts/cost-latency-throughput-and-caching.md)
- [Observability and troubleshooting](../concepts/observability-and-troubleshooting.md)
- [Bedrock model selection and runtime APIs](../concepts/bedrock-model-selection-and-runtime-apis.md)
- [RAG and vector search](../concepts/rag-knowledge-bases-vector-search.md)

This page is self-contained at exam depth even if those concept pages have not yet been opened.

## Complete official-skill map

| Skill | Required exam capability |
|---|---|
| 4.1.1 | Estimate and track tokens; prune context; compress prompts; limit responses without losing task quality |
| 4.1.2 | Choose models by cost-capability and price-to-performance; route simple and complex work appropriately |
| 4.1.3 | Maximize throughput and utilization with batching, capacity planning, monitoring, scaling, and Provisioned Throughput |
| 4.1.4 | Use prompt, semantic/result, deterministic, and edge caching safely |
| 4.2.1 | Balance latency, cost, and UX with pre-computation, latency-optimized inference, parallelism, streaming, and benchmarks |
| 4.2.2 | Improve retrieval speed and relevance with index/query optimization, hybrid search, and scoring/reranking |
| 4.2.3 | Optimize token throughput, concurrency, and batch inference |
| 4.2.4 | Tune model parameters and prove changes with A/B tests |
| 4.2.5 | Allocate FM capacity from token patterns and monitor/scaling signals |
| 4.2.6 | Profile end-to-end prompt, model, vector, network, and service communication latency |
| 4.3.1 | Build holistic operational, interaction, performance, quality, and business observability |
| 4.3.2 | Monitor tokens, latency, errors, prompt quality, hallucination/drift, logs, benchmarks, and cost anomalies |
| 4.3.3 | Integrate dashboards, business impact, compliance, forensic traceability, and user/model behavior |
| 4.3.4 | Measure tool calls, latency, failures, retries, loops, and multi-agent coordination |
| 4.3.5 | Operate vector stores with performance, index, ingestion, and data-quality monitoring |
| 4.3.6 | Diagnose GenAI-specific failures with golden datasets, output diffs, reasoning/tool traces, and specialized telemetry |

## The optimization unit

Do not optimize cost per API call in isolation. Optimize:

```text
cost per successful task
quality-adjusted latency
successful tasks per token budget
business value per dollar
```

A smaller model that causes retries, escalations, or user abandonment can cost more overall. A response cache that returns stale or unauthorized content is not an optimization.

Track at least:

- Input and output tokens.
- Cached read and write tokens where supported.
- Model and guardrail usage.
- Retrieval/reranking cost.
- Tool and orchestration calls.
- Retry and failure cost.
- Evaluation score and task-success rate.
- End-to-end latency and time to first token.

## Task 4.1: Cost and resource efficiency

### 4.1.1 Token efficiency

#### Count before invocation

`CountTokens` returns model-specific input token use for a compatible `InvokeModel` or `Converse` request. The Bedrock-native operation does not incur a charge. Support varies by model and endpoint, so check the selected model.

Use it to:

- Reject or defer requests above a tenant/application budget.
- Select a model or workflow based on context size.
- Avoid context-window errors.
- Estimate cost and token capacity.
- Compare prompt variants.

Tokenization differs by model; a character or word count is not an exact substitute.

#### Reduce input tokens

- Remove duplicate instructions and repeated retrieved chunks.
- Keep stable system instructions concise.
- Summarize older conversation turns and preserve critical state separately.
- Retrieve fewer, more relevant chunks; use metadata filters and reranking.
- Prune tool definitions to those needed for the current step.
- Avoid embedding full schemas/catalogs when a narrow subset is enough.
- Compress verbose source text only if evaluation shows no quality regression.
- Send references to large supported content instead of duplicating bytes where an API supports it.

#### Reduce output tokens

- Set a realistic `maxTokens`.
- Use concise output instructions and structured fields.
- Add stop sequences where appropriate.
- Return identifiers or structured decisions instead of verbose prose when the consumer is a machine.

Do not set a huge `maxTokens` “just in case.” Bedrock token-based throttling can reserve capacity from input plus configured output maximum, so an unnecessarily high value can reduce completed requests per minute even when responses are short.

#### Context-window management

Reserve space for:

```text
system + tools + conversation + retrieved context + current input + expected output
```

If the sum approaches the model limit, choose intentionally:

1. Remove irrelevant retrieved chunks.
2. Summarize/prune old turns.
3. Split the task or use hierarchical processing.
4. Use a model with a suitable context window if quality/cost justifies it.
5. Reject with a clear message instead of silent truncation.

### 4.1.2 Cost-effective model selection

Build a capability and cost matrix:

- Task quality and safety.
- Input/output modality.
- Context window.
- Tool/structured-output/streaming support.
- Region and lifecycle.
- Latency and throughput.
- Input/output token price and capacity option.
- Evaluation score per unit cost.

Patterns:

- Use a small model for classification, extraction, rewriting, and simple questions if it meets the quality gate.
- Escalate complex or low-confidence tasks to a stronger model.
- Use a specialized model for embeddings or reranking instead of a general text model.
- Use Intelligent Prompt Routing when supported models, family, language, and routing behavior fit the requirement.
- Use application or Step Functions routing when rules, providers, fallbacks, or audit paths must be explicit.

Evaluate the whole cascade. An inexpensive classifier plus two generation calls can cost more than one correct generation call.

### 4.1.3 Throughput and resource utilization

| Workload | Preferred capacity pattern | Why |
|---|---|---|
| Bursty interactive traffic | On-demand inference, possibly cross-Region profile | No fixed capacity commitment |
| Predictable sustained synchronous peak | Provisioned Throughput | Stable purchased capacity for the same model |
| Large asynchronous S3 dataset | Bedrock batch inference | Managed offline processing and S3 output |
| Temporary regional capacity pressure | Cross-Region inference profile | Uses capacity across allowed Regions |
| Customized Bedrock model | Eligible on-demand custom-model deployment, or Provisioned Throughput where required and supported | Availability depends on the model and Region; there is no single invocation mode for every custom model |
| Custom/open-weight endpoint | SageMaker real-time or batch with scaling/batching | Full serving control |

Provisioned Throughput is billed for purchased capacity. Size it from input/output token rate, concurrency, peak duration, and utilization—not request count alone. Invoke the provisioned model ARN as `modelId`.

Batch inference accepts JSONL input in S3 and writes results to S3. It is for asynchronous work, not an interactive chat response.

Current compatibility boundaries matter:

- Bedrock batch inference is not supported for provisioned models.
- Cross-Region inference profiles are an on-demand/batch routing mechanism and are not used with Provisioned Throughput.
- Batch records are independent and do not provide an interactive multi-turn tool loop.

### 4.1.4 Caching

#### Prompt caching

Use Bedrock prompt caching when a long prompt prefix repeats:

- Stable system instructions.
- Tool definitions.
- A repeatedly queried document or policy.
- Long shared conversation prefix.

Place cache checkpoints on a sufficiently large, static, contiguous prefix. Changes before the checkpoint cause misses. Separate stable content from per-request variables.

Economics:

- Cache reads are charged differently from normal input tokens.
- Cache writes can cost more than ordinary input, depending on model.
- A low reuse rate can make caching more expensive.
- Cache/token quota and TTL behavior are model-specific.
- Current Bedrock documentation states prompt caching is for supported on-demand inference, not batch inference.

#### Exact/result cache

Cache deterministic, reusable responses by a normalized fingerprint. The key must include every value that can change the answer:

```text
normalized request
+ model and inference settings
+ prompt/guardrail version
+ knowledge/data version
+ locale
+ authorization/tenant scope
```

CloudFront is useful for globally reusable deterministic API responses. Never share a cache entry across users or tenants when the response depends on identity or permissions.

#### Semantic cache

A semantic cache can reuse an answer for a similar query. It increases hit rate but risks:

- Semantically similar yet materially different questions.
- Stale policy/data.
- Cross-tenant leakage.
- Wrong personalization.
- Non-deterministic or high-risk answers.

Use conservative thresholds, authorization-aware keys, TTLs, source-version invalidation, and evaluation.

#### Pre-computation

Precompute predictable, finite questions or summaries. Do not precompute when inputs are private, highly dynamic, or too diverse.

## Task 4.2: Application performance

### 4.2.1 Latency, cost, and user experience

Separate:

- **Time to first token (TTFT):** responsiveness of streaming.
- **Generation time/output tokens per second:** speed after generation starts.
- **Model invocation latency:** request to last token.
- **End-to-end latency:** API, validation, retrieval, model, tools, network, and client rendering.

Response streaming lowers perceived latency; it usually does not reduce total generation work or token cost. Use `ConverseStream` or `InvokeModelWithResponseStream` when supported and carry chunks to the client with a compatible streaming path such as WebSocket, server-sent events, or supported response streaming.

Other controls:

- Run independent retrieval/tool/model work in parallel.
- Reuse SDK/HTTP clients and connection pools across Lambda invocations.
- Keep service calls in suitable Regions and avoid unnecessary hops.
- Use prompt caching for repeated prefixes.
- Reduce input and output tokens.
- Precompute predictable results.
- Benchmark p50, p95, p99, TTFT, and quality together.

Latency-optimized inference is model/Region-specific and current AWS documentation labels the feature preview. Verify support and fallback/price behavior rather than assuming it is universal.

### 4.2.2 Retrieval performance

| Symptom | Optimization |
|---|---|
| Exact error code/model number is missed | Hybrid keyword and vector search |
| Relevant result ranks below generic results | Reranking or custom scoring |
| Wrong business unit/version | Metadata filters and governed version metadata |
| Too many overlapping passages | Deduplicate, improve chunking, filter versions |
| High cross-shard query fan-out | Revisit shard count/size and index design |
| High latency with strong relevance | Tune search parameters, top-k, shard/index capacity |
| Fast retrieval with poor relevance | Improve embeddings, chunking, query processing, filters |

Reranking adds a step and cost but can reduce the number of chunks passed to the generator, improving overall quality, token cost, and latency.

Do not change prompts until retrieval-only evidence shows retrieval is healthy.

### 4.2.3 Throughput

Throughput is driven by tokens and concurrency:

```text
input tokens per minute
reserved/actual output tokens per minute
concurrent in-flight requests
request latency
retry amplification
```

Improve it by:

- Accurate `maxTokens`.
- Context pruning.
- Queuing and backpressure.
- Controlled concurrency.
- Standard SDK retries with exponential backoff and jitter.
- Batch inference for offline work.
- Cross-Region or provisioned capacity when requirements fit.
- Batching at custom SageMaker endpoints when the container supports it.

Retries must be bounded and side effects idempotent. Uncontrolled retries increase throttling.

### 4.2.4 Model parameters

| Parameter | Effect | Exam caution |
|---|---|---|
| `maxTokens` | Caps output and influences capacity planning | Too high wastes quota; too low truncates |
| `temperature` | Controls randomness | Lower can improve repeatability, not correctness |
| `topP` | Limits probability mass considered | Tune systematically, not with temperature blindly |
| `topK` | Limits candidate tokens for supported models | Model-specific, not a universal Converse field |
| Stop sequence | Ends generation at a marker | Can truncate legitimate content |

Use a representative dataset and A/B evaluation to select settings. For structured regulated summaries, lower randomness may help consistency. For ideation, more diversity may be useful.

### 4.2.5 Resource allocation

Capacity-plan with distributions, not averages:

- Prompt and completion token percentiles.
- Concurrency and arrival bursts.
- Time of day and tenant.
- Cache hit rate.
- Retry rate.
- Model-specific quotas.
- Expected quality escalation rate.

For multi-tenant fairness, estimate/request token use before invocation, enforce per-tenant budgets, and queue/defer excess work. Requests per minute alone is misleading when prompt sizes vary greatly.

For SageMaker/ECS/EKS custom serving, monitor GPU/CPU/memory, model-loading time, tokens per second, queue depth, and endpoint/container concurrency, then scale on the constraint that actually saturates.

### 4.2.6 End-to-end workflow optimization

Profile before changing architecture:

```text
client -> API -> validation -> retrieval -> rerank -> model -> tools -> post-check -> client
```

Use X-Ray/OTel subsegments and structured timing fields. Common improvements:

- Collapse redundant serial calls.
- Parallelize independent calls.
- Reuse connections.
- Reduce serialization and payload size.
- Select a direct managed integration where it removes custom hops.
- Make slow non-interactive work asynchronous.
- Cache stable intermediate results.
- Optimize vector index/shards rather than blaming the FM for retrieval latency.

## Task 4.3: Monitoring systems

### 4.3.1 Holistic observability

Observe four layers:

| Layer | Examples |
|---|---|
| Infrastructure/API | availability, errors, throttles, concurrency, queue depth |
| GenAI operation | tokens, TTFT, total latency, cache hits, stop reason, model/prompt version |
| Quality/safety | correctness, faithfulness, harmfulness, guardrail interventions, drift |
| Business | task completion, conversion, deflection, escalation, satisfaction, value/cost |

A system can have perfect API uptime and still fail because answers are wrong.

### 4.3.2 Bedrock monitoring

For `bedrock-runtime`, CloudWatch `AWS/Bedrock` includes high-value metrics such as:

| Metric | Meaning |
|---|---|
| `Invocations` | Successful runtime requests |
| `InputTokenCount` | Processed input tokens, excluding cache reads |
| `OutputTokenCount` | Generated output tokens |
| `InvocationLatency` | Request to last token |
| `TimeToFirstToken` | First-token latency for streaming operations |
| `InvocationClientErrors` | Client-side failures |
| `InvocationServerErrors` | Service-side failures |
| `InvocationThrottles` | Throttled calls |
| `CacheReadInputTokens` | Tokens served from prompt cache |
| `CacheWriteInputTokens` | Tokens written to prompt cache |
| `EstimatedTPMQuotaUsage` | Approximation, not sole capacity-planning truth |

The `bedrock-mantle` endpoint uses a different CloudWatch namespace and currently exposes a different metric surface. Do not assume every Bedrock endpoint publishes identical metrics.

### 4.3.3 Integrated observability

Use:

- CloudWatch dashboards and alarms for operational and custom business metrics.
- Bedrock model invocation logging for deliberately enabled supported runtime payload/metadata evidence.
- CloudWatch Logs Insights for structured request analysis.
- X-Ray/OTel for latency across API, Lambda, retrieval, model, and tools.
- CloudTrail for API actor/action audit.
- Amazon Quick Sight when stakeholders need recurring evaluation/business trends.

Correlate with a request/session ID and version metadata. Never use raw PII as a metric dimension or tag.

### 4.3.4 Tool and multi-agent performance

Measure per tool:

- Calls per task.
- Success/error/timeout rate.
- p50/p95/p99 latency.
- Retry count.
- Input-validation failures.
- Repeated identical calls.
- Tokens before/after call.
- Task-completion contribution.
- Human-approval wait separately from compute latency.

For multi-agent systems, record handoffs, cycles, routing choices, duplicated work, and final task completion. AgentCore Observability can publish sessions, traces, spans, logs, tokens, latency, and errors to CloudWatch when configured/instrumented.

### 4.3.5 Vector-store operations

Monitor:

- Source sync age and failed documents.
- Chunk counts and deletion/index statistics.
- Embedding failures and dimension/schema consistency.
- Index/search latency and error/throttle rate.
- Queue/rejection metrics.
- CPU, memory/JVM pressure, storage, node/cluster health.
- Shard count/size and skew.
- Retrieval relevance/coverage on a golden dataset.

Bedrock Knowledge Base ingestion logs can expose per-document final status and chunk statistics. OpenSearch slow/error/audit logs and CloudWatch metrics diagnose the vector layer.

### 4.3.6 FM-specific troubleshooting

Traditional uptime monitoring misses:

- Hallucination.
- Semantic drift.
- Nondeterminism.
- Prompt leakage.
- Context-window loss.
- Tool loops.
- Wrong-but-valid structured output.
- Retrieval/generation mismatch.

Use:

- Golden datasets.
- Versioned output comparisons and semantic diffs.
- RAG retrieval-only and end-to-end evaluations.
- Agent/tool traces.
- Guardrail and evaluation scores.
- Production feedback and sampled continuous evaluation.

## Architecture decision table

| Constraint | Choose | Do not confuse with |
|---|---|---|
| Long repeated prefix | Prompt caching | Response cache |
| Identical public deterministic answer | Fingerprint plus edge/result cache | Prompt caching |
| Similar questions may share safe answer | Carefully scoped semantic cache | Universal cross-user cache |
| Bursty interactive workload | On-demand/cross-Region | Fixed Provisioned Throughput by default |
| Predictable sustained peak | Provisioned Throughput | Batch inference |
| Huge asynchronous S3 workload | Batch inference | Thousands of synchronous Lambda calls |
| Users need visible output quickly | Streaming; optimize TTFT | Assuming total latency/cost falls |
| Exact code plus natural language | Hybrid search then rerank | Vector-only search |
| Varied prompt size across tenants | Token-based budgeting | Request-count allocation |
| Simple and complex requests | Evaluated routing/cascade | Always use largest model |
| Slow application, FM latency normal | X-Ray/end-to-end profiling | Change model first |

## Failure modes and troubleshooting

| Symptom | Evidence | Probable cause | Remediation |
|---|---|---|---|
| Cost rises, traffic stable | Token and cache metrics by version | Longer prompts/history or cache misses | Prune, summarize, fix checkpoint/key |
| Frequent context errors | `CountTokens`, prompt component sizes | History/retrieval exceeds model window | Budget, prune, split, or suitable model |
| Throttles with short actual responses | Configured `maxTokens`, throttle metric | Excess output reservation | Lower maximum to realistic bound |
| High TTFT, normal generation speed | TTFT vs total latency | Long prefix/retrieval/setup | Cache prefix, reduce context, profile retrieval |
| Good TTFT, slow completion | Output tokens and OTPS | Long output or model generation | Limit output or select evaluated faster model |
| Streaming enabled but UI still waits | Network/client trace | Backend buffers chunks | Use a true end-to-end streaming path |
| Prompt-cache hit rate low | Cache read/write metrics | Prefix changes before checkpoint | Move variables after stable prefix |
| Semantic cache serves wrong answer | Cache audit and source/version | Threshold/key/invalidation too broad | Narrow scope; include versions/authorization |
| Vector search latency spikes | Shard, queue, CPU/JVM, slow logs | Fan-out, pressure, or poor index | Rebalance/tune index and capacity |
| Model change lowers cost but harms users | Evaluation and business outcomes | Cost-only selection | Roll back; use quality-adjusted decision |
| Provisioned capacity underused | Utilization and traffic schedule | Overprovisioning | Resize or move bursty portion on demand |
| Agent cost explodes | Tool/trace loop metrics | Repeated tool calls/retries | Stop conditions, validation, circuit breaker |
| Dashboard is green, answer quality falls | Quality/feedback by version | Only infrastructure metrics exist | Continuous evaluation and quality alarms |

## Common exam traps

- Tokens, not only requests, drive cost and many quotas.
- `CountTokens` is model-specific; a word count is not equivalent.
- Streaming improves perceived latency, not necessarily total latency or cost.
- Provisioned Throughput is not automatically cheapest; it needs sustained/predictable utilization.
- Batch inference is not for interactive responses.
- Cross-Region inference and Provisioned Throughput are different capacity patterns.
- Prompt caching is not a response cache.
- A cache key that omits authorization or data version can leak or stale data.
- Lower temperature does not guarantee correctness.
- Bigger context can reduce quality and raise latency.
- More retrieved chunks can increase noise, cost, and latency.
- CloudWatch service metrics do not measure business success or factual quality automatically.
- Invocation logging is not enabled by default and can store sensitive payloads.
- `EstimatedTPMQuotaUsage` is approximate; it is not the sole capacity signal.
- An OpenSearch problem should not be “fixed” by changing the FM.

## Local mock references

| Topic | Questions |
|---|---|
| Tokens/context/cost | PE1-Q10, Q44, Q52; PE2-Q15, Q28, Q54 |
| Prompt/response caching | PE1-Q57; PE2-Q12 |
| Model routing and cascading | PE1-Q63, Q68, Q71; PE2-Q53 |
| Streaming and latency | PE1-Q16, Q41, Q52, Q70; PE2-Q62, Q66 |
| Batch and capacity | PE1-Q18, Q26, Q60; PE2-Q14 |
| Retrieval optimization | PE1-Q42, Q51, Q61, Q70; PE2-Q16, Q33, Q73 |
| Parameter/cost-quality evaluation | PE1-Q09, Q59; PE2-Q37, Q49, Q55 |
| Tool/vector/model observability | PE1-Q22, Q64, Q66; PE2-Q33, Q39, Q69 |

## Skill mastery check

You are ready for this domain when you can:

- Compute which prompt components consume the budget and decide what to prune.
- Choose on-demand, cross-Region, provisioned, or batch from workload shape.
- Distinguish prompt caching, exact result caching, semantic caching, and pre-computation.
- Explain TTFT, total latency, generation rate, and end-to-end latency.
- Diagnose whether latency is retrieval, model, tool, network, or client.
- Size capacity from token distributions and concurrency.
- Name the Bedrock runtime metrics that prove cost, throttling, and latency behavior.
- Add quality, safety, business, agent/tool, and vector metrics to infrastructure monitoring.

## Recall questions

1. Why can a `maxTokens` value of 4,096 reduce throughput when outputs average 150 tokens?
2. When is prompt caching more expensive than no caching?
3. What must an exact response-cache key contain for a RAG application?
4. Why can a semantic cache become a security vulnerability?
5. Which capacity option fits a predictable 45-minute weekday peak?
6. Which option fits 500,000 nightly asynchronous summaries in S3?
7. What is the difference between TTFT and invocation latency?
8. Why can streaming improve UX without lowering token cost?
9. Which metrics distinguish slow model generation from a slow prefix/retrieval phase?
10. Why is request count a weak multi-tenant capacity unit?
11. How does hybrid search improve exact-identifier retrieval?
12. What evidence should exist before changing chunking or shard count?
13. Which GenAI failures remain invisible on a normal uptime dashboard?
14. How would you prove that a smaller model is truly cheaper per successful task?
15. What tool metrics reveal an agent loop?
16. Why should `EstimatedTPMQuotaUsage` not be the only capacity-planning metric?

## Official sources

- [Official Domain 4 tasks and skills](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain4.html)
- [CountTokens](https://docs.aws.amazon.com/bedrock/latest/userguide/count-tokens.html)
- [How Bedrock tokens and quotas are counted](https://docs.aws.amazon.com/bedrock/latest/userguide/quotas-token-burndown.html)
- [Prompt caching](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html)
- [Intelligent Prompt Routing](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-routing.html)
- [Provisioned Throughput](https://docs.aws.amazon.com/bedrock/latest/userguide/prov-throughput.html)
- [Batch inference](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html)
- [Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html)
- [Latency-optimized inference](https://docs.aws.amazon.com/bedrock/latest/userguide/latency-optimized-inference.html)
- [Bedrock runtime CloudWatch metrics](https://docs.aws.amazon.com/bedrock/latest/userguide/monitoring-runtime-metrics.html)
- [Model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html)
- [Diagnose latency with output tokens per second](https://docs.aws.amazon.com/bedrock/latest/userguide/monitoring-runtime-otps.html)
- [Knowledge Base retrieval](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-how-retrieval.html)
- [Knowledge Base chunking](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html)
- [Bedrock reranking](https://docs.aws.amazon.com/bedrock/latest/userguide/rerank.html)
- [Knowledge Base ingestion logs](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-bases-logging.html)
- [OpenSearch operational best practices](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/bp.html)
- [AgentCore Observability](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability.html)
- [AWS X-Ray](https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html)
