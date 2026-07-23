# AIP-C01 Glossary

Status: Exam-oriented definitions  
Last verified: 2026-07-23

## A–F

**Action group** — A set of functions/APIs an Amazon Bedrock agent can invoke, commonly through Lambda.

**Agent** — A system that uses an FM to decide steps and tools toward a goal. Production agents also need state, permissions, stop rules, error handling, and observability.

**Agent trace** — Events describing orchestration, KB use, action groups, and failures. It is not provider-internal chain-of-thought.

**AgentCore** — Amazon Bedrock services for operating agents, including runtime, Gateway, Memory, and related capabilities.

**AIP-C01** — AWS Certified Generative AI Developer – Professional exam code.

**At-least-once delivery** — A message/event can be delivered more than once; consumers need idempotency.

**Batch inference** — Asynchronous processing of many model inputs, typically through S3 input/output.

**Canary deployment** — Expose a small traffic share to a candidate before wider promotion.

**Chain-of-thought (CoT)** — Intermediate reasoning tokens/steps. Do not assume they are available or appropriate to expose; use audit traces and evidence.

**Chunk** — A document segment indexed and retrieved in RAG.

**Circuit breaker** — Stops calls to an unhealthy dependency for a cooldown period after repeated failures.

**Context window** — Maximum model input plus output token budget under the model’s rules.

**Context relevance** — Whether retrieved context relates to the question.

**Context coverage** — Whether retrieved context contains the information needed for a correct answer.

**Contextual grounding** — Checking whether a response is supported by supplied source/context.

**Converse** — Bedrock common message-style inference API for supported models.

**ConverseStream** — Streaming form of the common Bedrock conversation interface.

**Cross-Region inference profile** — Bedrock routing abstraction that can use capacity across permitted Regions for supported models.

**Data lineage** — Traceability from source through transformations to output.

**DQDL** — AWS Glue Data Quality Definition Language for declarative quality rules.

**Embedding** — Numeric vector representing semantic or multimodal content.

**FM** — Foundation model.

**Flow** — Managed graph of prompts, conditions, KBs, Lambda, and other nodes in Amazon Bedrock.

**Function/tool calling** — FM emits a structured request for application-defined capability; the application validates and executes it.

## G–M

**Golden dataset** — Stable, curated benchmark used to detect regression.

**Guardrail** — Bedrock policy controls for content, topics, sensitive information, prompt attacks, grounding, and other supported checks.

**Hallucination** — Unsupported or fabricated model output. Fluency does not prevent it.

**Hierarchical chunking** — Index/retrieve smaller child chunks but return associated larger parent context.

**HNSW** — Graph-based approximate nearest-neighbor index commonly used for vector search.

**Human-in-the-loop** — A person reviews, edits, approves, or labels a decision/output.

**Hybrid search** — Combines lexical/keyword and vector semantic search.

**Idempotency** — Repeating a request with the same key does not duplicate a side effect.

**Inference profile** — Bedrock resource used to route supported model inference, including cross-Region profiles.

**Jailbreak** — Input crafted to bypass model safety behavior.

**Knowledge Base** — Bedrock managed RAG component for ingestion, vector integration, retrieval, and supported generation/citations.

**LLM** — Large language model.

**LLM-as-a-judge** — An LLM scores another model’s output against a rubric/reference. It must be calibrated for bias.

**LoRA** — Low-rank adaptation, a parameter-efficient model customization technique.

**MCP** — Model Context Protocol, a standard interface for agents/clients to discover and call tools/resources.

**Metadata filter** — Restricts retrieval by structured attributes such as date, tenant, owner, or jurisdiction.

**Model card** — Governance artifact describing intended use, limitations, risk, metrics, and model details.

**Model cascading** — Use a smaller model first and escalate selected requests to a stronger model.

**Model invocation logging** — Bedrock feature for recording supported model invocation data to configured destinations.

**Multimodal** — Handles more than one modality such as text, image, audio, or video.

## N–R

**nDCG** — Normalized Discounted Cumulative Gain, a ranking-quality metric rewarding relevant results near the top.

**On-demand inference** — Pay-per-use/shared capacity model invocation without dedicated throughput commitment.

**OpenSearch Serverless** — Managed serverless OpenSearch collections, including supported vector configurations.

**PII** — Personally identifiable information.

**Prompt** — Instructions and context supplied to an FM.

**Prompt attack/injection** — Untrusted content attempts to override instructions, reveal secrets, or cause unsafe tool/model behavior.

**Prompt caching** — Reuse processing of an eligible stable prompt prefix; not a completed-answer cache.

**Prompt Management** — Bedrock capability for reusable prompts, variables, variants, testing, and versions.

**Provisioned Throughput** — Reserved Bedrock model capacity for supported predictable workloads.

**RAG** — Retrieval Augmented Generation: retrieve external evidence and supply it to generation.

**Reranker** — Model/algorithm that reorders retrieved candidates for better relevance.

**Responsible AI** — Operational principles including fairness, safety, privacy, transparency, robustness, and accountability.

**Retrieve** — Knowledge Bases runtime operation returning relevant content and metadata without final generation.

**RetrieveAndGenerate** — Knowledge Bases runtime operation that combines retrieval with model generation and citations for supported non-Managed Knowledge Base configurations. The product named Managed Knowledge Bases uses `Retrieve` or `AgenticRetrieveStream`.

**Bedrock Mantle** — Amazon Bedrock endpoint family for supported OpenAI-compatible and Anthropic APIs, stateful conversation, and agentic capabilities. Bedrock-native `Converse` and `InvokeModel` remain on `bedrock-runtime`.

## S–Z

**Semantic cache** — Reuses results for meaning-equivalent requests; requires careful quality and authorization boundaries.

**Semantic drift** — Output behavior/meaning changes across model, prompt, data, or time.

**Server-sent events (SSE)** — One-way incremental events over HTTP, useful for browser token streaming.

**Sidecar metadata file** — File stored beside a source document containing Knowledge Base metadata and embedding/filter controls where supported.

**Step Functions callback** — Workflow task pauses until a trusted component returns a task token with success/failure.

**Strands Agents** — AWS open-source SDK for building model-driven agents with tools and related patterns.

**Structured output** — Output constrained/validated against a predictable schema such as JSON.

**Temperature** — Sampling parameter affecting randomness; lower is usually more deterministic, but evaluation is required.

**Token** — Model-specific unit of input/output text or content representation used for context and billing/capacity.

**Token bucket** — Admission/rate algorithm that allocates capacity units over time; useful per tenant.

**Tool schema** — Structured definition of tool name, purpose, arguments, types, and requirements.

**Top-k** — Restricts sampling to the k highest-probability next tokens.

**Top-p** — Nucleus sampling over the smallest token set reaching a cumulative probability.

**Trace** — Structured execution-path evidence across services, agent steps, or Flow nodes.

**TTFT** — Time to first token, measuring streaming responsiveness.

**Vector database** — Store/index optimized for similarity search over embeddings.

**Vector dimension** — Number of values in an embedding; must match the index field and affects storage/performance.

**WebSocket** — Bidirectional persistent network connection used for interactive streaming.

**Zero-shot/few-shot** — Prompting with no examples or a small set of examples.
