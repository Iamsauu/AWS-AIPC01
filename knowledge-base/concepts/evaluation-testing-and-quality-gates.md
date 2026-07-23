# Evaluation, Testing, and Quality Gates

Status: Verified core concepts; model- and Region-specific support must be rechecked  
Official tasks: 1.6.4, 3.4.2, 4.2.4, 5.1, 5.2.3  
Last verified: 2026-07-23

## Why this matters

A GenAI system can return fluent text while being factually wrong, ungrounded, unsafe, biased, inconsistent, or ineffective at the business task. Traditional exact-match accuracy is therefore insufficient.

Evaluation is a system:

`versioned dataset → candidate execution → multi-dimensional scoring → threshold → deployment gate → limited rollout → live feedback → continuous evaluation`

Prompt intuition and a few hand-picked examples are not evidence.

## Separate the component being evaluated

| Target | What to hold constant | What to measure |
|---|---|---|
| Foundation model | Dataset and prompt | Quality, safety, latency, tokens, price |
| Prompt | Model and inference settings | Task quality, style, structure, consistency |
| Inference parameters | Model and prompt | Determinism, completeness, latency, output size |
| Retrieval only | Generator excluded | Context relevance, coverage, rank, latency |
| RAG end to end | Source/version fixed | Faithfulness, answer relevance, citations |
| Agent | Tools and environment controlled | Task completion, tool choice, loops, errors |
| Release | Baseline and candidate | Regression, drift, safety, performance |

Do not change model, prompt, retrieval configuration, and dataset simultaneously and then claim to know what caused the result.

## Dataset design

A useful evaluation set is:

- Representative of production languages, intents, lengths, and difficulty.
- Stratified across important users, risk classes, and protected cohorts.
- Rich in edge cases, refusals, missing evidence, unsafe requests, and tool failures.
- Versioned and immutable for a given comparison.
- Separate from prompt-development examples to reduce overfitting.
- Annotated with expected responses, contexts, labels, criteria, or tool outcomes where relevant.
- Large enough to expose material regressions; exact size depends on risk and variance.

Use JSON Lines in S3 when required by a managed evaluation job. Follow the current evaluation type’s documented schema; model, RAG, human, and bring-your-own-response jobs do not all use an identical record shape.

## Quality dimensions

### Output quality

- Relevance: answers the actual request.
- Correctness/factual accuracy: claims agree with verified facts.
- Fluency: readable language.
- Coherence: internally logical and well organized.
- Consistency: repeated or equivalent inputs do not create unacceptable variation.
- Completeness: covers required information.
- Helpfulness: useful to the intended user.
- Format compliance: satisfies required JSON/schema/style.

### RAG quality

- Context relevance: retrieved material is relevant to the question.
- Context coverage: retrieved material contains the evidence needed for the reference answer.
- Faithfulness/groundedness: answer claims are supported by retrieved context.
- Citation correctness/coverage: citations point to evidence for the corresponding claim.
- Retrieval latency and rank quality.

Retrieval scores and citations are evidence, not a guarantee that the generated claim is true.

### Agent quality

- Task completion rate.
- Correct tool selection.
- Tool argument validity.
- Tool result use.
- Number of steps and repeated calls.
- Recovery from tool errors.
- Safe stop behavior.
- Human-escalation correctness.
- End-to-end latency and token use.

### Responsible AI

- Harmful-content rate.
- Prompt-attack resistance.
- PII leakage.
- Refusal correctness.
- Fairness across matched cohorts.
- Policy compliance.
- Transparency and source attribution.

### Business and operational quality

- User rating and preference.
- Resolution or conversion outcome.
- Human edit distance/time.
- Cost per successful task.
- Latency-to-quality ratio.
- Token efficiency.

## Evaluation methods

| Method | Strength | Limitation |
|---|---|---|
| Deterministic validation | Excellent for JSON, fields, rules, exact labels | Cannot judge nuanced prose |
| Reference comparison | Repeatable against SME answer | Many valid outputs may differ |
| LLM-as-a-judge | Scales semantic and qualitative scoring | Judge bias and model dependence |
| Human rating | Best for subjective/high-risk judgment | Slower and more expensive |
| Pairwise preference | Useful for candidate comparison | Does not provide absolute correctness |
| Production feedback | Measures real utility | Noisy and affected by user selection |
| Red-team/adversarial suite | Finds safety failures | Must be refreshed as attacks evolve |

Use multiple methods for high-risk workloads.

## Amazon Bedrock evaluation patterns

Know the role of:

- Model evaluation jobs for comparing models, prompts, or settings.
- Automated evaluation, including supported built-in metrics.
- LLM-as-a-judge for semantic quality and explanations.
- Human-based evaluation for subjective or consequential decisions.
- Bring-your-own/precomputed responses when evaluating an external or legacy candidate without calling it during the job.
- RAG retrieve-only evaluation to isolate retrieval.
- RAG retrieve-and-generate evaluation for end-to-end answer quality.
- Agent/AgentCore evaluations for task and trajectory behavior.

