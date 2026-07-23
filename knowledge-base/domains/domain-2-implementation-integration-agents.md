# Domain 2 — Implementation and Integration

Status: Verified  
Official weight: 26%  
Official tasks: 2.1–2.5  
Last verified: 2026-07-23

## Why this domain matters

Domain 2 tests whether you can turn a foundation model into a production application. A correct answer normally combines an FM with deterministic software controls, tools, state, APIs, queues, workflows, deployment automation, and observability.

The recurring professional-level rule is:

> Let the model interpret, classify, plan, and generate. Let deterministic services authorize, validate, branch, retry, stop, approve, write, and audit.

This page is the exam-shaped map. Detailed explanations live in:

- [Agents, tools, MCP, and AgentCore](../concepts/agents-tools-mcp-and-agentcore.md)
- [Enterprise integration and CI/CD](../concepts/enterprise-integration-and-cicd.md)
- [SageMaker custom-model deployment](../concepts/sagemaker-custom-model-deployment.md)
- [Bedrock model selection and runtime APIs](../concepts/bedrock-model-selection-and-runtime-apis.md)

## Official coverage map

The official Domain 2 page contains 25 skills. Use this table as a completeness checklist.

| Skill | What you must be able to design or implement | Canonical coverage |
|---|---|---|
| 2.1.1 | Autonomous systems with session state, durable memory, specialization, MCP, Strands Agents, and Agent Squad | Agent loop; memory and state; multi-agent patterns |
| 2.1.2 | Structured multi-step problem solving | ReAct-style workflow; Step Functions Task and Choice states |
| 2.1.3 | Safeguarded workflows | Iteration limits, timeouts, least privilege, risk branches, circuit breakers |
| 2.1.4 | Model coordination | Specialized models, parallel calls, ensembles, aggregation, routing |
| 2.1.5 | Human collaboration | Review, edit, approval, feedback, callback task tokens |
| 2.1.6 | Reliable tool integration | Typed schemas, validation, structured errors, idempotency, Strands tools |
| 2.1.7 | Model extension frameworks | MCP client/server contract; Lambda, ECS/Fargate, and AgentCore hosting |
| 2.2.1 | FM deployment by workload | Bedrock on-demand, Provisioned Throughput, batch, SageMaker endpoints |
| 2.2.2 | LLM-specific deployment | GPU memory, LMI containers, model loading, storage, startup timeouts |
| 2.2.3 | Optimized deployment | Smaller models, model cascading, routing, cost-to-quality |
| 2.3.1 | Enterprise connectivity | APIs, events, queues, data synchronization, legacy adapters |
| 2.3.2 | Existing-application enhancement | API Gateway, Lambda webhooks, EventBridge integration |
| 2.3.3 | Secure access | Federation, RBAC/ABAC where appropriate, least privilege, private access |
| 2.3.4 | Cross-environment design | Outposts, Wavelength recognition, private hybrid routing, residency |
| 2.3.5 | CI/CD and GenAI gateways | CodePipeline, CodeBuild, tests, rollback, central policy enforcement |
| 2.4.1 | Flexible FM interaction | Bedrock Runtime, SDKs, synchronous and queued asynchronous requests |
| 2.4.2 | Real-time interaction | ConverseStream/native streaming, SSE, WebSocket, chunked delivery |
| 2.4.3 | Resilience | Backoff with jitter, throttling, fallback, degradation, X-Ray |
| 2.4.4 | Intelligent routing | Static config, classification, Step Functions routing, prompt routers |
| 2.5.1 | GenAI API interfaces | Validation, token limits, timeouts, streaming, retry boundaries |
| 2.5.2 | Accessible interfaces | Amplify, OpenAPI, Bedrock Flows |
| 2.5.3 | Business enhancements | CRM adapters, document workflows, Bedrock Data Automation |
| 2.5.4 | Developer productivity | Amazon Q Developer for AWS/API help, code, refactoring, and scanning |
| 2.5.5 | Advanced applications | Strands, multi-agent systems, Step Functions, prompt chaining |
| 2.5.6 | Troubleshooting efficiency | Structured logs, Logs Insights, X-Ray, traces, Amazon Q Developer |

