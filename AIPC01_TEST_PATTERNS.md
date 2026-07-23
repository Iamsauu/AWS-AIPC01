# AWS AIP-C01 Practice-Test Patterns

Derived from the 150 questions in the two local Udemy practice exams. This is a
study map for this question bank, not proof of the official exam's current
weighting.

## 1. The master exam pattern

Translate requirement words into architecture constraints before comparing
answers:

| Requirement wording | Default pattern to consider |
|---|---|
| Least operational overhead | Prefer a managed, native AWS feature over custom code or self-managed infrastructure |
| Serverless, short, stateless | API Gateway + Lambda |
| Bursty work, one durable processing path | SQS + Lambda |
| One event, several current or future consumers | EventBridge rules |
| Immediate acknowledgement, slow work later | Accept request, enqueue/publish, process asynchronously |
| Branches, retries, timeouts, execution history | Step Functions |
| Human approval that may take hours | Step Functions Standard callback with task token |
| Change configuration without redeploying | AppConfig |
| Gradual rollout and automatic rollback | Canary/linear deployment + CloudWatch alarms |
| Fixed recurring high inference demand | Provisioned Throughput |
| Offline high-volume inference | Bedrock batch inference |
| Same model, temporary regional capacity issue | Cross-Region inference profile |
| Faster visible response | Streaming; measure time to first token |
| Common static prompt prefix | Prompt caching |
| Long prompts or per-tenant budgets | CountTokens before invocation |
| Reusable governed prompts | Bedrock Prompt Management |
| Editable model workflow with nodes and branches | Bedrock Flows |
| General AWS orchestration with strong control | Step Functions |
| Ground answers in changing documents | Bedrock Knowledge Bases/RAG, not fine-tuning |
| Safety policy around FM input/output | Bedrock Guardrails |
| PII in free text before FM invocation | Comprehend PII detection/redaction |
| Sensitive data already stored in S3 | Macie discovery |
| Private service traffic | VPC endpoints/PrivateLink; S3 gateway endpoint |
| Data column restrictions | Lake Formation |
| API activity audit | CloudTrail |
| Application/model telemetry | CloudWatch Logs and metrics |
| Request path and latency tracing | X-Ray |

The recurring distractor is a solution that is possible but requires custom
code, manual work, polling, or infrastructure when a managed service directly
implements the requirement.

## 2. Bedrock runtime API selection

Master these distinctions:

- **Converse**: the common messages-based interface for supported chat models.
- **ConverseStream**: Converse plus streamed output.
- **InvokeModel**: use a provider/model-specific native request body; a shared
  body does not automatically work across models.
- **InvokeModelWithResponseStream**: native-model streaming when supported.
- Check model capabilities such as streaming, modality, tool use, Region,
  lifecycle, context window, and inference type before selection.
- `bedrock:InvokeModel` is the core permission used by both InvokeModel and
  Converse calls in these questions.
- Set `contentType` to `application/json` and serialize the documented native
  schema for each `modelId`, especially for imported/customized models.
- Use `additionalModelRequestFields` with Converse when a model needs supported
  provider-specific parameters.
- Use the regional `bedrock-runtime` endpoint for inference, not a control-plane
  endpoint.

API integration patterns:

- Simple complete-response HTTPS service: API Gateway HTTP API, Lambda proxy,
  payload format 2.0, then Converse.
- Streaming: ConverseStream plus WebSocket, SSE, or API Gateway/Lambda response
  streaming.
- Long-running request: return a job ID, put work on SQS, and store the result
  for later retrieval.
- Validate and throttle at API Gateway; validate again in Lambda where business
  rules require it.

## 3. Prompt engineering and prompt lifecycle

- Use **Prompt Management** for reusable templates, variables, versions,
  variants, approval, and traceable rollout.
- Put role, tone, instructions, examples, and a strict JSON response contract
  in the prompt; do not rely on prose such as “return JSON.”
- Keep safety enforcement outside the prompt by attaching a Guardrail.
- Correlate user feedback and production records with prompt version and model
  ID.
