# Architecture Decision Tables

Status: Verified patterns; check current service/model support before implementation  
Last verified: 2026-07-23

## Bedrock versus SageMaker AI

| Requirement | Bedrock | SageMaker AI |
|---|---:|---:|
| Managed access to multiple FMs | Strong | Possible, more assembly |
| Fast FM application development | Strong | Moderate |
| Managed Guardrails/Knowledge Bases/Agents | Native | Custom/integrated |
| Custom/open-weight model serving control | Limited by supported offerings | Strong |
| Custom container/GPU topology | No | Yes |
| Training/fine-tuning pipeline control | Limited managed options | Strong |
| Lowest infrastructure ownership | Usually | No |

Decision: choose Bedrock for managed FM integration; choose SageMaker when model/serving customization is the hard requirement.

## Converse versus InvokeModel

| Requirement | Converse | InvokeModel |
|---|---:|---:|
| Common chat interface | Yes | No |
| Model-specific body/control | Limited additions | Yes |
| Specialized image/embedding model | Often not the fit | Typical |
| Easier provider switching | Better | Adapter required |
| Common streaming | ConverseStream | InvokeModelWithResponseStream |

## On-demand versus provisioned versus batch

| Workload | Choice |
|---|---|
| Bursty interactive | On-demand |
| Predictable sustained interactive | Provisioned Throughput |
| Large asynchronous offline | Batch inference |
| Regional capacity resilience | Cross-Region inference, if residency permits |

## Retrieve versus RetrieveAndGenerate

| Requirement | Retrieve | RetrieveAndGenerate |
|---|---:|---:|
| Inspect/debug retrieval | Best | Indirect |
| Custom final schema/prompt/workflow | Best | Less control |
| Supported non-Managed KB end-to-end RAG | More code | Best |
| Return generated answer with citations | Application builds it | Native output |

The product named **Managed Knowledge Bases** uses `Retrieve` or `AgenticRetrieveStream`; current `RetrieveAndGenerate` documentation excludes Managed Knowledge Bases.

## Knowledge Bases versus Q Business versus custom RAG

| Need | Choice |
|---|---|
| Managed RAG building block inside a custom app | Bedrock Knowledge Bases |
| Enterprise assistant with supported source connectors and user ACLs | Amazon Q Business |
| Unsupported store/algorithm or maximum retrieval control | Custom RAG |

Do not choose Q Business only because “documents” are mentioned. Its enterprise search/identity and application experience must match the requirement.

## Step Functions versus Bedrock Flows versus agent orchestration

| Requirement | Choice |
|---|---|
| Deterministic, auditable, long-running, callback, retry, compensation | Step Functions Standard |
| Configured prompt/condition/KB/Lambda graph managed in Bedrock | Bedrock Flows |
| Dynamic tool selection and flexible multi-step interaction | Agent |
| High-risk decision boundary | Deterministic workflow outside FM |

## Lambda versus ECS/Fargate versus EKS versus AgentCore Runtime

| Runtime need | Choice |
|---|---|
| Short, stateless, event-driven tool | Lambda |
| Longer/native dependencies/containers without server management | ECS on Fargate |
| Existing Kubernetes platform or cluster-specific requirement | EKS |
| Purpose-built managed agent/MCP runtime and features | AgentCore Runtime |

“Use EKS” is not correct merely because a container can run there.

## SQS versus EventBridge versus SNS

| Requirement | Choice |
|---|---|
| Buffer work, absorb bursts, worker polling, backpressure | SQS |
| Route one event to changing consumers by rules | EventBridge |
| Push fanout/notification | SNS |
| Ordered processing/deduplication requirement | Consider SQS FIFO if supported need |

Use idempotency with at-least-once delivery.

## API Gateway HTTP, REST, and WebSocket

| Requirement | Choice |
|---|---|
| Simple low-overhead HTTP/Lambda proxy | HTTP API |
| REST features, validation, mature transformations, response streaming design | REST API |
| Bidirectional persistent communication | WebSocket |
| Existing browser SSE contract | REST/Lambda response streaming where supported |

## OpenSearch versus Aurora pgvector

| Requirement | OpenSearch | Aurora PostgreSQL + pgvector |
|---|---:|---:|
| Search-first, hybrid keyword/vector | Strong | More custom |
| Large semantic search corpus | Strong | Depends on design |
| Complex relational transactions/joins | Weak | Strong |
| SQL metadata filtering | Moderate | Strong |
| Managed KB integration | Supported configurations | Supported configurations |

## Semantic, keyword, hybrid, and reranking

| Query | Pattern |
|---|---|
| Natural-language meaning | Semantic/vector |
| Exact error/CVE/product code | Keyword/filter |
| Both exact and descriptive | Hybrid |
| Good candidates in poor order | Reranker |
| Multiple intents | Query decomposition, retrieve per subquery, merge |

## Chunking

| Corpus/problem | Pattern |
|---|---|
| Uniform short text | Fixed size may suffice |
| Structured sections | Semantic/structure-aware |
| Precise clause needs surrounding context | Hierarchical child retrieval/parent return |
| Multi-modal/layout-rich | Specialized parser before chunking |

## Comprehend versus Macie versus Guardrails

| Need | Choice |
|---|---|
| Detect/redact PII in request text before FM | Comprehend and application preprocessing |
| Discover sensitive data already stored in S3 | Macie |
| Block/mask policy/safety content in FM input/output | Bedrock Guardrails |
| Custom deterministic validation | Lambda/application validator |

These controls complement rather than replace one another.

## CloudTrail versus CloudWatch versus X-Ray

| Question | Service |
|---|---|
| Who called/changed an AWS API/resource? | CloudTrail |
| What is the error/metric trend? | CloudWatch |
| Where did this distributed request spend time? | X-Ray |
| What did the agent/flow orchestrate? | Agent/Flow trace |
| Did answer quality regress? | Evaluation system |

## Data processing

| Input/requirement | Choice |
|---|---|
| Row-level data-quality rules | Glue Data Quality |
| Interactive preparation | SageMaker Data Wrangler |
| Existing custom container at scale | SageMaker Processing |
| Structured document/image extraction | Bedrock Data Automation |
| OCR/forms/tables | Textract |
| Speech/transcript/custom vocabulary | Transcribe |
| Text entities/PII/intent | Comprehend |

## Model configuration changes

| Requirement | Pattern |
|---|---|
| Change model/routing without code deploy | AppConfig |
| Stable prompt release | Prompt Management version |
| Controlled production approval | CI/CD/manual approval policy |
| Limited traffic rollout | Lambda alias/CodeDeploy canary or supported endpoint guardrail |
| Automatic configuration rollback | AppConfig/CodeDeploy plus CloudWatch alarm |

Prompt Management versions a prompt. Do not assume it alone supplies every organizational approval workflow.

## Private data path

Typical pattern:

- Compute in private subnets.
- Interface VPC endpoints for supported AWS APIs such as Bedrock Runtime.
- S3 gateway endpoint.
- Endpoint policies plus resource policies.
- Lake Formation for governed table/column access.
- Least-privilege IAM.
- Sanitized logging and explicit retention.

A private network path does not replace authorization.

## Human approval

| Duration/requirement | Pattern |
|---|---|
| Minutes to days, auditable | Step Functions Standard callback token |
| Ambiguous moderation only | Branch exception to human review |
| Every regulated output | Mandatory review step |
| Simple feedback after response | API Gateway/Lambda/DynamoDB |

## Exam reminder

Choose the simplest pattern that satisfies every requirement. “Least operational overhead” does not mean ignore security, validation, or failure handling.
