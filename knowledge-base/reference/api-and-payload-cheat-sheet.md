# Bedrock API and Payload Cheat Sheet

Status: Verified conceptually; always check the selected model’s current capabilities and schema  
Last verified: 2026-07-23

## Endpoint families

| Endpoint/service client | Purpose |
|---|---|
| `bedrock` control plane | Model discovery, configuration, evaluation and administrative operations |
| `bedrock-runtime` | Bedrock-native inference including Converse, InvokeModel, streaming, CountTokens, and ApplyGuardrail |
| `bedrock-mantle` | Supported OpenAI-compatible Responses/Chat Completions and Anthropic Messages APIs, with stateful and agentic capabilities |
| `bedrock-agent` build time | Create/manage agents, Knowledge Bases, prompts, and flows |
| `bedrock-agent-runtime` | Invoke agents, retrieve, retrieve-and-generate, invoke flows |
| AgentCore APIs | Runtime, Gateway, Memory, and evaluation capabilities |
| `sagemaker-runtime` | Invoke a model deployed to a SageMaker endpoint |

Using the wrong endpoint family is a common cause of implementation and exam errors.

AWS currently recommends `bedrock-mantle` for new applications whose required models and compatible APIs/features are supported. Use `bedrock-runtime` for Bedrock-native `Converse`/`InvokeModel`, unsupported Mantle models, and feature differences such as documented structured-output cases. The two endpoint families can coexist; select by API and capability, not price.

## Choose the inference API

| Need | API |
|---|---|
| Common message interface across supported chat models | `Converse` |
| Common streaming message interface | `ConverseStream` |
| Specialized model or provider-native parameters/body | `InvokeModel` |
| Specialized model streaming | `InvokeModelWithResponseStream` |
| Estimate tokens for the actual request/model | `CountTokens` |
| Evaluate text independently with a guardrail | `ApplyGuardrail` |

`Converse` reduces provider-specific application code, but it does not make all models identical. Model capability, Region, lifecycle, tool use, modality, and additional parameters still differ.

## Converse mental model

Illustrative structure:

```json
{
  "modelId": "model-or-inference-profile",
  "system": [
    {"text": "System instruction"}
  ],
  "messages": [
    {
      "role": "user",
      "content": [{"text": "Question"}]
    }
  ],
  "inferenceConfig": {
    "maxTokens": 300,
    "temperature": 0.1
  },
  "additionalModelRequestFields": {}
}
```

Remember:

- `messages` contains role/content turns.
- Content is represented as content blocks.
- Use `additionalModelRequestFields` only for supported model-specific options.
- Tool configuration and guardrail configuration add documented structures.
- A Prompt Management ARN can be used according to current supported prompt-invocation rules; pass prompt variables separately.
- Do not include fields the prompt resource already defines if the API disallows overriding them.

Use the current SDK/API reference rather than memorizing every JSON field.

## InvokeModel mental model

`InvokeModel` uses:

- `modelId`.
- `contentType`, normally `application/json` for JSON request bodies.
- `accept`.
- A provider/model-specific request body.

The correct body for one provider or modality is not necessarily valid for another. A stable client contract therefore needs a model-aware serializer/adapter:

`canonical app request → selected model schema → InvokeModel → normalized app response`

Do not send a generic `prompt` body to every model.

## Streaming

| Path | Use |
|---|---|
| `ConverseStream` | Supported chat models with a common stream format |
| `InvokeModelWithResponseStream` | Supported specialized/provider-native model streaming |
| API Gateway WebSocket | Bidirectional/long-lived client interaction |
| REST API + Lambda response streaming/SSE | Preserve HTTP/SSE client contract where supported |

Before enabling streaming:

- Check the model supports response streaming.
- Handle incremental content and final metadata.
- Handle mid-stream error/disconnect.
- Do not run output safety only after unsafe tokens are already shown.
- Measure TTFT and total latency separately.

Streaming improves perceived latency; it does not automatically reduce model cost.

## CountTokens

Use `CountTokens` with the selected model and the same logical input shape intended for inference.

Applications:

- Context-window validation.
- Per-tenant token admission.
- Cost estimates.
- Pruning decisions.
- Output-budget reservation.

Do not estimate capacity only from character count or request count. Tokenization varies by model and content.

## Guardrails

### Attached to inference

