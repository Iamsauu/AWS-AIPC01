# Enterprise Integration, Bedrock APIs, and CI/CD

Status: Verified  
Official tasks: 2.3, 2.4, 2.5; supporting 3.2–3.3, 4.2–4.3, and 5.2  
Last verified: 2026-07-23

## Why this matters

A production GenAI feature is part of a larger system. It must accept requests from existing clients, survive bursts and dependency failures, preserve a stable contract while models change, enforce enterprise identity and policy, and release prompts/models/code safely.

The exam repeatedly asks which managed integration service best expresses the requirement. The decision is driven by synchronization, fanout, durability, latency, residency, and operational overhead—not by the presence of an FM.

## Core concepts

### Translate wording into an integration constraint

| Requirement wording | Architectural implication |
|---|---|
| “Return the complete response now” | Synchronous API path |
| “Show text as it is generated” | Streaming model API plus streaming client transport |
| “Acknowledge within two seconds” | Validate minimally, enqueue/publish, return immediately |
| “Bursty and must not lose work” | Durable queue and idempotent worker |
| “Add consumers without changing producer” | Event bus and independent rules/targets |
| “May take hours or days” | Durable workflow/job pattern; not a held request |
| “One stable contract across providers” | Gateway plus provider-specific adapter |
| “Change routing without deployment” | Validated dynamic configuration such as AppConfig |
| “Review every step later” | Step Functions execution history plus structured logs |
| “No public internet path” | Private networking and VPC endpoints where supported |
| “Stay inside a geography” | Verify model/feature Region support and geographic routing |

### Synchronous, streaming, asynchronous, and batch

| Mode | Producer expectation | Typical pattern |
|---|---|---|
| Synchronous | One complete result within request deadline | API Gateway → Lambda/service → Bedrock Runtime |
| Streaming | Incremental output on one interactive request | API Gateway streaming/SSE or WebSocket → `ConverseStream` |
| Asynchronous application job | Immediate job ID; result later | API → SQS/workflow → worker → DynamoDB/S3 |
| Batch inference | Large offline dataset and S3 result | Bedrock batch inference or SageMaker Batch Transform |

Do not call every non-immediate operation “batch.” An SQS-backed application job and a managed batch inference job have different APIs, quotas, record formats, and completion models.

## AWS services and APIs

### Amazon API Gateway

#### HTTP API versus REST API versus WebSocket

| Need | Default choice |
|---|---|
| Low-cost, simple HTTP proxy/API with supported features | HTTP API |
| Request models/validation, broader REST feature set, existing REST contract | REST API |
| Bidirectional, long-lived messaging | WebSocket API |
| Existing REST browser supports server-sent events | REST API response streaming with streaming Lambda proxy integration |

Choose based on required features, not the word “API.” Validate authentication, content type, required JSON fields, body size, and rate limits at the edge; repeat business-rule validation in the application.

#### Request validation

API Gateway request models can reject malformed requests before Lambda/model invocation. Validation should cover:

- content type;
- required properties;
- property type/length/range;
- permitted enum values;
- request size;
- route-specific authentication and quota.

JSON schema validation cannot determine whether the user may access a business object or whether the model/tool supports the requested operation. That remains application logic.

#### Streaming paths

For an existing REST API and SSE-capable browser, current API Gateway supports a Lambda proxy integration with:

- `responseTransferMode: STREAM`;
- the Lambda `/response-streaming-invocations` integration URI;
- a Lambda function that writes SSE frames as Bedrock stream deltas arrive.

Use WebSocket when the client and server both need long-lived bidirectional messages. Do not force a WebSocket migration when server-to-client streaming over an existing REST contract is the actual requirement.

Streaming implementation must handle:

- content-type and event framing;
- first-token, delta, final, and error events;
- client disconnect/cancellation;
- backpressure;
- idle and maximum duration timeouts;
- partial-response observability;
- deduplication if a resumable protocol retries.

### AWS Lambda

Good fit:

- short, stateless request adapters;
- webhook verification;
- request normalization;
- Bedrock invocation;
- lightweight tool execution;
- queue/event consumers;
- callback endpoints.

Poor fit:

- multi-hour work;
- long-lived stateful MCP server;
- large native/CPU-heavy processing beyond practical limits;
- sleeping while a human reviews;
- relying on process memory for durable state.

