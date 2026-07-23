# Domain 5: Testing, Validation, and Troubleshooting

Status: Verified against the current blueprint  
Exam weight: 11%  
Official tasks: 5.1, 5.2  
Last verified: 2026-07-23

## Domain objective

Prove that models, prompts, retrieval, agents, and deployed applications meet defined quality, safety, performance, and business requirements—and diagnose failures from evidence instead of random prompt changes.

Canonical deep dives:

- [Evaluation, testing, and quality gates](../concepts/evaluation-testing-and-quality-gates.md)
- [Observability and troubleshooting](../concepts/observability-and-troubleshooting.md)
- [RAG and vector search](../concepts/rag-knowledge-bases-vector-search.md)
- [Bedrock model selection and runtime APIs](../concepts/bedrock-model-selection-and-runtime-apis.md)
- [SageMaker custom model deployment](../concepts/sagemaker-custom-model-deployment.md)

This page remains complete at exam depth without those pages.

## Complete official-skill map

| Skill | Required exam capability |
|---|---|
| 5.1.1 | Evaluate GenAI output with relevance, factual accuracy, consistency, fluency, safety, and business metrics |
| 5.1.2 | Compare models/configurations with Bedrock evaluations, A/B/canary tests, and cost-latency-quality analysis |
| 5.1.3 | Collect ratings, preferences, comments, annotations, and user-experience feedback with version metadata |
| 5.1.4 | Run continuous evaluation, regression suites, and automated deployment quality gates |
| 5.1.5 | Combine RAG, LLM-as-a-judge, deterministic, and human evaluation |
| 5.1.6 | Test retrieval relevance, context match/coverage, ranking, and latency independently of generation |
| 5.1.7 | Evaluate agent task completion, tool use, trajectory, reasoning/workflow quality, safety, latency, and efficiency |
| 5.1.8 | Create stakeholder reports, trends, and model comparisons |
| 5.1.9 | Validate deployments with synthetic workflows, hallucination/semantic-drift checks, canaries, and rollback criteria |
| 5.2.1 | Diagnose context overflow, content loss, truncation, chunking, and prompt-size issues |
| 5.2.2 | Diagnose FM API, endpoint, payload, permission, throttling, timeout, and SageMaker deployment problems |
| 5.2.3 | Troubleshoot prompt behavior systematically with fixed tests, versions, and controlled refinement |
| 5.2.4 | Isolate retrieval, embedding, vectorization, chunking, metadata, drift, ranking, and search-performance failures |
| 5.2.5 | Maintain prompts with templates, logs, traces, schema checks, version comparisons, and regression workflows |

## Evaluation starts with a contract

Define before testing:

1. Use case and user population.
2. Expected input distribution and edge cases.
3. Acceptable and prohibited behavior.
4. Quality, safety, fairness, latency, cost, and business metrics.
5. Threshold for each release criterion.
6. Required confidence/sample size.
7. Failure action: block, retry, fallback, human review, or rollback.

“The answer looks good” is not a release criterion.

## Evaluation layers

| Layer | Question | Example evidence |
|---|---|---|
| Deterministic | Is the contract mechanically valid? | JSON Schema, required fields, allowed values, SQL/tool policy |
| Model output | Is the answer correct, relevant, complete, safe, and useful? | Bedrock model evaluation, custom metrics, human rating |
| Retrieval | Did the system retrieve the right evidence? | Context relevance/coverage, recall@k, nDCG, latency |
| RAG generation | Did the answer use evidence faithfully and cite it correctly? | Faithfulness, citation precision/coverage |
| Agent | Did the agent complete the task with correct and efficient tools? | Goal success, tool trajectory, repeated calls, trace |
| Application | Does the deployed API work through the real path? | Synthetic canary, error/latency/availability |
| Business/user | Does it improve the intended outcome? | Resolution, conversion, escalation, satisfaction |

No single evaluator covers every layer.

## Dataset design

A strong dataset is:

- Representative of production tasks and token sizes.
- Stratified by use case, tenant/cohort, language, and risk.
- Rich in long-tail, ambiguous, adversarial, and failure cases.
- Versioned and immutable for a given comparison.
- Separate from prompt/model development examples where possible.
- Annotated with expected response, source context, assertions, tool trajectory, or rubric when available.
- Large enough to expose category-specific regressions.

Recommended partitions:

```text
development set -> prompt iteration
validation set  -> parameter/model selection
locked test set -> release decision
production sample -> continuous evaluation
```

Avoid benchmark leakage. If developers repeatedly optimize against the test set, it is no longer an unseen test.

For stochastic output, run repeated trials or use score distributions and thresholds. Exact string equality is appropriate only for genuinely deterministic fields.

## Task 5.1: Evaluation systems

### 5.1.1 Output-quality framework

Core dimensions:

| Dimension | Meaning |
|---|---|
| Correctness/factual accuracy | Claims are accurate against reference or authoritative facts |
| Completeness | All required aspects are addressed |
| Relevance | Response addresses the user request |
| Faithfulness/groundedness | Claims are supported by supplied context |
| Logical coherence | No contradictions or gaps |
| Fluency/style | Clear, grammatical, suitable tone |
| Consistency | Equivalent/repeated cases do not vary beyond tolerance |
| Helpfulness | Response supports the user's goal |
| Harmfulness/safety | Unsafe or prohibited content rate |
| Fairness | Outcome/quality disparity across relevant cohorts |

Add domain-specific deterministic metrics: required disclaimers, exact codes, prohibited claims, citation presence, valid schema, or calculation match.

Traditional lexical metrics can support narrow tasks, but overlap alone does not measure semantic correctness, usefulness, or hallucination.

### 5.1.2 Model and configuration comparison

Amazon Bedrock supports:

- Programmatic/automatic model evaluation with built-in or custom datasets.
- LLM-as-a-judge evaluation with built-in or custom metrics.
- Human evaluation with a private work team.
- Evaluation of selected Bedrock resources and pre-generated responses from external models.

For LLM-as-a-judge, a generator produces the response and an evaluator model scores it and explains the score. Results are written to S3; current dataset formats use JSONL in S3 and support optional reference responses and categories.

Compare:

- Model and model version.
- Prompt version.
- Inference parameters.
- Tool/retrieval configuration.
- Quality by category and cohort.
- Input/output tokens.
- TTFT and total latency.
- Cost per successful task.
- Safety and refusal behavior.

Offline evaluation selects candidates. A/B testing compares live behavior; a canary limits deployment risk. They are related but not interchangeable.

### 5.1.3 User-centered evaluation

Collect:

- Per-response thumbs/rating.
- Pairwise preference.
- Optional reason/comment.
- Edit distance between draft and final human-approved text.
- Escalation or abandonment.
- Task completion.

Store the model, prompt, guardrail, knowledge-base/data version, request category, and anonymized user/cohort context. Without version metadata, feedback cannot identify the regression.

Feedback limitations:

- It is sparse and self-selected.
- Users may reward confidence over correctness.
- Different groups rate differently.
- A click or abandonment can have many causes.

Use feedback as one signal, not ground truth.

### 5.1.4 Quality assurance and gates

A production release flow:

```text
change
  -> unit/schema/security tests
  -> locked regression dataset
  -> model/RAG/agent evaluation
  -> compare to baseline and thresholds
  -> human review for high-risk changes
  -> deploy canary
  -> synthetic and live quality monitoring
  -> promote or automatically roll back
```

Store test reports and artifacts by release. Fail the pipeline when a hard threshold fails. Do not merely publish a dashboard and continue deployment.

Continuous evaluation samples production traffic after privacy controls, monitors drift, and feeds reviewed failures back into the regression set.

### 5.1.5 Multi-perspective evaluation

Use the evaluator suited to the check:

| Evaluator | Strength | Limitation |
|---|---|---|
| Deterministic code | Fast, repeatable, exact | Cannot judge open-ended nuance |
| Traditional task metric | Comparable and cheap | Often misses semantic quality |
| LLM-as-a-judge | Scalable semantic rubric and explanations | Can be biased, inconsistent, or self-preferential |
| Human SME | High-context domain judgment | Slow, costly, variable |
| User feedback | Real-world preference/outcome | Biased and sparse |

