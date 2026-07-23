# Mock Question Coverage Map

Status: 150/150 questions mapped  
Sources: two local 75-question practice exams  
Last mapped: 2026-07-23

This map identifies the primary learning objective of each question. Most professional scenarios cross domains; the “domain” column names the dominant domain(s), not an official classification from AWS.

Do not use this as an answer key. Read the canonical page, explain every constraint, and reject every distractor.

## Practice Exam 1

| Question | Domain | Primary learning objective | Canonical page |
|---|---|---|---|
| PE1-Q01 | D1/D4 | Glue Data Quality, quarantine, quality metrics | [Data processing](../concepts/data-quality-and-multimodal-processing.md) |
| PE1-Q02 | D2 | MCP runtime choice: Lambda versus ECS/Fargate | [Agents/MCP](../concepts/agents-tools-mcp-and-agentcore.md) |
| PE1-Q03 | D2 | Fast webhook acknowledgment and EventBridge fanout | [Enterprise integration](../concepts/enterprise-integration-and-cicd.md) |
| PE1-Q04 | D5 | Baseline/candidate evaluation and release quality gate | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE1-Q05 | D3 | PII masking before and after inference | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE1-Q06 | D1/D3 | Prompt versions, controlled promotion, and audit | [Prompts](../concepts/prompt-engineering-management-and-flows.md) |
| PE1-Q07 | D3 | Managed harmful-input/topic/word filtering | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE1-Q08 | D2 | Step Functions ReAct loop with retries and history | [Agents](../concepts/agents-tools-mcp-and-agentcore.md) |
| PE1-Q09 | D4/D5 | Deterministic sampling settings validated by evaluation | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE1-Q10 | D1/D4 | Managed KB PoC plus token/latency measurement | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE1-Q11 | D5 | Pipeline evaluation plus production synthetic canary | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE1-Q12 | D3 | Model cards, data lineage, and decision evidence | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE1-Q13 | D3 | Defense-in-depth prompt and response safety | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE1-Q14 | D5 | Per-response user feedback with version metadata | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE1-Q15 | D3 | Guardrails plus deterministic read-only text-to-SQL | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE1-Q16 | D2/D4 | Streaming, latency-optimized inference, and TTFT | [Efficiency](../concepts/cost-latency-throughput-and-caching.md) |
| PE1-Q17 | D1 | Hierarchical chunking for precision plus context | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE1-Q18 | D2/D4 | Bedrock batch inference for offline throughput | [Efficiency](../concepts/cost-latency-throughput-and-caching.md) |
| PE1-Q19 | D2 | Long-running human approval with SFN callback/state | [Agents](../concepts/agents-tools-mcp-and-agentcore.md) |
| PE1-Q20 | D1/D2 | Stable MCP retrieval abstraction over vector stores | [Agents/MCP](../concepts/agents-tools-mcp-and-agentcore.md) |
| PE1-Q21 | D1/D2 | AppConfig model selection, staged rollout, rollback | [Model selection](../concepts/bedrock-model-selection-and-runtime-apis.md) |
| PE1-Q22 | D3/D4 | Agent trace, citations, and measured confidence evidence | [Observability](../concepts/observability-and-troubleshooting.md) |
| PE1-Q23 | D1/D2 | Generative AI Lens and reusable IaC standards | [Enterprise integration](../concepts/enterprise-integration-and-cicd.md) |
| PE1-Q24 | D1/D3 | Managed prompt, JSON contract, and Guardrails | [Prompts](../concepts/prompt-engineering-management-and-flows.md) |
| PE1-Q25 | D3 | VPC endpoints, Lake Formation, least privilege | [Security](../concepts/security-networking-and-access-control.md) |
| PE1-Q26 | D4 | Provisioned Throughput sizing and utilization | [Efficiency](../concepts/cost-latency-throughput-and-caching.md) |
| PE1-Q27 | D2 | API Gateway-to-SQS decoupling for legacy webhook | [Enterprise integration](../concepts/enterprise-integration-and-cicd.md) |
| PE1-Q28 | D3 | Output safety and deterministic database-backed answer | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE1-Q29 | D1/D3 | Central prompts, Guardrails, CloudTrail, and logs | [Prompts](../concepts/prompt-engineering-management-and-flows.md) |
| PE1-Q30 | D3/D5 | Fairness evaluation of prompt variants by cohort | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE1-Q31 | D1 | Event-driven KB source synchronization | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE1-Q32 | D3 | Policy Guardrail, deterministic compliance check, model card | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE1-Q33 | D1 | Supported KB sources and grounded RAG | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE1-Q34 | D2 | Amplify, OpenAPI, and configurable Bedrock Flow | [Enterprise integration](../concepts/enterprise-integration-and-cicd.md) |
| PE1-Q35 | D1/D2 | SharePoint/Confluence connectors and AppFlow replication | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE1-Q36 | D2/D3 | GenAI gateway CI/CD, security tests, canary rollback | [Enterprise integration](../concepts/enterprise-integration-and-cicd.md) |
| PE1-Q37 | D1/D2 | Bedrock Data Automation and CRM workflow | [Data processing](../concepts/data-quality-and-multimodal-processing.md) |
| PE1-Q38 | D1 | Entity extraction, normalization, and prompt input quality | [Data processing](../concepts/data-quality-and-multimodal-processing.md) |
| PE1-Q39 | D3 | Identity Center federation and runtime-only permissions | [Security](../concepts/security-networking-and-access-control.md) |
| PE1-Q40 | D1 | Lowest-overhead managed Knowledge Base RAG | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE1-Q41 | D2/D4 | Parallel specialized model calls and output merge | [Efficiency](../concepts/cost-latency-throughput-and-caching.md) |
| PE1-Q42 | D1/D4 | Multi-index/hierarchical routing and shard optimization | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE1-Q43 | D5 | RAG evaluation plus ongoing user feedback | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE1-Q44 | D4/D5 | Conversation compression, CountTokens, context pruning | [Efficiency](../concepts/cost-latency-throughput-and-caching.md) |
| PE1-Q45 | D1/D4 | Geographic cross-Region inference for resilience | [Efficiency](../concepts/cost-latency-throughput-and-caching.md) |
| PE1-Q46 | D1/D3 | Grounded RAG, grounding score, structured answer | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE1-Q47 | D2/D5 | ConverseStream and SageMaker payload formatting | [API cheat sheet](../reference/api-and-payload-cheat-sheet.md) |
| PE1-Q48 | D2 | Strands tool schema, validation, and structured error | [Agents](../concepts/agents-tools-mcp-and-agentcore.md) |
| PE1-Q49 | D1/D5 | Embedding dimension/model selection by measured relevance | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE1-Q50 | D2/D3 | Agent stop condition, circuit breaker, least privilege | [Agents](../concepts/agents-tools-mcp-and-agentcore.md) |
| PE1-Q51 | D1/D4 | Hybrid exact/vector retrieval and shard tuning | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE1-Q52 | D2/D4 | Token budget, streaming, and bounded SDK retries | [API cheat sheet](../reference/api-and-payload-cheat-sheet.md) |
| PE1-Q53 | D2 | AgentCore runtime/memory and routed specialized agents | [Agents](../concepts/agents-tools-mcp-and-agentcore.md) |
| PE1-Q54 | D2/D5 | SageMaker model download and startup health timeouts | [SageMaker](../concepts/sagemaker-custom-model-deployment.md) |
| PE1-Q55 | D5 | Automated multidimensional LLM-judge evaluation | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE1-Q56 | D2 | Validated stable API for sync and async requests | [Enterprise integration](../concepts/enterprise-integration-and-cicd.md) |
| PE1-Q57 | D4 | Deterministic shared-answer edge caching | [Efficiency](../concepts/cost-latency-throughput-and-caching.md) |
| PE1-Q58 | D1 | Topic indexes and OpenSearch neural query integration | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE1-Q59 | D4/D5 | Cost/latency/quality evaluation and canary deployment | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE1-Q60 | D2/D4 | Bedrock Provisioned Throughput for predictable peak | [Efficiency](../concepts/cost-latency-throughput-and-caching.md) |
| PE1-Q61 | D1 | Hybrid search followed by Bedrock reranking | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE1-Q62 | D1/D2 | LoRA registry, approval, canary, and rollback | [SageMaker](../concepts/sagemaker-custom-model-deployment.md) |
| PE1-Q63 | D2/D4 | Auditable Step Functions model routing and fallback | [Model selection](../concepts/bedrock-model-selection-and-runtime-apis.md) |
| PE1-Q64 | D5 | Evaluation reporting with S3, Glue, Athena, and Amazon Quick Sight | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE1-Q65 | D1 | Multimodal embeddings in a shared vector space | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE1-Q66 | D2/D5 | Agent evaluation plus trace-based loop analysis | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE1-Q67 | D3/D5 | Prompt-attack runtime defense and adversarial suite | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE1-Q68 | D2/D4 | Model cascading for routine and complex requests | [Efficiency](../concepts/cost-latency-throughput-and-caching.md) |
| PE1-Q69 | D3 | Guardrail masking, Macie discovery, S3 retention | [Security](../concepts/security-networking-and-access-control.md) |
| PE1-Q70 | D1/D4 | OpenSearch shard tuning and Lambda connection reuse | [Observability](../concepts/observability-and-troubleshooting.md) |
| PE1-Q71 | D4 | Intelligent Prompt Routing and token monitoring | [Efficiency](../concepts/cost-latency-throughput-and-caching.md) |
| PE1-Q72 | D1 | Managed multimodal processing and case-packet assembly | [Data processing](../concepts/data-quality-and-multimodal-processing.md) |
| PE1-Q73 | D3 | Real-time harmful-input and PII Guardrail | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE1-Q74 | D1 | Metadata schema for current, scoped retrieval | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE1-Q75 | D2/D3 | Outposts preprocessing and private Bedrock endpoint | [Security](../concepts/security-networking-and-access-control.md) |

