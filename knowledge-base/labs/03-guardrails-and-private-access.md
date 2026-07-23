# Lab 3 — Guardrails and Private Access

Status: Guided lab; verify feature, model, endpoint, and Region support before execution  
Estimated cost: Variable; Guardrails, model inference, interface endpoints, Athena scans, logs, and KMS can incur charges

## Objective

Build and test a protected Amazon Bedrock request path that:

- Evaluates untrusted input before retrieval or model inference.
- Applies the same versioned guardrail to model output.
- Masks selected personally identifiable information (PII) and blocks prohibited content.
- Uses IAM to deny direct model inference that omits the approved guardrail.
- Reaches Bedrock privately through the correct VPC endpoint.
- Enforces row- or column-level data access in AWS Lake Formation.
- Records useful operational evidence without deliberately storing raw sensitive text.
- Proves controls with negative tests, not only a successful request.

This is a training pattern, not a production reference implementation. Replace example thresholds and policies with controls approved for the workload.

## Architecture

```text
Authenticated test client
  -> private API / workload role
  -> application in private subnets
       1. ApplyGuardrail(source=INPUT)
            -> block, mask, or continue
       2. authorized retrieval/query
            -> Athena + Lake Formation row/column filter
            -> approved S3 data
       3. Converse/InvokeModel with approved guardrail ID + version
            -> model input and output guardrail evaluation
       4. deterministic output/schema and authorization validation
  -> sanitized response

Private dependencies:
  bedrock-runtime interface endpoint
  + S3 gateway endpoint
  + required Athena, Glue, KMS, STS, and Logs paths

Evidence:
  CloudTrail + redacted application logs + CloudWatch metrics
  + VPC Flow Logs/DNS checks + Athena/Lake Formation evidence
```

Use `bedrock-runtime` for Bedrock-native `ApplyGuardrail`, `Converse`, and `InvokeModel` calls. Add `bedrock` only if the workload must use Bedrock control-plane APIs, and add `bedrock-agent-runtime` if the lab is extended to Knowledge Bases or Agents runtime APIs.

## Prerequisites

- A sandbox AWS account and an approved test Region.
- Access to one supported Bedrock text model.
- Permission to create and version a guardrail.
- A VPC with private subnets, DNS support, and no public IP requirement.
- Permission to create interface/gateway endpoints and inspect VPC Flow Logs.
- A small non-sensitive S3 dataset cataloged in AWS Glue.
- Athena and Lake Formation configured for the test table.
- Separate administrator and runtime roles.
- An encrypted CloudWatch log group with short retention.
- Synthetic names, email addresses, account identifiers, and tenant rows only.

Record these values before testing:

```text
AWS Region:
Runtime role ARN:
Model or inference profile ID:
Guardrail ID:
Published guardrail version:
VPC endpoint IDs:
Athena workgroup:
Glue database/table:
Approved tenant/row:
Disallowed tenant/row:
```

Do not use the mutable `DRAFT` guardrail in a release test. Publish a numeric version so that the tested policy can be reproduced.

## Part A — Create and baseline the guardrail

Create a guardrail with controls appropriate to the synthetic test:

- Content filters for harmful categories and prompt attacks.
- One denied topic, such as instructions to reveal internal credentials.
- A word filter for a synthetic prohibited term.
- PII handling:
  - Mask an email address and person name.
  - Block a synthetic high-risk identifier.
- Separate blocked-input and blocked-output messages.

Start uncertain filters in detect mode, run a labeled test set, and then choose block or mask actions. A stricter threshold can increase false positives; it is not automatically safer for the business workflow.

Build a small baseline:

| Case | Expected action |
|---|---|
| Ordinary permitted question | No intervention |
| Direct prohibited topic | Block input |
| Prompt-injection attempt | Detect or block according to policy |
| Name and email in an allowed request | Mask configured PII |
| High-risk synthetic identifier | Block |
| Borderline legitimate content | No false-positive block |

Publish the tested guardrail as a numeric version and record its ARN/version with the test results.

## Part B — Guard input before retrieval

Call `ApplyGuardrail` before any retrieval, tool call, or model invocation:

```text
ApplyGuardrail(
  guardrailIdentifier = approved guardrail,
  guardrailVersion    = published numeric version,
  source              = INPUT,
  content             = untrusted user text
)
```

Handle the result explicitly:

1. If the action is an intervention, return the approved blocked message and do not retrieve or invoke the model.
2. If configured PII is masked, pass only the masked text to later stages unless the business workflow has a separately approved reversible-tokenization path.
3. If no intervention occurs, continue with the evaluated content.
4. Record only safe metadata such as request ID, guardrail version, action, policy category, and latency.

`ApplyGuardrail` is independent of model inference. This makes it useful before retrieval and for testing guardrails without paying for an unnecessary model call.

For debugging in a non-production account, `outputScope=FULL` can return detected and non-detected assessments for supported policies. Do not log raw assessment matches: PII trace fields can expose the original value.

## Part C — Apply the guardrail to model output

Invoke the approved model with the same guardrail ID and numeric version. For a Bedrock-native path, use `Converse`/`ConverseStream` or `InvokeModel`/`InvokeModelWithResponseStream` and include the guardrail configuration supported by that API.

Expected flow:

```text
evaluated input + authorized context
  -> model invocation with approved guardrail
  -> output guardrail evaluation
  -> deterministic application validation
  -> response
```

Test `source=OUTPUT` separately with `ApplyGuardrail` against synthetic candidate responses. Input and output require different tests because a safe prompt can still produce unsafe or sensitive output.

Guardrails do not replace:

- JSON/schema validation.
- Citation or factual-grounding checks.
- Authorization on retrieved records or tool actions.
- Business-rule validation.
- Tool-argument validation. PII filtering does not inspect `tool_use` function-call parameters in the same way as normal text output.

## Part D — Validate PII behavior

Use synthetic examples that include enough context for context-dependent PII detection:

```text
Please summarize the customer record for Alex Example.
Contact email: alex.example@example.test
Synthetic case identifier: TEST-9911-2233
```

Verify:

- The configured name/email action is mask, block, or detect as designed.
- Masked values are replaced by PII-type markers rather than copied onward.
- A blocked request never reaches retrieval or inference.
- A normal number or identifier is not incorrectly treated as PII.
- Output PII is also handled.
- Raw PII is absent from application logs, traces, exceptions, and request metadata.

Critical caveat: Guardrails PII masking changes the content sent to or returned from the model, but Bedrock model invocation logging can retain the original request. Guardrail trace matches can also contain the original detected value. Treat both as sensitive and restrict or disable raw-payload capture.

## Part E — Enforce the guardrail with IAM

