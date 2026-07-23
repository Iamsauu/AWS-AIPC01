# Lab 01 — Bedrock API Contracts, Token Admission, and Streaming

Status: Ready to run with account-specific values  
Related exam tasks: 2.4.1–2.4.3, 2.5.1, 4.1–4.3, 5.2  
Last verified: 2026-07-23

## Objective

Build and verify a small Amazon Bedrock Runtime client that:

1. selects a model that supports `Converse`, `ConverseStream`, and `CountTokens`;
2. rejects a request that exceeds an application token budget;
3. invokes the model synchronously with `Converse`;
4. invokes the same model with `ConverseStream`;
5. measures time to first token (TTFT), total latency, and token usage;
6. distinguishes retryable service failures from non-retryable contract errors;
7. optionally streams the response to a browser over an existing REST/SSE contract.

Do not optimize for a particular model ID. Model availability, APIs, and Regions change. The skill is to verify the capability and use the correct contract.

## Success criteria

The lab is complete only when you have evidence that:

- `CountTokens` and inference used the same model and input shape;
- an over-budget input was blocked before a paid inference call;
- `Converse` returned one complete response;
- `ConverseStream` delivered at least one text delta before the final event;
- client-measured TTFT is lower than client-measured total latency;
- CloudWatch shows the relevant Bedrock metrics;
- an intentionally malformed request failed without being retried unchanged;
- no sensitive prompt or response was retained unintentionally.

## Minimal architecture

```text
Local test client or test Lambda
  ├─ Bedrock control plane: GetFoundationModel
  └─ Bedrock Runtime endpoint
       ├─ CountTokens
       ├─ Converse
       └─ ConverseStream
            ↓
       CloudWatch AWS/Bedrock metrics

Optional browser path:

Browser using SSE
  → API Gateway REST API, responseTransferMode=STREAM
  → response-streaming Lambda
  → Bedrock ConverseStream
```

## Prerequisites

- An AWS account with permission to use Amazon Bedrock.
- Temporary AWS credentials through an approved federation method.
- An AWS Region that supports the selected model and APIs.
- Model access enabled where the selected model requires it.
- A current AWS SDK. The examples use Python/Boto3 for direct Bedrock calls.
- AWS CLI v2 for read-only capability checks and deployment inspection.
- Optional REST/SSE extension: a supported Node.js Lambda runtime and an API Gateway REST API.
- A test prompt containing no PII, PHI, secrets, customer content, or proprietary data.

### Least-privilege lab permissions

Use a lab role and scope resources to the selected model or approved inference profile wherever the action supports resource-level permissions.

Illustrative policy fragment:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "InvokeOneApprovedModel",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:CountTokens"
      ],
      "Resource": [
        "APPROVED_MODEL_OR_INFERENCE_PROFILE_ARN"
      ]
    }
  ]
}
```

`Converse` and `ConverseStream` use `bedrock:InvokeModel` authorization. Recheck the service authorization reference for the exact resource type used by your selected model/profile.

Do not grant `bedrock:*` merely to make the lab easier.

## Cost and safety guardrails before starting

- Set a local request limit, for example 20 total inference calls.
- Set a small task-specific output ceiling, for example 100–300 tokens.
- Use a short public-information prompt.
- Do not load test or intentionally trigger service throttling.
- `CountTokens` currently does not incur a charge, but model inference does.
- Model invocation logging is disabled by default and can retain full input/output when enabled. Do not enable it with sensitive content.
- If your organization requires a Bedrock Guardrail, include its identifier and version in every supported inference request; this lab does not authorize bypassing that policy.

## Step 1 — Select and verify a model

Use the current [models at a glance](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html) page and the model detail page to select a model that supports:

- the Region you will use;
- Converse;
- streaming;
- Bedrock Runtime `CountTokens`.

Some models, including some models available only through cross-Region inference, do not support `CountTokens` on `bedrock-runtime`. Choose a compatible model for this lab instead of quietly switching token-count contracts.

For a base foundation model, inspect metadata:

```bash
aws bedrock get-foundation-model \
  --region REGION \
  --model-identifier MODEL_ID