Feature names, schemas, supported judge models, Regions, and metrics can change. Verify them before implementation.

## LLM-as-a-judge controls

An LLM judge is not an objective oracle.

Reduce risk by:

- Defining a precise rubric and scale.
- Including reference answer/context where appropriate.
- Randomizing pair order for comparison.
- Testing position, verbosity, and self-preference bias.
- Calibrating against human-labeled samples.
- Tracking judge model and version.
- Re-evaluating if the judge changes.
- Reviewing disagreements and high-risk failures manually.

## Fairness and cohort evaluation

Create matched scenarios where only the cohort attribute changes. Use balanced samples, score each cohort separately, and compare:

- Recommendation/outcome rate.
- Tone or sentiment.
- Helpfulness and completeness.
- Refusal or escalation rate.
- Error and unsupported-claim rate.

An overall average can hide a serious cohort failure. Store prompt/model versions and evaluation criteria with results.

## Prompt and parameter evaluation

For deterministic or structured output:

- Compare lower temperature and narrower sampling settings.
- Validate every required field with schema checks.
- Measure consistency across repeated runs.
- Track missing-field and parse-failure rates.
- Control `maxTokens` so the output can finish but does not reserve unnecessary token capacity.

For creative output:

- Do not optimize only for consistency.
- Evaluate diversity, adherence, safety, and usefulness.

## Continuous quality pipeline

Recommended stages:

1. Static checks: prompt syntax, variables, schemas, tool definitions.
2. Unit tests: validators, redaction, routing, and tool adapters.
3. Offline benchmark: baseline versus candidate on a fixed dataset.
4. Safety tests: guardrail integration, adversarial, PII, and policy cases.
5. Quality gate: fail if any critical metric or cohort threshold fails.
6. Canary: expose a limited production cohort.
7. Live monitoring: errors, latency, tokens, ratings, edits, and safety.
8. Automatic rollback: CloudWatch alarm or failed deployment criterion.
9. Continuous/scheduled benchmark for drift.

CloudWatch Synthetics validates the deployed API journey and availability. It is complementary to semantic model evaluation, not a replacement.

## Human feedback architecture

A serverless pattern:

`UI rating/edit → API Gateway → Lambda validation → DynamoDB`

Store:

- Request/response correlation ID.
- Model, prompt, KB/index, and guardrail versions.
- Rating, reason, optional corrected response.
- Task/user cohort metadata that is safe to retain.
- Timestamp and deployment experiment.

Never store raw PII merely to make later analysis convenient.

## Reporting

For recurring stakeholder reporting:

`evaluation output in S3 → Glue Data Catalog → Athena → Amazon Quick Sight`

Separate:

- Release-gate metrics for engineers.
- Risk and compliance evidence.
- Business outcomes.
- Cost/latency trends.

## Failure modes

| Failure | Why it happens | Correction |
|---|---|---|
| Test set looks perfect, production fails | Dataset is narrow or leaked into development | Add production-like and holdout cases |
| Judge scores conflict with humans | Weak rubric or judge bias | Calibrate and combine methods |
| RAG answer fails but retrieval is unknown | Only end-to-end evaluation exists | Add retrieve-only evaluation |
| Average fairness passes, one cohort fails | Aggregation hides disparity | Report stratified metrics |
| New prompt is fluent but breaks parser | No deterministic structure test | Add schema validation gate |
| Canary detects HTTP 200 but poor answers | Only availability is checked | Add semantic regression evaluation |
| Model improvement raises cost sharply | Quality evaluated alone | Add token, latency, and business-value metrics |
| Repeated mock score rises | Wording memorized | Use unseen and transformed scenarios |

## Decision rules

- Use deterministic validators for deterministic requirements.
- Use retrieve-only evaluation before changing prompts when retrieval may be wrong.
- Use human evaluation for subjective, high-impact, or governance-sensitive judgments.
- Use LLM-as-a-judge for scale, but calibrate it.
- Use a fixed benchmark for fair candidate comparisons.
- Use canary plus rollback for production uncertainty.
- Track configuration versions with every result.

## Local mock references

- PE1: Q4, Q9, Q11, Q14, Q30, Q43, Q55, Q59, Q64, Q66.
- PE2: Q7, Q8, Q16, Q19, Q30, Q37, Q41, Q49, Q52, Q55, Q68.

See [mock-question coverage](../practice/mock-question-coverage-map.md).

## Recall questions

1. Why can a high semantic-similarity score not serve as factual confidence?
2. When should you run retrieve-only rather than retrieve-and-generate evaluation?
3. What must be held constant when comparing prompt versions?
4. Which controls reduce LLM-judge bias?
5. What belongs in a production quality gate before canary deployment?
6. Why is a CloudWatch Synthetics canary insufficient for answer-quality regression?
7. How would you detect fairness failure hidden by an overall average?
8. Which metadata is required to connect user feedback to a release?

## Official sources

- [Amazon Bedrock evaluations](https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation.html)
- [Official Domain 5](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain5.html)
- [CloudWatch Synthetics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Synthetics_Canaries.html)
