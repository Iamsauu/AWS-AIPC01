# Exam and Study Strategy

Status: Verified against the 2026-07-23 exam guide  
Official scope: Overall exam

## Exam facts

- Certification: AWS Certified Generative AI Developer – Professional.
- Exam code: AIP-C01.
- 75 total questions: 65 scored and 10 unscored.
- Question formats: multiple choice and multiple response.
- Treat multiple-response questions as all-or-nothing: identify every required choice.
- Unanswered questions are incorrect, and AWS states there is no penalty for guessing.
- Scaled score: 100–1,000.
- Passing score: 750.
- The scoring model is compensatory: passing depends on the total weighted result, not passing each domain separately.

An 85% mock score does not convert directly to an AWS scaled score. Use 85% on unseen, timed exams as a safety margin.

## Domain weights

| Domain | Weight | Approximate share of 65 scored questions |
|---|---:|---:|
| 1 | 31% | 20 |
| 2 | 26% | 17 |
| 3 | 20% | 13 |
| 4 | 12% | 8 |
| 5 | 11% | 7 |

The question counts are estimates, not promises. Domains 1–3 make up 77% of the blueprint, but weak operational or evaluation knowledge can still erase the desired safety margin.

## Read the scenario as a constraint system

Before reading the options, extract:

1. Workload: interactive, streaming, asynchronous, batch, agentic, RAG, or multimodal.
2. Scale: requests, tokens, concurrency, data volume, and burst pattern.
3. Quality: deterministic, creative, grounded, fair, structured, or human-reviewed.
4. Security: identity, least privilege, data classification, private paths, residency, and retention.
5. Reliability: retries, idempotency, fallback, Region failure, timeout, and recovery.
6. Operations: managed service preference, audit evidence, deployment, and monitoring.
7. Optimization: lowest cost, lowest perceived latency, highest throughput, or least operational overhead.

The correct option must satisfy every hard constraint. A technically possible option is wrong if it violates one required constraint.

## Interpret common phrases

| Scenario phrase | Likely direction |
|---|---|
| Least operational overhead | Prefer a managed/native capability |
| Immediate acknowledgment; work can finish later | Queue or event decoupling |
| Bursty asynchronous workload | SQS plus workers or managed batch |
| Add consumers without changing producer | EventBridge fanout |
| Long human approval | Step Functions Standard callback |
| Exact ID plus semantic description | Hybrid keyword/vector search |
| Current source content without retraining | RAG and synchronized ingestion |
| Same model; predictable sustained peak | Provisioned Throughput |
| Same model; temporary regional capacity issue | Cross-Region inference profile |
| Reduce time before users see output | Streaming and TTFT optimization |
| Same stable prompt prefix | Prompt caching |
| Auditable routing or agent path | Step Functions history or agent trace |
| Stored sensitive data discovery | Macie |
| Redact free text before inference | Comprehend/Guardrails preprocessing |
| Who changed or called an AWS resource | CloudTrail |
| Why one runtime request was slow | Structured logs and X-Ray |
| Quality before promotion | Fixed evaluation dataset and automated gate |

## Eliminate distractors systematically

Reject an option when it:

- Uses manual work for a recurring automated requirement.
- Adds EC2/EKS administration when a serverless or managed feature meets the need.
- Couples an incoming synchronous request to slow downstream work.
- Uses request count when tokens drive capacity.
- Uses vector-only retrieval for exact identifiers.
- Calls an FM the deterministic authority for financial, operational, or SQL results.
- Logs errors without metrics, alarms, traceability, or correlation.
- Detects PII after it has already been sent to the model.
- Claims that citations or vector scores guarantee factual correctness.
- Changes prompts without a baseline dataset and release gate.
- Retries non-idempotent side effects without an idempotency key.
- Uses the wrong endpoint or payload contract for the selected model.

## Study loop

For each topic:

1. Read the domain summary.
2. Read its canonical concept page.
3. Reproduce the decision table from memory.
4. Complete or reason through the related lab.
5. Answer recall questions without options.
6. Solve changed scenarios.
7. Review the local mock mappings.
8. Record the reason for every wrong distractor.

Do not repeat the same mock until its wording is memorized and call that readiness.

## Error log format

For every missed question, record:

| Field | Example |
|---|---|
| Question ID | PE2-Q73 |
| Miss type | Chose semantic instead of hybrid search |
| Hidden constraint | Exact model number plus natural-language symptom |
| Correct rule | Hybrid search, then rerank |
| Distractor failure | Vector-only retrieval misses exact tokens |
| Canonical page | RAG and vector search |
| New scenario | CVE ID plus descriptive vulnerability |

Classify misses as:

- Knowledge gap.
- Misread constraint.
- Service confusion.
- API/payload confusion.
- Security omission.
- Cost/latency tradeoff error.
- Overengineering.
- Changed or questionable mock behavior.

## Readiness gate

Book the exam only when all are true:

- At least 85% on two unseen, timed, full-length exams.
- At least 80% in each domain.
- At least 90% across Domains 1–3 combined.
- Can explain why every distractor is wrong.
- Can solve recall questions without seeing service-name options.
- Can draw or explain the six lab architectures.
- No unresolved Tier 1 gaps remain in the coverage matrix.

## Primary source

[Official AIP-C01 exam guide](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01.html)