- Store source prompt artifacts in versioned S3/source control when governance
  requires a source of truth.
- Use **Bedrock Flows** when non-developers need to edit prompt, condition,
  knowledge-base, and Lambda nodes without application deployment.
- Use **Advanced Prompt Optimization** only when the scenario asks for managed
  optimization across evaluation samples and target models.
- Lower temperature and narrow top-p/top-k for more deterministic output, but
  prove improvement through evaluation rather than assuming it.
- Limit `maxTokens` by task and trim irrelevant context.

Prompt caching pattern:

- Cache the long, identical prefix: system instructions, policies, taxonomy, or
  other shared context.
- Put the cache checkpoint after that stable prefix.
- Keep user input and other changing content outside the cached prefix.
- Meet the selected model's minimum cacheable token count.

## 4. Responsible AI, security, and privacy

### Guardrails

Use Bedrock Guardrails for inline input/output enforcement:

- harmful-content filters;
- denied topics;
- word/profanity filters;
- prompt-attack detection;
- sensitive-information and custom-regex masking;
- contextual grounding/relevance checks;
- automated reasoning policies when a formal policy document must be enforced.

Important implementation details:

- Apply the guardrail to every relevant invocation.
- Pass both guardrail identifier and version.
- Use guard content tags when only part of an input should be assessed.
- Use IAM conditions requiring the approved Guardrail identifier when policy
  enforcement must be organization-wide.
- Fail closed: a Lambda wrapper should block output when mandatory deterministic
  checks, such as a required disclaimer or JSON schema, fail.

### Defense in depth

- **Comprehend**: detect/redact PII or extract entities from incoming free text
  before it reaches the FM.
- **Guardrails**: filter or mask model inputs and outputs during inference.
- **Macie**: discover sensitive data in S3; it is not the inline prompt filter.
- **S3 Lifecycle**: enforce retention/deletion of prompt, response, or audit
  artifacts.
- Disable model invocation logging when raw sensitive content must not be
  retained; store only sanitized audit metadata.
- Use adversarial prompt suites and scheduled/pipeline tests to test jailbreak
  resistance rather than trusting prompt instructions alone.

### Deterministic data-backed answers

- Do not let an FM execute arbitrary SQL.
- Map intent to predefined parameterized `SELECT`-only templates, or strictly
  validate generated read-only SQL.
- Execute through a least-privilege Lambda role.
- Build the final factual response only from the returned result set.

## 5. Evaluation and continuous quality

Treat evaluation as a lifecycle:

1. Build a representative S3 JSONL dataset.
2. Include prompts and `referenceResponses` when ground truth exists.
3. Compare candidate model, prompt, and inference-parameter combinations.
4. Gate CI/CD promotion on thresholds.
5. Canary the selected version in production.
6. Capture user feedback and production regression signals.

Know the evaluation types:

- **LLM-as-a-judge**: semantic qualities such as correctness, relevance,
  faithfulness, helpfulness, coherence, tone, bias, and citation coverage.
- **Human evaluation**: subjective or high-stakes comparison, including
  bring-your-own/precomputed responses.
- **Model evaluation**: generated response quality for prompts/models.
- **Retrieve-only RAG evaluation**: retrieval context relevance and coverage.
- **Retrieve-and-generate RAG evaluation**: end-to-end faithfulness and answer
  quality.
- **Agent/AgentCore evaluation**: task completion, correctness, and tool-use
  effectiveness.

Special cases:

- For an external model that must not be called during evaluation, supply one
  precomputed response per prompt.
- For fairness, stratify/balance the dataset and publish cohort-level results.
- For regression testing, combine prerelease Bedrock evaluations with
  CloudWatch Synthetics checks against production.
- Store evaluation outputs in S3; use Glue Catalog + Athena + QuickSight when
  stakeholders need longitudinal reporting.
- Do not choose a model only from public benchmarks. Compare quality with
  modality, tool/stream support, Region availability, lifecycle status,
  latency, throughput, and price.

## 6. RAG and Knowledge Bases

### Basic decision

