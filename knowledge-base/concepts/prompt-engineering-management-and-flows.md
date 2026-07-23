# Prompt Engineering, Prompt Management, and Flows

Status: Verified  
Official tasks: 1.6.1–1.6.6  
Last verified: 2026-07-23

## Why this matters

A production prompt is a versioned behavioral contract. AIP-C01 tests instruction design, context management, deterministic control, Prompt Management governance, quality gates, and managed orchestration with Bedrock Flows.

## Core concepts

### Prompt contract

A strong prompt separates:

1. **role and scope** — who the model is and what it may do;
2. **task and success criteria** — the exact required outcome;
3. **trusted context** — clearly delimited evidence and its authority;
4. **constraints** — policy, format, length, tone, and forbidden behavior;
5. **examples** — representative demonstrations when needed;
6. **output specification** — preferably a machine-enforceable schema;
7. **uncertainty behavior** — what to do when evidence is missing, stale, or conflicting.

Do not hide critical constraints in a long prose paragraph. Give them explicit structure and test each one.

### Prompting techniques

| Technique | Use when | Risk/control |
|---|---|---|
| Zero-shot | Task is common and instructions are sufficient | Validate edge cases |
| Few-shot | Format, label boundary, tone, or reasoning pattern needs demonstration | Curate diverse examples; avoid leaking answers |
| Role/context framing | Domain vocabulary and scope matter | Role text is not authorization |
| Decomposition | Task has dependent steps | Validate intermediate results |
| Chain-of-thought-style instruction | Complex reasoning quality improves | Do not require disclosure of hidden reasoning; request concise rationale/evidence |
| Self-check/critique | A second pass catches defined errors | Extra tokens/latency; still needs external evaluation |
| RAG grounding | Answer must use current/private evidence | Verify retrieval and citations |
| Tool use | Deterministic data/action is required | Validate arguments and authorize actions |

### Inference controls

Lower temperature and a narrower sampling distribution usually improve repeatability. They do not make an answer factual. Change one control at a time and evaluate on a representative held-out set.

Set `maxTokens` for the task, not the model maximum. Too little truncates required output; too much increases worst-case cost, latency, and capacity consumption.

### Structured outputs

Current Bedrock structured outputs can constrain supported model responses to a supplied JSON schema through `outputConfig.textFormat` on `Converse`, or through documented model-native fields with `InvokeModel`. Strict tool use can constrain tool arguments on supported APIs/models.

This is stronger than saying “return JSON,” but it guarantees structure, not factual correctness or valid business semantics. Parse, apply business rules, and verify references after generation. The first use of a new schema can incur schema-compilation latency; subsequent requests can reuse the cached grammar. Check current model/API compatibility, including documented citation restrictions.

## How it works

### Context and conversation

Models do not remember prior API calls. The application persists conversation state, selects the useful turns, and sends them again.

```text
new message
  → authenticate and load permitted history
  → classify intent / detect ambiguity
  → clarify through deterministic workflow if needed
  → retrieve current evidence
  → fit history + evidence + instructions to token budget
  → invoke
  → validate and persist response metadata
```

DynamoDB fits keyed conversation state and turn metadata. Keep recent turns and a versioned summary of older content. Never include another tenant’s history, and do not trust a model-written summary without validation for critical facts.

For business-critical ambiguity, use a deterministic branch—for example Comprehend intent/classification plus Step Functions—rather than asking a model to decide silently. Record the user’s clarification and resume the correct branch.

### Prompt Management

Bedrock Prompt Management provides:

- reusable prompts addressed by ARN;
- variables;
- a mutable `DRAFT`;
- variants for model/config/prompt alternatives;
- numbered immutable version snapshots;
- integration with supported Bedrock features.

Develop against a draft, evaluate it, create a numbered version, pass an external release gate, and invoke the exact approved version. Do not let production implicitly track a mutable draft.

Current AWS Prompt Management documentation does **not** describe a native reviewer/approval-status workflow. Enforce approval with CI/CD or a manual deployment gate and retain CloudTrail plus sanitized application evaluation/release records. Versioning is not approval.

