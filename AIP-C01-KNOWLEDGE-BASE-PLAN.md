# AIP-C01 Knowledge Base Writing Plan

Status: Implemented and verified on 2026-07-23  
Entry point: [AIP-C01 Knowledge System](knowledge-base/README.md)

## 1. Objective

Create a source-backed Markdown knowledge base that covers the current exam scope for **AWS Certified Generative AI Developer – Professional (AIP-C01)** and supports an **85% target on unseen practice exams**.

“All knowledge” must be bounded correctly: the official exam guide says its topic list is non-exhaustive. The achievable goal is:

- Cover every official domain, task, and skill.
- Cover every recurring concept in the 150 local mock questions.
- Cover the purpose and decision boundary of every officially in-scope AWS service.
- Go deep on services and features that the blueprint and mocks emphasize.
- Clearly separate verified AWS behavior from mock-exam assumptions.

The knowledge base should teach architecture decisions, implementation, security, operations, and troubleshooting—not only definitions.

## 2. Source hierarchy

Use sources in this order:

1. Current official AIP-C01 exam guide and domain pages.
2. Current AWS service documentation and API references.
3. AWS Well-Architected, Prescriptive Guidance, and official AWS technical blogs.
4. The two local practice exams and their explanations.
5. Existing local summaries:
   - `AIP-C01-85-PERCENT-STUDY-CHECKLIST.md`
   - `AIPC01_TEST_PATTERNS.md`

Rules:

- A mock answer must not override current AWS documentation.
- Flag claims that cannot be confirmed.
- Record the date each page was last verified.
- Prefer precise links to the relevant AWS documentation page.
- Do not copy AWS documentation verbatim; explain it in exam-oriented language.

## 3. Recommended structure

A single large Markdown file will be difficult to maintain and revise. Use a linked Markdown library:

```text
knowledge-base/
├── README.md
├── 00-exam-and-study-strategy.md
├── 01-coverage-matrix.md
├── domains/
│   ├── domain-1-fm-integration-data-rag-prompts.md
│   ├── domain-2-implementation-integration-agents.md
│   ├── domain-3-safety-security-governance.md
│   ├── domain-4-cost-performance-operations.md
│   └── domain-5-evaluation-troubleshooting.md
├── concepts/
│   ├── bedrock-model-selection-and-runtime-apis.md
│   ├── rag-knowledge-bases-vector-search.md
│   ├── prompt-engineering-management-and-flows.md
│   ├── agents-tools-mcp-and-agentcore.md
│   ├── data-quality-and-multimodal-processing.md
│   ├── safety-privacy-and-responsible-ai.md
│   ├── security-networking-and-access-control.md
│   ├── evaluation-testing-and-quality-gates.md
│   ├── cost-latency-throughput-and-caching.md
│   ├── observability-and-troubleshooting.md
│   ├── enterprise-integration-and-cicd.md
│   └── sagemaker-custom-model-deployment.md
├── reference/
│   ├── aws-service-decision-catalog.md
│   ├── api-and-payload-cheat-sheet.md
│   ├── architecture-decision-tables.md
│   ├── metrics-errors-and-limits.md
│   ├── glossary.md
│   └── official-sources.md
├── practice/
│   ├── mock-question-coverage-map.md
│   ├── exam-traps-and-distractors.md
│   ├── recall-questions.md
│   └── scenario-drills.md
└── labs/
    ├── 01-bedrock-api-and-streaming.md
    ├── 02-rag-knowledge-base.md
    ├── 03-guardrails-and-private-access.md
    ├── 04-agent-tools-and-step-functions.md
    ├── 05-model-rag-agent-evaluation.md
    └── 06-observability-cost-and-performance.md
```

The five domain pages provide the exam-shaped learning path. Concept pages hold the canonical detailed explanation so the same topic is not copied into several domain files.

## 4. Standard page format

Every concept page should use the same structure:

```markdown
# Topic

Status: Draft | Verified | Needs recheck
Official tasks: 1.4, 1.5
Last verified: YYYY-MM-DD

## Why this matters
## Core concepts
## How it works
## AWS services and APIs
## Architecture patterns
## Decision table
## Security and governance
## Cost, latency, and reliability
## Failure modes and troubleshooting
## Common exam traps
## Local mock references
## Hands-on validation
## Recall questions
## Official sources
```

Headings may be combined or reworded when the page still contains every required content type. The contract is semantic, not a requirement to repeat artificial headings.

Each page must answer:

- What is it?
- When should it be used?
- When should it not be used?
- What alternative is the main distractor?
- Which requirement changes the answer?
- Which API, configuration, metric, or failure mode is exam-relevant?

## 5. Coverage depth

### Tier 1 — implementation depth

These require architecture, API/configuration, security, cost, monitoring, and troubleshooting coverage:

- Amazon Bedrock Runtime: Converse, ConverseStream, InvokeModel, streaming, CountTokens.
- Foundation model selection, inference profiles, Provisioned Throughput, batch inference, prompt caching, and intelligent routing.
- Bedrock Knowledge Bases, RAG, chunking, embeddings, metadata, retrieval, hybrid search, and reranking.
- Bedrock Guardrails, ApplyGuardrail, prompt attacks, contextual grounding, and IAM enforcement.
- Prompt Management, Prompt Flows/Bedrock Flows, versioning, optimization, and release governance.
- Bedrock agents, AgentCore, Strands Agents, AWS Agent Squad, MCP, memory, tools, and traces.
- Bedrock model, RAG, and agent evaluation.
- Step Functions agent loops, callbacks, retries, stop conditions, and circuit breakers.
- Lambda, API Gateway, SQS, EventBridge, DynamoDB, S3, CloudWatch, X-Ray, IAM, VPC endpoints, and Lake Formation.
- SageMaker custom/open-weight FM deployment and lifecycle.

### Tier 2 — architectural decision depth

Know when to select these and how they integrate:

- OpenSearch Service/Serverless, Aurora PostgreSQL with pgvector, and supported external vector stores.
- Glue, Glue Data Quality, Bedrock Data Automation, Comprehend, Transcribe, Textract, Macie, and SageMaker Processing.
- AppConfig, CloudTrail, Model Cards, Audit Manager, IAM Identity Center, KMS, Secrets Manager, and S3 Lifecycle.
- CodePipeline, CodeBuild, CodeDeploy, CloudFormation, CDK, Amplify, and Amazon Q Developer.
- ECS/Fargate, EKS, EC2, and AgentCore Runtime for different tool/model workloads.

### Tier 3 — recognition depth

For the remaining officially in-scope services, document:

- Primary purpose.
- One likely GenAI use.
- The closest competing service.
- One reason it would be correct or incorrect in an exam scenario.

Do not spend equal effort on every in-scope service.

## 6. Writing phases

### Phase 0 — inventory and scope lock

- Capture the current official exam version, weights, tasks, skills, technologies, and in-scope services.
- Inventory both local mock exams and explanations.
- Assign stable question IDs: `PE1-Q01` through `PE1-Q75`, and `PE2-Q01` through `PE2-Q75`.
- Record known mock/documentation conflicts in a review list.

Output:

- Initial `README.md`
- Official source registry
- Empty coverage matrix

### Phase 1 — build the coverage matrix

Create one row for every official exam skill.

Columns:

| Field | Purpose |
|---|---|
| Domain/task/skill | Official scope |
| Canonical knowledge page | Where it is explained |
| AWS services/features | Required technology |
| Local mock questions | Evidence and scenarios |
| Hands-on lab | Practical validation |
| Depth | Tier 1, 2, or 3 |
| Status | Missing, draft, verified |
| Last verified | Staleness control |

This matrix is the definition of coverage and prevents hidden gaps.

### Phase 2 — write Domain 1

Write in this order:

1. Requirements analysis and proof of concept.
2. FM selection, capability matrix, availability, lifecycle, and resilience.
3. Data validation and multimodal processing.
4. Embeddings and vector-store design.
5. RAG ingestion, chunking, metadata, retrieval, hybrid search, and reranking.
6. Prompt engineering, Prompt Management, Flows, optimization, and governance.

Quality gate:

- Every Domain 1 skill has a canonical explanation, decision rule, mock mapping, and official source.

### Phase 3 — write Domain 2

Write:

1. Agent loop, state, memory, tools, stopping conditions, and human review.
2. MCP, Strands, AgentCore Runtime/Gateway/Memory/Evaluations, and AWS Agent Squad.
3. Bedrock and SageMaker deployment patterns.
4. Enterprise, event-driven, hybrid, and gateway integration.
5. Bedrock API contracts and streaming.
6. Application integration, Amplify, Flows, CI/CD, and developer tools.

Quality gate:

- Every API decision includes supported use, payload shape, error class, retry behavior, and main distractor.

### Phase 4 — write Domain 3

Write:

1. Input and output safety controls.
2. Prompt-injection and jailbreak defense in depth.
3. Grounding, citations, structured output, and deterministic tool/SQL control.
4. PII detection, masking, stored-data discovery, retention, and private networking.
5. IAM, Identity Center, Lake Formation, encryption, and secrets.
6. Model cards, lineage, audit evidence, logging, Responsible AI, fairness, and transparency.

Quality gate:

- Clearly distinguish Guardrails, Comprehend, Macie, IAM, CloudTrail, CloudWatch, and invocation logging.

### Phase 5 — write Domains 4 and 5

Domain 4:

- Token accounting and context management.
- Prompt, semantic, and edge caching.
- Model routing/cascading.
- Latency, streaming, throughput, batching, and capacity.
- CloudWatch, X-Ray, traces, logs, cost, and business metrics.

Domain 5:

- Model, RAG, retrieval-only, agent, automated, and human evaluations.
- Dataset design and quality metrics.
- Regression, canary, continuous evaluation, and feedback.
- API, prompt, retrieval, agent-loop, throttling, latency, and SageMaker deployment troubleshooting.

Quality gate:

- Every common symptom maps to evidence, probable cause, and remediation.

### Phase 6 — cross-domain reference material

Create:

- AWS service decision catalog.
- API and request-payload cheat sheet.
- Architecture decision tables.
- Metrics and error guide.
- Exam distractor guide.
- Glossary.

High-value comparison tables:

- Bedrock versus SageMaker.
- Converse versus InvokeModel.
- On-demand versus Provisioned Throughput versus batch.
- Retrieve versus RetrieveAndGenerate.
- Knowledge Bases versus Q Business versus custom RAG.
- Lambda versus ECS/Fargate versus AgentCore Runtime.
- Step Functions versus Bedrock Flows versus agent-controlled orchestration.
- Comprehend versus Macie versus Guardrails.
- CloudTrail versus CloudWatch versus X-Ray.
- SQS versus EventBridge versus SNS.
- OpenSearch versus Aurora pgvector.

### Phase 7 — labs and active recall

Each lab must have:

- Objective.
- Minimal architecture.
- Prerequisites.
- Implementation steps.
- Verification evidence.
- Expected failures.
- Cleanup steps and cost warning.
- Exam lessons.

Add recall questions that require explanation, not only recognition. Add changed versions of mock scenarios so memorized wording does not inflate scores.

### Phase 8 — final coverage and accuracy audit

- Confirm every official skill has a completed row.
- Map all 150 mock questions to one or more topics.
- Confirm every in-scope service appears in the service catalog.
- Resolve or flag every mock/documentation conflict.
- Check internal links and source links.
- Remove duplicated or contradictory explanations.
- Verify terminology, API names, and current feature support.
- Run a cold-reader review: a topic must make sense without opening the mock question.

## 7. Definition of complete

The knowledge base is complete only when:

- [ ] 100% of official domains, tasks, and skills are mapped.
- [ ] 150/150 local mock questions are mapped.
- [ ] 100% of official in-scope services have at least recognition coverage.
- [ ] Every Tier 1 topic includes implementation, security, cost, monitoring, and troubleshooting.
- [ ] Every factual feature claim has an official source.
- [ ] Mock-only or uncertain claims are labeled.
- [ ] Every domain includes decision tables and active-recall questions.
- [ ] Six core hands-on labs are documented and verified.
- [ ] No broken internal links remain.
- [ ] The root README provides a study order and progress dashboard.

## 8. Recommended writing order

Use the exam weight and mock frequency:

1. RAG, Knowledge Bases, embeddings, vector search.
2. Bedrock Runtime APIs, model selection, and inference patterns.
3. Guardrails, security, privacy, and governance.
4. Agents, tools, MCP, AgentCore, and Step Functions.
5. Evaluation and release quality gates.
6. Prompt engineering, Prompt Management, and Flows.
7. Cost, latency, throughput, caching, and observability.
8. Data quality and multimodal processing.
9. SageMaker custom-model deployment.
10. Enterprise integration, CI/CD, frontend, and lower-frequency services.

## 9. Planned deliverables

The writing work should produce:

- Approximately 25–30 linked Markdown files.
- One canonical coverage matrix.
- Five domain guides.
- Twelve cross-domain concept guides.
- Six reference/decision guides.
- Six hands-on labs.
- A complete 150-question coverage map.
- Recall questions and scenario drills.
- A root study path with progress status.

This plan intentionally avoids a single very large file: modular files make gaps, contradictions, updates, and revision progress visible.