Initialize SDK and HTTP clients outside the handler where safe, enable connection reuse/keep-alive, and use connection pools to reduce downstream setup latency.

### Amazon SQS

Use SQS when:

- traffic arrives in bursts;
- the producer needs an immediate acknowledgement;
- one durable processing path should absorb backpressure;
- work can be retried independently;
- the producer must not depend on Bedrock availability.

Required design:

- idempotent consumer;
- visibility timeout longer than normal processing with appropriate margin;
- bounded redrive and dead-letter queue;
- partial batch response handling where appropriate;
- stable job/business ID;
- result/status store;
- monitoring for age, depth, failures, and DLQ.

An SQS event source is at-least-once. Exactly-once business behavior comes from idempotency, not from assuming one Lambda invocation.

### Amazon EventBridge

Use EventBridge when a business event should reach multiple independent or future consumers. The webhook handler publishes a compact event; rules route to downstream targets.

Good event fields:

- event ID and schema version;
- event type;
- source;
- occurred time;
- tenant/business object ID;
- pointer to large data rather than a sensitive full transcript;
- safe correlation ID.

Do not embed regulated content in events unless explicitly required and protected. Design each target for duplicate delivery and out-of-order events where relevant.

### Amazon SNS

Use SNS primarily for pub/sub notification and fanout. It can deliver to SQS, Lambda, HTTP, email, and other supported targets.

SNS is not:

- an approval-state store;
- a deterministic workflow;
- the default durable work buffer for one processing path;
- a replacement for idempotency.

### AWS Step Functions

Use Standard workflows for:

- branch-heavy document or agent processing;
- long-running operations;
- human callbacks;
- explicit retries and fallback;
- parallel model calls;
- audit-ready execution history.

Use Bedrock Flows when the core requirement is a managed, editable FM/prompt/Knowledge Base/Lambda graph. Use Step Functions when the requirement is general AWS orchestration, multi-day durability, callbacks, or stronger deterministic workflow control.

## Bedrock API integration contracts

### Endpoints

- Use the Regional **`bedrock-runtime`** endpoint for Bedrock-native `Converse`, `ConverseStream`, `InvokeModel`, and related runtime APIs.
- Use **`bedrock-mantle`** for supported OpenAI-compatible or Anthropic-compatible APIs and Mantle capabilities when they fit the application.
- Check model and feature availability because the two endpoint families are not identical.
- Use the Bedrock control plane for model metadata and resource management.
- Do not route runtime invocation to the wrong endpoint family.
- Check current model/API/Region compatibility before deployment.

### API decision table

| API | Correct use | Key request shape | Error/retry behavior | Main distractor |
|---|---|---|---|---|
| `Converse` | Synchronous chat/message interaction across supported models | `modelId`, `messages[]`, role/content blocks, optional system/inference config | Retry throttling/transient errors; fix validation errors | A provider-native body reused across models |
| `ConverseStream` | Message-style incremental output | Converse structure plus streamed event processing | Treat errors before and after first byte differently | `Converse` when immediate partial output is required |
| `InvokeModel` | Native model features, embeddings, image/specialized calls, exact provider schema | `modelId`, `contentType: application/json`, model-native JSON body | Serializer must match every model | Believing `InvokeModel` normalizes providers |
| `InvokeModelWithResponseStream` | Native-schema streaming where supported | Native request body and stream | Verify capability first | Unsupported client/API combination |
| `CountTokens` | Admission control and cost/context preflight | Use actual model and relevant request format | Not an invocation or output guarantee | Character/word estimation |
| `GetFoundationModel` | Inspect capabilities such as response streaming support | Model identifier on control plane | Cache carefully; lifecycle can change | Enabling a feature by assumption |
| `ApplyGuardrail` | Independently assess input or output content | Guardrail ID/version, source, content | Fail closed where policy mandates | Prompt-only safety |
| `Retrieve` | Application controls retrieval merging/generation | Knowledge Base retrieval request | Retry service failures; inspect results | `RetrieveAndGenerate` when custom control is required |
| `RetrieveAndGenerate` | Retrieval plus generation for supported non-Managed Knowledge Base configurations | retrieval and generation config | Inspect citations/references | Using it with the product named Managed Knowledge Bases |
| `InvokeAgent` | Invoke configured Bedrock agent | agent/alias/session input; optional trace | Separate trace and final response | Treating trace as model confidence |

