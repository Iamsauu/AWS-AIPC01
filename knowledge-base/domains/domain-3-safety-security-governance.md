# Domain 3: AI Safety, Security, and Governance

Status: Verified against the current blueprint  
Exam weight: 20%  
Official tasks: 3.1, 3.2, 3.3, 3.4  
Last verified: 2026-07-23

## Domain objective

Design GenAI systems that prevent harmful use, protect data, enforce access, preserve traceability, and operationalize Responsible AI.

This domain is about layered controls. A correct answer normally combines:

- A control at the API or input boundary.
- Managed model safeguards.
- Deterministic authorization and validation.
- Data privacy and lifecycle controls.
- Logging, monitoring, evaluation, and human review.

Canonical deep dives:

- [Safety, privacy, and Responsible AI](../concepts/safety-privacy-and-responsible-ai.md)
- [Security, networking, and access control](../concepts/security-networking-and-access-control.md)

## Complete official-skill map

| Skill | Required exam capability | Canonical detail |
|---|---|---|
| 3.1.1 | Protect FMs from harmful user input with Guardrails, real-time validation, Lambda, and Step Functions moderation | [Input safety](../concepts/safety-privacy-and-responsible-ai.md#amazon-bedrock-guardrails) |
| 3.1.2 | Prevent harmful output with output guardrails, toxicity evaluation, post-validation, and deterministic text-to-SQL | [Output safety](../concepts/safety-privacy-and-responsible-ai.md#harmful-output-prevention) |
| 3.1.3 | Reduce hallucination with governed RAG, verification, grounding/relevance signals, citations, and structured output | [Accuracy](../concepts/safety-privacy-and-responsible-ai.md#hallucination-and-accuracy-controls) |
| 3.1.4 | Build defense in depth with pre-processing, model guardrails, post-processing, and API response controls | [Defense in depth](../concepts/safety-privacy-and-responsible-ai.md#core-mental-model-defense-in-depth) |
| 3.1.5 | Detect prompt injection, jailbreaks, adversarial input, and security vulnerabilities; automate adversarial testing | [Prompt attacks](../concepts/safety-privacy-and-responsible-ai.md#prompt-injection-and-jailbreak-defense) |
| 3.2.1 | Protect FM environments with private connectivity, IAM, Lake Formation, encryption, and access monitoring | [Security](../concepts/security-networking-and-access-control.md) |
| 3.2.2 | Protect sensitive interactions with Comprehend, Macie, Bedrock privacy controls, Guardrails, and retention | [Privacy services](../concepts/safety-privacy-and-responsible-ai.md#service-boundary-guardrails-comprehend-and-macie) |
| 3.2.3 | Preserve utility through masking, tokenization, anonymization, and purpose-limited context | [Privacy patterns](../concepts/safety-privacy-and-responsible-ai.md#privacy-preserving-patterns) |
| 3.3.1 | Create compliance evidence with model cards, lineage, metadata, decision logs, and release controls | [Governance](../concepts/safety-privacy-and-responsible-ai.md#governance-and-compliance) |
| 3.3.2 | Track source, version, transformations, citations, and API actors with Glue, metadata, application logs, and CloudTrail | [Evidence services](../concepts/safety-privacy-and-responsible-ai.md#evidence-services) |
| 3.3.3 | Establish organization-wide ownership, policy, risk tiers, approved components, review, and enforcement | [Lifecycle](../concepts/safety-privacy-and-responsible-ai.md#governance-and-compliance) |
| 3.3.4 | Continuously detect misuse, drift, policy violations, bias drift, redaction failures, and unsafe output; alert and remediate | [Monitoring](#task-33-ai-governance-and-compliance) |
| 3.4.1 | Build transparent outputs with evidence, uncertainty, documented limits, and operator traces without exposing secrets or chain-of-thought | [Transparency](../concepts/safety-privacy-and-responsible-ai.md#transparency-without-exposing-secrets) |
| 3.4.2 | Evaluate fairness with paired/cohort datasets, A/B tests, LLM-as-a-judge plus human validation, and trend metrics | [Fairness](../concepts/safety-privacy-and-responsible-ai.md#fairness-evaluation) |
| 3.4.3 | Turn policy into Guardrails, deterministic compliance checks, model cards, approval gates, and fail-closed behavior | [Responsible AI](../concepts/safety-privacy-and-responsible-ai.md#responsible-ai-dimensions) |

## Task 3.1: Input and output safety controls

### 3.1.1 Harmful input

Know how to:

- Configure content, word, denied-topic, prompt-attack, and sensitive-information policies.
- Apply a guardrail to model inference or call `ApplyGuardrail` independently.
- Block before inference when possible to prevent influence and avoid model cost.
- Add a Lambda/custom classifier when organization-specific policy exceeds managed filters.
- Route ambiguous cases to a human workflow without delaying normal traffic.
- Return a stable blocked-response shape.

### 3.1.2 Harmful output

Use:

- Guardrails on output.
- Post-processing policy checks.
- Toxicity/safety evaluations before release and continuously after release.
- Schema validation for machine consumers.
- Human approval for high-impact messages.
- Parameterized, allowlisted SQL or tool actions.

An FM-generated `SELECT` string is not deterministic merely because it begins with `SELECT`. Validate statement type, table, column, filter, parameter, result size, and caller authorization, or map intent to pre-approved templates.

### 3.1.3 Accuracy and hallucination

Layer:

1. Governed and current data sources.
2. Correct ingestion and metadata.
3. Retrieval with filters, hybrid search, and reranking when needed.
4. Source citations.
5. Grounding/faithfulness evaluation.
6. Structured output.
7. Abstention and human review for insufficient/conflicting evidence.

Do not call vector similarity a factual confidence score.

### 3.1.4 Defense in depth

The canonical exam pattern is:

```text
API validation
  -> Comprehend/custom pre-filter
  -> Guardrails input
  -> model/RAG/tool workflow
  -> Guardrails output
  -> Lambda business and schema validation
  -> API response filtering
  -> redacted telemetry
```

Each stage addresses a different failure. Repeating the same classifier three times without distinct purpose is not defense in depth.

### 3.1.5 Advanced threats

For prompt injection and jailbreak:

- Mark untrusted input correctly for guardrail evaluation.
- Separate data from instructions.
- Never put secrets in context.
- Restrict tools and validate their inputs.
- Enforce authorization outside the model.
- Add maximum iterations, token/tool budgets, timeout, and circuit breaker.
- Test direct, indirect, encoded, multi-turn, and tool-manipulation attacks.
- Track detection and bypass rates by prompt/model/guardrail version.

## Task 3.2: Data security and privacy

### 3.2.1 Protected environments

Know the boundaries:

| Boundary | Control |
|---|---|
| Human workforce access | IAM Identity Center, permission sets, temporary credentials |
| Workload access | IAM roles and least privilege |
| Agent/tool access | Tool-specific roles, workload identity, external OAuth scopes |
| Bedrock private path | Correct interface VPC endpoint and endpoint policy |
| Data-lake row/column access | Lake Formation plus IAM/S3/KMS |
| Data at rest | Service encryption or customer-managed KMS key |
| Credentials | IAM roles; Secrets Manager/AgentCore credential provider for non-AWS secrets |
| API activity | CloudTrail |
| Runtime behavior | CloudWatch, invocation/application logs, X-Ray or agent trace |

Private networking does not replace authorization. IAM does not replace Lake Formation column permissions.

### 3.2.2 Sensitive information

Select by location and timing:

- **Live input/output:** Guardrails.
- **Text preprocessing:** Comprehend PII.
- **S3 discovery and posture:** Macie.
- **Logs:** CloudWatch Logs data-protection policy.
- **Retention:** S3 Lifecycle or log retention.

Guardrail masking does not automatically sanitize original invocation logs or trace match fields.

### 3.2.3 Privacy with utility

Prefer data minimization and stable placeholders to destroying needed context. For example:

```text
"Nguyen called from 090..." -> "<NAME_1> called from <PHONE_1>"
```

Keep any re-identification map outside the prompt in an encrypted, narrowly authorized store. Send only the minimum approved context and delete artifacts according to policy.

## Task 3.3: AI governance and compliance

### 3.3.1 Compliance framework

Maintain a versioned record of:

- Owner, intended use, prohibited use, risk tier.
- Model/provider/version and Region.
- Prompt, guardrail, tool, knowledge base, and embedding versions.
- Approved data sources and transformations.
- Evaluation datasets, metrics, thresholds, and results.
- Human approvals.
- Deployment and rollback state.
- Runtime decision metadata.

SageMaker Model Cards document intended use, risk, metrics, and limitations. They do not enforce runtime policy.

### 3.3.2 Source tracking

At ingestion, register and tag the source, owner, version, effective date, classification, and transformation. At retrieval/generation, preserve source IDs and citations. At runtime, record the model, prompt, guardrail, retrieved source IDs, policy outcome, and correlation ID. Use CloudTrail for AWS API actor/action evidence.

Glue Data Catalog is a catalog. Complete lineage may also require job metadata and explicit application records.

### 3.3.3 Organizational governance

Implement:

- Named owners and risk acceptance.
- Approved model/prompt/guardrail/tool catalog.
- Reusable infrastructure and policy templates.
- Separation of duties.
- CI/CD approval and automated checks.
- SCPs, permissions boundaries, and organization guardrail enforcement where supported.
- Exception and expiry process.
- Periodic Well-Architected/Responsible AI review.

Prompt Management versions prompts but is not, by itself, a complete approval workflow.

### 3.3.4 Continuous governance

Monitor:

- Guardrail interventions and attack categories.
- PII/redaction findings.
- Harmfulness, groundedness, correctness, and bias drift.
- Model, prompt, retrieval, and tool behavior.
- Model/API usage by unauthorized or unusual identities.
- Response retention and logging-policy conformance.
- Tool call loops and high-risk actions.

Use EventBridge/CloudWatch alarms and Step Functions/Lambda remediation to quarantine, block, roll back, or request review. Preserve the evidence needed to explain the automated action.

## Task 3.4: Responsible AI principles

### 3.4.1 Transparency

For users:

- Disclose AI involvement where appropriate.
- Present source evidence.
- State limitations and uncertainty.
- Offer human escalation.

For operators/auditors:

- Preserve agent or workflow traces.
- Record versioned decisions and sources.

Do not expose system prompts, secrets, private context, or provider-internal chain-of-thought. Traceability is not the same as chain-of-thought disclosure.

### 3.4.2 Fairness

Use a representative dataset with cohort labels and paired counterfactual cases. Compare quality, tone, refusal, recommendation, and error rates across groups. Track gaps over time. Use automated judges for scale and calibrated human evaluation for consequential decisions.

CloudWatch can store custom fairness metrics; it does not infer fairness automatically.

### 3.4.3 Policy compliance

Turn prose policy into:

- Guardrail configurations.
- Deterministic rules and Lambda checks.
- Model cards and user disclosures.
- Test cases and thresholds.
- CI/CD quality gates.
- Human approval paths.
- Fail-closed behavior for high-risk checks.

An audit framework or model card is evidence, not a legal guarantee.

## High-value service distinctions

| Service/capability | Best use | Not the best use |
|---|---|---|
| Bedrock Guardrails | Inline content, topic, prompt-attack, PII, grounding policies | S3 estate discovery or tool authorization |
| Amazon Comprehend | PII/entity analysis in text | Bedrock access control |
| Amazon Macie | Sensitive-data discovery and S3 posture | Synchronous prompt redaction |
| IAM | AWS API/resource authorization | Fine-grained semantic safety |
| IAM Identity Center | Workforce federation and account/application access | Customer application authorization engine |
| Lake Formation | Governed table/column/row/cell access | Model content filtering |
| KMS | Encryption-key control | Secret-value lifecycle |
| Secrets Manager | Secret storage/retrieval/rotation | AWS workforce federation |
| CloudTrail | Who called/changed an API and when | Full application reasoning or latency trace |
| CloudWatch | Metrics, logs, alarms, operational trends | API actor audit alone |
| X-Ray/agent trace | Request path, steps, latency, tool/retrieval behavior | Public chain-of-thought |
| Model Cards | Intended use, limits, risk, evaluation evidence | Runtime enforcement |
| Audit Manager | Best-practice control evidence | Compliance certification |

## Scenario decision table

| Hidden requirement | Choose | Reject |
|---|---|---|
| Stop unsafe request before model | Input Guardrails/pre-filter | Post-response-only filter |
| Preserve utility while hiding identity | Stable tokenization/masking | Delete every entity and lose context |
| Detect PII in stored audit transcripts | Macie | Comprehend used as S3 inventory |
| Answer must come from current approved source | RAG, source metadata, citations, grounding gate | Retrain for each policy update |
| Tool must never access other resources | Least-privilege role and server-side authorization | Prompt instruction |
| Data path must avoid public internet | Complete private endpoint/dependency design | “Private subnet” alone |
| Only approved data columns | Lake Formation and scoped IAM | Ask model not to query PHI |
| Regulated release must fail closed | Automated policy/evaluation gate | Dashboard-only monitoring |
| Auditable actor and resource change | CloudTrail | Model invocation log alone |
| Explain answer without leaking internals | Sources, limitations, trace for operators | Expose chain-of-thought |
| Organization must prevent guardrail bypass | Bedrock organization/account enforcement | Per-team convention |

## Failure-mode triage

| Symptom | First evidence | Likely fix |
|---|---|---|
| Harmful input reached model | Deployed guardrail config and input tags | Correct scope/version and add pre-filter |
| Safe request blocked | Category/threshold false-positive set | Tune threshold with validation dataset |
| PII in log | Invocation/application log sample | Pre-redact and apply log data protection |
| Wrong policy answer with citation | Retrieved chunk and source version | Fix source sync/filter/retrieval |
| Agent accessed wrong tenant | Identity-to-filter and tool authorization log | Server-derived tenant scope |
| Private request timed out | DNS, security group, flow logs, endpoint family | Correct endpoint and dependencies |
| Auditor cannot reproduce output context | Missing version/source identifiers | Emit decision metadata per response |
| Cohort gap appeared after prompt update | Fairness trend by version | Roll back and evaluate paired cases |
| Policy violation was detected but continued | Monitoring-only design | Add blocking/remediation workflow |

## Exam traps

- “Use Guardrails” is incomplete when the scenario explicitly requires custom pre-processing, deterministic tools, or post-validation.
- A harmless input can produce harmful output; protect both directions.
- RAG and citations reduce risk but do not prove truth.
- JSON shape is not semantic validity.
- A VPC endpoint is not an IAM grant.
- Macie is for S3 discovery, not inline model input.
- Model cards and Audit Manager provide evidence, not enforcement or legal attestation.
- CloudWatch Logs alone should not be described as an immutable ledger.
- CloudTrail is not an application decision trace.
- Do not expose provider-internal reasoning to satisfy transparency.
- Cross-Region inference must be checked against exact residency requirements.
- Lower temperature is not a safety, privacy, or fairness control.

## Local mock references

| Topic | Questions |
|---|---|
| Input/output Guardrails | PE1-Q07, Q15, Q24, Q28, Q73; PE2-Q03, Q18, Q20, Q42, Q63 |
| PII, masking, and stored discovery | PE1-Q05, Q13, Q69, Q75; PE2-Q21, Q29 |
| Hallucination, grounding, citations | PE1-Q22, Q32, Q43, Q46; PE2-Q45, Q57 |
| Prompt injection/adversarial tests | PE1-Q13, Q67, Q73; PE2-Q03, Q42, Q63 |
| Private access and data permissions | PE1-Q25, Q39, Q75; PE2-Q05, Q35, Q64 |
| Governance and evidence | PE1-Q06, Q12, Q29, Q36; PE2-Q04, Q32, Q46, Q59, Q67, Q74 |
| Fairness | PE1-Q30; PE2-Q08 |

## Skill mastery check

You are ready for this domain when you can:

- Place every safety/privacy control before, during, or after inference correctly.
- Explain why Guardrails, Comprehend, and Macie are not interchangeable.
- Design a private Bedrock path and list every dependent endpoint.
- Apply IAM, KMS, Lake Formation, and tool authorization without treating one as a substitute for another.
- Separate citations, relevance, grounding, and factual correctness.
- Define fair, transparent, policy-compliant release criteria.
- Map CloudTrail, CloudWatch, invocation logging, X-Ray, and model cards to distinct evidence.
- Identify and correct the questionable mock assumptions documented above.

## Recall questions

1. Design a five-layer defense for a public agent that reads account data and opens tickets.
2. Where must PII be removed if it must never reach the FM?
3. Which service finds PII already stored in S3, and why is it wrong for inline masking?
4. How do you prevent generated SQL from changing data?
5. What evidence separates bad retrieval from unfaithful generation?
6. Why is a vector score not a factual confidence score?
7. How do IAM and Lake Formation cooperate to hide PHI columns?
8. What does an interface endpoint solve, and what does it not solve?
9. How would you test whether a prompt treats two demographic cohorts differently?
10. What should a per-response governance log contain?
11. Which transparency information is useful to users, and which internal information must remain hidden?
12. Why is Prompt Management versioning not a complete production approval mechanism?
13. What must be added if CloudWatch logs are claimed to satisfy a strict immutable-record requirement?
14. How would organization-wide guardrail enforcement differ from application attachment?

## Official sources

- [Official Domain 3 tasks and skills](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain3.html)
- [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
- [Contextual grounding checks](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-contextual-grounding-check.html)
- [Bedrock structured outputs](https://docs.aws.amazon.com/bedrock/latest/userguide/structured-output.html)
- [Bedrock private VPC endpoints](https://docs.aws.amazon.com/bedrock/latest/userguide/vpc-interface-endpoints.html)
- [Bedrock IAM policy examples](https://docs.aws.amazon.com/bedrock/latest/userguide/security_iam_id-based-policy-examples.html)
- [Lake Formation permissions](https://docs.aws.amazon.com/lake-formation/latest/dg/managing-permissions.html)
- [Amazon Comprehend PII](https://docs.aws.amazon.com/comprehend/latest/dg/how-pii.html)
- [Amazon Macie](https://docs.aws.amazon.com/macie/latest/user/what-is-macie.html)
- [SageMaker Model Cards](https://docs.aws.amazon.com/sagemaker/latest/dg/model-cards.html)
- [Bedrock CloudTrail logging](https://docs.aws.amazon.com/bedrock/latest/userguide/logging-using-cloudtrail.html)
- [Bedrock model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html)
- [AWS Audit Manager GenAI Best Practices Framework](https://docs.aws.amazon.com/audit-manager/latest/userguide/aws-generative-ai-best-practices.html)
- [Responsible AI Lens](https://docs.aws.amazon.com/wellarchitected/latest/responsible-ai-lens/responsible-ai-lens.html)