## Task 2.1 — Agentic AI and tool integrations

### The production agent loop

An agent is not merely an FM with a long prompt. A controlled loop has:

1. Goal and policy instructions.
2. Current user input and authorized context.
3. Session state and, optionally, durable memory.
4. A bounded set of typed tools.
5. A planning or reason-act-observe cycle.
6. A deterministic stop policy.
7. A final response or a safe terminal outcome.

Use the FM for probabilistic selection and generation. Keep these outside the FM:

- authorization;
- input and output validation;
- loop and time limits;
- approval requirements;
- idempotency and transaction boundaries;
- circuit-breaker state;
- irreversible action execution;
- audit records.

Do not confuse an observable plan, tool trace, or Step Functions history with provider-internal chain of thought. The exam guide mentions structured reasoning, but production systems should expose concise decisions, tool inputs/outputs, and workflow state—not hidden model reasoning.

### Step Functions agent pattern

Use a **Standard** workflow when an agent process needs durable execution, an inspectable history, long waits, or human approval.

```text
Validate request
  → Load state and risk policy
  → Invoke model for next action
  → Choice
      ├─ answer → validate output → Succeed
      ├─ approved tool → invoke adapter → record observation → increment cycle → loop
      ├─ approval required → callback task token → continue or Fail
      └─ unsafe / max cycles / open circuit → Fail or safe response
```

Configure retries only for transient failures. Use `Catch` for deterministic fallback or compensation. A retry without a maximum can multiply tool calls and tokens.

### State and memory

| Need | Preferred pattern |
|---|---|
| State inside one deterministic workflow | Step Functions state plus external payload storage when large |
| Application session, slots, status, job result, custom TTL | DynamoDB |
| Short-term agent conversation plus extracted long-term preferences | AgentCore Memory |
| Ephemeral runtime context within one isolated AgentCore session | AgentCore Runtime session |
| Simple short-lived Lambda chat | Read/write recent turns in DynamoDB; summarize old history |

Never use a browser-provided session ID as authorization proof. The backend must bind an authenticated actor to an allowed session.

### Tools and MCP

A reliable tool definition contains:

- a unique, action-oriented name and unambiguous description;
- a strict JSON input schema with required fields, types, ranges, and enums;
- a stable output envelope;
- a declared timeout and retry policy;
- an idempotency key for side effects;
- structured errors such as `VALIDATION_ERROR`, `NOT_FOUND`, `THROTTLED`, and `DEPENDENCY_UNAVAILABLE`;
- a least-privilege execution role;
- sanitized logs and a correlation ID.

MCP standardizes how a compatible client discovers and invokes tools, prompts, and resources. It does not make a tool safe, authorized, durable, or idempotent.

| Workload | Likely host |
|---|---|
| Lightweight, short, stateless adapter | Lambda, commonly exposed through AgentCore Gateway |
| CPU-heavy/native dependencies/longer process | ECS with Fargate |
| Stateful streamable-HTTP MCP, long-running agent, session isolation | AgentCore Runtime |
| Existing OpenAPI, Smithy, Lambda, or MCP estate needing one governed endpoint | AgentCore Gateway |

For the current AgentCore Runtime MCP contract, a container listens on `0.0.0.0:8000/mcp`. Stateless streamable HTTP is the simpler default; stateful mode is for MCP interactions that require elicitation, sampling, progress, or session continuity.

### Human review

Use a callback task token when a review can take minutes, hours, or days:

1. Step Functions creates the token and pauses.
2. Store draft and status in DynamoDB or another system of record.
3. Notify the reviewer through the approved channel.
4. The reviewer calls an API in the workflow account.
5. Lambda validates identity, decision, and token ownership, then calls `SendTaskSuccess` or `SendTaskFailure`.
6. Record edited output, rating, reviewer, timestamp, and disposition.

