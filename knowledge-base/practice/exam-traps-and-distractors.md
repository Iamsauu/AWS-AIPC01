# Exam Traps and Distractor Patterns

Status: Derived from the official blueprint and 150 local mock questions

## 1. “Possible” is not “best”

EC2, EKS, custom Lambda code, or manual review can often be made to work. Reject them when the question asks for least operational overhead and a managed feature satisfies every requirement.

Do not apply this mechanically: managed simplicity does not excuse missing security, data access, or failure controls.

## 2. Request count versus token capacity

Trap: choose request-per-second scaling for requests with highly variable prompt size.

Correct reasoning:

- Estimate tokens using the selected model/request shape.
- Include output budget.
- Admit by token budget when tokens drive throttling.

## 3. Streaming versus total speed

Trap: claim streaming reduces total model compute or cost.

Correct reasoning: streaming reduces time to first visible output. Measure TTFT and total latency independently.

## 4. Retry versus capacity

Trap: add more retries to a sustained throttling problem.

Correct reasoning: bounded backoff handles transient errors. Capacity problems require lower token load, queue/batch, quota/capacity, provisioned throughput, or routing.

## 5. Cross-Region inference versus Provisioned Throughput

- Temporary regional capacity/resilience: cross-Region inference.
- Predictable sustained throughput in a required Region/model: Provisioned Throughput.
- Check data-residency boundaries before cross-Region routing.

## 6. Batch versus synchronous

Trap: invoke one request at a time for millions of offline records.

Use Bedrock batch inference or SageMaker Batch Transform when outputs can arrive asynchronously.

## 7. SQS versus EventBridge

- Need backlog, smoothing, worker consumption: SQS.
- Need rules and add/remove consumers without producer change: EventBridge.
- Need push notifications: SNS.

## 8. At-least-once delivery without idempotency

Trap: retry a CRM/ticket/payment side effect with no stable idempotency key.

Use a business event ID and make the downstream update idempotent.

## 9. Long approval with synchronous compute

Trap: make Lambda wait hours or poll continuously.

Use Step Functions Standard callback and task token.

## 10. FM as sole controller of high-risk workflow

Trap: let the model freely branch, execute SQL, or decide unsafe actions.

Use deterministic state/Choice/validation and least-privilege tools around the model.

## 11. Tool schema as a security boundary

Trap: assume the FM always produces valid parameters because a schema exists.

Validate in the tool implementation, authorize the caller/resource, return structured errors, and make side effects idempotent.

## 12. Unlimited agent loop

Trap: keep calling an unhealthy tool because the model requests it.

Use max cycles, timeouts, risk flags, structured error, and a circuit breaker.

## 13. MCP means “call Lambda directly”

Trap: bypass MCP while claiming a portable MCP tool interface.

Use an MCP server/client contract. Choose Lambda for lightweight stateless tools and a managed container/runtime for heavier/stateful/streaming tools.

## 14. Trace means chain-of-thought

Agent/Flow trace records orchestration events. Do not promise or expose provider-internal reasoning tokens.

## 15. Vector-only for exact identifiers

Exact codes can be semantically weak. Use keyword/metadata constraints plus vector search, then rerank when necessary.

## 16. Reranking before candidate retrieval

A reranker improves ordering of candidates; it cannot recover a relevant document that retrieval never returned. Fix retrieval/filters/candidate size first.

## 17. Broad chunks versus hierarchical chunks

Trap: increase fixed chunk size when users need a precise clause with surrounding evidence.

Hierarchical chunking can match a small child and return the larger parent context.

## 18. Metadata as content

Some metadata should influence semantic similarity; other fields should only filter. Treat dates, owner, tenant, or jurisdiction according to the use case and supported metadata configuration.

## 19. Retrieval score as factual confidence

Similarity means “close in embedding space,” not “the answer is true.” Use evidence, grounding/faithfulness checks, citations, deterministic validation, or insufficient-evidence behavior.

## 20. Fix prompts before isolating retrieval

If citations point to the wrong jurisdiction, inspect `Retrieve` results and metadata before changing generation prompts.

## 21. RAG without refresh

Trap: update source objects but never synchronize ingestion or confirm success.

Trigger/schedule ingestion and monitor terminal job state plus document/chunk failures.

## 22. Rebuild model when documents change

For changing factual content, prefer RAG/source synchronization over retraining the FM.

## 23. Same JSON body for every model

Converse provides a common message interface only for supported models. `InvokeModel` bodies remain model/provider specific. Use a model-aware adapter.

## 24. Converse for every modality

Specialized embedding/image APIs may require `InvokeModel` with the native schema. Choose by model capability, not preference.

## 25. High `maxTokens` “for safety”

An excessive output ceiling can consume token-based capacity and increase cost exposure. Set a task-specific ceiling; ensure it is high enough to finish required structured output.

## 26. Temperature is the whole quality strategy

Lower temperature can improve consistency, but cannot fix missing evidence, weak prompt structure, retrieval errors, or invalid tool schemas. Evaluate parameter candidates on a representative set.

## 27. Prompt Management equals approval workflow

Prompt Management provides prompts, variables, variants, and versions. Organizational approval/promotion should be enforced through IAM and a release workflow. Do not assume version creation alone equals governance approval.

## 28. Guardrails only on output

Unsafe input can influence the model before output filtering. Apply input controls before inference and output controls before returning content.

## 29. Guardrails replace application validation

Guardrails do not replace:

- Authorization.
- Tool parameter validation.
- Read-only SQL templates.
- Required disclaimers.
- JSON schema validation.
- Business-rule checks.

## 30. WAF equals FM Guardrail

WAF controls web-layer traffic patterns. Bedrock Guardrails evaluate supported semantic safety/policy conditions.

## 31. Macie for real-time prompt redaction

Macie discovers sensitive data in S3. Use Comprehend, Guardrails, and application preprocessing for live text.

## 32. PII masking that destroys utility

Use consistent placeholders where the model must retain relationships between entities. Apply output protection too.

## 33. Private network means authorized

VPC endpoints keep traffic off public paths. IAM, endpoint policies, S3 policies, and Lake Formation still determine access.

## 34. CloudTrail versus CloudWatch versus X-Ray

- CloudTrail: AWS API audit.
- CloudWatch: metrics/logs/alarms.
- X-Ray: request path and latency.
- Evaluation: semantic quality.

No single one replaces the others.

## 35. Log everything for audit

Raw prompts/responses can create a privacy breach. Use sanitized structured metadata, encryption, least-privilege log access, and retention. Enable invocation logging only when classification permits.

## 36. Synthetics means quality evaluation

A canary proves the deployed route works and can test deterministic properties. Use semantic/RAG/model evaluation for answer quality.

## 37. LLM judge as objective truth

Judges have bias. Use rubrics, calibration, references, human samples, and version tracking.

## 38. Overall average hides fairness

Report cohort-level outcomes using matched/stratified datasets.

## 39. Repeated mock score means readiness

Memorized wording inflates results. Require unseen timed exams and transformed scenarios.

## 40. Increasing startup timeouts fixes every SageMaker failure

Timeouts help only if the model is still legitimately downloading/loading. Insufficient disk, GPU memory, image errors, and incompatible serving configuration require different fixes.

## Five-step option test

For each candidate:

1. Does it satisfy every explicit requirement?
2. Does it violate security, residency, or authorization?
3. Does it fit interaction and failure behavior?
4. Is there a more managed option with equal capability?
5. Can I explain the operational evidence and rollback?