### Converse portability

`Converse` supplies a common message interface for supported models. Portability has limits:

- model capabilities still differ;
- supported content blocks differ;
- tool use, structured output, reasoning, latency features, and context limits differ;
- model-specific options may use `additionalModelRequestFields`;
- output must still be normalized and validated for the application contract.

Converse does not mean “every model supports every feature.”

### InvokeModel serialization

Build a model-aware adapter:

```text
canonical application request
  → resolve approved model/provider
  → validate model capability
  → serialize exact native request body
  → set application/json
  → InvokeModel
  → parse native response
  → normalize stable public response
```

An imported/customized model can require a different native body. Unsupported parameters might be rejected or ignored. Validate that important settings actually take effect.

### Token admission

Use `CountTokens` with the actual selected model and request shape. Admission should consider:

- counted input tokens;
- requested output ceiling;
- model context limit;
- tenant/user token budget;
- current rate/concurrency policy;
- reserved system/retrieval/tool context.

`CountTokens` does not predict the exact output length. A very large `maxTokens` can reduce token-based throughput even when typical outputs are short. Set a task-specific output ceiling.

### Resilient SDK behavior

Retry only errors that are likely transient:

- throttling;
- temporary service unavailability;
- retryable network errors;
- some timeouts when no partial/ambiguous side effect exists.

Do not retry unchanged:

- validation failures;
- unsupported model/API combinations;
- access denied;
- malformed content;
- policy/guardrail blocks;
- context limit errors.

Use:

- AWS SDK retry mode appropriate to the workload;
- exponential backoff with jitter;
- maximum attempts;
- end-to-end deadline;
- circuit breaker for repeated dependency failure;
- fallback only to an approved compatible model;
- correlation ID and X-Ray trace.

### Streaming retry boundary

Before the first byte, a bounded retry may be transparent. After partial text reaches the client, a full retry can duplicate or contradict output. Prefer:

- a terminal stream error event;
- a resumable protocol with sequence IDs if implemented;
- client-visible retry action;
- no silent concatenation of a new generation.

## Enterprise architecture patterns

### Pattern 1 — Synchronous common chat API

```text
client
  → API Gateway HTTP/REST API
  → Lambda adapter
      → validate/authenticate/authorize
      → CountTokens and policy checks
      → Converse on bedrock-runtime
      → validate normalized response
  → client
```

Use when the complete response fits the request deadline.

### Pattern 2 — Long request with job ID

```text
client → API Gateway → request Lambda
                         ├─ validate
                         ├─ create job record
                         └─ SQS message → return 202 + job ID

SQS → worker Lambda/service → Bedrock → result store
client → GET job status/result
```

The job ID must be authorized to the requesting user/tenant. Do not expose a predictable ID without an ownership check.

### Pattern 3 — Webhook to extensible consumers

```text
partner webhook
  → API Gateway
  → Lambda validates signature and replay window
  → EventBridge business event
      ├─ GenAI enrichment
      ├─ case-management integration
      └─ notification integration
```

If only one slow processing path exists and durability/backpressure is primary, SQS is simpler.

### Pattern 4 — Central GenAI gateway

```text
internal applications
  → API Gateway / gateway service
      → identity + tenant policy
      → request schema + token limit
      → approved prompt/guardrail enforcement
      → AppConfig routing policy
      → provider adapter
          ├─ Bedrock Converse
          ├─ Bedrock InvokeModel
          └─ approved external provider
      → normalized response + audit/metrics
```

The gateway should produce a stable public contract while preserving provider-specific behavior internally.

Centralize:

- model and feature allowlists;
- mandatory Guardrail version;
- prompt/version resolution;
- quotas and token budgets;
- PII-safe logging policy;
- retry/fallback policy;
- request/response normalization;
- routing reason and selected model;
- correlation and attribution metadata.

Do not centralize application-specific authorization decisions that require business context unless the gateway receives and verifies that context.

### Pattern 5 — Model routing

| Routing need | Pattern |
|---|---|
| Change one model ID/parameters without deployment | AppConfig configuration/feature flag |
| Route by explicit business class with audit path | classifier + Step Functions `Choice` |
| Simple versus complex within supported model family | Bedrock intelligent prompt routing |
| Provider outage/cross-provider policy | AppConfig + provider adapter + compatible fallback |
| Same model under Regional capacity pressure | cross-Region inference profile |
| Two independent artifacts | Step Functions `Parallel`, then merge |