- Use a **Knowledge Base** when documents change and answers must stay grounded;
  retraining is not required for content updates.
- Use **RetrieveAndGenerate** when Bedrock should retrieve and generate the
  answer.
- Use **Retrieve** when the application must inspect/merge chunks or control the
  final generation prompt itself.
- Return citation spans, retrieved references, document metadata, and locations
  when evidence must be visible.

### Chunking and query patterns

- **Hierarchical chunking**: retrieve precise child chunks, then return larger
  parent context.
- Narrow fact hidden in a long document: prefer smaller retrieval units.
- Multi-intent question: decompose into subqueries, retrieve for each, merge,
  then generate.
- Remove obsolete/duplicate documents and reduce redundant chunks before adding
  more model capacity.

### Retrieval quality

- Semantic-only search is weak for exact IDs and error codes.
- Use **hybrid search** for vector similarity plus keyword/identifier matching.
- Apply metadata filters before/with retrieval.
- Retrieve a wider candidate set and apply a Bedrock reranker when ranking
  quality is the problem.
- Inspect `retrievalResults` content, metadata, location, and score using a
  fixed test set.
- Evaluate retrieval separately before blaming the generator.

### Metadata and ingestion

- Use per-document `fileName.extension.metadata.json` sidecars.
- Keep sidecars under the supported size limit in the question bank (10 KB).
- Mark only semantically useful fields with `includeForEmbedding: true`.
- Keep revision date, owner, access control, and other filter-only fields out of
  embeddings.
- Standardize fields such as business unit, author, timestamp, and topic.
- On S3 create/overwrite/delete, trigger `StartIngestionJob`.
- Track ingestion until terminal success; starting a job does not prove the
  index is current.
- Monitor Knowledge Base retrieval/ingestion plus vector-store metrics and
  document/chunk logs.

### Data sources and access

- Knowledge Bases can use sources such as S3, Confluence, and SharePoint in
  these questions.
- Use AppFlow to copy Salesforce content into S3 when an S3 compliance copy is
  required.
- Preserve document-level permissions and pass user context during retrieval
  when results must respect source access.
- Amazon Q Business is the more direct pattern for an enterprise assistant with
  Identity Center, supported connectors, plugins, and user/group access.

### Embeddings and vector stores

- Validate embedding model, dimension, language/domain relevance, and storage
  cost empirically.
- Titan Text Embeddings V2 can use smaller dimensions when quality remains
  acceptable.
- Use multimodal embeddings when image and text must share a vector space.
- For OpenSearch Serverless with binary embeddings in the tested pattern, use a
  compatible `faiss` vector index with matching dimensions and mapped vector,
  text, and metadata fields.
- For many domains, use separate domain indexes and either hierarchical routing
  or fan-out/merge as the question requires.
- Prefer fewer, appropriately sized shards; excessive shards add k-NN
  coordination overhead.

## 7. Agents, tools, MCP, and memory

### Agents and traces

- Enable `InvokeAgent` tracing for per-request orchestration visibility.
- Persist trace events separately from the final `chunk.bytes` response.
- A trace explains knowledge-base selection, action-group calls, and failures;
  it is not provider-internal chain-of-thought.
- Return source attribution separately from the trace.

### Tool and workflow patterns

- Define strict typed tool schemas.
- Validate every parameter in the Lambda dispatcher.
- Return structured errors so an agent can ask for missing information or retry.
- Use least-privilege roles, short tool timeouts, bounded retries, maximum cycle
  counts, and explicit unsafe `Fail` states.
- Store circuit-breaker state with TTL in DynamoDB.
- Use Step Functions for an auditable reason-act loop when retries, timeout,
  branching, or human review must be explicit.

### MCP

- MCP supplies one stable tool contract across agent frameworks and backends.
- Short stateless MCP tool: Lambda is suitable.
- CPU-heavy, native-library, or long-running MCP server: ECS Fargate.
- Stateful streamable-HTTP MCP workload: AgentCore Runtime with the required MCP
  endpoint contract.
- A retrieval MCP server can hide OpenSearch/Aurora differences behind one
  `vector_search` tool.

