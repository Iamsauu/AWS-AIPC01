# Safety, Privacy, and Responsible AI

Status: Verified  
Official tasks: 3.1, 3.2, 3.3, 3.4  
Last verified: 2026-07-23

## Why this matters

A safe GenAI application is not a model plus one filter. It is a controlled system in which untrusted input, retrieved content, model output, tools, logs, and human decisions each have separate safeguards.

For exam scenarios, first identify the risk:

- Harmful or prohibited content.
- Prompt injection, jailbreak, or hidden-prompt extraction.
- Hallucination or unsupported output.
- PII exposure in live text, stored data, or logs.
- Unsafe tool or SQL execution.
- Bias or unequal quality across cohorts.
- Missing traceability, policy evidence, or human oversight.

Then place the control at the point where it can prevent the harm. A scanner that finds PII in S3 after the request is complete cannot prevent that PII from reaching a model.

## Core concepts

```text
Authenticated caller
  -> request limits and validation
  -> input classification, PII masking, and prompt-attack checks
  -> authorization and retrieval filtering
  -> guarded model invocation
  -> grounding and schema checks
  -> deterministic policy/tool validation
  -> output safety and PII checks
  -> human review when risk or uncertainty is high
  -> redacted logs, metrics, audit trail, and feedback
```

Each layer has a different purpose:

| Layer | Primary control | What it does not prove |
|---|---|---|
| API boundary | Authentication, quotas, schema and size validation | That the prompt is semantically safe |
| Pre-processing | Comprehend, regex, custom classifiers, Lambda | That model output will be safe |
| Bedrock Guardrails | Content, denied topics, prompt attacks, words, sensitive information, grounding | That a tool action is authorized |
| RAG | Approved context and source attribution | That every generated claim is true |
| Structured output | JSON Schema or strict tool schema | That field values are correct or permitted |
| Deterministic policy layer | Allowlist, parameterized template, business rules | That free-form prose is harmless |
| Human review | Judgment for ambiguous or high-impact cases | A scalable substitute for all automated controls |
| Monitoring | Detects violations, drift, and regressions | Prevention by itself |

## How it works

Safety decisions flow from the authenticated request through input controls, governed retrieval/tool authorization, model inference, output controls, and finally monitoring and human escalation. A control must run before the protected boundary when the requirement says the data or action must never cross that boundary.

## AWS services and APIs

### Amazon Bedrock Guardrails

A guardrail is a versioned collection of policies applied to model input, model output, or independently through a guardrail API. Exam-relevant safeguards include:

- Content filters for harmful text or images.
- Prompt-attack detection.
- Denied topics.
- Word filters.
- Sensitive-information filters for built-in PII types and custom regular expressions.
- Contextual grounding and relevance checks.

Attach a guardrail ID and version to `Converse`, `ConverseStream`, `InvokeModel`, or `InvokeModelWithResponseStream`, or use `ApplyGuardrail` before or after another model or service. Guardrails can also be associated with Bedrock agents, knowledge bases, and flow nodes.

#### Invocation behavior

1. Bedrock evaluates enabled input policies.
2. If the input is blocked, model inference is skipped and the configured blocked response is returned.
3. Otherwise, the model runs.
4. Bedrock evaluates the model response.
5. Bedrock blocks, replaces, or masks the response according to the policy.

This ordering has a cost implication: blocking input avoids model inference cost; blocking output occurs after the model has already generated it.

#### Guardrail decisions

| Requirement | Best starting control | Add when needed |
|---|---|---|
| Block profanity, violence, hate, or unsafe images | Content filter | Custom moderation workflow for unusual policy |
| Block a business-specific subject | Denied topic | Deterministic policy check for exact legal rules |
| Stop common prompt attacks before inference | Prompt-attack input filter | Sanitization, adversarial tests, tool isolation |
| Mask known PII in input and output | Sensitive-information policy with `ANONYMIZE` | Comprehend/custom regex for additional entities |
| Observe a category without blocking yet | Detection/no-action behavior and metrics | Change threshold after measured false positives |
| Apply safeguards to a non-Bedrock model | `ApplyGuardrail` around that call | Application-side post-validation |
| Select only user text for evaluation | Input tags/content-block guardrail configuration | Do not tag trusted system or retrieved text as user input |
| Reject unsupported answers against supplied context | Contextual grounding check | Citations, retrieval evaluation, abstention policy |