SNS can notify; it does not itself implement a durable approval state.

### Model coordination

| Requirement | Pattern |
|---|---|
| Two independent artifacts and lowest end-to-end latency | Step Functions `Parallel`, then deterministic merge |
| One request should use one specialist | Classify, `Choice`, invoke one model |
| Routine versus complex prompts with low custom logic | Bedrock intelligent prompt routing, if supported candidates fit |
| Cross-provider or custom routing rules | Stable API + provider adapter + AppConfig |
| Multiple specialists must collaborate | Supervisor/router plus specialized agents; bound delegation depth |
| Several outputs must be reconciled | Explicit aggregation rule and evaluation; do not assume voting is correct |

Every extra model or agent adds token cost, latency, failure modes, and an evaluation burden. Use a multi-agent system only when specialization or parallelism beats a simpler single-agent/tool design on measured outcomes.

## Task 2.2 — Model deployment strategies

### Primary deployment decision

| Requirement | Default answer | Why the nearest distractor is weaker |
|---|---|---|
| Bursty interactive use of a managed FM | Bedrock on-demand | Provisioned capacity can waste money during idle periods |
| Predictable sustained synchronous demand | Bedrock Provisioned Throughput | Lambda concurrency does not create Bedrock model capacity |
| Large offline managed-FM workload | Bedrock batch inference | One Lambda invocation per record creates throttling and orchestration overhead |
| Same on-demand model, temporary Regional capacity pressure | Cross-Region inference profile | Client-built failover adds routing logic; confirm residency first |
| Custom/open-weight interactive model | SageMaker real-time endpoint | Bedrock base-model invocation does not host arbitrary containers |
| Offline supported Bedrock customized model | Bedrock batch inference when the current model/Region table lists custom-model support | One synchronous invocation per record adds avoidable orchestration |
| Offline arbitrary custom model/container over S3 data | SageMaker Batch Transform | A permanent endpoint pays for idle capacity |
| Very large LLM on SageMaker | LMI container plus correct GPU/storage/loading configuration | A generic container without large-model serving support is operationally harder |

For current details, use [SageMaker custom-model deployment](../concepts/sagemaker-custom-model-deployment.md).

### Cost-to-quality deployment

Do not choose the smallest model blindly. Establish a representative benchmark, then compare:

- correctness, task completion, safety, and format compliance;
- input/output tokens;
- latency and time to first token;
- throughput and throttling;
- infrastructure or model-unit utilization;
- failure and fallback rates.

Model cascading routes common easy requests to a smaller model and escalates only qualifying requests. A fallback is for failure; an escalation is for required quality. Record which occurred.

## Task 2.3 — Enterprise integration

### Event and API selection

| Requirement | Service or pattern |
|---|---|
| Synchronous HTTPS request and complete response | API Gateway → Lambda/service → Bedrock Runtime |
| Immediate acknowledgement, one durable work path | API Gateway service integration or Lambda → SQS → worker |
| One event, multiple present/future consumers | EventBridge rules |
| Fanout notification | SNS |
| Long, branch-heavy, auditable workflow | Step Functions Standard |
| Store job/session/circuit state with TTL | DynamoDB |
| Legacy provider sends signed webhook | API Gateway → signature-validating Lambda → queue/event bus |

At-least-once delivery is normal. Consumers must be idempotent. Use a stable business identifier—not a new random ID created on every retry—as the idempotency key.

### Central GenAI gateway

A gateway can centralize:

- authentication and authorization;
- request schema validation and size limits;
- tenant and token budgets;
- approved model and prompt routing;
- mandatory Guardrail application;
- provider-specific request/response normalization;
- throttling, retry policy, and graceful fallback;
- sanitized logging, tracing, and attribution;
- versioned configuration and controlled rollout.

A gateway is not automatically the answer. Direct Bedrock calls are simpler when a single trusted application needs no shared policy or compatibility layer.

### Hybrid and residency

