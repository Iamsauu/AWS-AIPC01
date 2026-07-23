# Official AIP-C01 Coverage Matrix

Status: Verified  
Last blueprint verification: 2026-07-23

This matrix is the completeness contract for the knowledge system. Each official skill has a canonical destination. “Verified” means the target page and its internal links were inspected for the required implementation, tradeoffs, failure modes, and sources.

Abbreviations:

- D = domain guide
- FM = foundation model
- KB = Knowledge Base
- PM = Prompt Management
- SFN = Step Functions
- CW = CloudWatch

## Domain 1 — Foundation Model Integration, Data Management, and Compliance

[Official Domain 1 source](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain1.html)

| Skill | Required knowledge | Canonical page | Status |
|---|---|---|---|
| 1.1.1 | Architecture aligned to business and technical constraints | [D1](domains/domain-1-fm-integration-data-rag-prompts.md) | Verified |
| 1.1.2 | Technical PoCs measuring feasibility, performance, and value | [D1](domains/domain-1-fm-integration-data-rag-prompts.md) | Verified |
| 1.1.3 | Standard components, Well-Architected, GenAI Lens | [Enterprise integration](concepts/enterprise-integration-and-cicd.md) | Verified |
| 1.2.1 | Assess FM capability, benchmark, limitation, and use-case fit | [Model selection](concepts/bedrock-model-selection-and-runtime-apis.md) | Verified |
| 1.2.2 | Dynamic model/provider selection without code changes | [Model selection](concepts/bedrock-model-selection-and-runtime-apis.md) | Verified |
| 1.2.3 | Resilience, circuit breakers, cross-Region inference, degradation | [Efficiency](concepts/cost-latency-throughput-and-caching.md) | Verified |
| 1.2.4 | FM customization lifecycle, LoRA/adapters, registry, rollout, rollback | [SageMaker deployment](concepts/sagemaker-custom-model-deployment.md) | Verified |
| 1.3.1 | Automated data validation and operational quality metrics | [Data processing](concepts/data-quality-and-multimodal-processing.md) | Verified |
| 1.3.2 | Text, image, audio, and tabular processing workflows | [Data processing](concepts/data-quality-and-multimodal-processing.md) | Verified |
| 1.3.3 | Model-specific input and conversation formatting | [API cheat sheet](reference/api-and-payload-cheat-sheet.md) | Verified |
| 1.3.4 | Input normalization, entity extraction, and quality enhancement | [Data processing](concepts/data-quality-and-multimodal-processing.md) | Verified |
| 1.4.1 | Advanced vector architectures and managed integrations | [RAG](concepts/rag-knowledge-bases-vector-search.md) | Verified |
| 1.4.2 | Metadata frameworks for precision and context | [RAG](concepts/rag-knowledge-bases-vector-search.md) | Verified |
| 1.4.3 | Scalable indexes, shards, multi-index and hierarchy | [RAG](concepts/rag-knowledge-bases-vector-search.md) | Verified |
| 1.4.4 | Connect document systems, KBs, and wikis | [RAG](concepts/rag-knowledge-bases-vector-search.md) | Verified |
| 1.4.5 | Incremental updates, change detection, sync, and refresh | [RAG](concepts/rag-knowledge-bases-vector-search.md) | Verified |
| 1.5.1 | Fixed, semantic, and hierarchical document segmentation | [RAG](concepts/rag-knowledge-bases-vector-search.md) | Verified |
| 1.5.2 | Embedding selection, dimensions, domain fit, and batching | [RAG](concepts/rag-knowledge-bases-vector-search.md) | Verified |
| 1.5.3 | OpenSearch, Aurora pgvector, and managed KB vector search | [RAG](concepts/rag-knowledge-bases-vector-search.md) | Verified |
| 1.5.4 | Semantic, keyword, hybrid search, and reranking | [RAG](concepts/rag-knowledge-bases-vector-search.md) | Verified |
| 1.5.5 | Query expansion, decomposition, and transformation | [RAG](concepts/rag-knowledge-bases-vector-search.md) | Verified |
| 1.5.6 | Function, MCP, and standardized retrieval interfaces | [Agents/MCP](concepts/agents-tools-mcp-and-agentcore.md) | Verified |
| 1.6.1 | Role, instruction, output templates, and guardrail controls | [Prompts](concepts/prompt-engineering-management-and-flows.md) | Verified |
| 1.6.2 | Context, intent, conversation history, and clarification | [Prompts](concepts/prompt-engineering-management-and-flows.md) | Verified |
| 1.6.3 | Prompt repository, variables, versions, governance, and audit | [Prompts](concepts/prompt-engineering-management-and-flows.md) | Verified |
| 1.6.4 | Prompt QA, edge cases, and regression testing | [Evaluation](concepts/evaluation-testing-and-quality-gates.md) | Verified |
| 1.6.5 | Structured prompts, output contracts, iterative refinement | [Prompts](concepts/prompt-engineering-management-and-flows.md) | Verified |
| 1.6.6 | Prompt chains, branches, reusable components, preprocessing | [Prompts](concepts/prompt-engineering-management-and-flows.md) | Verified |