#### Important limitations

- Guardrails reduce risk; they do not guarantee safety.
- Current Guardrails documentation excludes reasoning content blocks from normal guardrail filtering.
- Prompt-attack detection requires the input portion to be marked correctly. For `InvokeModel`, use the documented input tags; for Converse, use guardrail configuration on content blocks.
- PII masking covers model input and returned text, but does not automatically sanitize model invocation logs.
- Guardrail trace fields can contain the original matched PII value.
- Sensitive-information filters do not detect PII inside model `tool_use` parameters. Validate tool inputs separately.
- Contextual grounding needs a source, query, and candidate response. It is designed for supported summarization, paraphrasing, and question-answering patterns; do not assume it is a universal truth oracle.
- For streaming, unsafe or ungrounded content can be partially emitted before a whole-response determination. Choose buffering or a stricter UX when no unsafe token may be shown.
- Feature, language, model, and Region support changes. Recheck before implementation.

## Architecture patterns

### Prompt injection and jailbreak defense

Treat user input and retrieved documents as untrusted data, not instructions. A system prompt such as “ignore malicious instructions” is useful but is not a security boundary.

#### Threats

- Direct injection: the user asks the model to ignore or reveal instructions.
- Indirect injection: a retrieved document or web page contains malicious instructions.
- Jailbreak: encoded, role-played, multi-turn, or obfuscated attempts to bypass safety.
- Tool manipulation: the prompt induces an unauthorized or destructive action.
- Prompt leakage: the model exposes system instructions, secrets, or hidden context.
- Resource abuse: very long, recursive, or repeated prompts consume tokens and tools.

#### Required controls

1. Separate system instructions, trusted context, and user content structurally.
2. Apply prompt-attack and content filtering to untrusted portions.
3. Do not place credentials or secrets in the prompt.
4. Retrieve only authorized documents and treat retrieved text as data.
5. Give the agent a narrow tool allowlist with explicit schemas.
6. Validate tool name, parameters, resource identifiers, and user authorization outside the model.
7. Use least-privilege IAM for every tool.
8. Require confirmation or human approval for high-impact actions.
9. Add maximum steps, timeouts, budgets, circuit breakers, and idempotency.
10. Run an adversarial suite after changes and monitor attacks in production.

An FM may propose an action; deterministic code must decide whether that action is allowed.

### Harmful-output prevention

Output safety is separate from input safety. A benign request can still yield toxic, private, fabricated, or policy-violating output.

Use:

- Output-side Guardrails.
- A post-processing Lambda for organization-specific validation.
- JSON Schema or strict tool definitions for machine-readable shape.
- Allowlisted values and business-rule validation.
- Human review for high-impact communication or action.
- A safe fallback response when checks fail.

For deterministic text-to-SQL, do not let the FM emit arbitrary executable SQL. Map recognized intent to parameterized, `SELECT`-only templates; enforce table, column, row, statement-type, and result-size policies before execution.

### Hallucination and accuracy controls

Hallucination controls address different failure points:

| Failure | Evidence | Control |
|---|---|---|
| Model lacks current private facts | Answer unsupported by corpus | RAG from governed sources |
| Retriever found wrong or stale chunks | Low context relevance/coverage; wrong metadata | Fix ingestion, filters, search, chunking, reranking |
| Model ignores correct context | High retrieval quality but low faithfulness | Prompt, model, generation, grounding check |
| Output is malformed | JSON parse/schema failure | Bedrock structured outputs or strict tool schema |
| Claim is high impact | Uncertain or conflicting evidence | Abstain and route to human review |

