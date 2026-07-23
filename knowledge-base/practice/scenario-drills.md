# Scenario Drills

Solve each scenario before reading the answer key. State:

1. Architecture.
2. Why it satisfies every constraint.
3. Why the strongest alternative is inferior.
4. Metrics and failure controls.

## Scenarios

### 1. Variable token capacity

A multi-tenant policy assistant receives short questions and very long case summaries. Request count is stable, but tenants intermittently receive throttling. Each tenant has a contractual per-minute capacity allocation. The allocation check must not itself incur model inference cost.

Design admission control.

### 2. Exact identifiers and symptoms

An operations RAG assistant retrieves incident runbooks. Engineers query both exact error codes and natural-language symptoms. Exact matches are frequently absent from the top results, although the correct chunks exist.

Improve retrieval without replacing the vector store.

### 3. Slow first response

A browser chat uses a large stable system policy, dynamic retrieved evidence, and a user question. Users wait too long before seeing text. Total answer quality is acceptable and the model must remain unchanged.

Reduce perceived latency and repeated processing.

### 4. Regulated agent loop

An agent plans restoration actions and calls read-only operational tools. It sometimes repeats a failing tool call. High-risk observations must end the workflow. Some plans require an engineer’s approval that can take two days.

Design the orchestration and permissions.

### 5. Sensitive case assistant

Case documents in S3 and staff questions contain identifiers and bank data. Raw values must not reach the FM or logs. Answers should retain enough entity relationships to be useful, and all artifacts must expire after 30 days.

Design privacy controls.

### 6. Prompt release regression

Three applications use one standardized prompt. A proposed version improves tone but sometimes omits a mandatory JSON field. The risk team requires approval evidence and rapid rollback.

Design the release path.

### 7. Wrong jurisdiction

A legal RAG assistant cites documents from the wrong jurisdiction after a migration. The team is unsure whether retrieval or generation is responsible.

Describe the first diagnostic sequence.

### 8. Predictable peak

A same-Region compliance summarizer uses one required Bedrock model. Every weekday at a known time, traffic increases tenfold and throttles. Requests are interactive.

Choose a capacity pattern and monitoring.

### 9. Offline transcript volume

Millions of transcripts arrive nightly in S3. Summaries are needed by morning and can be stored back in S3. Individual synchronous calls frequently throttle.

Choose the inference pattern.

### 10. Legacy webhook

An on-premises application can only send HTTPS and needs acknowledgment within two seconds. Events arrive in bursts. FM output later updates a CRM that supports idempotency keys.

Design the integration.

### 11. Multi-provider gateway

Internal clients have one request contract. The platform must route among two Bedrock models and an approved external provider by tenant, content type, and outage status without client redeployment.

Design the gateway.

### 12. Stateful MCP tool

An MCP tool runs a long simulation, emits progress, and may ask for missing parameters during the same interaction. The platform wants a managed agent runtime and portable clients.

Choose the runtime/interface characteristics.

### 13. Fairness concern

An HR-writing assistant may change tone for demographic cohorts in otherwise equivalent cases. The model and two prompt versions are candidates.

Design an evaluation that can expose disparity.

### 14. Large custom LLM startup

A 70 GB open-weight LLM on a SageMaker real-time endpoint fails its health check while still downloading and loading. The instance’s initial storage is too small.

Identify the complete deployment correction.

### 15. Enterprise knowledge assistant

Employees need one assistant over SharePoint, S3, and Jira. The organization already uses IAM Identity Center. Source-system permissions must be honored with minimal custom authorization code, and approved actions are required.

Choose between a custom Bedrock RAG app and an enterprise assistant service.

### 16. Deterministic account answer

A chatbot must answer “What are my last five transactions?” and must never fabricate amounts or run write SQL. It must also block unsafe language.

Separate the FM and deterministic responsibilities.

### 17. Source freshness

Policy objects are replaced and deleted throughout the day. The KB occasionally answers from obsolete versions. Re-embedding unchanged documents should be minimized.

Design synchronization and completion tracking.

### 18. Observability without PHI

An API Gateway/Lambda/Bedrock note assistant returns wrong output, validation errors, and latency spikes. Logs contain only strings, and raw PHI cannot be retained.

Design the evidence model.

### 19. Model comparison

Three models and two prompts must be compared on quality, human preference, tokens, latency, and real production behavior before final selection.

Design the evaluation sequence.

### 20. Private governed data

A Lambda in AWS reads Athena tables over S3 and invokes Bedrock. No traffic may use public internet paths; application teams may read only approved columns.

Design network and data authorization.

## Answer key

### 1