Outposts can run preprocessing next to on-premises data; use supported private routing into a VPC and a Bedrock Runtime interface endpoint. Only approved/de-identified content should leave the local environment. A geographic cross-Region inference profile can constrain processing to its documented geography. Do not use a global profile when jurisdiction limits destination Regions.

AWS Wavelength is a recognition-level blueprint topic for edge applications that need ultra-low-latency access near carrier networks. It is not a generic solution for keeping records in a hospital or factory data center; Outposts is the closer fit there.

## Task 2.4 — FM API integrations

### Bedrock endpoint and API contract

Use the Regional **Bedrock Runtime (`bedrock-runtime`)** data-plane endpoint for Bedrock-native `Converse` and `InvokeModel` inference. Use **Bedrock Mantle (`bedrock-mantle`)** for its supported OpenAI-compatible or Anthropic-compatible APIs and capabilities. Current AWS guidance recommends Mantle for new compatible applications, but the endpoint families do not have identical model or feature support and can coexist. Do not send runtime requests to the Bedrock control-plane endpoint.

| API | Use | Request contract | Main distractor |
|---|---|---|---|
| `Converse` | Synchronous message-style interaction across supported models | Common `messages`/content blocks; system and inference config; optional `additionalModelRequestFields` | A single provider-native body sent to every `InvokeModel` model |
| `ConverseStream` | Common message-style streaming | Same conversation structure; consume typed stream events | `Converse` when partial output is required |
| `InvokeModel` | Specialized modality or exact native model interface | `application/json`; body must match selected model ID | Assuming unsupported parameters are translated automatically |
| `InvokeModelWithResponseStream` | Native-schema streaming where supported | Provider/model-specific body and streamed response | Enabling streaming without checking model support |
| `CountTokens` | Preflight token estimate for the actual model/request shape | Input must match the chosen invocation style | Counting characters or words |
| `GetFoundationModel` | Inspect model capabilities such as streaming support | Control-plane metadata request | Trial-and-error in production |

With Converse, provider-specific options belong in supported `additionalModelRequestFields`. With `InvokeModel`, build a model-aware serializer. Customized/imported models can have a different native schema; unsupported fields may be ignored or rejected.

### Streaming transport

Streaming reduces **perceived latency** and time to first token; it does not guarantee lower total generation time.

| Client contract | Pattern |
|---|---|
| Existing REST client supports SSE | API Gateway REST API response streaming → Lambda → `ConverseStream` → SSE frames |
| Bidirectional, long-lived interaction | API Gateway WebSocket API |
| Simple server-to-browser incremental output | SSE or supported response streaming |
| Client can wait for full result | Synchronous HTTP API/REST API |

Handle client disconnects, partial outputs, backpressure, maximum duration, and final/error events. Do not retry an entire streamed response after bytes have reached the client unless the protocol can identify and deduplicate replayed content.

### Resilience and routing

- Let AWS SDK retry policies handle retryable throttling and transient service failures with exponential backoff and jitter.
- Do not retry validation, authorization, or unsupported-model errors unchanged.
- Apply an end-to-end deadline; each nested timeout must fit inside it.
- Bound retries and use a fallback only if it still satisfies safety, Region, feature, and quality constraints.
- Correlate API Gateway, Lambda/service, model call, queue message, and workflow execution with one safe identifier.
- Use X-Ray for service-path timing and structured CloudWatch logs for request context.
- Reduce an oversized `maxTokens`; token-based capacity can be reserved or limited by configured output ceilings even when typical answers are short.

## Task 2.5 — Application patterns and developer tools

### Frontend, contracts, and managed flows

- **Amplify**: rapid authenticated web/mobile delivery, UI primitives, conversation routes, and owner-scoped data patterns. Verify the current Gen 2 feature and Region support.
- **OpenAPI**: versioned API-first contract, request/response models, generated clients, and API Gateway import.
- **Bedrock Flows**: managed graph of input, prompt, condition, Knowledge Base, and Lambda nodes; useful when prompt/workflow configuration changes more often than application code.
- **Step Functions**: stronger fit for general AWS orchestration, long-lived execution, callbacks, explicit retries, and audit history.
- **Bedrock Data Automation**: managed multimodal extraction and structured outputs for document-processing/business workflows.
- **Amazon Q Developer**: IDE assistance for AWS SDK use, code generation, refactoring, project guidance, tests, and security scanning. It does not replace code review, deterministic tests, or runtime observability.