Every route should record a reason code, config version, selected endpoint/model, fallback status, and quality/latency outcome.

### Pattern 6 — Hybrid/on-premises preprocessing

For data that must be normalized or de-identified on premises:

```text
on-premises source
  → EC2/container on Outposts subnet
  → local processing/de-identification
  → private VPC routing
  → Bedrock Runtime interface VPC endpoint in parent VPC/Region
  → geographic inference profile if multiple allowed Regions are needed
```

Key facts:

- Outposts extends a VPC to on-premises hardware.
- A rack local gateway connects the Outpost subnet and local network.
- Direct VPC routing uses instance private IPs and requires non-overlapping CIDRs.
- A VPC endpoint is Region-side; do not assume it is directly reachable from the on-premises network as a requester-managed interface without the supported routing path.
- Send only the approved transformed payload.
- Verify every destination Region allowed by the inference profile.

Use Wavelength as a recognition-level option for carrier-edge, ultra-low-latency applications. It is not the default on-premises data-residency solution.

## Application integration and developer tools

### AWS Amplify Gen 2 AI

Amplify AI routes can accelerate web/mobile GenAI interfaces:

- conversation routes for asynchronous multi-turn interactions;
- generation routes for synchronous structured generation;
- authentication and route authorization;
- streaming UI hooks/components;
- conversation/message storage;
- owner-based access to a user's conversation history.

The current Amplify AI kit architecture uses managed AWS services such as AppSync, DynamoDB, Lambda, and Bedrock. Confirm current availability, generated resources, data retention, and authorization before using it in regulated workloads.

Owner-based authorization is not a substitute for authorization inside a tool that accesses a separate business system.

### OpenAPI

Use OpenAPI to:

- version the API contract;
- define requests, responses, and errors;
- import/configure API Gateway;
- generate clients;
- test compatibility;
- separate the public schema from provider-native payloads.

Do not expose raw Bedrock/provider request bodies as the enterprise public contract unless clients truly require them.

### Amazon Bedrock Flows

Flows supports managed nodes and connections for FM application workflows, including prompt, condition, Knowledge Base, and Lambda nodes. Use `InvokeFlow` tracing during test to inspect per-node inputs and outputs.

Use Flows when:

- prompt/workflow designers need a managed visual/configuration surface;
- the graph centers on prompts, retrieval, conditions, and lightweight Lambda logic;
- changes should not require application deployment.

Use Step Functions when:

- approvals last hours/days;
- general AWS orchestration dominates;
- explicit error/timeout/retry paths and execution durability are central;
- regulated state transitions need a workflow system of record.

### Amazon Bedrock Data Automation

Use Bedrock Data Automation in business workflows that extract structured information from multimodal documents. Pair it with:

- S3 source/output;
- Step Functions for surrounding document process;
- Lambda for lightweight transformations;
- validation/quarantine;
- human review where extraction confidence or consequence requires it.

It is not a replacement for all OCR/ETL or for deterministic validation.

### Amazon Q Developer

Use Amazon Q Developer in supported IDEs for:

- AWS architecture and SDK questions;
- chat and workspace context;
- inline code suggestions;
- code generation and refactoring;
- project rules/guidance;
- security and quality review;
- testing and troubleshooting assistance.

It can identify issues and propose changes. It does not prove that generated code compiles, meets the API contract, passes security review, or behaves correctly. CI tests and human review remain required.

## CI/CD for GenAI systems

### What must be versioned

- application and tool code;
- infrastructure as code;
- OpenAPI schema;
- prompt template/version/variables;
- model or inference profile ID;
- inference parameters;
- Guardrail ID/version;
- retrieval/chunking/index configuration;
- tool schemas and descriptions;
- routing/AppConfig configuration;
- evaluation dataset and metric thresholds;
- deployment and rollback policy.

Without a complete release manifest, “roll back the prompt” might still leave a new model, tool schema, or retrieval configuration active.

### Recommended pipeline