Pass the approved guardrail identifier and version through the supported inference API. Use the documented tagged content mechanism when only selected input blocks should be evaluated.

### Independent evaluation

`ApplyGuardrail` evaluates content without invoking an FM:

- `source=INPUT` for user/input content.
- `source=OUTPUT` for generated/external output.

Use it in pre/post pipelines or when the model invocation path cannot attach the guardrail in the needed way.

IAM conditions can require an approved guardrail for supported runtime actions, preventing production callers from bypassing it.

## Knowledge Bases

| API | Use |
|---|---|
| `Retrieve` | Return chunks/metadata/scores; application controls generation |
| `RetrieveAndGenerate` | Retrieval plus FM answer/citations for supported non-Managed Knowledge Base configurations |
| `AgenticRetrieveStream` | Multi-step retrieval and optional citation-backed synthesis for Managed Knowledge Bases |
| `StartIngestionJob` | Synchronize a data source with the vector store |
| Ingestion job status APIs | Confirm terminal success/failure |

The AWS product named **Managed Knowledge Bases** is a specific Knowledge Base type. Current `RetrieveAndGenerate` documentation explicitly excludes it; use `Retrieve` or `AgenticRetrieveStream`.

Use `Retrieve` to isolate retrieval quality or inject a custom final response schema/workflow. Inspect:

- Retrieved text/content.
- Metadata.
- Source location.
- Score.

Do not interpret score as factual certainty.

## Agents and flows

### InvokeAgent

Use the runtime API with trace enabled when you need orchestration evidence. Separate:

- Final response chunks.
- Trace events describing orchestration, KB, action group, and failure behavior.

Trace is not provider-internal chain-of-thought.

### InvokeFlow

Invoke a prepared Bedrock Flow and enable tracing during tests to inspect per-node behavior. Flows suit configured prompt/condition/KB/Lambda pipelines.

## Tool schemas

Every tool definition should have:

- Stable name and description.
- Explicit JSON input schema.
- Required fields and types.
- Bounds/enums/patterns where useful.
- Structured success and error response.
- Idempotency semantics.

The tool implementation must still validate parameters. A schema helps the model but is not a security boundary.

## SageMaker InvokeEndpoint

Send the exact media type and schema expected by the deployed container. A Bedrock Converse body is not a SageMaker endpoint body.

Common failure:

- `Content-Type: application/json` is set, but the JSON fields do not match the model container’s expected contract.

## Request validation and resilience

At API Gateway:

- Validate required JSON fields/schema.
- Authenticate/authorize.
- Throttle clients.
- Assign a correlation ID.

In compute:

- Validate semantic constraints and authorization.
- Estimate tokens.
- Sanitize or redact.
- Invoke with bounded exponential backoff for transient failures.
- Validate output schema and safety.
- Make side effects idempotent.

Do not retry invalid schemas, forbidden requests, or policy blocks.

## Common mismatches

| Symptom | Likely mismatch |
|---|---|
| Missing required messages | Provider prompt sent to Converse |
| Generation field ignored | Field belongs to another model/schema |
| JSON parse error in SageMaker | Container input schema/content type wrong |
| Streaming call rejected | Model/API does not support streaming |
| Guardrail bypass | Caller did not pass identifier/version and IAM did not require it |
| Retrieval returns data but no citations | Used Retrieve and expected generation output |
| Prompt ARN invocation fails | Unsupported API/model or conflicting fields |

## Recall questions

1. Why is Converse common but not model-independent?
2. When is InvokeModel preferable?
3. What does CountTokens need to model accurately?
4. What is the difference between attached Guardrails and ApplyGuardrail?
5. When should an application call Retrieve instead of RetrieveAndGenerate?
6. Why must a tool implementation validate parameters even with JSON Schema?
7. What does agent trace expose, and what does it not expose?

## Official sources

- [Bedrock Runtime API](https://docs.aws.amazon.com/bedrock/latest/APIReference/welcome.html)
- [Bedrock Runtime and Mantle endpoints](https://docs.aws.amazon.com/bedrock/latest/userguide/endpoints.html)
- [Converse](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference-call.html)
- [Model-specific parameters](https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters.html)
- [Knowledge Bases runtime](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-retrieve-generate.html)
- [`RetrieveAndGenerate` API and Managed Knowledge Base exclusion](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_RetrieveAndGenerate.html)