## Practice Exam 2

| Question | Domain | Primary learning objective | Canonical page |
|---|---|---|---|
| PE2-Q01 | D2 | HTTP API, Lambda proxy, Converse, and IaC | [API cheat sheet](../reference/api-and-payload-cheat-sheet.md) |
| PE2-Q02 | D2 | Stateful streamable-HTTP MCP on AgentCore Runtime | [Agents/MCP](../concepts/agents-tools-mcp-and-agentcore.md) |
| PE2-Q03 | D3 | ApplyGuardrail input/output defense-in-depth pipeline | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE2-Q04 | D1/D3 | Prompt variables, versions, promotion, and audit | [Prompts](../concepts/prompt-engineering-management-and-flows.md) |
| PE2-Q05 | D2/D3 | Q Business connectors, identity, and source ACLs | [Service catalog](../reference/aws-service-decision-catalog.md) |
| PE2-Q06 | D1/D4 | Titan embedding dimension/storage/relevance decision | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE2-Q07 | D5 | LLM-judge evaluation of precomputed external responses | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE2-Q08 | D3/D5 | Stratified fairness evaluation across prompt versions | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE2-Q09 | D1/D4 | Cross-Region inference profile | [Efficiency](../concepts/cost-latency-throughput-and-caching.md) |
| PE2-Q10 | D2 | SQS event source and idempotent CRM update | [Enterprise integration](../concepts/enterprise-integration-and-cicd.md) |
| PE2-Q11 | D3 | Bedrock agent trace without hidden chain-of-thought | [Observability](../concepts/observability-and-troubleshooting.md) |
| PE2-Q12 | D4 | Bedrock prompt caching for stable prefix | [Efficiency](../concepts/cost-latency-throughput-and-caching.md) |
| PE2-Q13 | D1 | Query decomposition with Retrieve and custom generation | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE2-Q14 | D2/D4 | SageMaker Batch Transform MultiRecord/concurrency | [SageMaker](../concepts/sagemaker-custom-model-deployment.md) |
| PE2-Q15 | D4 | CountTokens and per-tenant token buckets | [Efficiency](../concepts/cost-latency-throughput-and-caching.md) |
| PE2-Q16 | D5 | Retrieve-only RAG evaluation plus X-Ray | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE2-Q17 | D2/D5 | Model-aware InvokeModel serialization | [API cheat sheet](../reference/api-and-payload-cheat-sheet.md) |
| PE2-Q18 | D3 | Guardrails on normal path, human review for exceptions | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE2-Q19 | D1 | Defensible model capability/quality/operations matrix | [Model selection](../concepts/bedrock-model-selection-and-runtime-apis.md) |
| PE2-Q20 | D3 | Guardrail, model card, and fail-closed wrapper | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE2-Q21 | D3 | Macie, Comprehend, Guardrails, and S3 expiry | [Security](../concepts/security-networking-and-access-control.md) |
| PE2-Q22 | D1 | OpenSearch Serverless binary-vector KB configuration | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE2-Q23 | D1/D3 | Managed prompt variables and separate Guardrail | [Prompts](../concepts/prompt-engineering-management-and-flows.md) |
| PE2-Q24 | D1/D5 | Prompt versions, JSON contract, and feedback correlation | [Prompts](../concepts/prompt-engineering-management-and-flows.md) |
| PE2-Q25 | D2/D5 | SageMaker LMI, uncompressed model source, storage/timeouts | [SageMaker](../concepts/sagemaker-custom-model-deployment.md) |
| PE2-Q26 | D1 | KB metadata sidecars and embedding/filter fields | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE2-Q27 | D2 | Amplify conversation routes and traced Bedrock Flows | [Enterprise integration](../concepts/enterprise-integration-and-cicd.md) |
| PE2-Q28 | D1/D4 | Prompt caching and metadata-filtered retrieval context | [Efficiency](../concepts/cost-latency-throughput-and-caching.md) |
| PE2-Q29 | D3 | Custom PII regex masking and sanitized audit | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE2-Q30 | D1/D5 | Advanced Prompt Optimization with evaluation samples | [Prompts](../concepts/prompt-engineering-management-and-flows.md) |
| PE2-Q31 | D1/D2 | Native/OpenAI-compatible APIs and capacity selection | [Model selection](../concepts/bedrock-model-selection-and-runtime-apis.md) |
| PE2-Q32 | D2/D5 | CodeBuild reports and supported pipeline rollback | [Enterprise integration](../concepts/enterprise-integration-and-cicd.md) |
| PE2-Q33 | D4/D5 | KB/OpenSearch metrics and ingestion logs | [Observability](../concepts/observability-and-troubleshooting.md) |
| PE2-Q34 | D2 | Cross-account reviewer with same-account callback completion | [Agents](../concepts/agents-tools-mcp-and-agentcore.md) |
| PE2-Q35 | D2/D3 | Outposts, private endpoint, EU inference profile | [Security](../concepts/security-networking-and-access-control.md) |
| PE2-Q36 | D1/D4 | Domain vector indexes, shards, and global top-k merge | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE2-Q37 | D1/D5 | Managed PoC evaluation of model/prompt candidates | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE2-Q38 | D2 | Stable multi-provider adapter with AppConfig routing | [Enterprise integration](../concepts/enterprise-integration-and-cicd.md) |
| PE2-Q39 | D4/D5 | Sanitized structured logs and X-Ray correlation | [Observability](../concepts/observability-and-troubleshooting.md) |
| PE2-Q40 | D2 | Auditable agent loop and long human callback | [Agents](../concepts/agents-tools-mcp-and-agentcore.md) |
| PE2-Q41 | D5 | Caller-initiated AgentCore trajectory evaluation | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE2-Q42 | D3 | Guardrail thresholds, detect mode, and PII masking | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE2-Q43 | D1/D2 | Bedrock Flows for prompt sequence and branching | [Prompts](../concepts/prompt-engineering-management-and-flows.md) |
| PE2-Q44 | D1 | Managed KB ingestion with OpenSearch Serverless | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE2-Q45 | D3 | User citations plus agent orchestration trace | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE2-Q46 | D3 | Model cards, evaluation evidence, lineage, invocation logs | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE2-Q47 | D5 | Inspect Retrieve output to isolate ranking issue | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE2-Q48 | D2 | AgentCore Gateway Lambda tool schema and dispatch | [Agents](../concepts/agents-tools-mcp-and-agentcore.md) |
| PE2-Q49 | D4/D5 | Evaluate deterministic parameters for structured output | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE2-Q50 | D2 | Amazon Q Developer in IDE and release preparation | [Service catalog](../reference/aws-service-decision-catalog.md) |
| PE2-Q51 | D1 | Nested JSON Glue Data Quality and quarantine | [Data processing](../concepts/data-quality-and-multimodal-processing.md) |
| PE2-Q52 | D5 | Synthetics deployed-path test plus RAG evaluation | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE2-Q53 | D2 | Auditable classification and model routing in SFN | [Model selection](../concepts/bedrock-model-selection-and-runtime-apis.md) |
| PE2-Q54 | D4 | Reduce oversized `maxTokens` to improve token throughput | [Efficiency](../concepts/cost-latency-throughput-and-caching.md) |
| PE2-Q55 | D5 | Automated/human evaluation, cost metrics, and canary | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE2-Q56 | D2 | Converse common API and capability-checked streaming | [API cheat sheet](../reference/api-and-payload-cheat-sheet.md) |
| PE2-Q57 | D1/D3 | Citations, retrieval evidence, and groundedness checks | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE2-Q58 | D1/D4 | KB ingestion refresh and terminal status tracking | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE2-Q59 | D3 | Audit Manager Generative AI evidence framework | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE2-Q60 | D2/D3 | SFN cycle limit, circuit breaker, least-privilege tools | [Agents](../concepts/agents-tools-mcp-and-agentcore.md) |
| PE2-Q61 | D2 | Converse for chat and InvokeModel for specialized models | [API cheat sheet](../reference/api-and-payload-cheat-sheet.md) |
| PE2-Q62 | D2/D4 | ConverseStream, optimized latency, and TTFT metrics | [Efficiency](../concepts/cost-latency-throughput-and-caching.md) |
| PE2-Q63 | D3 | Guardrail prompt-attack blocking and tagged input | [Safety](../concepts/safety-privacy-and-responsible-ai.md) |
| PE2-Q64 | D3 | Private endpoints, Lake Formation, and access alarms | [Security](../concepts/security-networking-and-access-control.md) |
| PE2-Q65 | D2 | AgentCore Memory actor/session strategy and flush | [Agents](../concepts/agents-tools-mcp-and-agentcore.md) |
| PE2-Q66 | D2/D4 | REST/Lambda response streaming with SSE | [API cheat sheet](../reference/api-and-payload-cheat-sheet.md) |
| PE2-Q67 | D1/D2 | Reusable CDK constructs and GenAI Lens reviews | [Enterprise integration](../concepts/enterprise-integration-and-cicd.md) |
| PE2-Q68 | D5 | Human comparison with bring-your-own legacy responses | [Evaluation](../concepts/evaluation-testing-and-quality-gates.md) |
| PE2-Q69 | D4 | Runtime metrics, invocation logs, request metadata | [Observability](../concepts/observability-and-troubleshooting.md) |
| PE2-Q70 | D1/D2 | AgentCore Gateway managed KB retrieval tools | [Agents](../concepts/agents-tools-mcp-and-agentcore.md) |
| PE2-Q71 | D2 | DynamoDB session state and deterministic clarification workflow | [Agents](../concepts/agents-tools-mcp-and-agentcore.md) |
| PE2-Q72 | D1 | BDA, Transcribe, and SageMaker Processing by modality | [Data processing](../concepts/data-quality-and-multimodal-processing.md) |
| PE2-Q73 | D1 | KB hybrid search and Bedrock reranking configuration | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |
| PE2-Q74 | D3 | IAM requirement for approved Guardrail on runtime calls | [Security](../concepts/security-networking-and-access-control.md) |
| PE2-Q75 | D1/D3 | Managed KB connectors, document ACLs, and refresh | [RAG](../concepts/rag-knowledge-bases-vector-search.md) |

## Coverage totals

- Practice Exam 1: 75/75 mapped.
- Practice Exam 2: 75/75 mapped.
- Total: **150/150 mapped**.