```text
Source
  → Build
  → Unit tests
  → API/schema/tool contract tests
  → Security and IaC scans
  → Prompt/guardrail adversarial tests
  → Model/RAG/agent regression evaluation
  → Package immutable release manifest
  → Deploy non-production
  → Integration and synthetic journey tests
  → Manual approval if required
  → Canary/linear production deployment
  → Bake alarms and quality checks
  → Promote or rollback
```

### CodePipeline

CodePipeline orchestrates source, build, test, approval, and deployment actions.

Current rollback facts:

- an eligible stage can be manually rolled back;
- a stage can be configured for automatic rollback on failure;
- rollback creates a new pipeline execution using artifacts and variables from the selected successful execution;
- a source stage cannot be rolled back;
- the target execution must meet pipeline-structure/version constraints;
- a rollback execution itself is not a rollback target.

Do not invent unsupported “whole pipeline rewind” semantics.

### CodeBuild

Use CodeBuild for:

- unit/integration tests;
- prompt regression runners;
- Guardrail input/output tests;
- tool-schema contract tests;
- OpenAPI compatibility;
- dependency, SAST, secrets, and IaC scanning;
- evaluation-job submission/result gate logic;
- published test reports.

A buildspec can configure one to five report groups. A report group publishes test results; it does not itself decide the business quality threshold. The build command must fail or the pipeline condition must evaluate the result appropriately.

Never put provider API keys or secrets directly in the buildspec. Use IAM roles, Secrets Manager, or Parameter Store as appropriate and prevent secret values from entering logs.

### CodeDeploy

For Lambda:

- publish immutable function versions;
- route an alias;
- use canary or linear traffic shifting;
- associate CloudWatch alarms;
- configure automatic rollback.

Use alarms for:

- error rate;
- throttling;
- p95/p99 latency;
- invalid output/schema rate;
- safety intervention or policy-failure rate where appropriate;
- business/task success degradation.

An infrastructure-only alarm can miss a semantically bad prompt/model release. Include quality/synthetic evidence.

### CloudFormation and AWS CDK

Use infrastructure as code for repeatability across accounts/environments:

- APIs, Lambda, queues, event rules, workflows;
- roles and policies;
- VPC endpoints;
- alarms and dashboards;
- deployment groups/pipelines;
- prompt/model configuration references where supported.

Do not hard-code account IDs, model availability, or Region-specific feature assumptions without validation.

### AppConfig

Use AppConfig for dynamic:

- model IDs or inference profile ARNs;
- inference settings;
- routing rules and thresholds;
- feature/fallback flags;
- provider endpoint selection.

Use validators, staged deployment strategies, and CloudWatch alarms. During a deployment, an alarm or insufficient-data condition can trigger automatic rollback when configured. AppConfig rollback reverts configuration; it does not roll back application code or model artifacts.

## Decision table

| Requirement | Choose | Why not the distractor |
|---|---|---|
| Compact webhook and many future consumers | EventBridge | Hard-coded Lambda fanout couples producer to consumers |
| Burst and one processing path | SQS | EventBridge is not primarily a backpressure work queue |
| Notify a reviewer | SNS plus durable workflow state | SNS alone does not capture approval |
| Full response, simple serverless API | HTTP API + Lambda + Converse | WebSocket adds unnecessary lifecycle complexity |
| Preserve REST and stream to SSE client | REST Lambda response streaming + ConverseStream | No need to migrate protocol |
| Long document result later | Queue/job ID/result store | Holding an API request risks timeout |
| Change model without code release | AppConfig | Environment variables require deployment/config lifecycle and lack staged rollback by themselves |
| Cross-provider stable contract | API Gateway + adapter | Converse only normalizes supported Bedrock models |
| Auditable content-based routing | Step Functions classifier + Choice | Hidden routing inside one Lambda has weaker execution history |
| Prompt-centered editable branch graph | Bedrock Flows | Step Functions can work but adds general orchestration management |
| Multi-day approval | Step Functions Standard callback | Flow/Lambda wait is the wrong durability mechanism |
| Existing app rapid chat UI | Amplify AI conversation route | Building auth/history/streaming from scratch adds overhead |
| On-premises preprocessing and private Bedrock call | Outposts + supported VPC routing + interface endpoint | Wavelength is carrier edge, not on-premises residency |

## Security and governance

### Identity and authorization