### AgentCore

- **Runtime**: host routed/long-running agent or MCP workloads.
- **Gateway**: expose Lambda tools or managed Knowledge Base connectors with
  explicit tool schemas; handle target-prefixed tool names.
- **Memory**: persist short- and long-term/user-preference memory with clear
  `actorId` and per-chat `sessionId`.
- Use DynamoDB yourself when the question specifically asks for application
  session records, slots, TTL history, or custom access patterns.

## 8. Orchestration and event architecture

- **SQS**: buffer bursty work, decouple one processing path, retry delivery, and
  acknowledge the producer immediately.
- **EventBridge**: publish one business event to multiple independently
  evolvable consumers.
- **Step Functions Standard**: long-lived, auditable workflows and human
  callbacks.
- **Parallel state**: call independent models simultaneously, then merge.
- **Choice state**: classify and route by complexity, risk, or intent.
- **Callback task token**: wait for approval without polling.
- **Lambda idempotency**: use a stable business identifier when writing to an
  external API that supports idempotency keys.
- **DynamoDB**: workflow/session state, feedback, job results, circuit breakers,
  or conversation summaries.
- **SNS**: notification, not durable work buffering.

## 9. Performance, capacity, and cost

- Measure input/output tokens, latency, throttles, errors, invocation count, and
  TimeToFirstToken.
- **CountTokens** before invocation for admission control, pruning, and
  per-tenant token buckets.
- Include the output token limit when reserving a tenant's budget.
- Compress older conversation turns into a running summary and keep only recent
  turns.
- Reduce excessive RAG chunks and cap task-specific output length.
- **Streaming** lowers perceived latency, not necessarily total generation time.
- **Latency-optimized inference** targets response latency; verify improvement
  with TimeToFirstToken and InvocationLatency.
- **Provisioned Throughput** fits predictable, sustained demand and is invoked
  with the provisioned model ARN.
- **Cross-Region inference profiles** absorb temporary regional capacity
  pressure while respecting the required geography.
- **Batch inference** fits asynchronous Bedrock jobs with S3 input/output.
- **SageMaker Batch Transform MultiRecord** improves GPU utilization for many
  small offline records; tune concurrency and payload size.
- **Intelligent Prompt Routing/model cascading** sends simple tasks to a cheaper
  model and complex tasks to a stronger model.
- Reuse SDK/HTTP clients outside the Lambda handler and enable keep-alive.
- Deterministic response caching is valid only when prompts and model settings
  are normalized into a stable cache key and stale/non-deterministic responses
  are acceptable.

## 10. Data and multimodal processing

- **Glue Data Quality**: automated DQDL rules, row-level pass/fail, quarantine,
  and CloudWatch metrics.
- DQDL is case-sensitive; flatten nested structures when row rules require it.
- **Bedrock Data Automation**: evolving PDFs/images and other supported
  unstructured media to structured output using blueprints.
- **Transcribe**: audio transcription, custom vocabulary, and PII redaction.
- **SageMaker Processing**: reuse an existing custom container for specialized
  transforms.
- **Glue ETL**: structured transformation such as CSV to normalized JSON.
- Use Step Functions around asynchronous extraction and Lambda for final
  external API updates or packet assembly.

## 11. Governance, networking, and observability

### Identity and least privilege

- Use IAM Identity Center federation for workforce SSO and short-lived
  credentials.
- Limit developer roles to approved runtime actions/models; do not grant Bedrock
  management permissions unnecessarily.
- Give each tool adapter only the table, secret, model, or data actions it
  actually needs.

### Private networking and data governance

- Interface endpoints: Bedrock Runtime, Athena, Glue, and other supported AWS
  APIs.
- Gateway endpoint: S3.
- Private subnet with no internet gateway when public transit is prohibited.
- Apply endpoint policies and bucket policies, not only security groups.
- Use Lake Formation for table/column-level access.
- Outposts handles required local preprocessing/storage; send only de-identified
  data to Bedrock through PrivateLink.

### Audit and governance artifacts

