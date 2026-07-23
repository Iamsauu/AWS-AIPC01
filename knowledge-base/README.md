# AIP-C01 Knowledge System

Status: Built and verified  
Certification: AWS Certified Generative AI Developer – Professional  
Exam code: AIP-C01  
Last blueprint check: 2026-07-23

## Purpose

This is an exam-oriented knowledge system for designing, implementing, securing, operating, evaluating, and troubleshooting production GenAI applications on AWS.

It is not a collection of copied service descriptions. Each topic is organized around four questions:

1. What requirement does this solve?
2. Why is this service or pattern preferable to the alternatives?
3. What can fail, and what evidence identifies the cause?
4. Which security, quality, cost, latency, and operational tradeoffs change the answer?

The current AWS exam guide is authoritative. Local mock questions provide scenarios and distractor patterns, but they do not override AWS documentation.

## Verified inventory

- **98/98** official skills mapped and inspected.
- **150/150** local mock questions mapped to canonical topics.
- **106/106** officially in-scope service entries covered.
- **12** detailed concept guides, **5** domain guides, **6** reference guides, and **6** hands-on labs.
- **36** linked Markdown files with internal links and code fences validated.

The scope and writing contract are documented in the [knowledge-base plan](../AIP-C01-KNOWLEDGE-BASE-PLAN.md).
The researched implementation plan for an active-revision application is in the [revision web-tool plan](../AIP-C01-REVISION-WEB-TOOL-PLAN.md).

## Start here

1. [Exam and study strategy](00-exam-and-study-strategy.md)
2. [Official coverage matrix](01-coverage-matrix.md)
3. Study the five domain guides in order of weight.
4. Use concept pages when a domain guide links to deeper material.
5. Use the reference pages while solving scenarios.
6. Complete the labs, then use recall questions and unseen scenario drills.
7. Check every missed mock question in the [mock coverage map](practice/mock-question-coverage-map.md).

## Official domain guides

| Domain | Weight | Guide |
|---|---:|---|
| 1. FM integration, data management, and compliance | 31% | [Domain 1](domains/domain-1-fm-integration-data-rag-prompts.md) |
| 2. Implementation and integration | 26% | [Domain 2](domains/domain-2-implementation-integration-agents.md) |
| 3. AI safety, security, and governance | 20% | [Domain 3](domains/domain-3-safety-security-governance.md) |
| 4. Operational efficiency and optimization | 12% | [Domain 4](domains/domain-4-cost-performance-operations.md) |
| 5. Testing, validation, and troubleshooting | 11% | [Domain 5](domains/domain-5-evaluation-troubleshooting.md) |

## Canonical concept guides

| Concept | Guide |
|---|---|
| Model selection and Bedrock runtime APIs | [Bedrock model selection and APIs](concepts/bedrock-model-selection-and-runtime-apis.md) |
| RAG, Knowledge Bases, embeddings, and vector search | [RAG and vector search](concepts/rag-knowledge-bases-vector-search.md) |
| Prompts, Prompt Management, optimization, and Flows | [Prompt systems](concepts/prompt-engineering-management-and-flows.md) |
| Agents, tools, MCP, Strands, and AgentCore | [Agentic systems](concepts/agents-tools-mcp-and-agentcore.md) |
| Data quality and multimodal processing | [Data processing](concepts/data-quality-and-multimodal-processing.md) |
| Safety, privacy, and Responsible AI | [Safety and Responsible AI](concepts/safety-privacy-and-responsible-ai.md) |
| IAM, private networking, and data access | [Security and access control](concepts/security-networking-and-access-control.md) |
| Evaluation and release quality gates | [Evaluation systems](concepts/evaluation-testing-and-quality-gates.md) |
| Cost, latency, throughput, routing, and caching | [Efficiency and performance](concepts/cost-latency-throughput-and-caching.md) |
| Monitoring, tracing, and diagnosis | [Observability and troubleshooting](concepts/observability-and-troubleshooting.md) |
| Enterprise integration and delivery pipelines | [Enterprise integration and CI/CD](concepts/enterprise-integration-and-cicd.md) |
| Custom and open-weight model deployment | [SageMaker deployment](concepts/sagemaker-custom-model-deployment.md) |

## Fast reference

- [AWS service decision catalog](reference/aws-service-decision-catalog.md)
- [Bedrock API and payload cheat sheet](reference/api-and-payload-cheat-sheet.md)
- [Architecture decision tables](reference/architecture-decision-tables.md)
- [Metrics, errors, and limits](reference/metrics-errors-and-limits.md)
- [Glossary](reference/glossary.md)
- [Official source registry](reference/official-sources.md)

## Practice and active recall

- [All 150 mock-question mappings](practice/mock-question-coverage-map.md)
- [Exam traps and distractors](practice/exam-traps-and-distractors.md)
- [Recall questions](practice/recall-questions.md)
- [Scenario drills](practice/scenario-drills.md)

## Hands-on validation

| Lab | Capability |
|---|---|
| [1. Bedrock API and streaming](labs/01-bedrock-api-and-streaming.md) | Converse, model-aware requests, CountTokens, streaming |
| [2. RAG Knowledge Base](labs/02-rag-knowledge-base.md) | Ingestion, chunking, metadata, retrieval, citations |
| [3. Guardrails and private access](labs/03-guardrails-and-private-access.md) | Safety, PII, IAM, VPC endpoints |
| [4. Agent tools and Step Functions](labs/04-agent-tools-and-step-functions.md) | Tool schemas, loops, callback, circuit breaker |
| [5. Model, RAG, and agent evaluation](labs/05-model-rag-agent-evaluation.md) | Benchmarks, quality gates, human feedback |
| [6. Observability, cost, and performance](labs/06-observability-cost-and-performance.md) | Logs, metrics, traces, tokens, TTFT |

## Coverage dashboard

The completed verification checked that:

- Every official skill is present in the [coverage matrix](01-coverage-matrix.md).
- Every local mock question has at least one canonical topic mapping.
- Every officially in-scope service has recognition coverage.
- Tier 1 topics include implementation, security, cost, monitoring, and failure handling.
- Uncertain or mock-only claims are labeled.
- Internal links and cited official sources have been checked.

## Maintenance rules

- Put each fact in one canonical concept page; domain guides should summarize and link.
- Add the verification date when behavior is Region-, model-, quota-, or feature-dependent.
- Never treat retrieved vector similarity as factual confidence.
- Never expose provider-internal chain-of-thought; use citations and orchestration traces.
- Never assume a managed feature exists because a mock answer says so.
- Recheck model availability, API support, quotas, pricing, and Region coverage before production use.