- Use workforce federation through IAM Identity Center where applicable.
- Give applications temporary credentials/roles rather than long-term access keys.
- Scope `bedrock:InvokeModel` and related actions to approved resources where service support allows.
- Separate caller role, adapter role, workflow role, and tool roles.
- Enforce tenant/business authorization before retrieval or tool execution.
- Use API Gateway authorizers/IAM as appropriate for the client type.
- Sign and verify webhooks; enforce timestamp/replay window.

### Private networking

Use interface VPC endpoints/PrivateLink for supported services such as Bedrock Runtime when private service traffic is required. Use appropriate endpoints and policies for S3 and other dependencies.

Private networking does not automatically enforce data residency. Model routing, storage/log destinations, and cross-Region profiles must also comply.

### Logging

Log:

- safe correlation ID;
- authenticated principal/tenant pseudonymous ID;
- release/config/prompt/model/Guardrail version;
- route and fallback reason;
- token counts and latency;
- validation and policy outcome;
- dependency status and workflow execution ID.

Do not log:

- raw PHI/PII unless explicitly approved and protected;
- secrets or access tokens;
- callback task tokens;
- raw tool credentials;
- full prompts by default in sensitive workloads.

Use CloudTrail for AWS API audit, CloudWatch for application metrics/logs, X-Ray for service-path timing, and model invocation logging only under an approved content-retention policy.

## Cost, latency, and reliability

### Cost

- API Gateway/ Lambda/event/workflow charges matter, but model tokens usually dominate at scale.
- Event fanout multiplies downstream work.
- SQS batching can improve worker efficiency.
- Step Functions transitions grow with loops and fine-grained state machines.
- Streaming changes perceived latency, not necessarily token cost.
- Provisioned Throughput only pays off when utilization and predictability justify fixed capacity.
- AppConfig/intelligent routing can move routine traffic to smaller models, but must pass quality gates.

### Reliability

- Use DLQs and alarms for asynchronous work.
- Make every event/queue consumer idempotent.
- Use bounded SDK retries with jitter.
- Set API, Lambda, SDK, and workflow timeouts as one end-to-end budget.
- Use geographic/global inference profiles only after checking residency.
- Store job status before enqueueing or use a transaction/outbox pattern when loss between writes matters.
- Fail closed for mandatory security and policy controls.
- Test fallback models for capability and format compatibility before an incident.

## Failure modes and troubleshooting

| Symptom | Inspect first | Probable cause | Remediation |
|---|---|---|---|
| API returns 4xx validation | Gateway validation and request body | Schema/content-type mismatch | Fix client contract; do not retry unchanged |
| Bedrock validation exception | endpoint, API, model ID, message/native body | Wrong runtime endpoint or serializer | Correct endpoint and model-aware shape |
| Parameter silently has no effect | model parameter docs and request field location | Unsupported/misplaced field | Supported native field or `additionalModelRequestFields` |
| Client gets no partial text | model API and Gateway integration mode | Synchronous API or buffered integration | ConverseStream plus configured response streaming/SSE |
| Stream duplicates after error | retry logs and sequence handling | Full retry after partial delivery | Terminal error or resumable protocol |
| Queue work duplicates CRM update | message receive count and external record | Missing idempotency | Stable interaction ID and conditional/external idempotency |
| Queue age grows | age/depth, worker errors, Bedrock throttles | Insufficient consumer/model capacity | Tune batching/concurrency; quotas/capacity; DLQ |
| Webhook times out | handler trace | Synchronous model call in webhook path | Verify/publish/enqueue then acknowledge |
| Event consumer sees wrong schema | event version and rule transformations | Schema drift | Version events and compatibility tests |
| Gateway fallback violates policy | route log and model capabilities | Unapproved fallback | Capability/residency/safety allowlist |
| Cross-Region request denied | SCP/IAM and profile destinations | One destination Region blocked | Allow all required destinations or use compliant profile |
| Outposts path cannot reach service | route tables, endpoint, DNS, CIDRs | Unsupported/incorrect routing or overlap | Correct direct VPC/local gateway and endpoint path |
| Prompt/model release is semantically bad | release manifest and evaluation | Infra tests passed, quality gate absent | Regression dataset and semantic threshold |
| CodePipeline cannot roll back | stage type and pipeline version | Source/ineligible target | Roll forward or select eligible prior execution |
| CodeBuild reports absent | buildspec report paths/permissions | Misconfigured report group/files | Correct paths, format, IAM, and failing commands |
| Lambda p95 increases | X-Ray subsegments | Client/connection created each invoke | Reuse SDK clients and keep-alive |
| Logs cannot correlate request | fields across hops | No common ID or structured format | Propagate safe correlation ID and JSON logs |

