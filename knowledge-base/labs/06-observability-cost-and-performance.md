# Lab 6 — Observability, Cost, and Performance

Status: Guided lab; verify current metric names and API support  
Estimated cost: Low to variable depending on model calls, logs, traces, and canary frequency

## Objective

Instrument a small GenAI API to:

- Correlate one request across services.
- Distinguish model, retrieval, and application latency.
- Track tokens, errors, throttles, TTFT, and quality signals.
- Detect regression with alarms and a synthetic journey.
- Test token, caching, streaming, and connection optimizations.

## Architecture

```text
Test client / Synthetics
          |
      API Gateway
          |
        Lambda
      /    |    \
safety  Retrieve  Bedrock Runtime
          |
   structured CloudWatch Logs
   custom/service metrics
   X-Ray trace
```

## Prerequisites

- Non-production API Gateway/Lambda/Bedrock application.
- Optional small Knowledge Base/OpenSearch test index.
- CloudWatch and X-Ray permissions.
- Test data with no PII/PHI.
- Budget alarm or explicit cost limit.

## Part A — Correlation and structured logs

At the API boundary:

- Generate or accept a correlation ID.
- Return it to the client.
- Pass it through Lambda, retrieval, model, and tool operations.

Log sanitized JSON events with:

- Correlation ID.
- Prompt/model/guardrail/KB versions.
- Safe tenant/product/experiment identifiers.
- Input/output token counts.
- Retrieval result count.
- Validation/safety outcome.
- Component and total latency.
- Error class and retry count.

Do not log raw prompts, system instructions, retrieved sensitive text, or model output unless explicitly approved.

Create Logs Insights queries for:

- Error count by prompt/model version.
- p95 application latency by model.
- Invalid JSON by release.
- Tool retries by request.
- Token totals by safe tenant identifier.

## Part B — Metrics and alarms

Create a dashboard for applicable:

- Bedrock invocations, latency, tokens, errors, throttles, TTFT.
- API/Lambda request, error, duration.
- Queue depth/age if asynchronous.
- Retrieval latency/failure/empty results.
- Safety intervention count.
- Format-validation failure.
- User/benchmark quality signal.

Create alarms:

- Error/throttle rate over a minimum sample.
- p95 latency or TTFT.
- Token/cost anomaly.
- Invalid output spike.
- Ingestion/retrieval failure.

Configure notifications only to a test destination.

## Part C — Distributed tracing

Enable X-Ray for API Gateway and Lambda and instrument relevant SDK/downstream segments where supported.

Run requests and identify:

- Gateway time.
- Lambda initialization/handler time.
- Safety preprocessing.
- Retrieval.
- Model invocation/TTFT.
- Post-validation.

The slowest component, not the loudest service name, determines the optimization.

## Part D — Invocation logging decision

Review the workload classification.

If prompt/response logging is permitted:

- Configure Bedrock model invocation logging in the test Region.
- Encrypt and restrict the destination.
- Apply retention.
- Add safe request metadata.

If not permitted:

- Keep it disabled.
- Retain only sanitized application metadata and aggregate metrics.

Document the reason.

## Part E — Performance experiments

Use one fixed benchmark request set.

### Experiment 1: context budget

Compare:

- Full history and many retrieved chunks.
- Running summary + recent turns + filtered chunks.

Measure tokens, quality, TTFT, and total latency.

### Experiment 2: task-specific output

Compare a realistic `maxTokens` with an unnecessarily large ceiling. Observe throughput/cost-related behavior while confirming outputs are not truncated.

### Experiment 3: streaming

Compare Converse with ConverseStream or the supported equivalent.

Measure:

- TTFT.
- Total latency.
- User-visible incremental behavior.

### Experiment 4: stable prefix

If the model supports prompt caching, add a cache checkpoint after the stable system prefix. Keep dynamic content after it. Compare cache token metrics, TTFT, and input cost.

### Experiment 5: connection reuse

Compare clients initialized:

- Inside the Lambda handler.
- Outside the handler with keep-alive/connection reuse.

Inspect warm-invocation latency.

### Experiment 6: retrieval

Compare:

- Large unfiltered result set.
- Metadata-filtered smaller set.
- Hybrid search for exact IDs.
- Reranking on a bounded candidate set.

Measure relevance and latency together.

## Part F — Synthetic journey

Create a CloudWatch Synthetics canary or equivalent test that:

- Calls the deployed API.
- Checks status and latency.
- Validates deterministic response fields/safe behavior.
- Publishes failure metrics.

Do not use a canary alone to claim semantic quality. Pair it with scheduled evaluation on a benchmark.

## Failure injection

Test:

- Invalid model body.
- Context budget exceeded.
- Simulated transient timeout.
- Sustained throttle or local admission rejection.
- Empty retrieval.
- Wrong document version.
- Invalid tool parameters.
- Output schema failure.

For each, record:

- Log event.
- Trace segment.
- Metric/alarm.
- Retry or non-retry behavior.
- User-safe response.

## Validation evidence

- Dashboard screenshot/export.
- Example sanitized log query result.
- Trace showing component latency.
- Alarm test.
- Before/after token, TTFT, latency, and quality table.
- Documented invocation-logging decision.

## Cost and privacy cautions

- Set log retention; verbose debug logs can cost and leak data.
- Limit canary frequency and evaluation size.
- Do not emit raw sensitive content as metric dimensions or request metadata.
- Delete unused dashboards/canaries/log groups after the lab if not retained.

## Cleanup

- Disable/delete the test canary.
- Remove temporary alarms/dashboard if not retained.
- Disable test invocation logging or apply retention.
- Remove test-only tracing and permissions where appropriate.
- Delete temporary data and expire logs under policy.

## Exam lessons

- CloudWatch, X-Ray, CloudTrail, traces, and evaluations answer different questions.
- TTFT and total latency are separate.
- Token metrics explain both cost and capacity.
- Optimize the measured component.
- Invocation logging must respect privacy.
- Synthetics validates the deployed route, not nuanced semantic quality.

## Sources

- [Bedrock runtime CloudWatch metrics](https://docs.aws.amazon.com/bedrock/latest/userguide/monitoring-runtime-metrics.html)
- [Bedrock model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html)
- [AWS X-Ray](https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html)