Domain 1 coverage: **28/28 skills verified**.

## Domain 2 — Implementation and Integration

[Official Domain 2 source](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain2.html)

| Skill | Required knowledge | Canonical page | Status |
|---|---|---|---|
| 2.1.1 | Autonomous agents, state, session, long-term memory, multi-agent | [Agents](concepts/agents-tools-mcp-and-agentcore.md) | Verified |
| 2.1.2 | Structured reasoning and ReAct-style problem solving | [Agents](concepts/agents-tools-mcp-and-agentcore.md) | Verified |
| 2.1.3 | Stopping conditions, timeouts, boundaries, circuit breakers | [Agents](concepts/agents-tools-mcp-and-agentcore.md) | Verified |
| 2.1.4 | Specialized-model coordination, ensembles, and selection | [Model selection](concepts/bedrock-model-selection-and-runtime-apis.md) | Verified |
| 2.1.5 | Human review, callback, feedback, and augmentation | [Agents](concepts/agents-tools-mcp-and-agentcore.md) | Verified |
| 2.1.6 | Tool schemas, validation, error handling, and reliable operation | [Agents](concepts/agents-tools-mcp-and-agentcore.md) | Verified |
| 2.1.7 | Lambda/ECS MCP servers and consistent client access | [Agents](concepts/agents-tools-mcp-and-agentcore.md) | Verified |
| 2.2.1 | On-demand, provisioned, SageMaker, and hybrid deployment | [SageMaker deployment](concepts/sagemaker-custom-model-deployment.md) | Verified |
| 2.2.2 | LLM containers, GPU memory, token capacity, model loading | [SageMaker deployment](concepts/sagemaker-custom-model-deployment.md) | Verified |
| 2.2.3 | Smaller models, cascading, and resource-quality balance | [Efficiency](concepts/cost-latency-throughput-and-caching.md) | Verified |
| 2.3.1 | Legacy APIs, event-driven integration, and synchronization | [Enterprise integration](concepts/enterprise-integration-and-cicd.md) | Verified |
| 2.3.2 | API Gateway, Lambda webhooks, and EventBridge integrations | [Enterprise integration](concepts/enterprise-integration-and-cicd.md) | Verified |
| 2.3.3 | Federation, RBAC, and least-privilege FM/data access | [Security](concepts/security-networking-and-access-control.md) | Verified |
| 2.3.4 | Hybrid, Outposts, edge, routing, and jurisdiction constraints | [Security](concepts/security-networking-and-access-control.md) | Verified |
| 2.3.5 | GenAI gateway, CI/CD, tests, scans, controls, and rollback | [Enterprise integration](concepts/enterprise-integration-and-cicd.md) | Verified |
| 2.4.1 | Synchronous APIs, async SQS, SDKs, and request validation | [API cheat sheet](reference/api-and-payload-cheat-sheet.md) | Verified |
| 2.4.2 | Bedrock streaming, WebSockets, SSE, and incremental delivery | [API cheat sheet](reference/api-and-payload-cheat-sheet.md) | Verified |
| 2.4.3 | Backoff, throttling, fallback, degradation, and X-Ray | [Observability](concepts/observability-and-troubleshooting.md) | Verified |
| 2.4.4 | Static, content-based, and metric-based model routing | [Model selection](concepts/bedrock-model-selection-and-runtime-apis.md) | Verified |
| 2.5.1 | GenAI API interfaces, token limits, retries, and streaming | [API cheat sheet](reference/api-and-payload-cheat-sheet.md) | Verified |
| 2.5.2 | Amplify, OpenAPI, and no-code/managed workflow interfaces | [Enterprise integration](concepts/enterprise-integration-and-cicd.md) | Verified |
| 2.5.3 | CRM, document processing, Lambda, SFN, and Data Automation | [Enterprise integration](concepts/enterprise-integration-and-cicd.md) | Verified |
| 2.5.4 | Amazon Q Developer coding, refactoring, tests, and optimization | [Service catalog](reference/aws-service-decision-catalog.md) | Verified |
| 2.5.5 | Strands, Agent Squad, SFN patterns, and prompt chaining | [Agents](concepts/agents-tools-mcp-and-agentcore.md) | Verified |
| 2.5.6 | Logs Insights, X-Ray, and GenAI error recognition | [Observability](concepts/observability-and-troubleshooting.md) | Verified |