## Common exam traps

1. **“Asynchronous” does not automatically mean EventBridge.** Use SQS for one durable buffered work path.
2. **“Multiple consumers” is the EventBridge clue.**
3. **SNS notification is not approval orchestration.**
4. **API Gateway validation does not perform business authorization.**
5. **WebSocket is not required for every stream.** SSE over REST response streaming can preserve an existing contract.
6. **Converse is portable only across supported capabilities.**
7. **InvokeModel requires a model-specific body.**
8. **CountTokens is model/request aware; words and characters are not reliable substitutes.**
9. **Retrying a validation error wastes time and tokens.**
10. **A fallback model must still satisfy safety, modality, tool, format, and residency requirements.**
11. **Cross-Region inference is not the same as a cross-provider fallback.**
12. **A global profile can violate geographic constraints.**
13. **Outposts provides local AWS infrastructure; Wavelength is carrier-edge infrastructure.**
14. **Bedrock Flows and Step Functions overlap but have different centers of gravity.**
15. **Amazon Q Developer output is not a passed test or security approval.**
16. **CodePipeline stage rollback has eligibility constraints and cannot roll back a source stage.**
17. **Infrastructure health alone cannot detect semantic response regression.**
18. **AppConfig rollback does not restore code.**

## Local mock references

### Practice Exam 1

- **PE1-Q03:** signed webhook, immediate acknowledgement, EventBridge fanout.
- **PE1-Q16, Q52:** streaming, token budget, SDK retry.
- **PE1-Q18:** managed batch instead of per-record Lambda.
- **PE1-Q19:** callback-style human approval and feedback.
- **PE1-Q21:** AppConfig model selection, gradual rollout, alarm rollback.
- **PE1-Q27:** legacy webhook → API Gateway service integration → SQS.
- **PE1-Q34:** Amplify, OpenAPI, and Bedrock Flows.
- **PE1-Q36:** centralized gateway with CodePipeline/CodeBuild/CodeDeploy.
- **PE1-Q41:** Step Functions Parallel for independent model calls.
- **PE1-Q45:** geographic cross-Region inference.
- **PE1-Q47:** ConverseStream messages and SageMaker endpoint input schema.
- **PE1-Q52, Q56:** streaming versus synchronous/queued job API patterns.
- **PE1-Q59:** evaluation plus Lambda canary deployment.
- **PE1-Q63:** auditable Step Functions model route and fallback.
- **PE1-Q68, Q71:** model cascading and intelligent prompt routing.
- **PE1-Q70:** SDK client reuse and connection pooling.
- **PE1-Q75:** Outposts local processing and private Bedrock endpoint.

### Practice Exam 2

- **PE2-Q01:** HTTP API/Lambda proxy/Converse and infrastructure as code.
- **PE2-Q09:** cross-Region inference profile for same-model capacity.
- **PE2-Q10:** SQS Lambda consumer and CRM idempotency key.
- **PE2-Q17:** model-aware `InvokeModel` serializer.
- **PE2-Q27:** Amplify conversation route and Flow tracing.
- **PE2-Q31:** preserve Bedrock-native and documented compatibility API paths.
- **PE2-Q32:** CodeBuild test reports and CodePipeline eligible-stage rollback.
- **PE2-Q34:** same-account callback endpoint for cross-account reviewer portal.
- **PE2-Q35:** Outposts direct VPC routing, private endpoint, EU profile.
- **PE2-Q38:** AppConfig plus provider-normalizing adapter.
- **PE2-Q39:** structured sanitized logs plus X-Ray.
- **PE2-Q43:** managed prompt/condition sequence with Bedrock Flows.
- **PE2-Q50:** Amazon Q Developer IDE assistance and security scans.
- **PE2-Q53:** classifier plus Step Functions `Choice` and one generation model.
- **PE2-Q54:** task-specific output ceiling for token throughput.
- **PE2-Q56, Q61:** Converse versus native InvokeModel and capability checks.
- **PE2-Q62:** ConverseStream, latency optimization, TTFT monitoring.
- **PE2-Q66:** REST Lambda response streaming and SSE.

