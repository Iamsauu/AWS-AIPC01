# Lab 5 — Model, RAG, and Agent Evaluation

Status: Guided lab; verify feature, model, metric, and Region support before execution  
Estimated cost: Variable; evaluation and FM invocations incur charges

## Objective

Build an evaluation workflow that:

- Compares two model/prompt configurations.
- Isolates retrieval quality from generation.
- Measures an agent’s task/tool behavior.
- Gates a candidate release.
- Preserves versioned evidence.

## Architecture

```text
Versioned JSONL datasets in S3
        |
        +--> Bedrock model evaluation --> S3 results
        +--> RAG retrieve-only evaluation --> S3 results
        +--> agent replay/evaluation --> result store
                                            |
                                 Glue/Athena or analysis
                                            |
                             quality gate + CloudWatch
```

## Prerequisites

- AWS account and an approved test Region.
- Access to candidate Bedrock models.
- S3 bucket with encryption, least-privilege access, and lifecycle policy.
- A small Knowledge Base or retrieval system.
- A non-production agent/tool workflow.
- No sensitive production data.

## Part A — Create the benchmark

Create three datasets:

### Model/prompt dataset

Include:

- Normal requests.
- Long/ambiguous requests.
- Required JSON output cases.
- Safety/refusal cases.
- Reference responses or scoring criteria.
- Cohort-matched fairness cases.

### Retrieval dataset

For each query:

- SME reference answer.
- Expected source/document IDs where useful.
- Important exact identifiers.
- Expected context coverage.

### Agent dataset

For each task:

- Initial request.
- Allowed tools.
- Expected final outcome.
- Required/forbidden tool calls.
- Maximum acceptable steps.
- Tool-failure variation.

Version datasets in S3. Record a dataset ID and content hash in evaluation metadata.

## Part B — Compare model/prompt configurations

Choose:

- Baseline model + prompt version + inference settings.
- Candidate model/prompt/settings.

Hold the dataset constant.

Run supported Bedrock model evaluation jobs using:

- Deterministic validation for required JSON fields.
- Supported automated metrics.
- LLM-as-a-judge with a precise rubric.
- A human sample for calibration or subjective quality.

Collect:

- Relevance/helpfulness.
- Factuality.
- Format-valid rate.
- Safety/policy failure.
- Cohort-level fairness.
- Input/output tokens.
- Invocation latency.

Do not select a candidate from quality average alone.

## Part C — Isolate RAG retrieval

Run a retrieve-only evaluation, or invoke `Retrieve` over the fixed dataset.

Inspect:

- Context relevance.
- Context coverage.
- Expected source hit.
- Rank/top-k behavior.
- Exact identifier retrieval.
- Latency.

Then run end-to-end retrieve-and-generate evaluation and measure:

- Faithfulness.
- Answer relevance.
- Citation correctness/coverage.
- Unsupported-claim rate.

If retrieval-only fails, correct chunking, embeddings, metadata, filters, hybrid search, or index freshness before changing the generation prompt.

## Part D — Evaluate the agent

Replay the agent tasks in an isolated environment.

Capture:

- Final result.
- Task completion.
- Trace/trajectory.
- Tool choices and arguments.
- Repeated calls.
- Error recovery.
- Step count.
- Tokens and latency.
- Safety termination.

Inject failures:

- Missing required tool parameter.
- Tool timeout.
- Tool returns a high-risk result.
- Tool remains unavailable across retries.

Verify:

- Agent asks for missing information.
- Structured error is handled.
- Maximum cycle stops the workflow.
- Circuit breaker suppresses repeated unhealthy calls.
- High-risk result ends/escalates correctly.

## Part E — Define the release gate

Example policy:

- 100% required JSON validity.
- No critical safety or PII leakage.
- No material cohort regression.
- Faithfulness above the approved threshold.
- Retrieval coverage not below baseline.
- Agent completion not below baseline.
- p95 latency and cost per successful task within budget.

Thresholds must come from workload risk and baseline evidence, not this example.

Pipeline behavior:

1. Run tests/evaluations.
2. Publish report.
3. Fail if any critical threshold fails.
4. Require approval where policy demands.
5. Canary the candidate.
6. Collect live metrics/feedback.
7. Promote or roll back.

## Validation evidence

Retain:

- Dataset version/hash.
- Candidate configuration versions.
- Evaluation job identifiers.
- Rubric/judge model version.
- Raw and summarized results.
- Human calibration result.
- Gate decision and approver.
- Canary outcome.

## Expected failure tests

- Candidate outputs fluent but invalid JSON.
- Retrieval returns wrong version due to missing metadata filter.
- Judge favors verbose answers.
- Overall fairness passes while one cohort regresses.
- Agent retries a deterministic tool-validation error.

Document which evidence exposes each failure.

## Cost and safety cautions

- Use small datasets first to validate schemas and permissions.
- Estimate model/evaluation calls before a large run.
- Never send unapproved sensitive data to a judge model.
- Treat human evaluation output as governed data.
- Apply S3 retention and access policies.

## Cleanup

- Delete temporary evaluation jobs/resources if no longer needed.
- Expire temporary S3 inputs/outputs according to policy.
- Remove test-only IAM permissions.
- Stop or delete temporary custom endpoints/vector resources.

## Exam lessons

- Evaluation target determines the evaluation type.
- Retrieve-only testing isolates RAG failures.
- LLM-as-judge scales but needs calibration.
- Human judgment remains important for subjective/high-risk decisions.
- A quality gate must include structure, safety, quality, cost, and latency.
- Canary deployment complements offline evaluation.

## Sources

- [Amazon Bedrock evaluations](https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation.html)
- [Official Domain 5](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain5.html)