```

Record:

```text
Region:
Model ID or inference profile:
Converse supported:
Response streaming supported:
CountTokens on bedrock-runtime supported:
Maximum context/output limits:
Access date:
```

The AWS CLI does **not** support Bedrock streaming operations such as `ConverseStream`. Use an AWS SDK for the streaming step.

## Step 2 — Define one canonical test request

Keep the test deterministic enough to compare API behavior:

```python
SYSTEM = [{"text": "Answer concisely. Return plain text. Do not invent facts."}]

MESSAGES = [
    {
        "role": "user",
        "content": [
            {
                "text": (
                    "Explain in two sentences why streaming reduces perceived "
                    "latency but might not reduce total generation time."
                )
            }
        ],
    }
]

INFERENCE_CONFIG = {
    "maxTokens": 180,
    "temperature": 0.0,
}
```

Use exactly the same `SYSTEM`, `MESSAGES`, and selected model for token counting and inference.

## Step 3 — Count tokens and enforce admission

SDK example:

```python
import boto3

REGION = "REGION"
MODEL_ID = "MODEL_ID_OR_PROFILE"
INPUT_TOKEN_BUDGET = 800

runtime = boto3.client("bedrock-runtime", region_name=REGION)

token_response = runtime.count_tokens(
    modelId=MODEL_ID,
    input={
        "converse": {
            "system": SYSTEM,
            "messages": MESSAGES,
        }
    },
)

input_tokens = token_response["inputTokens"]

if input_tokens > INPUT_TOKEN_BUDGET:
    raise ValueError(
        f"INPUT_TOKEN_BUDGET_EXCEEDED: {input_tokens} > {INPUT_TOKEN_BUDGET}"
    )
```

Admission should normally evaluate more than input tokens:

```text
inputTokens
+ task-specific maximum output
+ reserved tool/retrieval/system allowance
≤ model context limit and application budget
```

Do not count characters or words as a substitute. Tokenization is model-specific.

### Evidence

Save a sanitized record:

```json
{
  "modelId": "record-the-selected-id",
  "inputTokens": 42,
  "inputBudget": 800,
  "maxOutputTokens": 180,
  "admitted": true
}
```

## Step 4 — Invoke synchronously with Converse

```python
from time import monotonic

sync_started = monotonic()

sync_response = runtime.converse(
    modelId=MODEL_ID,
    system=SYSTEM,
    messages=MESSAGES,
    inferenceConfig=INFERENCE_CONFIG,
    requestMetadata={
        "lab": "aip-c01-api-streaming",
        "mode": "sync"
    },
)

sync_total_ms = (monotonic() - sync_started) * 1000
sync_text = "".join(
    block.get("text", "")
    for block in sync_response["output"]["message"]["content"]
)

sync_usage = sync_response.get("usage", {})
sync_metrics = sync_response.get("metrics", {})

print(sync_text)
print(
    {
        "clientTotalMs": round(sync_total_ms, 1),
        "usage": sync_usage,
        "serviceMetrics": sync_metrics,
        "requestId": sync_response["ResponseMetadata"]["RequestId"],
    }
)
```

Validate:

- the response role/content structure was parsed rather than treated as a raw string;
- the response fits the requested two-sentence/plain-text contract;
- `usage.inputTokens` agrees with the token count when the exact input/model was used;
- the request ID is captured;
- the client total and service metrics are not mislabeled as TTFT.

## Step 5 — Invoke with ConverseStream

```python
from time import monotonic

stream_started = monotonic()
first_text_at = None
parts = []
stream_metadata = {}
stream_error = None

stream_response = runtime.converse_stream(
    modelId=MODEL_ID,
    system=SYSTEM,
    messages=MESSAGES,
    inferenceConfig=INFERENCE_CONFIG,
    requestMetadata={
        "lab": "aip-c01-api-streaming",
        "mode": "stream"
    },
)