Use `RetrieveAndGenerate` citations or preserve source metadata when building custom RAG. Citations show which chunks were used; they are not proof that the claims are correct. Vector similarity is retrieval relevance, not factual confidence.

Bedrock contextual grounding produces grounding and relevance scores against supplied source material. Set thresholds from representative validation data, not intuition. A higher threshold generally blocks more unsupported output but can increase false positives.

For critical calculations, balances, eligibility, or status:

- Obtain the value from an authoritative system or deterministic query.
- Let the FM explain the result.
- Do not let the FM invent or calculate the authoritative value when exactness is required.

### Structured output

Bedrock structured outputs can constrain supported models to a JSON Schema and can validate strict tool inputs. This improves parseability and reduces retry loops.

Remember:

- A valid schema does not make the values true.
- Unsupported schema features cause a client error.
- The first use of a new schema can add compilation latency; successful grammars are cached.
- Support is API- and model-specific.
- Current documentation states that structured outputs and citations are incompatible for Anthropic models.

Use both schema validation and semantic/business validation.

### Privacy-preserving patterns

#### Data minimization

Send only the fields and context required for the task. Avoid full records when a de-identified excerpt is enough.

#### Reversible tokenization

When the model needs entity continuity but not the identity:

1. Detect PII.
2. Replace each entity with a stable placeholder such as `<NAME_1>`.
3. Keep the mapping in an encrypted, access-controlled store outside the prompt.
4. Invoke the model with placeholders.
5. Re-identify only in an authorized downstream step if required.

This preserves relationships better than deleting all detected text.

#### Service boundary: Guardrails, Comprehend, and Macie

| Need | Service | Why |
|---|---|---|
| Mask/block PII in live model input or text output | Bedrock Guardrails | Native inline safeguard around inference |
| Locate PII in a live English/Spanish text request before inference | Amazon Comprehend real-time PII detection | Returns detected entities and locations |
| Detect or redact a collection of text documents | Amazon Comprehend asynchronous analysis | Text-oriented batch processing |
| Discover sensitive data already stored in S3 | Amazon Macie | S3 inventory, policy findings, and sensitive-data discovery |
| Mask sensitive values when viewing logs | CloudWatch Logs data protection policy | Audit and de-identify matching log fields |
| Delete artifacts after a retention period | S3 Lifecycle or log-group retention | Lifecycle management, not content detection |

Macie is not an inline request redactor. Comprehend is not a general S3 security-posture service. Guardrails do not scan an S3 estate.

#### Bedrock data handling

AWS states that Bedrock inputs and outputs are not used to train Amazon or third-party base models. Current Bedrock documentation describes zero operator access and default zero data retention, with documented model-specific abuse-detection exceptions. This does not remove the customer's obligations to:

- Minimize and classify data.
- Verify model, endpoint, Region, and cross-Region behavior.
- Configure encryption and access.
- Avoid logging raw sensitive content.
- Apply retention and deletion policies to customer-controlled stores.

### Logging without creating a privacy incident

Prefer structured metadata over raw prompts:

- Correlation/request ID.
- Tenant or product pseudonym.
- Model ID or inference profile.
- Prompt version.
- Guardrail ID/version and action.
- Token counts, latency, stop reason, error class.
- Retrieved source IDs, not full confidential passages.
- Evaluation and policy outcome.

If model invocation logging is enabled, understand that it can capture requests and responses. Guardrail PII masking does not sanitize the original input field in those logs. Use CloudWatch Logs data-protection policies, restrictive IAM, KMS where required, a retention policy, and selective logging. Do not put PII in metric dimensions, resource names, or tags.

## Security and governance

### Responsible AI dimensions

AWS identifies dimensions that include fairness, explainability, privacy and security, safety, controllability, veracity and robustness, governance, and transparency.

Translate these into measurable controls:

| Dimension | Example evidence |
|---|---|
| Fairness | Cohort-balanced dataset; disparity metrics; equivalent-case tests |
| Explainability | User-facing evidence, decision factors, limitations |
| Privacy | Data inventory, consent/purpose, masking, retention, access review |
| Security | Threat model, least privilege, private access, red-team results |
| Safety | Harm and prompt-attack rates; guardrail configuration and tests |
| Controllability | Stop conditions, human override, rollback, kill switch |
| Veracity/robustness | Faithfulness, correctness, adversarial and perturbation tests |
| Governance | Owner, risk tier, approvals, lineage, audit evidence |
| Transparency | AI disclosure, citations, uncertainty, intended-use statement |

#### Fairness evaluation

Build paired or stratified cases where irrelevant demographic attributes change and task facts remain constant. Compare:

- Acceptance, recommendation, or escalation rates.
- Error and refusal rates.
- Helpfulness, correctness, and tone.
- False-positive and false-negative rates.
- Distribution of quality scores by cohort.

Use prompt/model variants with a fixed dataset. LLM-as-a-judge can scale evaluation, but validate the judge against humans because the judge can also be biased. CloudWatch stores custom fairness metrics and trends; it does not create fairness metrics automatically.

#### Transparency without exposing secrets

Show:

- Source citations or evidence.
- Model limitations and intended use.
- Whether content is AI-generated.
- Confidence or groundedness only when the score has a defined basis.
- How a user can appeal or request human review.

Do not expose:

- Secrets or hidden system prompts.
- Provider-internal chain-of-thought.
- Tool credentials.
- Raw private context.

Agent and orchestration traces can support operators and auditors. They are not the same as exposing internal chain-of-thought to end users.

### Governance and compliance

Governance is a lifecycle, not a document:

1. Define owner, intended use, prohibited use, risk tier, and obligations.
2. Register approved models, prompts, data sources, guardrails, and tools.
3. Define quantitative release criteria.
4. Test safety, fairness, privacy, quality, security, latency, and cost.
5. Approve and deploy immutable versions through CI/CD.
6. Record runtime metadata and API activity.
7. Monitor drift, violations, abuse, and feedback.
8. Alert, remediate, roll back, and preserve evidence.
9. Reassess when models, prompts, data, tools, or policy change.

#### Evidence services

| Evidence question | AWS capability |
|---|---|
| What is the model for, where should it not be used, and what are its limitations? | SageMaker Model Cards |
| Which source and transformations produced the governed data? | Glue Data Catalog, job metadata, tags, and application lineage records |
| Who called or changed an AWS API, and when? | AWS CloudTrail |
| What happened inside the application request? | Structured CloudWatch Logs and X-Ray/agent traces |
| How is the system performing now? | CloudWatch metrics, dashboards, alarms |
| How does implementation compare with an AWS best-practices control framework? | AWS Audit Manager GenAI Best Practices Framework |

Audit Manager evidence helps prepare for audits; it is not a legal certification or proof of compliance by itself.

#### Model cards

Record at least:

- Owner and model/version.
- Intended and prohibited uses.
- Risk rating.
- Training/customization context when known.
- Evaluation datasets, metrics, and results.
- Limitations, bias and safety observations.
- Approval status and release history.
- Monitoring and rollback plan.

SageMaker Model Cards can document models hosted inside or outside SageMaker. They are governance evidence, not a runtime enforcement mechanism.

## Cost, latency, and reliability

- Blocking a harmful input before inference avoids the model call; blocking an output happens after generation.
- Every additional classifier or human step adds cost and latency, so keep the normal safe path synchronous and route only ambiguous/high-risk cases for extended review.
- Tune thresholds with representative data to balance missed harm against false positives.
- Buffer streaming output when policy requires whole-response validation before any token is shown.
- Make safety fallbacks explicit and observable so a filter outage does not silently become an allow-all path.

## Decision table

