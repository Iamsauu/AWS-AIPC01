# Cost, Latency, Throughput, Routing, and Caching

Status: Verified principles; pricing, quotas, model support, and Region support are volatile  
Official tasks: 1.2.3, 2.2.1–2.2.3, 4.1, 4.2  
Last verified: 2026-07-23

## The core resource model

GenAI capacity is not described well by requests alone. Two requests can differ by orders of magnitude in input and output tokens.

Plan with:

- Requests per minute.
- Input tokens per minute.
- Maximum and typical output tokens.
- Concurrent in-flight invocations.
- Time to first token (TTFT).
- Total invocation latency.
- Model-specific quotas/capacity.
- Retrieval and tool latency.

Use `CountTokens` with the selected model and planned request shape before invocation when admission control or context validation matters. Combine the input estimate with an appropriate output budget.

## Token-efficiency hierarchy

Before changing infrastructure:

1. Remove duplicated instructions.
2. Retrieve fewer, more relevant chunks.
3. Filter obsolete or unauthorized document versions.
4. Summarize old conversation turns and retain recent turns.
5. Prune context by token budget.
6. Set a task-specific `maxTokens`.
7. Use prompt caching for a repeated stable prefix.
8. Route simple tasks to a smaller model.
9. Cache safe deterministic results where appropriate.

Do not reduce tokens blindly. Measure quality after each change.

## Context-window management

A safe assembly process:

1. Reserve output tokens.
2. Add mandatory system/safety instructions.
3. Add current user request.
4. Add the most relevant context.
5. Add recent conversation turns.
6. Add summarized older history if budget remains.
7. Call `CountTokens`.
8. Prune by explicit priority before invocation.

Silent truncation can remove the instruction, evidence, or closing schema that matters most. Diagnose which section was lost.

## Caching patterns

| Cache | Best use | Major safety condition |
|---|---|---|
| Bedrock prompt cache | Stable long prefix reused across requests | Prefix and model-specific eligibility must match |
| Application exact cache | Identical deterministic request/configuration | Key includes every response-affecting input |
| Semantic cache | Equivalent low-risk questions | Similarity threshold and tenant/data boundaries |
| CloudFront/edge cache | Public/shareable deterministic response | Never leak personalized or authorized content |
| Retrieval cache | Repeated stable query over unchanged corpus | Invalidate on source/index updates |

### Prompt caching

Place a cache checkpoint after the stable reusable prefix. Keep dynamic user and retrieved content after it. Confirm:

- Model supports prompt caching.
- Prefix meets current minimum token requirements.
- Prefix bytes/content remain stable enough for cache hits.
- Cached content has no cross-tenant data leakage.
- Cache-read/write token metrics and pricing produce a real benefit.

Prompt caching reduces repeated prefix processing; it is not a cache of the completed response.

### Exact and edge caching

Build a deterministic key from:

- Normalized request.
- Model and version/profile.
- Prompt version.
- Inference parameters.
- Retrieval/index version where relevant.
- Tenant/authorization context.
- Guardrail version.

If any omitted field changes the response or its authorization, the cache is unsafe or incorrect.

## Model selection and routing

### Model cascading

Use a smaller/lower-cost model for routine tasks, escalating only when:

- Confidence/evaluation criterion is insufficient.
- Query complexity classifier selects a harder path.
- Tool/reasoning requirements need a more capable model.
- Validation fails.

### Intelligent Prompt Routing

Use managed routing where supported when the allowed model set and requirements fit. Monitor quality and tokens; lower custom logic does not remove the need for evaluation.

### Explicit Step Functions routing

Use when routing must be:

- Auditable.
- Based on deterministic business attributes.
- Multi-step.
- Able to retry/fallback.
- Easy to change and test as workflow logic.

### AppConfig routing

Keep model IDs, provider endpoints, feature flags, weights, and failover state in validated runtime configuration when changes must not require code deployment. Use staged deployment and CloudWatch-triggered rollback.

## Inference capacity choices

| Pattern | Best fit | Main tradeoff |
|---|---|---|
| Bedrock on-demand | Bursty/unpredictable interactive demand | Quotas and variable shared capacity |
| Bedrock Provisioned Throughput | Predictable sustained synchronous demand | Capacity commitment/cost |
| Bedrock batch inference | Large asynchronous offline prompts in S3 | Not interactive |
| Cross-Region inference | Capacity/resilience across allowed Regions | Residency and profile/model support |
| SageMaker real-time endpoint | Custom/open-weight model, serving control | Endpoint/GPU operations |
| SageMaker Batch Transform | Offline custom-model data processing | Job startup and batch workflow |