When using a Prompt Management prompt ARN with `Converse`, supply its declared `promptVariables`. Current API restrictions prohibit also specifying `system`, `inferenceConfig`, `toolConfig`, or `additionalModelRequestFields`; put supported configuration in the managed prompt.

### Testing and feedback loop

```text
production failures and rated samples
  → classify error type
  → update prompt/retrieval/model candidate
  → offline regression on held-out data
  → safety and format gate
  → canary/cohort release
  → compare versioned metrics
  → promote or rollback
```

Test normal, boundary, missing-context, conflicting-context, malicious-instruction, multilingual, long-input, malformed-data, and denied-action cases. Keep evaluation data separate from examples used in the prompt.

A Lambda validator can enforce JSON shape, allowed values, citations, and deterministic business rules. An LLM judge can score rubric qualities but must be calibrated against human judgments and protected from seeing candidate identities/order when that could bias it.

### Prompt optimization

Simple prompt optimization is a quick heuristic rewrite for one prompt/model and does not replace evaluation.

Advanced Prompt Optimization is evaluation-driven: supply a JSONL dataset, choose target models and evaluation method, and compare generated candidates. Current documentation supports up to five target models per job. Always run held-out validation before promotion; optimization can overfit its examples.

### Bedrock Flows

Flows provide a visual/API-defined graph of nodes and connections. Useful nodes include:

- input and output;
- prompt;
- condition;
- Knowledge Base;
- Lambda;
- agent;
- iterator/collector;
- inline code where supported.

Use a Flow for managed GenAI chains, branching, reusable preprocessing/postprocessing, Knowledge Base calls, and bounded service logic. Enable trace during testing to observe node inputs/outputs and routing.

To deploy, create an immutable Flow version and point an alias at it. Shift or replace the alias target only after evaluation; rollback by restoring the last known-good version.

Use Step Functions Standard instead when the workflow needs durable execution, broad AWS integration, explicit retries/catches, or human approval lasting hours or days. A Flow can invoke logic; it is not a general-purpose human workflow engine.

### Prompt caching

Prompt caching can reduce cost and latency for a sufficiently large repeated prefix on supported models. Put stable system instructions, tools, and shared context before cache checkpoints; place user-specific and rapidly changing content after them. Minimum cacheable tokens, checkpoint count, write/read pricing, and TTL behavior are model-specific. Measure hit rate rather than assuming savings.

## AWS services and APIs

- Bedrock Prompt Management: prompts, variables, variants, drafts, versions.
- Bedrock Runtime: prompt ARN invocation, structured outputs, Guardrail attachment.
- Bedrock Flows: nodes, connections, versions, aliases, tracing.
- Bedrock Guardrails: content, denied-topic, sensitive-information, grounding, and other supported policies.
- Bedrock Evaluations and Prompt Optimization: measured prompt/model comparison.
- Step Functions: durable branching, retries, and approval workflows.
- Lambda: deterministic pre/post-processing and validation.
- Comprehend: intent/entity/PII signals.
- DynamoDB: conversation state and versioned summaries.
- CloudWatch and CloudTrail: operational telemetry and API audit.

## Architecture patterns

### Governed prompt release

```text
Prompt Management DRAFT
  → versioned test dataset
  → quality/safety/format evaluation
  → CI/CD or manual approval
  → numbered prompt version
  → canary/cohort
  → promote or rollback
```

### Clarification workflow

```text
request → deterministic intent/ambiguity check
            ├─ clear → governed prompt
            └─ unclear → ask bounded question → persist answer → resume
```

### Flow chain

```text
Input → preprocess Lambda → retrieval/query prompt
          → condition
             ├─ sufficient → answer prompt → validate Lambda → Output
             └─ insufficient → explicit no-evidence response
```

## Decision table