| Scenario constraint | Correct direction | Main distractor |
|---|---|---|
| Harmful prompt must be blocked before model | Input Guardrails or `ApplyGuardrail` | Output-only scan |
| Unknown moderation cases need review | Managed filter plus threshold-based human workflow | Send all traffic to humans |
| Exact PII must never reach model | Pre-inference detection/tokenization plus inline guardrail | Macie after storage |
| PII may remain in S3 audit records | Macie findings plus remediation | Comprehend used as S3 posture scanner |
| Output must be valid JSON | Structured output/schema | “Return JSON” prompt only |
| SQL must be read-only and deterministic | Parameterized allowlisted `SELECT` templates | Execute arbitrary generated SQL |
| Answer must cite approved policy | Governed RAG plus source metadata/citations | Fine-tune on changing policy |
| Unsupported answer must be withheld | Grounding/faithfulness threshold and abstention | Treat vector score as confidence |
| All teams must use a policy guardrail | Organization/account enforcement plus application guardrails | Hope each application attaches it |
| Bias must be measured | Paired/cohort dataset and tracked disparity metrics | Ask the model whether it is fair |
| Auditors need API actor and time | CloudTrail | CloudWatch metrics |
| Operators need request-path diagnosis | Structured logs/traces | CloudTrail alone |

## Failure modes and troubleshooting

| Symptom | Likely cause | Evidence | Remediation |
|---|---|---|---|
| Prompt attacks pass through | User content not tagged or filter disabled | Guardrail trace/config version | Mark untrusted blocks; test deployed version |
| Safe requests are blocked | Threshold too strict or denied topic too broad | False-positive set by category | Tune with representative dataset |
| PII appears in invocation logs | Raw requests logged before masking | Log sample and logging configuration | Redact before logging; data-protection policy |
| PII appears in tool parameters | Guardrail text masking does not cover `tool_use` values | Tool trace/input | Validate/redact tool arguments |
| Grounded answer is still wrong | Wrong/stale retrieved evidence | Retrieved chunks, source version | Fix ingestion/filter/retrieval; do not tune prompt first |
| Citation exists but claim unsupported | Model cited an irrelevant passage | Citation precision/coverage review | RAG evaluation and claim-level validation |
| Agent performs unauthorized action | Model output treated as authorization | Tool call and IAM policy | External authorization and least-privilege tool role |
| Fairness score improves but user harm rises | Metric or dataset is incomplete | Cohort outcomes and complaints | Add harm-specific metrics and human review |
| Audit cannot tie answer to release | Missing model/prompt/guardrail/source versions | Application decision log | Emit versioned identifiers per response |
| Evidence can be altered or deleted | Logs lack protection/retention controls | Resource policy and retention config | Controlled archive, restrictive delete, Object Lock where required |

## Common exam traps

- “Guardrails” is not always the complete answer. Defense in depth requires pre-processing, authorization, post-validation, and monitoring when the scenario names those risks.
- A system prompt is not an access-control mechanism.
- RAG reduces unsupported answers but does not eliminate hallucination.
- Citations are attribution, not factual proof.
- JSON Schema validates shape, not truth.
- Lower temperature improves repeatability, not factuality or fairness.
- Macie discovers sensitive data in S3; it is not the inline PII solution.
- Comprehend detects PII in text; it does not enforce Bedrock model access.
- CloudTrail records API activity; it does not normally contain the full application decision.
- CloudWatch stores metrics and logs; it does not automatically make logs immutable.
- A model card documents risk; it does not block a policy violation.
- “Native Bedrock privacy” does not justify sending unnecessary sensitive data.

## Local mock references

- Input/output safety and defense in depth: PE1-Q07, PE1-Q13, PE1-Q15, PE1-Q24, PE1-Q28, PE1-Q67, PE1-Q73; PE2-Q03, PE2-Q18, PE2-Q20, PE2-Q42, PE2-Q63.
- PII and retention: PE1-Q05, PE1-Q25, PE1-Q69, PE1-Q75; PE2-Q21, PE2-Q29, PE2-Q35, PE2-Q64.
- Grounding and transparency: PE1-Q22, PE1-Q32, PE1-Q43, PE1-Q46; PE2-Q11, PE2-Q45, PE2-Q57.
- Governance, lineage, and fairness: PE1-Q12, PE1-Q29, PE1-Q30, PE1-Q36; PE2-Q04, PE2-Q08, PE2-Q32, PE2-Q46, PE2-Q59, PE2-Q67, PE2-Q74.

