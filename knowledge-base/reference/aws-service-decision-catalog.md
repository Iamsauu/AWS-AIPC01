# AWS Service Decision Catalog

Status: Covers the official in-scope list verified 2026-07-23  
Depth: Tier 1 services require implementation knowledge; Tier 2 require architectural decisions; Tier 3 require recognition.

The official list is non-exhaustive and can change. This catalog explains why a service might appear in a GenAI architecture; it does not imply equal exam probability.

## Analytics

| Service | Depth | Exam role and boundary |
|---|---:|---|
| Amazon Athena | 2 | Serverless SQL over S3; query governed data or evaluation results. Not a transactional database. |
| Amazon EMR | 3 | Managed big-data frameworks for very large custom Spark/Hadoop processing. More operations than Glue for common managed ETL. |
| AWS Glue | 2 | Serverless ETL, Data Catalog, crawlers, lineage integration, and Data Quality/DQDL. |
| Amazon Kinesis | 3 | Real-time streaming ingestion/processing. Use for ordered high-throughput streams, not simple task queues. |
| Amazon OpenSearch Service | 1 | Search, log analytics, vector/hybrid search, shards/index control. |
| Amazon Quick Sight (formerly Amazon QuickSight) | 2 | Dashboards, evaluation reporting, analytics, and stakeholder visualization within the current Amazon Quick family. |
| Amazon MSK | 3 | Managed Apache Kafka for existing Kafka/event-streaming requirements; heavier than native queues/events. |

## Application integration

| Service | Depth | Exam role and boundary |
|---|---:|---|
| Amazon AppFlow | 2 | Managed SaaS-to-AWS data transfer, such as Salesforce content to S3. |
| AWS AppConfig | 1 | Validated runtime configuration, feature flags, model/routing changes, staged deployment, and rollback. |
| Amazon EventBridge | 1 | Rule-based event routing and fanout to changing consumers. Not a work queue/backpressure layer. |
| Amazon SNS | 2 | Push fanout and notifications, including human-review alerts. |
| Amazon SQS | 1 | Durable queue, burst absorption, backpressure, asynchronous workers, dead-letter queues. |
| AWS Step Functions | 1 | Deterministic orchestration, retries, branches, parallel work, long callbacks, agent loops, and audit history. |

## Compute

| Service | Depth | Exam role and boundary |
|---|---:|---|
| AWS App Runner | 3 | Managed deployment of web/container services with low infrastructure ownership. |
| Amazon EC2 | 3 | Full VM control for workloads that do not fit managed/serverless choices; highest host operations. |
| AWS Lambda | 1 | Stateless event/API/tool adapters, preprocessing, validators, and short orchestration tasks. |
| Lambda@Edge | 3 | Run limited logic at CloudFront edge locations; not general FM inference. |
| AWS Outposts | 2 | AWS infrastructure on premises for local preprocessing/data residency and hybrid access. |
| AWS Wavelength | 3 | Ultra-low-latency compute at telecom edge; select only for explicit edge/mobile latency needs. |

## Containers

| Service | Depth | Exam role and boundary |
|---|---:|---|
| Amazon ECR | 2 | Store container images for ECS/EKS/SageMaker/custom runtimes. |
| Amazon ECS | 2 | AWS-native container orchestration; pair with Fargate for managed compute. |
| Amazon EKS | 3 | Managed Kubernetes when Kubernetes is a requirement or existing platform, not the lowest-overhead default. |
| AWS Fargate | 2 | Serverless compute for ECS/EKS containers; useful for heavier/longer MCP tools. |

## Customer engagement

| Service | Depth | Exam role and boundary |
|---|---:|---|
| Amazon Connect | 3 | Cloud contact center; GenAI agent assist, call workflows, and transcripts. |

## Databases

| Service | Depth | Exam role and boundary |
|---|---:|---|
| Amazon Aurora | 2 | Managed MySQL- and PostgreSQL-compatible relational database; choose Aurora PostgreSQL with pgvector for relational/vector use cases. |
| Amazon DocumentDB | 3 | Managed document database compatible with MongoDB workloads; not a default vector store. |
| Amazon DynamoDB | 1 | Serverless key-value/document storage for sessions, job state, feedback, idempotency, memory metadata, TTL, and circuit breakers. |
| DynamoDB Streams | 3 | Change stream from DynamoDB for event-driven synchronization. |
| Amazon ElastiCache | 2 | Managed in-memory caching/session/rate state. Use when low-latency shared cache is required. |
| Amazon Neptune | 3 | Managed graph database for knowledge graphs and relationship traversal. |
| Amazon RDS | 2 | Managed relational engines; choose when relational semantics are primary. |