Provisioned Throughput is not the automatic answer to every throttle. First check request shape, output limits, token rate, retry behavior, and whether work can be batched.

## Throughput improvements

- Set realistic output-token ceilings.
- Batch asynchronous workloads.
- Increase safe concurrency within quotas.
- Use Provisioned Throughput for predictable sustained demand.
- Use cross-Region inference when capacity and residency allow.
- Queue bursts to smooth downstream load.
- Apply per-tenant token buckets.
- Reuse connections and SDK clients.
- Parallelize independent calls.
- Use smaller models where evaluated quality is sufficient.

Retries do not create capacity. They can amplify an overload if backoff and admission control are missing.

## Latency decomposition

Measure:

`client → gateway → compute cold/startup → preprocessing → retrieval → model queue/TTFT → token generation → post-processing → client`

Optimization depends on the slow component.

| Evidence | Likely action |
|---|---|
| High TTFT, repeated static prefix | Prompt caching, reduce prefix/context |
| Full response slow but TTFT acceptable | Reduce output, faster model, parallelize independent work |
| User waits for first byte | ConverseStream/SSE/WebSocket |
| Retrieval dominates | Filters, fewer chunks, index/shard/query tuning |
| Lambda downstream connection overhead | Initialize SDK clients outside handler, keep-alive |
| Predictable model throttling | Provisioned Throughput |
| Regional capacity disruption | Cross-Region inference |
| Independent specialized outputs | Parallel state/calls, then merge |

Streaming reduces perceived latency. It does not necessarily reduce compute cost or total completion time.

## Latency-optimized inference

Use only for supported models/Regions and validate:

- TTFT improvement.
- Total latency.
- Output quality.
- Cost.
- Capacity behavior.

Do not claim support from memory; check model documentation.

## Vector retrieval performance

Tune:

- Metadata filters before broad vector comparison.
- Number of retrieved candidates.
- Hybrid keyword/vector search only where it improves the task.
- Reranking candidate count.
- Index/shard count and size.
- Domain segmentation.
- Embedding dimension/representation after relevance evaluation.
- Duplicate/obsolete chunk removal.
- Client connection reuse.

Fewer shards can reduce cross-shard coordination, but there is no universal shard count. Size from corpus, load, availability, and current OpenSearch guidance.

## Cost-quality measurement

For each candidate, calculate or compare:

- Input and output tokens per successful task.
- Model price.
- Retrieval and orchestration cost.
- Latency and timeout rate.
- Quality score.
- Human review/edit time.
- Business success.

A smaller model is not cheaper if lower quality causes repeated calls, escalation, or human rework.

## Failure modes

| Failure | Cause | Correction |
|---|---|---|
| Throttles despite few requests | Requests have large token budgets | Track tokens; lower context/output |
| High cache miss rate | Prefix changes or key includes volatile fields | Stabilize prefix/key design |
| Cached answer leaks data | Tenant/auth context omitted | Partition cache or disable it |
| Streaming still feels slow | Preprocessing/retrieval delays before model | Trace the pre-model path |
| Provisioned capacity underused | Poor sizing or burst-only traffic | Re-evaluate on-demand/batch mix |
| Retry storm | No backoff/admission/circuit breaker | Add jitter, queue, token bucket, breaker |
| Lower-cost model hurts outcome | Routing based only on price | Evaluate cost-to-quality |
| Context pruning causes hallucination | Evidence removed | Prioritize grounding evidence and retest |

## Local mock references

- PE1: Q10, Q16, Q18, Q21, Q26, Q41, Q44, Q45, Q49, Q52, Q57, Q59, Q60, Q63, Q68, Q70, Q71.
- PE2: Q6, Q9, Q12, Q14, Q15, Q28, Q31, Q38, Q53, Q54, Q55, Q62, Q66, Q69.

## Recall questions

1. Why can requests per minute be a poor capacity measure?
2. When does prompt caching help, and what does it not cache?
3. What must be included in an exact-response cache key?
4. When is Provisioned Throughput better than queueing and batch?
5. Why does streaming improve perceived but not necessarily total latency?
6. How do you distinguish model latency from retrieval latency?
7. When should routing use AppConfig, Step Functions, or a managed prompt router?
8. Why can a smaller model increase total cost?

## Official sources

- [Official Domain 4](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain4.html)
- [Prompt caching](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html)
- [Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html)
- [Provisioned Throughput](https://docs.aws.amazon.com/bedrock/latest/userguide/prov-throughput.html)
- [Batch inference](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html)