- CloudTrail: who called which AWS API and when.
- CloudWatch structured logs: correlation ID, sanitized attributes, prompt
  version, model ID, latency, validation outcome, and request metadata.
- X-Ray: API Gateway → Lambda → Bedrock/Retrieve latency path.
- Bedrock invocation logging: prompts/responses and attribution where allowed.
- SageMaker Model Cards: intended use, limitations, versions, and evaluation
  evidence.
- Glue Data Catalog metadata: source and transformation lineage.
- Audit Manager Generative AI Best Practices Framework: automated and manual
  compliance evidence.
- AWS Well-Architected Generative AI Lens + reusable CDK/CloudFormation:
  consistent multi-account architecture.

## 12. CI/CD and safe change patterns

- CodePipeline orchestrates source, test, approval, and deployment.
- CodeBuild runs tests/security scans and publishes report groups.
- CodeDeploy shifts Lambda alias traffic canary/linearly.
- CloudWatch alarms trigger automatic rollback.
- AppConfig changes model IDs, inference settings, endpoints, routing, or
  failover flags without code redeployment and supports controlled rollout.
- Prompt/model evaluation is a release gate; CloudWatch Synthetics is a
  production regression detector.
- Keep a known-good baseline before relying on automatic rollback.

## 13. SageMaker-specific patterns

- Large LLM serving: use an LMI container rather than building a serving stack.
- Uncompressed large artifacts: `ModelDataSource` with `CompressionType: None`.
- Size EBS volume and increase model download and container startup health-check
  timeouts.
- Register adapted models in Model Registry and require approval.
- Use real-time endpoint deployment guardrails with canary/linear rollout and
  CloudWatch-based rollback.
- Batch Transform MultiRecord is for offline batches; real-time endpoints are
  for interactive inference.

## 14. Frontend and developer productivity

- Amplify provides a fast authenticated/accessibility-oriented web path.
- Amplify Gen 2 AI conversation routes/UI primitives fit streamed multi-turn
  chat with owner-scoped persisted conversation history.
- OpenAPI + API Gateway creates a reusable API-first backend contract.
- Amazon Q Developer is the pattern for IDE chat, inline suggestions,
  refactoring, project guidance, and security scans.

## 15. Highest-priority revision order

Approximate question coverage based on correct-answer keyword occurrences:

1. Lambda and serverless integration — 46/150 questions
2. CloudWatch observability — 29/150
3. Bedrock Guardrails — 23/150
4. API Gateway — 19/150
5. Knowledge Bases/RAG — 18/150
6. Step Functions — 13/150
7. Bedrock Model Evaluation — 11/150
8. Converse/ConverseStream — 9/150
9. Prompt Management and OpenSearch — 8/150 each
10. Glue and SageMaker — 7/150 each
11. AgentCore and Comprehend — 6/150 each
12. CountTokens — 5/150
13. SQS, EventBridge, and MCP — 3/150 each

These are occurrences in correct answers, not official domain weights. A service
can appear in a question without being the tested decision.

## 16. Final readiness checklist

You are ready for this question bank when you can explain, without notes:

- Converse vs InvokeModel vs their streaming variants.
- SQS vs EventBridge vs Step Functions.
- Guardrails vs Comprehend vs Macie.
- Prompt Management vs Flows vs Step Functions.
- Retrieve vs RetrieveAndGenerate.
- Semantic vs hybrid search vs reranking.
- Hierarchical chunking and metadata sidecars.
- Model, RAG, agent, LLM-judge, and human evaluation.
- CountTokens, prompt caching, streaming, Provisioned Throughput, batch
  inference, and cross-Region profiles.
- CloudTrail vs CloudWatch Logs/metrics vs X-Ray.
- VPC interface endpoints, S3 gateway endpoints, Lake Formation, and IAM least
  privilege.
- Agent trace vs source citation vs prohibited exposure of hidden
  chain-of-thought.
- Canary rollout, alarms, rollback, AppConfig, and evaluation gates.
- Bedrock managed inference vs SageMaker LMI/real-time/Batch Transform.

