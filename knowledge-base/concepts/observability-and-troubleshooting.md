# Observability and Troubleshooting

Status: Verified core architecture; metric names and service support must be checked for the deployed configuration  
Official tasks: 2.4.3, 2.5.6, 3.3.4, 4.3, 5.2  
Last verified: 2026-07-23

## Observability model

A production GenAI request can cross:

`client → API → compute → safety → retrieval → model → tools → post-validation → response`

Three evidence types are needed:

- Metrics: what changed and when?
- Logs: what happened in this request?
- Traces: where did time/failure occur across components?

Evaluation adds a fourth:

- Quality evidence: was the output useful, grounded, safe, and correct?

## Correlation and version identity

Assign one correlation ID at the boundary and carry it through logs, tool calls, traces, and feedback.

Record safe metadata:

- Correlation and trace ID.
- Product, tenant, environment, and experiment identifiers.
- Model ID/inference profile.
- Prompt and guardrail version.
- KB/data source/index version where practical.
- Tool name and attempt count.
- Token counts, latency, status, validation result.
- Retrieval result count and sanitized quality scores.

Use Bedrock `requestMetadata` where supported for attribution instead of hiding metadata in prompt text.

Do not log raw PHI/PII, secrets, full system prompts, or sensitive retrieved context. Log hashes, categories, counts, and redacted samples only when permitted.

## Metrics by layer

### Bedrock runtime

Monitor applicable metrics such as:

- Invocation volume.
- Invocation latency.
- Input/output token counts.
- Throttles.
- Client/server errors.
- Time to first token for streaming.

Metric availability and exact names depend on API/model. Verify current documentation.

### Application/API

- Request count, p50/p95/p99 latency.
- Validation and schema failure.
- Timeout/retry count.
- Queue depth and age.
- Cold starts where material.
- Status by model/prompt/tenant.

### RAG

- Retrieve latency and errors.
- Result count and empty-retrieval rate.
- Context relevance/coverage on a golden set.
- Ingestion job status.
- Documents/chunks added, deleted, failed.
- Index/search throttling, latency, memory, shard pressure.
- Source staleness.

### Agents/tools

- Task completion.
- Step/loop count.
- Repeated tool calls.
- Tool parameter-validation failure.
- Tool latency, timeout, retry, and error.
- Human escalation.
- Circuit-breaker state.
- Multi-agent handoff/coordination.

### Quality and business

- Groundedness/unsupported-claim rate.
- Format-valid response rate.
- User rating and human edit rate.
- Safety intervention rate.
- Fairness/cohort metrics.
- Cost per successful outcome.

## AWS evidence sources

| Source | Best purpose | Not sufficient for |
|---|---|---|
| CloudWatch metrics | Trends, thresholds, alarms | Per-request detail |
| Structured CloudWatch Logs | Request metadata and searchable events | Cross-service timing alone |
| Logs Insights | Aggregate/search JSON fields | Distributed trace |
| X-Ray | Service path and latency | Semantic output quality |
| CloudTrail | AWS API/control-plane audit | Application conversation debugging |
| Bedrock invocation logging | Supported runtime request/response evidence | All custom application context |
| Agent/Flow trace | Orchestration, KB/tool choices, failures | Provider-internal chain-of-thought |
| CloudWatch Synthetics | Deployed journey availability and latency | Broad semantic quality |
| Evaluation jobs | Quality/regression evidence | Live production health by themselves |

## Structured logging

Prefer one JSON object per significant event:

```json
{
  "correlationId": "generated-id",
  "event": "bedrock_invocation_completed",
  "modelId": "approved-model",
  "promptVersion": "12",
  "guardrailVersion": "4",
  "inputTokens": 900,
  "outputTokens": 180,
  "latencyMs": 1420,
  "validation": "passed",
  "safetyAction": "none"
}
```

This is illustrative. Do not emit sensitive payloads. Use a consistent schema so Logs Insights can compare releases.

## Model invocation logging

Configure in each account/Region where required and supported. Decide:

- CloudWatch Logs or S3 destination.
- Data classification and encryption.
- Access control.
- Retention.
- Whether prompts/responses may be stored.
- Sanitization or disabling for sensitive workloads.

Invocation logging does not replace application logs, because the application knows the prompt version, user-safe business context, retrieval steps, and post-validation outcome.

## Tracing

Use X-Ray to locate latency across API Gateway, Lambda, SDK calls, and supported downstream components. Instrument:

- Boundary request.
- Preprocessing.
- Retrieve/vector search.
- Model invocation.
- Tool invocation.
- Post-processing.

Agent trace and Bedrock Flow trace explain orchestration choices and node/tool execution. Store them only with appropriate access and retention. Trace is not an entitlement to expose hidden provider reasoning.

## Troubleshooting method