### CI/CD release path

```text
Source
  → build and static/security checks
  → prompt, tool-schema, guardrail, and API-contract tests
  → model/RAG/agent regression evaluation
  → package infrastructure and immutable configuration
  → approval when risk requires it
  → canary/linear deployment
  → CloudWatch bake alarms
  → promote or automatic rollback
```

CodePipeline coordinates. CodeBuild runs tests and can publish reports. CodeDeploy can shift Lambda/ECS traffic and roll back based on deployment failure or alarms. CodePipeline also supports configured rollback of eligible non-source stages to a prior successful execution; do not claim that a source stage can be rolled back.

## Cross-cutting decision table

| If the requirement changes to… | Change the design to… |
|---|---|
| Approval may take five days | Standard workflow callback; not polling or a long Lambda wait |
| Existing REST client must remain REST and supports SSE | REST response streaming; not a forced WebSocket migration |
| One webhook must add future consumers | EventBridge; not a hard-coded list in the handler |
| Bursts must not couple producer to Bedrock | SQS buffer and return a job ID/acknowledgement |
| Same prompt contract across message models | Converse where supported |
| Image/embedding model needs native parameters | InvokeModel with exact model schema |
| Model changes without code deployment | AppConfig or another validated configuration source |
| Every route must be auditable | Step Functions execution history plus structured route outcome |
| Human edits must improve future evaluation | Persist draft, approved edit, rating, version metadata |
| Tool can create an external record | Idempotency key, authorization, approval if high risk, structured result |
| Data cannot leave an EU geography | Geographic profile and verified destinations; not global routing |
| Custom 70 GB open-weight LLM | SageMaker LMI, uncompressed model data, adequate EBS and timeouts |

## Failure modes and troubleshooting

| Symptom | Evidence to inspect first | Likely cause | Corrective action |
|---|---|---|---|
| Bedrock validation error | API, model ID, request body, role/content blocks | Wrong API shape or model schema | Use valid Converse structure or model-native serializer |
| Generation setting ignored | Model parameter documentation | Field unsupported or misplaced | Use documented native field or `additionalModelRequestFields` |
| Streaming unavailable | Model capability metadata | Selected model/API does not stream | Check capability before enabling; choose supported path |
| Duplicate external writes | Queue delivery and idempotency record | At-least-once delivery without stable key | Use business ID and conditional/idempotent write |
| Agent repeatedly calls tool | Trace, tool error, loop count | Ambiguous schema/error or no stop rule | Structured error, clarification path, max cycles, circuit breaker |
| Workflow cost spikes | Transitions, retries, token counts | Retry loop or over-decomposition | Bound retry/cycles and simplify orchestration |
| Callback never resumes | Token destination, account, timeout, IAM | Wrong token/account or lost reviewer state | Same-account callback endpoint, persist status, handle timeout |
| Bedrock throttling at predictable peaks | input/output tokens, `maxTokens`, concurrency | Insufficient throughput or oversized output ceiling | Right-size requests; Provisioned Throughput if sustained |
| Intermittent on-demand capacity | Region/profile metrics | Regional pressure | Cross-Region inference if residency allows |
| Lambda downstream latency | X-Ray subsegments, connection timing | New clients/connections per invocation | Reuse SDK/HTTP clients and keep-alive |
| Gateway release regresses | test reports, alarm state, deployment history | Missing quality gate or unsafe rollout | Regression gate, canary/linear shift, rollback |
| SageMaker endpoint startup fails | endpoint and container logs | Download/load exceeds storage or timeouts | LMI/model source, EBS, download and health-check settings |
| Batch GPU underused | records/request and concurrency | One-record calls | MultiRecord plus tuned payload and concurrency |
| Wrong tool handler via Gateway | tool name and target prefix | Dispatcher expects unprefixed name | Validate schema and normalize documented target-prefixed name |