for event in stream_response["stream"]:
    if "contentBlockDelta" in event:
        delta = event["contentBlockDelta"].get("delta", {})
        text = delta.get("text")
        if text:
            if first_text_at is None:
                first_text_at = monotonic()
            parts.append(text)
            print(text, end="", flush=True)

    elif "metadata" in event:
        stream_metadata = event["metadata"]

    elif "throttlingException" in event:
        stream_error = ("THROTTLED", event["throttlingException"])
        break

    elif "modelStreamErrorException" in event:
        stream_error = ("MODEL_STREAM_ERROR", event["modelStreamErrorException"])
        break

stream_finished = monotonic()

client_ttft_ms = (
    None
    if first_text_at is None
    else (first_text_at - stream_started) * 1000
)
client_total_ms = (stream_finished - stream_started) * 1000
stream_text = "".join(parts)

print()
print(
    {
        "clientTtftMs": (
            None if client_ttft_ms is None else round(client_ttft_ms, 1)
        ),
        "clientTotalMs": round(client_total_ms, 1),
        "metadata": stream_metadata,
        "error": stream_error,
    }
)
```

Production code must handle every documented event and exception type for the SDK version, not only the subset shown above.

Validate:

- at least one text delta appeared before loop completion;
- TTFT is measured from before the API call to the first text delta;
- total latency is measured to stream completion;
- final text is assembled in order;
- usage and service latency data are read from the final metadata event where provided;
- a stream error is not appended to the user-visible answer as if it were model text.

## Step 6 — Compare synchronous and streaming behavior

Use at least five low-volume paired runs. Do not claim a performance improvement from one call.

| Run | Sync total ms | Stream TTFT ms | Stream total ms | Input tokens | Output tokens | Errors |
|---:|---:|---:|---:|---:|---:|---|
| 1 |  |  |  |  |  |  |
| 2 |  |  |  |  |  |  |
| 3 |  |  |  |  |  |  |
| 4 |  |  |  |  |  |  |
| 5 |  |  |  |  |  |  |

Expected conclusion:

- streaming should let the user see content before completion;
- total generation time can be similar, lower, or higher;
- model, prompt, Region, load, output length, and network affect results;
- TTFT and total latency are separate service objectives.

## Step 7 — Optional REST/SSE integration

Use this extension only if the browser already supports SSE and the requirement is to preserve a REST contract.

### Lambda streaming pseudocode

Use a supported Node.js runtime and the documented Lambda response-streaming API:

```javascript
export const handler = awslambda.streamifyResponse(
  async (event, rawStream) => {
    const stream = awslambda.HttpResponseStream.from(rawStream, {
      statusCode: 200,
      headers: {
        "content-type": "text/event-stream",
        "cache-control": "no-cache",
      },
    });

    // Validate/authenticate request and perform token admission first.
    const bedrockEvents = await callConverseStreamWithAwsSdk(event);

    for await (const modelEvent of bedrockEvents) {
      const text = extractTextDelta(modelEvent);
      if (text) {
        stream.write(`event: delta\ndata: ${JSON.stringify({ text })}\n\n`);
      }
    }

    stream.write(`event: done\ndata: {}\n\n`);
    stream.end();
  }
);
```

The helpers are intentionally pseudocode. Use the current SDK event structure and perform all authentication, validation, Guardrail, and error handling required by your application.

### API Gateway integration

For a REST API Lambda proxy integration:

- use the `/response-streaming-invocations` Lambda integration URI;
- set `responseTransferMode` to `STREAM`;
- use `AWS_PROXY`;
- configure the Lambda timeout to cover the full request;
- configure API Gateway access logs.

Illustrative OpenAPI integration:

```yaml
x-amazon-apigateway-integration:
  type: aws_proxy
  httpMethod: POST
  uri: arn:aws:apigateway:REGION:lambda:path/2021-11-15/functions/LAMBDA_ARN/response-streaming-invocations
  responseTransferMode: STREAM
  timeoutInMillis: 90000
