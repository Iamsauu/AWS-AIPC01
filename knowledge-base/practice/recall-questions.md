# Active Recall Questions

Use these without answer options. Write or speak the answer before checking the key.

## Domain 1

1. Which dimensions belong in a defensible FM selection matrix?
2. How do on-demand, Provisioned Throughput, batch, and SageMaker endpoint deployment differ?
3. When does cross-Region inference solve a problem that Provisioned Throughput does not?
4. How can AppConfig enable model switching and safe rollback?
5. Why must a PoC measure business value as well as model quality?
6. How do Glue Data Quality and Data Wrangler differ?
7. Which services would process PDF tables, call audio, noisy text, and nested JSON?
8. Why must model input formatting be model/API aware?
9. What tradeoff does embedding dimension create?
10. When is OpenSearch preferable to Aurora pgvector?
11. Explain the complete RAG pipeline.
12. Compare fixed, semantic, and hierarchical chunking.
13. When should metadata influence embeddings, and when should it only filter?
14. Why does exact identifier search need a lexical component?
15. What problem does reranking solve?
16. Compare `Retrieve` and `RetrieveAndGenerate`.
17. How do you keep a KB current without unnecessary full rebuilds?
18. What does a citation prove, and what does it not prove?
19. What belongs in a robust prompt template?
20. What do Prompt Management versions provide, and what approval control remains external?

## Domain 2

21. What components make an agent safe to operate?
22. When is Step Functions better than an FM-controlled loop?
23. How does a callback task avoid polling for human approval?
24. What should a tool schema and tool implementation each validate?
25. How does a circuit breaker prevent token waste?
26. When should an MCP server run on Lambda, Fargate, or AgentCore Runtime?
27. How do session memory and long-term memory differ?
28. Why should an AgentCore/agent memory session be explicitly closed or flushed?
29. Compare Bedrock Flows, Step Functions, and agents.
30. What makes an event-driven integration loosely coupled?
31. When do you choose SQS, EventBridge, or SNS?
32. Why does a webhook handler often need to acknowledge before invoking the FM?
33. How do you make an asynchronous side effect idempotent?
34. Describe a centralized GenAI gateway.
35. Which tests belong before a canary deployment?
36. Compare Converse and InvokeModel.
37. What must be checked before enabling ConverseStream?
38. Which errors should the SDK retry?
39. How do HTTP API, REST API, and WebSocket API differ?
40. What conditions justify SageMaker LMI rather than Bedrock?

## Domain 3

41. Describe a defense-in-depth FM request path.
42. Compare attached Guardrails and ApplyGuardrail.
43. How can IAM prevent production callers from omitting the approved Guardrail?
44. What is the difference between a prompt attack and ordinary harmful content?
45. Why are read-only parameterized SQL templates safer than generated free-form SQL?
46. Compare Macie, Comprehend, and Guardrails for PII.
47. How can masking preserve model utility?
48. What controls are still required with a VPC endpoint?
49. How do IAM Identity Center and Cognito differ?
50. What belongs in a model card?
51. Compare CloudTrail, invocation logging, and application logs as audit evidence.
52. How should runtime logs handle sensitive prompts?
53. How do citations and agent traces provide different transparency?
54. Why is a vector similarity score not model confidence?
55. How do you evaluate fairness without hiding cohort failures?

## Domain 4

56. Why can a low request rate still cause throttling?
57. How should a context budget be assembled?
58. What does prompt caching reuse?
59. Which fields belong in a safe deterministic response cache key?
60. When is semantic caching too risky?
61. Compare model cascading, managed intelligent routing, and explicit SFN routing.
62. Why do retries amplify overload?
63. Which changes improve throughput before buying provisioned capacity?
64. How do TTFT and total latency differ?
65. Why can streaming leave total latency unchanged?
66. How can Lambda connection reuse improve a RAG path?
67. What vector-index properties can increase query latency?
68. Why is quality-per-cost better than token price alone?
69. Which metrics detect a prompt-cache configuration failure?
70. What evidence supports a cross-Region inference decision?

## Domain 5

71. How do you isolate retrieval quality from answer generation?
72. What makes an evaluation dataset representative?
73. Compare exact validation, reference scoring, LLM-as-judge, and human evaluation.
74. How do you calibrate an LLM judge?
75. Which metrics matter for RAG?
76. Which metrics matter for an agent?
77. What belongs in a release quality gate?
78. Why is a canary complementary to offline evaluation?
79. What metadata should be stored with user feedback?
80. How do you diagnose an unsupported answer?
81. How do you diagnose ignored generation parameters?
82. How do you diagnose repeated tool loops?
83. How do you distinguish retrieval latency from model latency?
84. Why should one variable change at a time during troubleshooting?
85. What qualifies two unseen mock exams as a better readiness signal?

## Short answer key