### Mock claims to treat cautiously

- PE1-Q06 describes a Prompt Management “approval workflow.” Prompt Management provides reusable prompts, variants, versions, and deployment references. Implement approval as a separate source-control/CI/CD/IAM process; do not assume Prompt Management itself is a complete approval engine.
- PE1-Q12 asks for an immutable decision record but its selected answer relies on CloudWatch Logs. CloudWatch Logs can have encryption, retention, deletion protection, and restrictive IAM, but those controls alone do not mean a cryptographically immutable ledger. For a formal WORM requirement, design a controlled archive such as S3 Object Lock and verify the governing requirement.
- Mock wording that calls a retrieval or grounding score “confidence” must not be generalized to model truth confidence. Name the score by what it actually measures.

## Hands-on validation

Use [Lab 3: Guardrails and private access](../labs/03-guardrails-and-private-access.md) to validate:

1. A blocked harmful input does not invoke the model.
2. PII masking occurs on both input and output.
3. A custom identifier is handled.
4. Prompt attacks are tested with the correct input tagging.
5. Raw PII does not appear in application or invocation logs.
6. A grounded and ungrounded answer produce different policy outcomes.
7. Tool inputs are validated outside Guardrails.

## Recall questions

1. Why is Guardrails not an authorization boundary for agent tools?
2. When should `ApplyGuardrail` be used instead of attaching a guardrail to inference?
3. Why can Guardrails mask a response but still leave PII in model invocation logs?
4. What is the precise difference between Macie, Comprehend, and Guardrails?
5. How would you preserve entity relationships while preventing identities from reaching the FM?
6. Why does a valid citation not prove that an answer is correct?
7. Which evidence distinguishes bad retrieval from bad generation?
8. Why is valid JSON not enough for safe SQL or tool execution?
9. What fairness dataset would detect a tone difference caused only by a demographic attribute?
10. What do CloudTrail, CloudWatch, X-Ray, and a model card each prove?
11. How would you enforce one guardrail across multiple AWS accounts?
12. What additional control is needed when a model response is streamed but no unsafe token may be displayed?

## Official sources

- [AIP-C01 Domain 3](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain3.html)
- [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
- [How Guardrails works](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-how.html)
- [ApplyGuardrail](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-use-independent-api.html)
- [Prompt input tagging](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-tagging.html)
- [Sensitive-information filters](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-sensitive-filters.html)
- [Contextual grounding checks](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-contextual-grounding-check.html)
- [Guardrail enforcement with AWS Organizations](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-enforcements.html)
- [Bedrock structured outputs](https://docs.aws.amazon.com/bedrock/latest/userguide/structured-output.html)
- [Knowledge Base retrieval and citations](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-how-retrieval.html)
- [Amazon Comprehend PII](https://docs.aws.amazon.com/comprehend/latest/dg/how-pii.html)
- [What is Amazon Macie?](https://docs.aws.amazon.com/macie/latest/user/what-is-macie.html)
- [CloudWatch Logs data-protection policies](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/cloudwatch-logs-data-protection-policies.html)
- [Bedrock abuse detection and data retention](https://docs.aws.amazon.com/bedrock/latest/userguide/abuse-detection.html)
- [SageMaker Model Cards](https://docs.aws.amazon.com/sagemaker/latest/dg/model-cards.html)
- [AWS Audit Manager GenAI Best Practices Framework](https://docs.aws.amazon.com/audit-manager/latest/userguide/aws-generative-ai-best-practices.html)
- [Responsible AI in the Generative AI Lens](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/responsible-ai.html)
- [AWS Responsible AI Lens](https://docs.aws.amazon.com/wellarchitected/latest/responsible-ai-lens/responsible-ai-lens.html)