## Developer tools

| Service | Depth | Exam role and boundary |
|---|---:|---|
| AWS Amplify | 2 | Rapid authenticated frontend, UI, data, and GenAI conversation integration. |
| AWS CDK | 2 | Define reusable infrastructure constructs in programming languages. |
| AWS CLI | 3 | Script and operate AWS APIs; not an application runtime. |
| AWS CloudFormation | 2 | Declarative IaC and repeatable environment deployment. |
| AWS CodeArtifact | 3 | Managed package/artifact repositories for build dependencies. |
| AWS CodeBuild | 2 | Builds, security tests, prompt/guardrail tests, and test reports. |
| AWS CodeDeploy | 2 | Deployment automation; canary/linear traffic shifting is relevant to Lambda and ECS deployment configurations, while EC2/on-premises deployments use different configurations. |
| AWS CodePipeline | 1 | Source-to-test-to-approval-to-deploy orchestration and release gates. |
| Kiro | 3 | AWS agentic coding service/IDE for specs, code, tests, debugging, and development workflows. |
| AWS SDKs and tools | 1 | Authenticated service APIs, retry configuration, model runtime integration. |
| AWS X-Ray | 1 | Distributed request tracing and latency isolation. |

## Machine learning and GenAI

| Service/feature | Depth | Exam role and boundary |
|---|---:|---|
| Amazon Augmented AI (A2I) | 3 | Managed human-review workflows for supported ML/custom decisions. Compare with SFN callback for general orchestration. |
| Amazon Bedrock | 1 | Managed FMs and GenAI application capabilities. |
| Bedrock AgentCore | 1 | Managed agent runtime, Gateway, Memory, identity/observability/evaluation-related capabilities. Verify current subservice support. |
| Bedrock Knowledge Bases | 1 | Managed RAG ingestion, vector integration, retrieval, generation, and citations. |
| Bedrock Prompt Management | 1 | Prompt variables, variants, versions, testing, and stable deployment references. |
| Bedrock Prompt Flows / Flows | 1 | Managed nodes for prompts, conditions, KBs, Lambda, and configured orchestration. |
| Amazon Comprehend | 2 | NLP entities, language, sentiment, custom classification, and real-time PII detection. |
| Amazon Kendra | 3 | Enterprise intelligent search with connectors and relevance; distinguish from custom vector RAG. |
| Amazon Lex | 3 | Managed conversational interface with intents/slots; useful for deterministic dialog capture. |
| Amazon Q Business | 2 | Enterprise assistant/search with supported connectors, identity, source permissions, and plugins. |
| Amazon Q Business Apps | 3 | User-created/enterprise applications on Q Business data and capabilities. |
| Amazon Q Developer | 2 | IDE/CLI coding assistance, AWS guidance, refactoring, tests, and security scans. |
| Amazon Quick | 3 | AI workspace for connected knowledge, agents, research, analytics, workflows, and actions. |
| Amazon Rekognition | 3 | Managed image/video labels, moderation, faces, and text detection; not generative multimodal reasoning. |
| Amazon SageMaker AI | 1 | Build, process, evaluate, deploy, and operate custom ML/FMs. |
| SageMaker Clarify | 3 | Bias/explainability analysis for ML workflows; distinguish from Bedrock FM evaluation. |
| SageMaker Data Wrangler | 2 | Interactive data preparation/analysis, not the automatic nightly-validation default. |
| SageMaker Ground Truth | 3 | Data labeling and human annotations. |
| SageMaker JumpStart | 2 | Prebuilt models/solutions and model deployment starting points. |
| SageMaker Model Monitor | 2 | Monitor deployed SageMaker model/data quality and drift. |
| SageMaker Model Registry | 1 | Version, metadata, approval status, lineage, and release of custom models. |
| SageMaker Neo | 3 | Compile/optimize models for supported inference targets. |
| SageMaker Processing | 2 | Managed custom data/evaluation processing using scripts or containers. |
| SageMaker Unified Studio | 3 | Governed development environment spanning data, analytics, and ML workflows. |
| Amazon Textract | 2 | OCR and structured extraction from forms/tables/documents. |
| Amazon Titan | 1 | AWS FMs including text and embedding/multimodal embedding capabilities. Check exact model support. |
| Amazon Transcribe | 2 | Speech-to-text, custom vocabulary, streaming/batch, and supported PII redaction. |

## Management and governance