Domain 2 coverage: **25/25 skills verified**.

## Domain 3 — AI Safety, Security, and Governance

[Official Domain 3 source](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain3.html)

| Skill | Required knowledge | Canonical page | Status |
|---|---|---|---|
| 3.1.1 | Harmful-input filtering and real-time validation | [Safety](concepts/safety-privacy-and-responsible-ai.md) | Verified |
| 3.1.2 | Harmful-output prevention, moderation, deterministic results | [Safety](concepts/safety-privacy-and-responsible-ai.md) | Verified |
| 3.1.3 | Grounding, verification, confidence evidence, JSON schema | [Safety](concepts/safety-privacy-and-responsible-ai.md) | Verified |
| 3.1.4 | Defense-in-depth pre-, model-, and post-processing controls | [Safety](concepts/safety-privacy-and-responsible-ai.md) | Verified |
| 3.1.5 | Prompt injection, jailbreak, sanitization, and adversarial tests | [Safety](concepts/safety-privacy-and-responsible-ai.md) | Verified |
| 3.2.1 | VPC endpoints, IAM, Lake Formation, and access monitoring | [Security](concepts/security-networking-and-access-control.md) | Verified |
| 3.2.2 | PII discovery, runtime filtering, and retention policies | [Security](concepts/security-networking-and-access-control.md) | Verified |
| 3.2.3 | Masking, anonymization, utility preservation, and Guardrails | [Safety](concepts/safety-privacy-and-responsible-ai.md) | Verified |
| 3.3.1 | Model cards, lineage, source attribution, and decision logs | [Safety](concepts/safety-privacy-and-responsible-ai.md) | Verified |
| 3.3.2 | Data catalogs, tags, provenance, and CloudTrail audit | [Security](concepts/security-networking-and-access-control.md) | Verified |
| 3.3.3 | Organization-wide policies, regulatory and Responsible AI alignment | [D3](domains/domain-3-safety-security-governance.md) | Verified |
| 3.3.4 | Misuse/drift/policy monitoring, alerting, redaction, audit readiness | [Observability](concepts/observability-and-troubleshooting.md) | Verified |
| 3.4.1 | Explanations, uncertainty metrics, citations, and agent traces | [Safety](concepts/safety-privacy-and-responsible-ai.md) | Verified |
| 3.4.2 | Fairness metrics, A/B tests, and LLM-as-a-judge | [Evaluation](concepts/evaluation-testing-and-quality-gates.md) | Verified |
| 3.4.3 | Policy Guardrails, model cards, and automated compliance checks | [Safety](concepts/safety-privacy-and-responsible-ai.md) | Verified |

Domain 3 coverage: **15/15 skills verified**.

## Domain 4 — Operational Efficiency and Optimization

[Official Domain 4 source](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain4.html)