1. Modality, quality, task/tool support, context, APIs, streaming, Region, lifecycle, latency, throughput, cost, safety, language/domain fit.
2. Bursty managed; predictable reserved; offline asynchronous; custom serving control.
3. It routes around regional capacity disruption; it does not reserve dedicated capacity.
4. Externalize validated model/routing config, deploy gradually, alarm, roll back.
5. A technically good answer can fail economics or user outcome.
6. Automated rule pipeline versus interactive preparation.
7. BDA/Textract; Transcribe; Comprehend; Glue Data Quality/processing.
8. Provider/model schemas and supported parameters differ.
9. Storage/latency versus information/relevance; evaluate before reducing.
10. Search/hybrid scale versus relational filtering/transactions.
11. Source, parse, chunk, embed, index, retrieve, filter/rerank, prompt, generate, cite, evaluate.
12. Uniform size; structure/meaning; child match with parent context.
13. Semantic descriptive fields may embed; date/owner/tenant often filter.
14. Embeddings can miss exact rare tokens.
15. It improves ordering of already retrieved candidates.
16. Chunks/control versus managed final answer/citations.
17. Change-triggered or scheduled ingestion plus terminal status monitoring.
18. It points to source evidence, not guaranteed factuality.
19. Role, task, context, constraints, examples, output schema, failure behavior.
20. Reusable snapshots; organizational promotion/approval via IAM/pipeline.
21. Instructions, tools, state, permissions, stop rules, errors, human review, trace/evaluation.
22. Deterministic high-risk branches, callbacks, retries, and audit.
23. Workflow pauses on token and resumes from trusted callback.
24. Schema guides structure; implementation enforces semantics, auth, idempotency.
25. It stops repeated calls during dependency failure.
26. Lightweight stateless; heavy/container; managed agent-native/stateful needs.
27. Current conversation versus durable preferences/facts across sessions.
28. Buffered writes may otherwise be lost.
29. Managed prompt graph; durable deterministic orchestration; dynamic tool reasoning.
30. Producer and consumers communicate through event contract, not direct dependency.
31. Buffer; route; notify.
32. External latency and burst must not block acknowledgment.
33. Stable business key plus deduplicating/conditional downstream update.
34. Stable validated API, auth, throttling, routing, safety, audit, observability.
35. Unit/schema, security, prompt regression, guardrail, integration, quality.
36. Common supported chat interface versus provider-native body.
37. Model capability, API support, client path, midstream safety/errors.
38. Transient throttles/timeouts/service failures, bounded with jitter.
39. Simple HTTP; richer REST; bidirectional persistent.
40. Unsupported/custom model or specialized serving/GPU control.
41. Validate, redact, input Guardrail, restricted model/tools, output Guardrail, deterministic validation.
42. In-call policy versus independent input/output evaluation.
43. IAM condition requiring guardrail identifier/version on supported actions.
44. Instruction manipulation versus unsafe semantic content.
45. It constrains operation and parameters deterministically.
46. S3 discovery; live NLP detection; model input/output policy.
47. Stable placeholders preserve entity relationships.
48. IAM, endpoint/resource policies, encryption, governed data permissions.
49. Workforce AWS access versus application end-user identity.
50. Intended use, limits, risk, version, metrics, evidence.
51. AWS API audit; supported FM invocation evidence; runtime business context.
52. Redact/minimize, encrypt, restrict, expire, or disable payload logging.
53. Source support versus orchestration/tool path.
54. Similarity is not truth probability.
55. Matched/stratified cases and separate cohort metrics.
56. Token rate/output budget can be high.
57. Reserve output, mandatory instructions, user, best evidence, recent/summarized history, count/prune.
58. Eligible stable prefix processing.
59. Request, model, prompt, parameters, data/index, tenant/auth, guardrail.
60. Personalized/sensitive/high-risk or weak equivalence boundaries.
61. Escalation; managed complexity route; deterministic auditable route.
62. More in-flight work competes for exhausted capacity.
63. Trim, limit output, batch, queue, cache, route, reuse connections.
64. First visible token versus complete response.
65. Same generation work is delivered incrementally.
66. Avoid repeated TCP/TLS/client construction overhead.
67. Excess shards/fanout, pressure, dimensions, candidates, duplicates, filters.
68. Cheap low-quality calls can create retries/escalation/rework.
69. Cache read/write tokens, hit ratio, TTFT/input cost.
70. Allowed Regions, model/profile support, residency, failure objective.
71. Run Retrieve/retrieve-only evaluation without generation.
72. Production-like, stratified, edge/safety cases, versioned holdout.
73. Deterministic; known target; scalable semantic; subjective/high-risk.
74. Rubric, reference, randomization, human calibration, judge version.
75. Context relevance/coverage, faithfulness, citations, rank, latency.
76. Completion, tool selection/args, loops, recovery, safety, cost/latency.
77. Static, functional, safety, benchmark thresholds, cohort limits, rollback.
78. Offline predicts quality; canary exposes limited real production behavior.
79. Model/prompt/KB/guardrail versions, safe cohort/experiment, rating/correction, correlation.
80. Inspect Retrieve results, metadata, citations, grounding, then prompt.
81. Verify API/model schema and supported fields.
82. Trace tool errors/args/cycle count; add validation/stops/breaker.
83. Distributed trace and component metrics.
84. Preserve causal attribution.
85. Timed, truly unseen, domain-balanced, no reused wording.