| Requirement | Prefer | Avoid |
|---|---|---|
| Reusable variables and numbered snapshots | Prompt Management | Prompt copied into application code |
| Native JSON shape guarantee | Structured outputs | “Return valid JSON” only |
| Reusable GenAI chain with branches | Bedrock Flows | Large bespoke Lambda orchestrator |
| Durable long-running approval | Step Functions Standard | Treating Flow or Prompt Management as native approval |
| Conversation memory | Authorized DynamoDB history + summary | Assuming model remembers requests |
| Critical ambiguous intent | Deterministic classify/clarify branch | Silent model guess |
| Production prompt release | External approval + exact version ARN | Mutable draft |
| Compare variants | Same dataset/rubric and controlled cohort | Uncontrolled traffic comparison |
| Repeated stable prefix | Prompt caching after compatibility test | Caching dynamic user data as prefix |
| Enforce policy | Guardrails plus application controls | Prompt prose alone |
| Quick candidate rewrite | Simple optimization | Claiming it proves improvement |
| Data-driven multi-model optimization | Advanced optimization + held-out test | Training/evaluating on identical examples only |

## Security and governance

- Treat retrieved documents and user text as untrusted data; delimit them and instruct the model not to follow embedded commands.
- A role instruction does not grant authorization. Enforce identity, tenant, resource, and action policy in application/tool layers.
- Attach the approved Guardrail identifier/version at invocation where required.
- Restrict who can edit drafts, create versions, move Flow aliases, or invoke production artifacts.
- Use exact prompt, Flow, Guardrail, model, and evaluation versions for reproducibility.
- Keep secrets and raw PII out of prompt templates, variables, traces, and logs.
- Sanitize Flow trace and model-invocation telemetry before broad operator access.
- Validate tool arguments and reauthorize every side effect; never trust generated parameters.
- Audit API changes with CloudTrail and release decisions with an external change record.

## Cost, latency, and reliability

| Driver | Control |
|---|---|
| Long repeated prefix | Prompt caching if minimums and reuse justify it |
| Growing chat history | Preserve recent turns, summarize/prune older turns |
| Verbose response | Explicit schema/length and task-sized `maxTokens` |
| Self-critique/multiple prompts | Use only where measured quality gain exceeds extra calls |
| Flow node fanout | Bound iteration and branches; instrument node latency |
| New structured-output schema | Warm/test schema compilation before latency-sensitive release |
| Prompt optimization jobs | Limit to justified models/data and gate candidates |
| Model or node transient error | Bounded retry/backoff and explicit degraded output |

Token reduction must preserve instructions and evidence. Blind truncation often removes the constraint that prevents the most expensive failure.

## Failure modes and troubleshooting

| Symptom | Likely cause | Evidence | Fix |
|---|---|---|---|
| Same input changes unexpectedly | sampling/model/prompt version drift | request metadata and version | Pin versions/settings; regression test |
| JSON still fails business logic | schema guarantees shape only | validator failure | Add semantic rules and reference checks |
| Prompt ARN request rejected | disallowed fields supplied with managed prompt | runtime request | Move configuration into prompt and pass variables only |
| Chat contradicts earlier fact | history omitted or summary corrupted | assembled context | Persist/select history and validate summary |
| Wrong branch taken | model inferred critical intent | trace and classifier output | Deterministic clarification path |
| Production changed without approval | application invoked DRAFT/latest | ARN/version and release logs | Pin numbered approved version |
| Flow gives wrong output | condition mapping/node input mismatch | Flow trace | Inspect node data and connection expression |
| Flow update not visible | alias still targets prior version | version/alias | Point alias to tested version |
| Cache hit rate is low | dynamic content before checkpoint or prefix too short | cache metrics/token layout | Move stable prefix first; verify thresholds |
| LLM judge prefers one style | biased rubric/order/model | human calibration | Randomize/blind, refine rubric, calibrate |
| Prompt injection overrides policy | source/user text treated as instruction | retrieved content and prompt | Delimit, least privilege, Guardrails, validation |

## Common exam traps

1. Lower temperature improves repeatability, not factuality.
2. More examples are not always better; they consume context and can bias the model.
3. A JSON instruction is weaker than schema-constrained structured output.
4. Structured output does not validate truth or business rules.
5. Models have no memory across API requests.
6. Prompt Management versions prompts; current documentation does not provide a native approval-state workflow.
7. A production ARN should target an approved numbered version, not mutable `DRAFT`.
8. Prompt prose is not an authorization or safety boundary.
9. Flows are strong for GenAI chains; Step Functions Standard is stronger for durable human approval.
10. Enabling Flow trace helps diagnose routing but can expose sensitive content.
11. Prompt caching only helps repeated stable prefixes that meet model-specific rules.
12. Optimization generates candidates; held-out evaluation decides promotion.
13. “A/B testing” in a mock may require application/cohort routing even if Flows execute variants.
14. Request fields allowed with an inline prompt may be prohibited with a Prompt Management ARN.