| Service | Depth | Exam role and boundary |
|---|---:|---|
| AWS Auto Scaling | 2 | Scaling plans for supported resources such as EC2 Auto Scaling groups, ECS, DynamoDB, Aurora, and Spot Fleet. SageMaker endpoint autoscaling is configured through Application Auto Scaling. |
| AWS Chatbot | 3 | Operational notifications and commands through chat channels; naming/integration can evolve. |
| AWS CloudTrail | 1 | AWS API and resource-change audit: who did what and when. |
| Amazon CloudWatch | 1 | Metrics, dashboards, alarms, logs, anomaly detection, and custom application signals. |
| CloudWatch Logs | 1 | Structured runtime logs and Logs Insights analysis. |
| CloudWatch Synthetics | 2 | Scheduled canaries for deployed user journeys, availability, and latency. |
| AWS Cost Anomaly Detection | 2 | Detect unusual spend patterns. |
| AWS Cost Explorer | 2 | Analyze historical/forecast cost and usage. |
| Amazon Managed Grafana | 3 | Managed Grafana dashboards across observability sources. |
| AWS Service Catalog | 3 | Govern and distribute approved infrastructure products. |
| AWS Systems Manager | 3 | Fleet/config/parameter/operations management for compute environments. |
| AWS Well-Architected Tool | 2 | Record architecture reviews; apply the Generative AI Lens. |

## Migration and transfer

| Service | Depth | Exam role and boundary |
|---|---:|---|
| AWS DataSync | 3 | Scheduled/accelerated file/object transfer between storage systems and AWS. |
| AWS Transfer Family | 3 | Managed SFTP/FTPS/FTP/AS2 access to S3/EFS. |

## Networking and content delivery

| Service | Depth | Exam role and boundary |
|---|---:|---|
| Amazon API Gateway | 1 | Authenticated/validated/throttled HTTP, REST, and WebSocket APIs; Lambda/service integrations. |
| AWS AppSync | 3 | Managed GraphQL and real-time application data APIs. |
| Amazon CloudFront | 2 | Global edge delivery/cache; safe only for shareable correctly keyed GenAI responses. |
| Elastic Load Balancing | 3 | Distribute traffic to compute/container targets; not an API contract/validation service. |
| AWS Global Accelerator | 3 | Anycast static IPs and optimized global path to regional endpoints. |
| AWS PrivateLink | 1 | Private interface endpoint access to supported services without public internet paths. |
| Amazon Route 53 | 3 | DNS, routing, and health checks. |
| Amazon VPC | 1 | Network isolation, subnets, routing, security groups, and endpoints. |

## Security, identity, and compliance

| Service | Depth | Exam role and boundary |
|---|---:|---|
| Amazon Cognito | 2 | Application user identity, tokens, and federation; distinct from workforce IAM Identity Center. |
| AWS Encryption SDK | 3 | Client-side envelope encryption library and data-key handling. |
| AWS IAM | 1 | Authentication/authorization, least privilege, roles, conditions, and resource scoping. |
| IAM Access Analyzer | 2 | Identify external/public access and validate IAM policies. |
| IAM Identity Center | 2 | Workforce SSO, groups, permission sets, and temporary credentials. |
| AWS KMS | 2 | Managed encryption keys and key policies. |
| Amazon Macie | 2 | Discover/classify sensitive data stored in S3. |
| AWS Secrets Manager | 2 | Store, retrieve, and rotate secrets; scope tool roles to specific secrets. |
| AWS WAF | 2 | Web request filtering/rate rules at API/edge; not an FM semantic guardrail. |

## Storage

| Service/feature | Depth | Exam role and boundary |
|---|---:|---|
| Amazon EBS | 2 | Block storage for EC2/SageMaker endpoint model artifacts and containers. |
| Amazon EFS | 3 | Shared elastic file system for multiple compute clients. |
| Amazon S3 | 1 | Source documents, datasets, artifacts, batch input/output, logs, and durable object storage. |
| S3 Intelligent-Tiering | 3 | Automatic storage-cost tiering for changing access patterns. |
| S3 Lifecycle | 2 | Retention, transition, and expiry for prompts, responses, and audit data. |
| S3 Cross-Region Replication | 3 | Replicate objects for resilience/compliance when destination residency is allowed. |

## High-priority confusions

- Macie scans S3; it does not synchronously redact a chat prompt.
- WAF filters web traffic; Guardrails evaluate FM safety/policy content.
- CloudTrail audits API activity; CloudWatch debugs/alarms on runtime behavior.
- Kendra/Q Business provide enterprise search experiences; Knowledge Bases is a RAG component.
- Data Wrangler is interactive; Glue Data Quality is suited to recurring rule validation.
- EKS is not the lowest-overhead answer unless Kubernetes is a requirement.
- A VPC endpoint provides a private path; IAM/Lake Formation still control access.

## Official source

[AIP-C01 in-scope AWS services](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/aip-01-in-scope-services.html)