Calibrate automated judges against human-labeled examples. Mitigate order/position bias with randomized presentation and judge prompts. Do not use the same model’s self-score as unquestioned truth.

### 5.1.6 Retrieval quality

Evaluate retrieval **before** generation by calling `Retrieve` or the relevant retriever directly.

Bedrock RAG evaluation types:

- Retrieve only.
- Retrieve and generate.

Current Bedrock built-in retrieve-only metrics include:

- Context relevance.
- Context coverage, which requires ground truth.

Useful custom/offline ranking metrics include:

- Precision@k.
- Recall@k.
- Mean reciprocal rank (MRR).
- nDCG@k.
- Hit rate.
- Retrieval latency.

Retrieve-and-generate metrics include correctness, completeness, helpfulness, logical coherence, faithfulness, citation precision, citation coverage, and harmfulness.

Interpretation:

| Retrieval | Generation | Diagnosis |
|---|---|---|
| Poor | Poor | Fix ingestion/retrieval first |
| Good | Poor | Fix prompt/model/grounding |
| Poor | Apparently good | Model may be answering from prior knowledge; unsafe for governed RAG |
| Good | Good | Optimize cost/latency without crossing thresholds |

### 5.1.7 Agent evaluation

Measure:

- Goal/task completion.
- Correct final response.
- Tool selection accuracy.
- Tool input validity.
- Expected versus actual tool trajectory.
- Number of steps and repeated calls.
- Safe stopping and policy compliance.
- Recovery from tool error.
- Latency and tokens per successful task.
- Human intervention rate.

Current AgentCore Evaluations supports agent/session/trace/tool assessment with built-in, LLM-as-a-judge, and custom code evaluators, plus online, on-demand, batch, dataset, and simulation patterns. Ground-truth inputs can include expected responses, assertions, and expected tool trajectories for supported evaluators.

For classic Bedrock agents, preserve invocation traces to inspect knowledge-base and action-group decisions. A successful final sentence can hide wasteful loops or an unsafe attempted tool call.

### 5.1.8 Reporting

Report:

- Baseline and candidate by metric.
- Category/cohort distribution, not only global average.
- Confidence interval or sample count.
- Failure examples and severity.
- Token, latency, and cost tradeoffs.
- Release decision and threshold.
- Trend by version/time.

An AWS-native reporting pattern is evaluation output in S3, Glue Data Catalog/Athena for query, and Amazon Quick Sight for dashboards. CloudWatch is suitable for operational and continuous time-series alarms.

### 5.1.9 Deployment validation

Use synthetic user workflows to exercise the deployed API, authentication, retrieval, model, guardrail, and response contract. A model evaluation job alone does not prove the real API path works.

Canary checks:

- Availability and latency.
- Required schema and fields.
- Expected semantic assertions.
- Hallucination/faithfulness threshold.
- Prompt/model/guardrail version.
- Error and throttle rate.

Gradually shift traffic with a Lambda alias/CodeDeploy, AppConfig, SageMaker deployment guardrail, or appropriate delivery mechanism. Automatically roll back on operational or AI-quality gates.

## Task 5.2: Troubleshooting

## Evidence-first troubleshooting loop

1. Reproduce with a versioned request and correlation ID.
2. Identify the failing layer.
3. Compare with a known-good baseline.
4. Inspect the smallest relevant evidence.
5. Change one variable.
6. Rerun the fixed evaluation set.
7. Roll back if the threshold remains violated.

Do not begin by “improving the prompt” when retrieval or the API contract is broken.

### 5.2.1 Content handling, overflow, and truncation

Symptoms:

- Missing beginning/middle/end of source.
- Response stops abruptly.
- Required fields disappear.
- Validation error for context length.
- Older instructions are ignored.

Inspect:

- `CountTokens` for the exact model/request.
- Tokens by system, tools, history, retrieval, user, and output budget.
- Configured `maxTokens`.
- Stop reason/sequence.
- Retrieved chunk count and overlap.
- Application-side truncation, serialization, API Gateway/Lambda limits.

Remediate:

- Prune irrelevant context and duplicate chunks.
- Summarize old conversation with critical state preserved separately.
- Use smaller/hierarchical/dynamic chunks based on the evidence.
- Split documents and aggregate summaries.
- Limit tool definitions.
- Increase output maximum only when output—not input context—is truncated.
- Select a suitable context model only after cost/quality evaluation.

Silent truncation is worse than a clear reject/abstain because it can produce plausible incomplete answers.

### 5.2.2 FM integration and deployment

#### Bedrock API error triage

| Error/symptom | Likely cause | Correct response |
|---|---|---|
| `AccessDeniedException` / 403 | IAM/SCP/endpoint policy, model access, KMS | Fix authorization; do not retry blindly |
| `ValidationException` / 400 | Wrong endpoint/payload, missing field, unsupported parameter, size/schema | Compare with API/model contract |
| `ResourceNotFound` / 404 | Wrong ID/ARN/Region/version | Verify resource and Region |
| `ThrottlingException` / 429 | Token/request quota or capacity | Backoff+jitter, reduce token reservation, capacity option |
| 5xx / `ServiceUnavailable` | Transient service issue | Bounded SDK retry, fallback where designed |
| Timeout/reset | Client timeout, long stream, NAT/endpoint idle behavior, downstream delay | Trace path; adjust client/network only after locating cause |
| Output ignores parameters | Provider-native payload field mismatch | Map internal request to selected model schema |

API checks:

- `bedrock-runtime` versus `bedrock-mantle` versus agent runtime endpoint.
- `Converse` common message shape versus `InvokeModel` provider-specific body.
- Exact `modelId`/inference profile/provisioned ARN.
- Model support for messages, tools, images, structured output, and streaming.
- `contentType`/`accept`.
- SDK version and Region.
- Required IAM actions.

Retry throttling and transient failures with bounded exponential backoff and jitter. Do not retry non-retryable authorization or schema errors without changing the request. Ensure side effects are idempotent.

#### SageMaker deployment

For a custom/open-weight endpoint inspect:

- Endpoint and container CloudWatch logs.
- `/ping` and `/invocations` behavior.
- Image architecture and dependencies.
- Model artifact path, format, and download.
- GPU/CPU memory and disk.
- Container startup health-check timeout.
- Model data download timeout.
- Request content type and payload.

If a large model is still downloading/loading when health checks fail, increase the documented startup and download timeouts after confirming the container is otherwise healthy. A larger timeout does not fix an out-of-memory model or broken `/ping`.

### 5.2.3 Prompt engineering

Classify the symptom:

- Wrong task or missing instruction.
- Conflicting system/user/context instructions.
- Bad example or label ambiguity.
- Context order/separator problem.
- Unsupported output format.
- Excess randomness.
- Retrieved evidence absent or noisy.
- Model capability mismatch.

Systematic refinement:

1. Freeze model, data, parameters, and dataset.
2. Reproduce failures by category.
3. Simplify instructions and define role/task/constraints/output.
4. Separate trusted context from user content.
5. Add a minimal high-quality example only when it addresses the failure.
6. Use structured output for supported machine contracts.
7. Compare version to baseline.
8. Change model/parameters only as a separate experiment.

### 5.2.4 Retrieval systems

Isolate with retrieval-only calls and inspect actual chunks.

| Symptom | Likely cause | Evidence/fix |
|---|---|---|
| No result for known source | Ingestion failed/stale or filter excludes it | Ingestion logs, source sync, metadata |
| Semantically related but wrong chunks | Chunking/embedding/query issue | Golden retrieval set; retune and reindex |
| Exact codes missed | Vector-only search | Hybrid keyword/vector search |
| Correct chunk ranked low | Search scoring/top-k | Rerank and measure nDCG/recall |
| Wrong document version | Missing version metadata/filter | Effective-date/version filter and sync |
| Dimension/schema error | Embedding model/index mismatch | Verify vector dimension/type/mapping |
| Latency spikes | Shards, pressure, queue/rejections | OpenSearch metrics/slow logs; index/capacity tuning |
| Generation unsupported despite good retrieval | Prompt/model ignores evidence | Faithfulness evaluation and generation fix |