| Skill | Required knowledge | Canonical page | Status |
|---|---|---|---|
| 4.1.1 | Token tracking, budgets, compression, pruning, output limits | [Efficiency](concepts/cost-latency-throughput-and-caching.md) | Verified |
| 4.1.2 | Cost-capability tiers and price-to-performance | [Efficiency](concepts/cost-latency-throughput-and-caching.md) | Verified |
| 4.1.3 | Batching, capacity planning, utilization, scaling, provisioned use | [Efficiency](concepts/cost-latency-throughput-and-caching.md) | Verified |
| 4.1.4 | Semantic, edge, deterministic, and prompt caching | [Efficiency](concepts/cost-latency-throughput-and-caching.md) | Verified |
| 4.2.1 | Latency-cost tradeoffs, precompute, parallelism, streaming | [Efficiency](concepts/cost-latency-throughput-and-caching.md) | Verified |
| 4.2.2 | Index, query, filter, hybrid search, and scoring optimization | [RAG](concepts/rag-knowledge-bases-vector-search.md) | Verified |
| 4.2.3 | Token throughput, batch inference, and concurrent invocation | [Efficiency](concepts/cost-latency-throughput-and-caching.md) | Verified |
| 4.2.4 | Parameter tuning, A/B testing, temperature, top-k/top-p | [Evaluation](concepts/evaluation-testing-and-quality-gates.md) | Verified |
| 4.2.5 | FM token-capacity allocation, utilization, and scaling | [Efficiency](concepts/cost-latency-throughput-and-caching.md) | Verified |
| 4.2.6 | API profiling, vector-query tuning, latency, service communication | [Observability](concepts/observability-and-troubleshooting.md) | Verified |
| 4.3.1 | Operational, trace, FM interaction, and business observability | [Observability](concepts/observability-and-troubleshooting.md) | Verified |
| 4.3.2 | Tokens, prompts, hallucinations, quality, anomalies, invocation logs | [Observability](concepts/observability-and-troubleshooting.md) | Verified |
| 4.3.3 | Dashboards, compliance, forensic, user, and behavior insights | [Observability](concepts/observability-and-troubleshooting.md) | Verified |
| 4.3.4 | Tool calls, performance, multi-agent coordination, baselines | [Observability](concepts/observability-and-troubleshooting.md) | Verified |
| 4.3.5 | Vector-store monitoring, index optimization, and data quality | [Observability](concepts/observability-and-troubleshooting.md) | Verified |
| 4.3.6 | Golden sets, output diffs, reasoning-path traces, GenAI failure modes | [Observability](concepts/observability-and-troubleshooting.md) | Verified |

Domain 4 coverage: **16/16 skills verified**.

## Domain 5 — Testing, Validation, and Troubleshooting

[Official Domain 5 source](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain5.html)

| Skill | Required knowledge | Canonical page | Status |
|---|---|---|---|
| 5.1.1 | Relevance, factuality, consistency, fluency, and GenAI quality | [Evaluation](concepts/evaluation-testing-and-quality-gates.md) | Verified |
| 5.1.2 | Bedrock evaluations, A/B/canary, multi-model and value analysis | [Evaluation](concepts/evaluation-testing-and-quality-gates.md) | Verified |
| 5.1.3 | Ratings, feedback, annotation, and user-centered improvement | [Evaluation](concepts/evaluation-testing-and-quality-gates.md) | Verified |
| 5.1.4 | Continuous evaluation, regression, and deployment quality gates | [Evaluation](concepts/evaluation-testing-and-quality-gates.md) | Verified |
| 5.1.5 | RAG evaluation, LLM-as-judge, and human feedback | [Evaluation](concepts/evaluation-testing-and-quality-gates.md) | Verified |
| 5.1.6 | Retrieval relevance, context matching, and latency | [Evaluation](concepts/evaluation-testing-and-quality-gates.md) | Verified |
| 5.1.7 | Agent task completion, tool effectiveness, and reasoning quality | [Evaluation](concepts/evaluation-testing-and-quality-gates.md) | Verified |
| 5.1.8 | Stakeholder reports and model comparison visualization | [Evaluation](concepts/evaluation-testing-and-quality-gates.md) | Verified |
| 5.1.9 | Synthetic workflows, hallucination, drift, and release validation | [Evaluation](concepts/evaluation-testing-and-quality-gates.md) | Verified |
| 5.2.1 | Context overflow, dynamic chunking, prompt design, truncation | [Troubleshooting](concepts/observability-and-troubleshooting.md) | Verified |
| 5.2.2 | API validation, error logging, and response analysis | [Troubleshooting](concepts/observability-and-troubleshooting.md) | Verified |
| 5.2.3 | Prompt tests, version comparison, and systematic refinement | [Evaluation](concepts/evaluation-testing-and-quality-gates.md) | Verified |
| 5.2.4 | Embedding, drift, chunking, vectorization, and search diagnosis | [RAG](concepts/rag-knowledge-bases-vector-search.md) | Verified |
| 5.2.5 | Template tests, Logs, X-Ray, schema checks, and prompt maintenance | [Troubleshooting](concepts/observability-and-troubleshooting.md) | Verified |

Domain 5 coverage: **14/14 skills verified**.

## Summary

| Domain | Skills verified |
|---|---:|
| 1 | 28/28 |
| 2 | 25/25 |
| 3 | 15/15 |
| 4 | 16/16 |
| 5 | 14/14 |
| **Total** | **98/98** |

Final audit completed on 2026-07-23.