## Common exam traps

1. **Agent autonomy is not authorization.** A model deciding to call a tool does not grant permission.
2. **MCP is a protocol, not a runtime.** Choose Lambda, ECS/Fargate, AgentCore Runtime, or another host separately.
3. **Memory is not workflow state.** Long-term preferences belong in purpose-built memory; payment status and approval state belong in an authoritative deterministic store.
4. **SNS is not a work queue.** Use SQS for durable buffered processing.
5. **EventBridge is not the default for one work path.** Its advantage is event routing and multiple independently evolving consumers.
6. **Streaming is not batch inference.** Streaming is incremental interactive output; batch is asynchronous offline throughput.
7. **Provisioned Throughput is not autoscaling Lambda.** It purchases model capacity.
8. **A global inference profile can violate residency.** Select a documented geographic profile when required.
9. **One `InvokeModel` JSON body is not portable across providers.** Native payloads differ.
10. **A Step Functions retry is not a circuit breaker.** A circuit breaker remembers dependency health across attempts/executions.
11. **Human review is not a Lambda sleep.** Use a callback or another durable approval mechanism.
12. **Multi-agent is not automatically better.** It increases cost, latency, and coordination risk.
13. **Prompt/agent traces are not hidden chain of thought.** Store observable execution events and concise rationales.
14. **CodePipeline rollback has constraints.** A source stage cannot be rolled back, and a target must be eligible under the current pipeline structure.
15. **Amazon Q Developer is assistance, not evidence that code is correct or secure.**

## Local mock references

These questions are learning evidence, not authoritative product documentation.

### Practice Exam 1

- **PE1-Q02, Q20:** MCP abstraction; Lambda versus ECS/Fargate tool hosting.
- **PE1-Q03, Q27:** webhook acknowledgement; EventBridge fanout versus SQS buffering.
- **PE1-Q08, Q50:** Step Functions reason-act loop, stopping conditions, circuit breaker, IAM.
- **PE1-Q16, Q47, Q52:** Bedrock request structure, streaming, token admission, retry.
- **PE1-Q18, Q26, Q60:** batch inference versus Provisioned Throughput.
- **PE1-Q19:** long human review and feedback persistence.
- **PE1-Q21, Q36, Q59:** AppConfig, gateway CI/CD, canary and rollback.
- **PE1-Q34:** Amplify, OpenAPI, and Bedrock Flows.
- **PE1-Q41, Q63, Q68, Q71:** parallel models, deterministic routing, cascading, intelligent prompt routing.
- **PE1-Q45:** geographic cross-Region inference.
- **PE1-Q48, Q53:** Strands tool schema; routed agents with AgentCore Runtime and Memory.
- **PE1-Q54, Q62:** SageMaker large-model startup and governed LoRA releases.
- **PE1-Q66:** agent evaluation and repeated-tool-call analysis.
- **PE1-Q70:** SDK client reuse and downstream latency.
- **PE1-Q75:** Outposts preprocessing and private Bedrock access.

### Practice Exam 2

- **PE2-Q01, Q17, Q56, Q61:** Converse versus InvokeModel, endpoint, capability, and serialization.
- **PE2-Q02:** stateful streamable-HTTP MCP on AgentCore Runtime.
- **PE2-Q09, Q35:** cross-Region inference, hybrid routing, and residency.
- **PE2-Q10:** SQS-triggered work with an idempotent CRM write.
- **PE2-Q11:** `InvokeAgent` trace versus final response.
- **PE2-Q14, Q25:** Batch Transform tuning and LMI deployment.
- **PE2-Q27, Q43:** Amplify and Bedrock Flows.
- **PE2-Q31, Q38:** API compatibility and cross-provider gateway routing.
- **PE2-Q32:** CodePipeline/CodeBuild reports and eligible-stage rollback.
- **PE2-Q34, Q40:** Standard workflow callback and safe multi-day reasoning loop.
- **PE2-Q39:** structured, sanitized logs plus X-Ray.
- **PE2-Q41:** caller-initiated AgentCore evaluation using session spans.
- **PE2-Q48, Q70:** AgentCore Gateway tool schemas, target names, and managed Knowledge Base tools.
- **PE2-Q50:** Amazon Q Developer.
- **PE2-Q53, Q54:** auditable model routing and token-based throughput.
- **PE2-Q60, Q65, Q71:** bounded agents, AgentCore Memory, and deterministic session workflows.
- **PE2-Q62, Q66:** ConverseStream and REST/SSE response streaming.