Changing chunking or embedding model generally requires re-ingestion/reindexing. Measure before paying that cost.

### 5.2.5 Prompt maintenance and observability

Maintain:

- Prompt templates in Prompt Management or source control.
- Immutable production version references.
- Prompt/model/parameter version in every request log.
- Input/output schema validation.
- Redacted structured logs.
- X-Ray/OTel subsegments for request-path latency.
- Regression suite and release gate.
- Rollback target.

X-Ray traces application calls and latency; it does not understand prompt meaning automatically. Add safe annotations/metadata and correlate with evaluation results.

Prompt Management provides drafts, variants, and versions. Approval and quality gates belong in the delivery process; do not assume a native Prompt Management approval workflow.

## Symptom-to-evidence matrix

| Symptom | First evidence | Avoid first |
|---|---|---|
| New prompt gives worse answers | Baseline/candidate evaluation by category | Random rewrite |
| RAG answer cites wrong jurisdiction | `Retrieve` results and metadata filters | Change temperature |
| Correct source retrieved, answer invents claim | Faithfulness and prompt/model comparison | Reindex immediately |
| Same transcript yields different JSON | Schema results and parameter trials | Human spot check only |
| Agent loops tools | Trace, trajectory, repeated-call count | Increase timeout |
| 429 despite low request count | Tokens, `maxTokens`, quota/capacity | Add unbounded retries |
| 400 after model switch | Endpoint and provider payload contract | Retry |
| SageMaker health check fails while loading | Container log and startup/download timing | Change request prompt |
| Production API is down but offline evaluation passes | Synthetic canary and X-Ray | Rerun only model eval |
| Overall score stable, one cohort regresses | Category/cohort report | Trust global average |

## Evaluation decision table

| Requirement | Best approach | Main distractor |
|---|---|---|
| Open-ended semantic quality at scale | LLM-as-a-judge calibrated with humans | BLEU/ROUGE alone |
| High-impact domain judgment | SME human evaluation plus deterministic checks | Unvalidated model self-score |
| External model responses already generated | Supply responses to Bedrock judge evaluation | Require invoking external model again |
| Determine whether retriever is bad | Retrieve-only evaluation | End-to-end answer score only |
| Assess complete RAG answer | Retrieve-and-generate metrics | Retrieval relevance alone |
| Verify agent uses tools correctly | Trace/trajectory/task evaluation | Final response text only |
| Block a bad release | Automated threshold in CI/CD | Dashboard with no gate |
| Exercise production-like API journey | Synthetic canary | Offline model evaluation only |
| Gradually expose candidate | Canary traffic shift with alarms/rollback | Immediate 100% replacement |
| Learn user preference | Versioned rating/pairwise feedback | Treat clicks as factual correctness |

## Common exam traps

- One quality metric cannot represent correctness, safety, fairness, latency, and business value.
- LLM-as-a-judge is scalable but not ground truth.
- Human evaluation is not always “too much overhead” when the task is high risk or subjective.
- Exact string matching is brittle for valid open-ended output.
- A global average can hide a severe category or cohort regression.
- Offline model evaluation does not test authentication, API routing, retrieval permissions, or client streaming.
- A canary is a release-risk mechanism; it does not replace an evaluation dataset.
- User feedback needs model/prompt metadata and bias-aware interpretation.
- Evaluate retrieval independently before changing generation.
- Context overflow is not solved by increasing output `maxTokens`.
- Do not retry 400/403 errors without fixing the request or permission.
- 429 can be token-reservation pressure, not request-rate pressure.
- X-Ray does not automatically explain prompt semantics.
- Increasing a health-check timeout does not fix an unhealthy or oversized container.
- A prompt version is not approved merely because it exists.

## Local mock references