Call `CountTokens` with the selected model and assembled request, add the output budget, and admit through a per-tenant token bucket. Queue/defer requests that exceed the budget. Track admitted/rejected tokens and throttles.

Why not request rate: prompt sizes differ, so it allocates the wrong resource.

### 2

Use hybrid keyword/vector search. Extract exact codes into keyword clauses or metadata filters, retrieve a sufficient candidate set, then use a Bedrock reranker if ordering remains weak. Measure top-k relevance and latency.

### 3

Use prompt caching for the eligible stable prefix, keep dynamic retrieval/user text after the checkpoint, stream with `ConverseStream` through the client-compatible response path, and monitor TTFT plus total latency. Also remove redundant retrieved chunks.

### 4

Use a Step Functions Standard state machine with Task/Choice/Fail states, max cycles, bounded retries, a DynamoDB-backed circuit breaker, and callback task token for approval. Give each Lambda tool a role limited to approved read APIs. Persist execution/agent evidence.

### 5

Use Macie for S3 discovery, preprocess live text with Comprehend PII detection and consistent placeholders, apply Guardrails to input/output, avoid raw invocation logging or store only sanitized fields, encrypt/restrict logs, and apply S3 Lifecycle expiry.

### 6

Store/version the prompt in Prompt Management, run schema validation and Bedrock evaluation on a fixed dataset, require a pipeline/manual approval, deploy by canary/staged configuration, alarm on invalid JSON/quality, and roll back to the prior prompt version.

### 7

Call `Retrieve` for a fixed prompt set and inspect content, jurisdiction metadata, source location, rank/score, and source/index version. Confirm ingestion completion. Fix metadata/filtering/ranking before changing the generation prompt.

### 8

Use Bedrock Provisioned Throughput sized from token and request characteristics, invoke the provisioned model ARN, and monitor utilization, latency, throttles, errors, and input/output tokens.

### 9

Use Bedrock batch inference with S3 input/output. Validate input, submit/monitor the job, and quarantine failures. It avoids one synchronous request per transcript.

### 10

API Gateway validates/authenticates and writes to SQS, returning immediately. Lambda workers consume messages, invoke Bedrock, and update the CRM with the event/interaction ID as idempotency key. Use DLQ and bounded retries.

### 11

API Gateway fronts a Lambda/provider adapter with a stable contract. AppConfig stores validated providers, model IDs, routing rules, and failover flags with staged rollout/rollback. Log selected route and version. Use SFN if multi-step audit/fallback requires workflow history.

### 12

Use a stateful streamable-HTTP MCP server on AgentCore Runtime (subject to current supported contract), official MCP clients, progress messages, and elicitation/state handling. A short stateless Lambda MCP server does not fit the long interactive tool.

### 13

Create matched/stratified prompts across cohorts, run both prompt versions with the same model/settings, use a calibrated rubric/LLM judge plus human sample, report tone/helpfulness/outcome/error metrics per cohort, and gate on maximum disparity.

### 14

Use a SageMaker LMI container, an appropriate uncompressed `ModelDataSource`/S3 prefix when supported, sufficient EBS volume, sufficient GPU/memory topology, increased model-download and startup-health timeouts, and deployment logs/alarms. Timeout alone cannot fix insufficient storage.

### 15

Prefer Amazon Q Business with IAM Identity Center, supported connectors, user/group assignment, source-aware permissions, and plugins/actions. A custom Bedrock RAG app would require more custom ACL propagation and application code.

### 16

Use the FM only to classify intent or map to an approved parameterized read-only query/template. Application code authorizes the user, executes the query, and formats amounts from the result. Guardrails filter unsafe input/output. Never let free-form generated SQL execute.

### 17

Use source change events or a scheduled refresh to start KB ingestion. Track the job to terminal success and alarm on document/chunk failures. Preserve S3 as source of truth and avoid marking the update complete early.

### 18

Emit sanitized JSON logs with correlation ID, model/prompt/guardrail versions, safe request attributes, tokens, validation result, and latency. Enable X-Ray across API Gateway/Lambda/SDK segments. Use CloudWatch Logs Insights and metrics; do not log raw PHI.

### 19

Run every model/prompt pair on one versioned benchmark, combine deterministic/LLM-judge metrics with human preference, collect CountTokens and latency/cost, select viable candidates, canary a limited cohort, collect live feedback, and promote or roll back.

### 20

Put Lambda in private subnets; use interface VPC endpoints for Bedrock Runtime, Athena, and Glue where supported plus an S3 gateway endpoint. Apply endpoint/resource policies, least-privilege IAM, and Lake Formation column permissions. Send sanitized audit events/metrics to CloudWatch.