## Hands-on validation

Build or diagram these before declaring Domain 2 ready:

1. A synchronous `Converse` endpoint with schema validation, a correlation ID, token preflight, and structured errors.
2. A streaming path that distinguishes first-token latency, completion, client disconnect, and error events.
3. A Step Functions agent loop with `Choice`, `Retry`, `Catch`, maximum cycles, a risky-tool `Fail` state, and a callback approval.
4. Two tools with the same schema contract: one stateless/lightweight and one long-running; explain the hosting decision.
5. A bursty webhook path using SQS and an event-fanout path using EventBridge; implement idempotency reasoning for both.
6. A CI/CD diagram with security, prompt, guardrail, tool-contract, and quality tests plus a rollback trigger.
7. A SageMaker large-model deployment checklist and a Batch Transform tuning experiment.

## Recall questions

1. Which decisions must never be delegated solely to an FM?
2. When is AgentCore Memory a better fit than DynamoDB, and when is it not?
3. How do Step Functions `Retry`, `Catch`, a stopping condition, and a circuit breaker differ?
4. Why can a callback task wait safely while a Lambda sleep cannot?
5. What makes a tool call idempotent across an SQS redelivery?
6. What does MCP standardize, and what responsibilities remain with the application?
7. Why would a stateful MCP server be required for elicitation or progress?
8. When should a tool run on Lambda, ECS/Fargate, or AgentCore Runtime?
9. How do `Converse` and `InvokeModel` request contracts differ?
10. What must be checked before enabling streaming for a model?
11. Why might reducing `maxTokens` increase completed requests per minute?
12. When is WebSocket preferable to SSE, and when is REST response streaming preferable?
13. How do SQS, EventBridge, and SNS differ in a webhook architecture?
14. What must a centralized GenAI gateway enforce?
15. How do AppConfig rollback and code deployment rollback solve different problems?
16. When is Provisioned Throughput preferable to on-demand or batch inference?
17. Why is a geographic profile safer than a global profile for residency?
18. What LLM-specific issues make a SageMaker endpoint fail before serving traffic?
19. Why should multi-agent performance be measured against a simpler baseline?
20. Which evidence would you collect to diagnose a repeated-tool-call loop?

## Official sources

- [AIP-C01 Domain 2 task and skill statements](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain2.html)
- [Making inference requests in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/inference.html)
- [Inference request parameters by model](https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters.html)
- [Bedrock batch inference](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html)
- [Bedrock Provisioned Throughput](https://docs.aws.amazon.com/bedrock/latest/userguide/prov-throughput.html)
- [Bedrock cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html)
- [Bedrock intelligent prompt routing](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-routing.html)
- [Step Functions error handling](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-error-handling.html)
- [Step Functions callback task-token pattern](https://docs.aws.amazon.com/step-functions/latest/dg/callback-task-sample-sqs.html)
- [AgentCore developer guide](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/)
- [API Gateway Lambda response streaming](https://docs.aws.amazon.com/apigateway/latest/developerguide/response-streaming-lambda-configure.html)
- [CodePipeline stage rollback](https://docs.aws.amazon.com/codepipeline/latest/userguide/stage-rollback.html)
- [CodeDeploy deployment configurations](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-configurations.html)
- [SageMaker deployment guardrails](https://docs.aws.amazon.com/sagemaker/latest/dg/deployment-guardrails.html)