## Local mock references

Mocks are useful patterns but are not authoritative specifications.

| Focus | Questions |
|---|---|
| Prompt/model evaluation gate | PE1-Q4, Q55, Q59; PE2-Q8, Q49 |
| Determinism and token limits | PE1-Q9, Q44; PE2-Q54 |
| Roles, schemas, Guardrails | PE1-Q24, Q29; PE2-Q23, Q24 |
| Prompt governance | PE1-Q6; PE2-Q4 |
| Flows and branching | PE1-Q30, Q34; PE2-Q27, Q43 |
| Advanced Prompt Optimization | PE2-Q30 |
| Conversation/clarification | PE2-Q71 |

Documentation correction: PE1-Q6 overstates Prompt Management as supplying an approval workflow. PE2-Q4’s external manual approval around a versioned prompt matches the documented feature boundary. PE1-Q30’s “A/B” pattern still needs an explicit traffic/cohort and measurement design; a Flow does not automatically make an experiment fair.

## Hands-on validation

1. Create a prompt with role, delimited context, constraints, variables, schema, and insufficient-evidence behavior.
2. Compare temperature settings on the same held-out dataset and measure consistency plus quality.
3. Use structured output, then deliberately produce a schema-valid but business-invalid value and catch it.
4. Create Prompt Management `DRAFT`, two variants, and a numbered version; invoke the exact version with variables.
5. Try prohibited `Converse` fields with a prompt ARN and record the corrected request.
6. Build a Flow with prompt, Knowledge Base, condition, Lambda, and output nodes; inspect its trace.
7. Deploy a Flow version through an alias, then roll back the alias.
8. Persist conversation history in DynamoDB and prove tenant isolation and summarization.
9. Run Advanced Prompt Optimization and independently reject or promote results using held-out data.
10. Test prompt caching with stable versus dynamic prefix placement and compare cache metrics.

## Recall questions

1. Which seven sections form a production prompt contract?
2. When does few-shot prompting help, and what does it cost?
3. Why does low temperature not guarantee factuality?
4. What does a structured-output schema guarantee and not guarantee?
5. Where should conversation memory live?
6. When should ambiguity be handled outside the model?
7. What is mutable in Prompt Management, and what is an immutable snapshot?
8. Where does approval live if Prompt Management has no documented approval state?
9. Which fields cannot accompany a Prompt Management ARN in `Converse`?
10. When is a Bedrock Flow preferable to Step Functions?
11. When is Step Functions Standard preferable to a Flow?
12. How do Flow versions and aliases enable rollback?
13. Why must optimization results face a held-out evaluation?
14. Which prefix layout maximizes prompt-cache reuse?
15. Why can a Flow trace create a security risk?

## Official sources

- [Design a prompt](https://docs.aws.amazon.com/bedrock/latest/userguide/design-a-prompt.html)
- [Prompt engineering guidelines](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-engineering-guidelines.html)
- [Prompt Management](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-management.html)
- [Create prompts and versions](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-management-create.html)
- [Deploy and use Prompt Management prompts](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-management-deploy.html)
- [Structured outputs](https://docs.aws.amazon.com/bedrock/latest/userguide/structured-output.html)
- [Bedrock Flows](https://docs.aws.amazon.com/bedrock/latest/userguide/flows.html)
- [Flow nodes](https://docs.aws.amazon.com/bedrock/latest/userguide/flows-nodes.html)
- [Trace a Flow](https://docs.aws.amazon.com/bedrock/latest/userguide/flows-trace.html)
- [Deploy a Flow](https://docs.aws.amazon.com/bedrock/latest/userguide/flows-deploy.html)
- [Advanced Prompt Optimization jobs](https://docs.aws.amazon.com/bedrock/latest/userguide/advanced-prompt-optimization-jobs.html)
- [Evaluate optimized prompts](https://docs.aws.amazon.com/bedrock/latest/userguide/advanced-prompt-optimization-evaluation.html)
- [Prompt caching](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html)