1. Reproduce with a fixed request and configuration versions.
2. Locate the failing layer.
3. Inspect structured error and request metadata.
4. Separate retrieval from generation.
5. Compare candidate to last-known-good configuration.
6. Change one variable.
7. Re-run a focused test and the regression suite.
8. Promote through a gate/canary, not directly.

## Symptom-to-evidence matrix

| Symptom | Evidence to inspect | Likely causes | Remediation |
|---|---|---|---|
| Bedrock validation exception | API, model ID, body, roles, content type | Wrong Converse/native schema | Model-aware serializer |
| Settings ignored | Model docs and response metadata | Unsupported/wrong field | Correct native or additional fields |
| Context-window error | CountTokens and assembled sections | History/retrieval/output budget too large | Summarize/prune/filter |
| Missing closing JSON | Stop reason and output limit | `maxTokens` too low | Raise task-specific limit |
| Repeated inconsistent fields | Settings and repeated benchmark | Sampling too broad | Evaluate lower randomness |
| Unsupported answer | Retrieve results and citations | Missing/irrelevant context | Fix retrieval and suppress unsupported text |
| Wrong document version | Metadata, source object, ingestion status | Stale sync or no version filter | Refresh and filter |
| Exact code missed | Query and search type | Vector-only search | Hybrid keyword/vector |
| Retrieval slow | AOSS/OpenSearch metrics, shards, candidates | Fanout, memory, large candidate set | Tune indexes/filter/results |
| Tool loop | Agent trace, tool errors, cycle count | Invalid args or no stop/breaker | Validate, structured error, max cycles |
| Model throttles | Token rate, output limits, concurrency | Token capacity exhausted | Reduce, queue/batch/provision |
| High TTFT | Context size, cache, model metric | Static prefix or large context | Prompt cache/trim/optimized mode |
| End-to-end slow, model normal | X-Ray segments | Retrieval/tool/connection overhead | Optimize slow segment |
| Lambda downstream overhead | Repeated connection setup | Client built inside handler | Reuse client/keep-alive |
| SageMaker startup health failure | Container/model logs | Large download/load, small volume, timeout | LMI/source/volume/timeouts |
| Release quality regression | Prompt/model version and benchmark | Un-gated change | Roll back and evaluate |
| PII in logs | Log samples/config | Raw payload logging | Redact/disable/restrict/expire |

## Error classes and retry

Retry only transient failures:

- Throttling.
- Temporary service unavailability.
- Network timeout.

Use exponential backoff with jitter and a bounded attempt count. Do not retry:

- Invalid schema.
- Unauthorized access.
- Unsupported model/API.
- Deterministic policy rejection.

For side effects, require idempotency before retry. For repeatedly unhealthy tools, open a circuit breaker instead of spending more tokens on the same failing loop.

## Alarms and automated action

Alarm on:

- Error/throttle rate.
- p95/p99 latency and TTFT.
- Queue backlog/age.
- Token or cost anomaly.
- Safety intervention spike.
- Retrieval failure/staleness.
- Tool loop/error rate.
- Quality metric below release threshold.

Actions can notify, stop a pipeline, roll back AppConfig/CodeDeploy, open a circuit breaker, or route to a safe fallback. Avoid automatic actions based on a noisy semantic metric without calibration.

## Golden datasets and output diffs

Use a fixed golden set to detect:

- Hallucination/grounding regression.
- Semantic drift.
- Style/format breakage.
- Tool-use change.
- Cohort fairness change.

Exact string diffs are useful for structured output but noisy for valid prose. Combine structural validation with semantic scoring and human review where needed.

## Local mock references

- PE1: Q1, Q4, Q6, Q10–Q14, Q21–Q23, Q25–Q26, Q29–Q31, Q36, Q42–Q45, Q50–Q55, Q59–Q64, Q66–Q70.
- PE2: Q4, Q8, Q11, Q16, Q19–Q21, Q28–Q33, Q39–Q42, Q45–Q47, Q49, Q51–Q55, Q58–Q60, Q62–Q64, Q67, Q69.

## Recall questions

1. What question does each of CloudWatch, X-Ray, CloudTrail, and evaluation answer?
2. Why is invocation logging insufficient as the only application log?
3. Which metadata should accompany every model response?
4. How do you isolate retrieval from generation?
5. Which errors are retryable, and why are validation errors not?
6. What evidence distinguishes high model latency from high end-to-end latency?
7. Why is agent trace not chain-of-thought?
8. When should an alarm roll back a deployment?

## Official sources

- [Bedrock runtime CloudWatch metrics](https://docs.aws.amazon.com/bedrock/latest/userguide/monitoring-runtime-metrics.html)
- [Model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html)
- [CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)
- [AWS X-Ray](https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html)
- [Official Domains 4 and 5](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain4.html)