| Topic | Questions |
|---|---|
| Model/prompt evaluation and regression | PE1-Q04, Q09, Q11, Q55, Q59; PE2-Q07, Q24, Q30, Q37, Q49, Q52, Q55 |
| User/human feedback | PE1-Q14, Q19, Q43; PE2-Q55, Q68 |
| RAG evaluation | PE1-Q43, Q46; PE2-Q16, Q47, Q52, Q57 |
| Agent evaluation | PE1-Q66; PE2-Q41, Q60 |
| Reporting | PE1-Q64; PE2-Q46, Q69 |
| API/payload troubleshooting | PE1-Q47, Q52, Q56; PE2-Q17, Q39, Q54, Q56, Q61 |
| Retrieval troubleshooting | PE1-Q17, Q31, Q51, Q61; PE2-Q33, Q47, Q58, Q73 |
| SageMaker deployment | PE1-Q54, Q62; PE2-Q14, Q25 |
| Deployment validation | PE1-Q11, Q21, Q59, Q62; PE2-Q32, Q52, Q55 |

## Skill mastery check

You are ready for this domain when you can:

- Define a locked, representative, category-aware evaluation dataset.
- Choose deterministic, LLM judge, human, user, retrieval, RAG, or agent evaluation correctly.
- Build a release gate and production canary with rollback.
- Read retrieval and generation metrics without conflating them.
- Diagnose 400, 403, 404, 429, 5xx, timeout, and SageMaker startup failures.
- Prove whether a context issue is input overflow or output truncation.
- Debug an agent from task result, tool trajectory, and trace.
- Maintain prompt versions with correlation, regression, and rollback.

## Recall questions

1. Why is an LLM-as-a-judge not sufficient for every high-risk release?
2. What belongs in a locked GenAI regression dataset?
3. How do retrieve-only and retrieve-and-generate evaluations differ?
4. Which metrics reveal good retrieval but unfaithful generation?
5. What agent result can look correct while the workflow is still defective?
6. Why must feedback store prompt and model versions?
7. What is the difference between an A/B test and a deployment canary?
8. Why can an offline evaluation pass while the production feature is broken?
9. How do you determine whether a response was truncated by output limit or lost input context?
10. Which Bedrock errors should be retried, and which should not?
11. What checks come before changing a provider-native `InvokeModel` payload?
12. What evidence proves a SageMaker model is simply slow to load rather than unhealthy?
13. Why should retrieval be isolated before prompt tuning?
14. What change normally follows an embedding-dimension mismatch?
15. Why does X-Ray need application annotations to help prompt maintenance?
16. How would you report a candidate that is cheaper overall but fails one safety cohort?

## Official sources

- [Official Domain 5 tasks and skills](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain5.html)
- [Amazon Bedrock evaluations](https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation.html)
- [LLM-as-a-judge](https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation-judge.html)
- [Model evaluation metrics](https://docs.aws.amazon.com/bedrock/latest/userguide/model-evaluation-metrics.html)
- [Model evaluation prompt datasets](https://docs.aws.amazon.com/bedrock/latest/userguide/model-evaluation-prompt-datasets-judge.html)
- [Human model evaluation](https://docs.aws.amazon.com/bedrock/latest/userguide/model-evaluation-type-human.html)
- [RAG evaluation metrics](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-evaluation-metrics.html)
- [AgentCore Evaluations](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/evaluations.html)
- [AgentCore evaluation types](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/evaluations-types.html)
- [AgentCore ground-truth evaluation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/ground-truth-evaluations.html)
- [Bedrock API error troubleshooting](https://docs.aws.amazon.com/bedrock/latest/userguide/troubleshooting-api-error-codes.html)
- [CountTokens](https://docs.aws.amazon.com/bedrock/latest/userguide/count-tokens.html)
- [Knowledge Base ingestion logs](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-bases-logging.html)
- [Knowledge Base chunking](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html)
- [Prompt Management](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-management.html)
- [SageMaker model deployment troubleshooting](https://docs.aws.amazon.com/sagemaker/latest/dg/deploy-model-troubleshoot.html)
- [SageMaker large-model endpoint parameters](https://docs.aws.amazon.com/sagemaker/latest/dg/large-model-inference-hosting.html)
- [AWS SDK retry behavior](https://docs.aws.amazon.com/sdkref/latest/guide/feature-retry-behavior.html)
- [CloudWatch Synthetics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Synthetics_Canaries.html)
- [AWS X-Ray](https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html)