```

Current API Gateway documentation says REST response streaming:

- is supported only for proxy integration types;
- does not support request streaming;
- cannot use features that require the full response to be buffered, such as API Gateway endpoint caching and VTL response transformation;
- has duration, idle-timeout, and bandwidth considerations that must be rechecked before production.

### Client validation

```bash
curl --no-buffer -i \
  -H 'content-type: application/json' \
  -d '{"message":"Use the public lab prompt"}' \
  'https://API_ID.execute-api.REGION.amazonaws.com/prod/chat'
```

Evidence:

- SSE `delta` frames appear before the `done` frame;
- `content-type` is `text/event-stream`;
- API Gateway access logs show `STREAMED`;
- `timeToFirstContent` is less than total integration latency.

The API Gateway console test can buffer a stream. Validate the deployed endpoint with a streaming-capable client.

## Step 8 — Observe CloudWatch

In the `AWS/Bedrock` namespace, inspect the selected model dimension:

- `Invocations`;
- `InvocationLatency`;
- `TimeToFirstToken`;
- `InputTokenCount`;
- `OutputTokenCount`;
- `InvocationClientErrors`;
- `InvocationServerErrors`;
- `InvocationThrottles`.

`TimeToFirstToken` applies to streaming operations. CloudWatch data is aggregated and can arrive after the client test; it will not necessarily equal one client-side measurement exactly.

If model invocation logging is already approved and enabled, verify the operation, model ID, request ID, token counts, and safe `requestMetadata`. Do not enable full-content logging solely for this lab.

## Validation evidence checklist

- [ ] Selected model detail or capability output saved.
- [ ] IAM policy scoped to the approved model/profile.
- [ ] Token count and admission decision recorded.
- [ ] Synchronous response structure and request ID recorded.
- [ ] Five paired timing runs recorded.
- [ ] First delta observed before stream completion.
- [ ] Stream final metadata captured.
- [ ] CloudWatch metric screenshot/export captured.
- [ ] Malformed request failure captured.
- [ ] Optional SSE frames and API Gateway access-log timings captured.
- [ ] No raw sensitive content present in evidence.

## Expected failure tests

Run only low-volume, controlled failures.

### Failure 1 — Token budget exceeded

Lower `INPUT_TOKEN_BUDGET` below the measured count.

Expected:

- application returns `INPUT_TOKEN_BUDGET_EXCEEDED`;
- no `Converse`/`ConverseStream` call is made;
- Bedrock invocation count does not increase for the rejected request.

### Failure 2 — Invalid Converse structure

In a test copy, use an invalid role or omit required content.

Expected:

- a validation/client error;
- no retry of the same invalid request;
- safe error returned to the caller;
- `InvocationClientErrors` can increase.

### Failure 3 — Unsupported streaming capability

Do not invoke a random paid model. Instead, identify from model metadata a model or configuration that does not support the desired streaming path and verify that deployment/config validation prevents enabling it.

Expected:

- capability gate rejects configuration before production traffic.

### Failure 4 — Partial stream interruption

Cancel the test client after receiving several deltas.

Expected:

- client records an incomplete response;
- server detects/discovers disconnect as supported;
- the application does not silently issue a full retry and concatenate duplicate text;
- metrics/logs distinguish incomplete delivery from a valid final response.

API Gateway warns that Lambda can continue running after a connection closes. Account for this in cancellation and cost design.

### Failure 5 — Simulated transient dependency error

Do not generate real throttling. In an adapter unit/integration test, simulate a throttling exception before the first byte.

Expected:

- bounded SDK/application retry with backoff and jitter;
- no unbounded loop;
- retry stops at the end-to-end deadline;
- after partial delivery, the same retry policy is not blindly applied.

### Failure 6 — Logging policy violation

Attempt to add a field named `rawPrompt` to the lab evidence logger and verify the logger/test rejects it.

Expected:

- only approved sanitized attributes are emitted.

## Troubleshooting guide

| Symptom | Check first | Likely correction |
|---|---|---|
| Access denied | action, role, resource ARN, model access | Add only required permission/access |
| `CountTokens` validation error | model support and input union | Choose supported model; use exactly one `converse` or `invokeModel` member |
| Converse validation error | role/content/message shape | Correct common message schema |
| Parameter ignored | model support and field location | Use supported `inferenceConfig` or model-specific field |
| No stream deltas | model capability and API | Use supported model with `ConverseStream` |
| CLI cannot stream | tool limitation | Use an AWS SDK |
| Browser receives everything at end | API integration transfer mode/client buffering | Configure REST proxy streaming and use `--no-buffer`/SSE client |
| API returns 500 in streaming mode | Lambda output metadata/delimiter contract | Use documented response-streaming helper/format |
| TTFT is null | no text event before error/end | Inspect event types and model output |
| Token count differs | input/model was not identical | Count the exact request shape and selected model |
| Repeated text after failure | full stream retried after partial delivery | Terminal error or sequence-aware resume |

## Cost and safety cautions

- Bedrock inference is billable; pricing is model- and Region-specific.
- A high `maxTokens` can reserve/consume token-based capacity beyond typical output needs.
- API Gateway response streaming has separate cost and quota considerations.
- CloudWatch Logs, metrics, and optional S3 invocation logs can incur cost.
- Invocation logging can retain entire content and must follow organizational privacy/retention policy.
- Never put API keys, AWS credentials, or secrets in source, prompts, `requestMetadata`, or screenshots.
- Do not treat model output as trusted HTML/Markdown; sanitize at the UI boundary.
- Streaming does not weaken output safety requirements. If output must be fully validated before release, token-by-token display might be incompatible with that requirement.

## Cleanup

For every resource, verify ownership before deleting it. Do not remove shared resources.

1. Stop the local test client.
2. Delete the optional API Gateway stage/API created only for this lab.
3. Delete the optional Lambda function and its dedicated log group after exporting required evidence.
4. Delete dedicated alarms/dashboards created only for the lab.
5. Detach and delete the dedicated lab IAM role/policy.
6. If this lab alone enabled model invocation logging, remove its configuration and destinations according to retention policy. Do not disable organization-wide logging.
7. Delete dedicated S3 log objects only when retention policy permits.
8. Confirm there are no recurring canaries, schedules, provisioned capacity, or other continuing charges.

## Exam lessons

1. `Converse` is the common synchronous message API for supported models.
2. `ConverseStream` is the common message-style streaming API; the AWS CLI does not support the streaming operation.
3. `InvokeModel` uses a model-native payload and is not interchangeable with Converse.
4. Check model/API/Region capability before enabling a feature.
5. `CountTokens` is model-specific and should use the exact planned request shape.
6. Reject over-budget input before inference.
7. A small task-specific output ceiling improves cost and token-based throughput control.
8. Streaming reduces perceived latency; measure TTFT separately from total latency.
9. REST response streaming/SSE can preserve an existing REST contract; WebSocket is not mandatory.
10. Retry transient errors with bounds and jitter; do not retry malformed requests unchanged.
11. A retry after partial stream delivery can duplicate output.
12. CloudWatch metrics, application timing, and invocation logs provide different evidence.

## Official sources

- [Inference using the Converse API](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference.html)
- [ConverseStream API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_ConverseStream.html)
- [Count tokens before inference](https://docs.aws.amazon.com/bedrock/latest/userguide/count-tokens.html)
- [CountTokens API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_CountTokens.html)
- [Bedrock inference request parameters by model](https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters.html)
- [Bedrock runtime CloudWatch metrics](https://docs.aws.amazon.com/bedrock/latest/userguide/monitoring-runtime-metrics.html)
- [Bedrock model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html)
- [API Gateway response streaming](https://docs.aws.amazon.com/apigateway/latest/developerguide/response-transfer-mode.html)
- [API Gateway Lambda response-streaming integration](https://docs.aws.amazon.com/apigateway/latest/developerguide/response-transfer-mode-lambda.html)
- [Configure Lambda response streaming in API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/response-streaming-lambda-configure.html)
- [Troubleshoot API Gateway response streaming](https://docs.aws.amazon.com/apigateway/latest/developerguide/response-streaming-troubleshoot.html)