Give the runtime role only the model, inference profile, and guardrail permissions it needs. Add an explicit deny using the `bedrock:GuardrailIdentifier` condition key for the supported inference APIs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInferenceWithoutApprovedGuardrailVersion",
      "Effect": "Deny",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "bedrock:GuardrailIdentifier": "arn:aws:bedrock:REGION:ACCOUNT_ID:guardrail/GUARDRAIL_ID:VERSION"
        }
      }
    },
    {
      "Sid": "AllowApplyApprovedGuardrail",
      "Effect": "Allow",
      "Action": "bedrock:ApplyGuardrail",
      "Resource": "arn:aws:bedrock:REGION:ACCOUNT_ID:guardrail/GUARDRAIL_ID"
    }
  ]
}
```

This fragment is incomplete by itself: the role also needs narrowly scoped allow statements for the chosen model/inference action. Replace every placeholder, validate the current action/resource behavior, and keep administrator permissions on a separate role.

Run four tests:

| Request | Expected result |
|---|---|
| Approved guardrail and exact numeric version | Allowed |
| No guardrail | Explicit deny |
| Different guardrail | Explicit deny |
| Approved guardrail but wrong version | Explicit deny |

Current service limitations matter:

- Guardrail enforcement with this condition applies to the documented direct inference APIs.
- Do not reuse this role indiscriminately for `RetrieveAndGenerate` or `InvokeAgent`; their service-side model calls can lack the guardrail condition and fail.
- The enforced guardrail must be in the same account as the calling role because Guardrails do not currently provide cross-account resource-based access.
- An endpoint policy is an additional boundary; it does not grant missing IAM permission.

## Part F — Make the path private

Create a `com.amazonaws.REGION.bedrock-runtime` interface endpoint in the private subnets used by the application:

- Enable private DNS.
- Allow inbound TCP 443 on the endpoint security group only from the application security group.
- Attach a restrictive endpoint policy for the runtime role and required Bedrock actions/resources.
- Remove any test NAT/public route dependency only after all required service paths are inventoried.

Add the paths required by the complete workload:

- S3 gateway endpoint for source/query-result buckets.
- Athena and Glue connectivity needed by the query path.
- KMS, STS, and CloudWatch Logs connectivity where the application calls those services.
- Private database/vector endpoints if the lab is extended.

Validate:

1. The standard Regional Bedrock runtime hostname resolves to private endpoint IPs from the workload.
2. A permitted request succeeds with private DNS enabled.
3. VPC Flow Logs show the expected endpoint path.
4. Removing endpoint security-group access causes a network failure.
5. Denying the call in IAM causes `AccessDenied`, proving that network reachability did not bypass authorization.

A private subnet alone does not prove a private service path.

## Part G — Enforce data-level access with Lake Formation

Create a synthetic table with at least:

| tenant_id | public_note | restricted_note |
|---|---|---|
| tenant-a | Allowed A | Restricted A |
| tenant-b | Allowed B | Restricted B |

Configure the runtime/query role to:

- Use only the designated Athena workgroup and query-results location.
- Receive Lake Formation `SELECT` only for the approved columns.
- Receive a Lake Formation data filter that returns only the approved tenant row.
- Access only the required Glue catalog resources, S3 prefixes, and KMS keys.

Test:

| Query | Expected result |
|---|---|
| Approved column for tenant A | Returns tenant A data |
| Restricted column | Denied or unavailable according to the grant |
| Tenant B row | No unauthorized row returned |
| Direct S3 path outside approved prefix | Denied |

Both IAM and Lake Formation checks must pass. A model instruction such as “do not reveal tenant B” is not data authorization, and a caller-supplied metadata filter is not trusted tenant identity.

## Part H — Add sanitized logging

Create a structured application event that contains operational metadata only:

```json
{
  "request_id": "generated-id",
  "principal_hash": "non-reversible-test-hash",
  "guardrail_id": "approved-id",
  "guardrail_version": "numeric-version",
  "input_action": "NONE_OR_INTERVENED",
  "output_action": "NONE_OR_INTERVENED",
  "retrieval_allowed": true,
  "model_id": "approved-model-or-profile",
  "latency_ms": 0,
  "input_tokens": 0,
  "output_tokens": 0,
  "error_class": null
}
```

Do not include prompt text, retrieved chunks, generated text, authorization tokens, PII matches, secrets, or raw tool arguments.

Use:

- CloudTrail for AWS API caller and configuration-change evidence.
- CloudWatch metrics/logs for sanitized operational events and alarms.
- CloudWatch Logs data-protection policies as defense in depth, not as permission to log unnecessary raw data.
- VPC Flow Logs for network-path evidence.
- Athena/query history and application authorization decisions for data-access evidence.

If Bedrock model invocation logging is enabled for the lab, use a separate encrypted destination, least-privilege readers, short retention, and synthetic data. It is disabled by default, can capture full `bedrock-runtime` request/response content, and is not a substitute for sanitized application logging.

## Validation checklist

- [ ] Published guardrail version recorded.
- [ ] Ordinary input allowed.
- [ ] Prohibited input blocked before retrieval.
- [ ] Configured PII masked or blocked.
- [ ] Unsafe/sensitive output intercepted.
- [ ] Direct inference without guardrail denied by IAM.
- [ ] Wrong guardrail/version denied.
- [ ] Bedrock runtime hostname resolves privately.
- [ ] Endpoint policy and IAM both tested.
- [ ] Lake Formation excludes the prohibited row and column.
- [ ] Application logs contain no raw synthetic PII.
- [ ] CloudTrail, metrics, and network evidence captured.
- [ ] False-positive and false-negative observations recorded.

Retain the test case ID, expected/actual action, guardrail version, policy result, endpoint evidence, Lake Formation result, and sanitized request ID.

## Expected failure tests

| Failure injected | Expected evidence | Correct response |
|---|---|---|
| Omit guardrail from inference | IAM `AccessDenied`/CloudTrail | Add approved ID/version; never bypass deny |
| Use `DRAFT` or wrong version | Policy mismatch or inconsistent result | Pin the tested numeric version |
| Blocked prompt still retrieves | Retrieval trace/query history | Move input guard before retrieval |
| Raw email appears in log | Log inspection/data-protection finding | Stop raw logging; sanitize before emission |
| Cross-tenant row returned | Query result and authorization log | Fix Lake Formation filter and trusted tenant mapping |
| Bedrock request times out | DNS/Flow Logs/security groups | Fix endpoint family, private DNS, or TCP 443 path |
| Endpoint reachable but API denied | CloudTrail/IAM evaluation | Fix IAM/resource/endpoint policy; network is not authorization |
| Safe business request blocked | Guardrail assessment and labeled case | Calibrate threshold/policy against the test set |
| PII appears in tool arguments | Agent trace/schema validation | Validate/redact tool parameters separately |
| S3 read fails after IAM allow | KMS/CloudTrail denial | Add the required narrow KMS key authorization |

## Cost and privacy cautions

- Guardrail evaluation is billable; testing input and output separately adds calls.
- Interface endpoints have hourly and data-processing charges. Multiple endpoint families and Availability Zones multiply the fixed portion.
- Athena cost depends on scanned data; use small columnar test data, partitions, and a workgroup scan limit.
- CloudWatch Logs, VPC Flow Logs, S3, and KMS incur storage/request costs. Set deliberate retention and lifecycle policies.
- Do not reduce privacy protection merely to lower cost. Minimize data and calls first.
- Synthetic data is still preferable because raw invocation logs and guardrail traces can contain original input.
- Delete-mode cleanup can conflict with evidence-retention requirements; follow the account’s approved retention policy.

## Cleanup

After preserving only approved test evidence:

1. Remove test IAM role grants and the guardrail-enforcement test policy.
2. Delete or disable temporary guardrails and unpublished drafts that are no longer required.
3. Revoke Lake Formation grants and delete the synthetic table/data.
4. Delete temporary Athena query results according to policy.
5. Delete temporary interface endpoints and security-group rules if they are not shared.
6. Disable temporary model invocation logging and expire its destinations.
7. Delete temporary log groups, Flow Logs destinations, and KMS grants only when retention policy permits.
8. Confirm no synthetic prompt/output artifacts remain in S3 or logs.

Do not delete shared VPC endpoints, keys, buckets, or organization controls.

## Exam lessons

- Guard input before retrieval to prevent unsafe or sensitive text from reaching downstream systems.
- Guard output as well; safe input does not guarantee safe model output.
- `ApplyGuardrail` can evaluate text independently of an FM invocation.
- PII masking does not sanitize original invocation logs or PII trace matches.
- IAM can require a specific guardrail and numeric version for supported direct inference APIs.
- A VPC endpoint supplies a private path; IAM, endpoint policies, KMS, and data permissions still apply.
- Choose the endpoint family that matches the API.
- Private access is incomplete until every dependency has an approved route.
- Lake Formation enforces fine-grained data access; prompts do not.
- Tests must include denies, cross-tenant cases, network failures, false positives, and logging inspection.

## Official sources

- [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
- [Use the ApplyGuardrail API](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-use-independent-api.html)
- [Sensitive information filters and PII limitations](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-sensitive-filters.html)
- [Enforce specific guardrails with IAM](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-permissions-id.html)
- [Amazon Bedrock VPC interface endpoints](https://docs.aws.amazon.com/bedrock/latest/userguide/vpc-interface-endpoints.html)
- [Lake Formation permissions overview](https://docs.aws.amazon.com/lake-formation/latest/dg/lf-permissions-overview.html)
- [Managing Lake Formation permissions](https://docs.aws.amazon.com/lake-formation/latest/dg/managing-permissions.html)
- [Bedrock model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html)
- [CloudWatch Logs data-protection policies](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/cloudwatch-logs-data-protection-policies.html)
- [AIP-C01 Domain 3](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain3.html)
