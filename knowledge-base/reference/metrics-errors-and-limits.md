# Metrics, Errors, and Limits

Status: Conceptually verified; exact names, quotas, and limits are configuration-dependent  
Last verified: 2026-07-23

## Do not memorize volatile numbers blindly

Model context windows, embedding dimensions, batch limits, quotas, Region support, cache requirements, and prices change. For the exam, understand the decision. For production, query current model/service documentation and Service Quotas.

## Bedrock runtime metrics

Know the purpose of:

- Invocation count.
- Invocation latency.
- Input token count.
- Output token count.
- Throttles.
- Client errors.
- Server errors.
- Time to first token for streaming where published.

Use dimensions to separate model, Region, API, product, tenant, prompt version, and experiment through native dimensions plus safe application metadata.

## Derived metrics

Calculate:

- Tokens per successful task.
- Cost per successful task.
- TTFT p50/p95/p99.
- End-to-end latency minus model latency.
- Retry amplification.
- Cache-hit ratio.
- Quality per 1,000 tokens.
- Human edit/review time.
- Grounded answer rate.
- Invalid-JSON rate.
- Tool calls per successful agent task.

## RAG metrics

- Ingestion job success/failure.
- Document and chunk added/deleted/failed.
- Source-to-index freshness.
- Retrieve latency/error/throttle.
- Empty result rate.
- Context relevance and coverage on a benchmark.
- Faithfulness/citation coverage end to end.
- OpenSearch indexing/search latency, pressure, storage, and shard behavior.

## Agent/tool metrics

- Completion and escalation rate.
- Steps per task.
- Repeated identical tool call.
- Invalid argument rate.
- Tool timeout/error/retry.
- Circuit-breaker activation.
- Memory read/write/flush outcome.
- Multi-agent handoff count and failure.

## Evaluation metrics

| Dimension | Example evidence |
|---|---|
| Relevance | Judge/human score |
| Factuality | Reference or evidence-backed score |
| Faithfulness | Claim supported by retrieved context |
| Consistency | Repeated-run semantic/field variance |
| Fluency/coherence | Judge/human rubric |
| Safety | Intervention or policy-failure rate |
| Fairness | Cohort comparison |
| Agent performance | Completion/tool-use score |

## Common error classes

### Validation/client error

Examples:

- Wrong request body.
- Missing message/content field.
- Unsupported parameter.
- Wrong model ID or endpoint.
- Token/context limit exceeded.

Action: fix the request. Do not retry unchanged.

### Authorization/access error

Examples:

- Missing `bedrock:InvokeModel`.
- Model resource not allowed.
- Guardrail condition not satisfied.
- VPC endpoint/resource policy denies access.
- Lake Formation permission missing.

Action: inspect identity policy, resource/endpoint policy, SCP/permission boundary, and governed data permissions. Do not “fix” by granting broad wildcard access.

### Throttling

Evidence:

- Token or request quota exhausted.
- Concurrency/capacity spike.

Action:

- Backoff with jitter.
- Reduce context/output.
- Queue/batch.
- Request quota/capacity changes.
- Use Provisioned Throughput or cross-Region profile if appropriate.

### Transient service/network failure

Action:

- Bounded retry with exponential backoff/jitter.
- Idempotency for side effects.
- Circuit breaker/fallback for repeated failure.

### Policy/safety intervention

Action:

- Return controlled blocked/clarification response.
- Record sanitized policy outcome.
- Do not retry the same content to bypass the policy.

## Context and output limits

Budget:

`system + history + retrieved context + user/tool content + reserved output`

Use the model-specific context limit and `CountTokens`. An output limit that is too high can reduce available throughput; one that is too low truncates structured output.

## Embedding limits

Validate:

- Input text/token limit.
- Supported output dimension/representation.
- Vector field dimension matches.
- Distance metric/index compatibility.
- Batch size and payload limit.

Changing dimension after ingestion usually requires rebuilding/reindexing.

## Lambda considerations

Know qualitatively:

- Execution timeout and payload/streaming constraints.
- Memory affects CPU allocation.
- Package/container and native dependency implications.
- Concurrency and downstream quotas.
- Reuse clients outside the handler.

Do not memorize a number unless current AWS documentation and the question require it.

## SageMaker LLM startup

Large-model endpoint failures can involve:

- Download duration.
- Model artifact compression.
- Storage volume size.
- Container startup health timeout.
- Model data download timeout.
- GPU memory.
- Tensor parallelism/serving container settings.

Increasing a timeout cannot solve insufficient disk or GPU memory.

## Alarms

High-value alarms:

- Error or throttle percentage.
- p95/p99 latency and TTFT.
- Token/cost anomaly.
- Queue age/backlog.
- Ingestion failure/staleness.
- Safety-intervention spike.
- Invalid output or quality-gate failure.
- Repeated tool-call/loop rate.

Alarm thresholds need baselines and minimum sample counts to avoid noise.

## Exam traps

- Low request count does not imply low token use.
- HTTP 200 does not imply a good answer.
- High retrieval score does not imply factual correctness.
- Retry does not solve deterministic validation failure.
- Increasing every timeout does not fix resource insufficiency.
- A generic cache key can leak tenant data.
- A model’s maximum context is not the recommended prompt size.

## Official sources

- [Bedrock runtime monitoring](https://docs.aws.amazon.com/bedrock/latest/userguide/monitoring-runtime-metrics.html)
- [Amazon Bedrock quotas](https://docs.aws.amazon.com/bedrock/latest/userguide/quotas.html)
- [Service Quotas](https://docs.aws.amazon.com/servicequotas/latest/userguide/intro.html)