## Hands-on validation

1. Define an OpenAPI contract for synchronous and job-based endpoints with common errors.
2. Trace a request from API Gateway through Lambda to Bedrock with a safe correlation ID.
3. Build an SQS job design with idempotency, DLQ, status lookup, and authorization.
4. Design an EventBridge webhook flow with schema versioning and two consumers.
5. Demonstrate Converse and model-aware InvokeModel request adapters.
6. Demonstrate REST/SSE streaming and explain retry behavior before and after first byte.
7. Create a release manifest covering code, prompt, model, Guardrail, tools, routing, and evaluation dataset.
8. Design CodePipeline stages with CodeBuild reports, approval, canary, alarms, and eligible rollback.
9. Draw the exact Outposts-to-Bedrock private routing path and list data-residency checks.

## Recall questions

1. When does SQS beat EventBridge, and when does EventBridge beat SQS?
2. Why is an SQS-triggered Lambda still required to be idempotent?
3. What is SNS responsible for in a human-review design?
4. Which API Gateway type best fits a complete response, SSE, or bidirectional session?
5. What error behavior changes after streaming bytes reach a client?
6. Which Bedrock APIs use a common message contract?
7. When is `InvokeModel` the correct choice?
8. Why can an imported model ignore generation settings?
9. What should token admission calculate?
10. Which errors should not be retried unchanged?
11. What belongs in a centralized GenAI gateway?
12. How do AppConfig routing and Step Functions routing differ?
13. Why must a fallback model be prequalified?
14. What is the difference between a geographic and global inference profile?
15. When is Outposts preferable to Wavelength?
16. What makes Bedrock Flows preferable to Step Functions in a scenario?
17. What must be included in a GenAI release manifest?
18. What can CodeBuild reports prove, and what can they not prove?
19. What are the limits of CodePipeline rollback?
20. Why does a canary need semantic quality signals, not only HTTP errors?
21. What can Amazon Q Developer accelerate, and what evidence is still required?

## Official sources

- [AIP-C01 Domain 2](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain2.html)
- [Making Amazon Bedrock inference requests](https://docs.aws.amazon.com/bedrock/latest/userguide/inference.html)
- [Bedrock Runtime and Mantle endpoints](https://docs.aws.amazon.com/bedrock/latest/userguide/endpoints.html)
- [Bedrock model-specific inference parameters](https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters.html)
- [Bedrock model availability and API compatibility](https://docs.aws.amazon.com/bedrock/latest/userguide/models.html)
- [Bedrock inference profiles](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles.html)
- [Bedrock cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html)
- [API Gateway Lambda response streaming](https://docs.aws.amazon.com/apigateway/latest/developerguide/response-streaming-lambda-configure.html)
- [Step Functions error handling](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-error-handling.html)
- [Bedrock Flow node types](https://docs.aws.amazon.com/bedrock/latest/userguide/flows-nodes.html)
- [Bedrock Flow tracing](https://docs.aws.amazon.com/bedrock/latest/userguide/flows-trace.html)
- [Amplify AI architecture and routes](https://docs.amplify.aws/react/ai/concepts/architecture/)
- [Amplify AI conversation UI](https://docs.amplify.aws/react/frontend/ai/conversation/ai-conversation/)
- [Amplify conversation history and owner access](https://docs.amplify.aws/react/frontend/ai/conversation/history/)
- [CodePipeline stage rollback](https://docs.aws.amazon.com/codepipeline/latest/userguide/stage-rollback.html)
- [CodeBuild test reports](https://docs.aws.amazon.com/codebuild/latest/userguide/report-create.html)
- [CodeDeploy Lambda deployments and rollback](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-steps-lambda.html)
- [AWS AppConfig deployment monitoring and rollback](https://docs.aws.amazon.com/appconfig/latest/userguide/monitoring-deployments.html)
- [AWS Outposts local gateway routing](https://docs.aws.amazon.com/outposts/latest/userguide/routing.html)
- [Amazon Q Developer in IDEs](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/q-in-IDE.html)
- [Amazon Q Developer project rules](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/context-project-rules.html)
- [Amazon Q Developer code reviews](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/code-reviews.html)
