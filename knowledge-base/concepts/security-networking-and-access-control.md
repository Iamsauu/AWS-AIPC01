# Security, Networking, and Access Control

Status: Verified  
Official tasks: 3.2, 3.3; supports 2.1, 2.2, 2.3, 4.3, and 5.2  
Last verified: 2026-07-23

## Why this matters

GenAI security is conventional cloud security plus new trust boundaries:

- Prompts and retrieved text are untrusted input.
- Models are probabilistic components, not authorization engines.
- Agents can turn text into side effects.
- Context, traces, and invocation logs can contain more sensitive data than the final response.
- Model, prompt, guardrail, vector store, and data-source permissions are separate.

The exam usually asks for the smallest set of controls that satisfies identity, least privilege, private connectivity, data-level authorization, encryption, residency, logging, and operational overhead.

## Core concepts

For every scenario, answer these questions in order:

1. **Who is the caller?** Workforce user, customer, AWS workload, agent, or service.
2. **How is the caller authenticated?** Federation, IAM role, SigV4, JWT/OIDC, or OAuth.
3. **What is the caller allowed to do?** AWS API action, model/resource, data row/column, or external tool scope.
4. **Which network path is allowed?** Public endpoint, interface endpoint, hybrid private path, or specific geography.
5. **What data may cross each boundary?** Raw, masked, tokenized, or aggregated.
6. **How is data protected at rest and in transit?**
7. **What evidence proves the control worked?**

Do not start with a VPC diagram and assume it solves authorization.

## How it works

### Identity types

| Caller | Preferred identity | Avoid |
|---|---|---|
| Employee using console/CLI/SDK | Federation through IAM Identity Center and permission sets | Long-lived IAM access keys |
| Lambda, ECS task, EC2, or EKS workload | Workload IAM role with temporary credentials | Embedded access key |
| AWS service acting for Bedrock feature | Dedicated service role with narrow trust and resource access | Reusing an administrator role |
| Customer-facing application user | Application identity/JWT, mapped to server-side authorization | Passing an unverified user ID in prompt text |
| Agent calling AWS tools | Agent/runtime identity plus least-privilege tool roles | One broad role for every tool |
| Agent calling third-party API | OAuth user-delegated or machine identity; managed secret/token vault | Putting OAuth/API token in context |

IAM Identity Center centralizes workforce users and groups, account assignments, permission sets, and temporary CLI/SDK credentials. Use an organization instance for production multi-account access unless the scenario explicitly requires an isolated supported application.

AgentCore Identity provides workload identities and credential handling for agents, including OAuth patterns. It does not replace resource-side authorization: the downstream API must still enforce scopes and policy.

## AWS services and APIs

### Authorization layers

Authorization is the intersection of all applicable policy layers:

- AWS Organizations service control policy (SCP): maximum available permissions in member accounts.
- Resource control or organization policy where supported.
- IAM identity policy.
- Permissions boundary.
- Session policy.
- Resource policy.
- VPC endpoint policy.
- KMS key policy/grant.
- Service-specific data permissions, such as Lake Formation.

An `Allow` in an endpoint policy cannot override an explicit deny or create permission that the principal lacks.

### IAM least privilege for Bedrock

Scope:

- Actions: inference, token counting, retrieval, agent invocation, or resource management as required.
- Resources: approved foundation models, inference profiles, provisioned models, prompts, guardrails, agents, knowledge bases, or evaluation resources.
- Conditions: account, Region, tags, VPC endpoint, TLS, or inference profile where supported.

High-value runtime details:

- Bedrock-native `Converse` authorization is based on the underlying model-invocation permissions. Denying `bedrock:InvokeModel` for a model also blocks Converse access to it.
- Streaming needs the corresponding streaming permission path.
- Control-plane builders should not automatically receive runtime production invocation access.
- Runtime callers should not automatically be able to create/delete prompts, guardrails, agents, knowledge bases, or model capacity.
- A service role assumed by Bedrock needs a trust policy for the service and only the source/output/model/data permissions required by that feature.

Use IAM Access Analyzer to validate and refine policies. Use permissions boundaries when delegating role creation. Use SCPs for organization-wide maximum permissions, not as the application’s only authorization logic.

### Tool authorization

For every agent tool:

1. Bind the tool to a narrow role or workload identity.
2. Allow only required APIs and specific resources.
3. Validate tool name and input schema.
4. Derive tenant/user scope from authenticated claims, not model output.
5. Recheck authorization in the tool.
6. Use idempotency for side effects.
7. Require human approval for high-impact operations.
8. Log the policy decision without logging secrets.

Prompt instructions such as “only read records for this user” are not enforcement.

### Data-level access control

IAM controls access to AWS resources; a data service can enforce finer granularity.

AWS Lake Formation provides centralized permissions over Data Catalog resources and can enforce table-, column-, row-, and cell-level access for integrated analytics services such as Athena. A common protected RAG/query path is:

```text
Authenticated application role
  -> Athena workgroup
  -> Glue Data Catalog table
  -> Lake Formation column/row permissions
  -> approved S3 data
```

Important:

- Lake Formation permissions and the required IAM permissions must both be correct.
- Restrict the Bedrock/tool role to the Athena workgroup, database/table, S3 prefixes, query-result bucket, and KMS keys it needs.
- Never rely on asking the FM not to select a PHI column.
- For structured-data questions, use governed queries and deterministic controls.

### Tenant and document authorization

For multi-tenant RAG:

- Derive tenant and user identity from a verified token.
- Apply server-side metadata filters or document ACL filters.
- Prevent clients from choosing arbitrary knowledge base IDs or filters.
- Consider separate indexes or stores when isolation requirements are strong.
- Test negative cases: one tenant must not retrieve another tenant’s chunk.
- Propagate user identity only through supported, authenticated paths.

Retrieval relevance filters are not authorization unless the filter value is trusted and enforced.

### Private connectivity to Amazon Bedrock

Amazon Bedrock supports interface VPC endpoints powered by AWS PrivateLink. Important endpoint families include:

| Capability | Endpoint suffix |
|---|---|
| Bedrock control plane | `bedrock` |
| Bedrock-native runtime (`Converse`, `InvokeModel`) | `bedrock-runtime` |
| Responses/compatibility and newer inference APIs | `bedrock-mantle` |
| Agent build-time APIs | `bedrock-agent` |
| Agent/Knowledge Base runtime APIs | `bedrock-agent-runtime` |

An interface endpoint creates private IP addresses in selected subnets. With private DNS enabled, SDK calls to the normal Regional service name resolve to the endpoint. Without private DNS, configure the endpoint URL explicitly.

Configure:

- Endpoint subnets in required Availability Zones.
- Endpoint security group allowing HTTPS from workload security groups.
- Private DNS.
- A restrictive endpoint policy.
- Workload DNS and routing.
- IAM permissions independently of the endpoint policy.

### “No public internet” means the whole path

A private Bedrock endpoint is incomplete if the workload still uses a NAT gateway for another required dependency. Inventory all calls:

- Bedrock runtime/control/agent endpoints.
- S3 gateway endpoint.
- DynamoDB gateway endpoint.
- Interface endpoints for services such as Secrets Manager, KMS, STS, CloudWatch Logs, Athena, Glue, SQS, or API Gateway private APIs as required.
- Private database/vector-store endpoints and security groups.
- Container registry endpoints when private workloads pull images.

Verify DNS resolution and VPC flow logs; do not infer private routing from “Lambda is in a private subnet.”

### Hybrid and on-premises paths

For data that must be processed on premises:

1. Keep the raw source in the required location.
2. De-identify or transform it before the cloud boundary.
3. Connect the on-premises network to the VPC through a supported private hybrid path such as Direct Connect, Site-to-Site VPN, or Outposts connectivity according to the architecture.
4. Reach Bedrock through the appropriate interface endpoint.
5. Send only the approved transformed payload.

PrivateLink avoids the public internet between the VPC and Bedrock; it does not itself connect an on-premises data center to the VPC.

### Region and data-residency controls

Distinguish:

- **Single-Region inference:** processing in the chosen Region, subject to model availability/capacity.
- **Geographic cross-Region inference:** may process in Regions within the selected geography.
- **Global cross-Region inference:** may process in supported commercial Regions worldwide.

If residency requires “EU only,” a geographic EU inference profile can be appropriate if the documented destination Regions and organizational policy are acceptable. Global inference is not.

Cross-Region inference requires IAM/SCP permission for every possible destination Region in the profile. Blocking one destination with an SCP can make the profile fail. CloudTrail in the source Region records the processing Region in documented event metadata.

Do not equate “same geography” with “same country,” and do not equate “encrypted” with “residency compliant.”

### Encryption

#### In transit

Bedrock API requests use TLS. PrivateLink changes the network path, not the need for TLS. Use TLS for service-to-service and external tool calls.

#### At rest

Use service-default encryption when it meets the requirement. Use a customer-managed KMS key when the organization needs control over:

- Key policy and separation of duties.
- Cross-account use.
- Disablement or rotation policy.
- Auditability.
- Specific regulatory requirements.

Common encrypted resources include S3 source/output buckets, CloudWatch log groups, Secrets Manager secrets, vector databases, evaluation results, custom model artifacts, prompts/guardrails where supported, and state stores.

KMS access has two sides:

- The caller/service must have IAM permission.
- The key policy or grant must authorize use.

A workload can have `s3:GetObject` yet fail because it lacks `kms:Decrypt`.

#### S3 controls

- Keep Block Public Access enabled.
- Prefer bucket policies that restrict principals, require TLS, and limit prefixes.
- Use SSE-S3 or SSE-KMS according to control needs.
- Use separate input, output, quarantine, and audit prefixes/buckets where access differs.
- Apply Lifecycle retention and deletion rules.
- Use versioning and Object Lock only when the recovery/WORM requirement calls for them.
- Use access logs, CloudTrail data events, Macie, and AWS Config when required.

### Secrets and credentials

AWS Secrets Manager stores, retrieves, and can rotate database credentials, OAuth tokens, API keys, and other secrets.

Use it when:

- A third-party tool requires a secret.
- Credentials must rotate without redeploying application code.
- Access needs a dedicated IAM policy and audit trail.

Do not:

- Put secrets in prompts, tool descriptions, environment files, source code, logs, traces, resource tags, or exception messages.
- Give the model permission to retrieve arbitrary secrets.
- Reuse one broad secret for unrelated tools or tenants.

For AWS services, prefer IAM roles and temporary credentials over a stored access key.

### Logging, monitoring, and evidence

| Question | Best evidence |
|---|---|
| Who called or changed an AWS API? | CloudTrail |
| Was an invocation slow, throttled, or failing? | CloudWatch metrics and application logs |
| What path did the request take? | X-Ray/OTel or AgentCore/agent trace |
| What input/output did supported Bedrock Runtime calls process? | Bedrock model invocation logging, if deliberately enabled |
| Which data did the query retrieve? | Application decision log, Athena/query history, source identifiers |
| Was S3 sensitive data found? | Macie finding |
| Did traffic use the expected endpoint? | DNS checks, VPC Flow Logs, CloudTrail source context where available |

Bedrock model invocation logging is disabled by default and can publish request/response data and metadata to CloudWatch Logs or S3 for supported `bedrock-runtime` operations. Treat it as a sensitive data store:

- Enable only required modalities and environments.
- Restrict read access.
- Apply encryption and retention.
- Protect raw text with redaction/data-protection policies.
- Separate operational metadata from content.

CloudTrail and model invocation logging are complementary. CloudTrail is primarily API activity/audit evidence; invocation logging can contain payload-level evidence.

## Architecture patterns

### Protected RAG application

```text
Enterprise IdP
  -> IAM Identity Center / application JWT
  -> private API or authenticated API Gateway
  -> Lambda/ECS role in private subnets
       -> Comprehend/Guardrails pre-check
       -> Bedrock Agent Runtime or Knowledge Base via interface endpoint
       -> vector store / Athena with server-side ACL or Lake Formation
       -> Bedrock Runtime via interface endpoint
       -> Guardrails and deterministic output validation
  -> redacted response

Evidence:
  CloudTrail + CloudWatch metrics/logs + X-Ray/agent traces
Data controls:
  KMS + S3 policies/lifecycle + Macie + Secrets Manager
```

## Decision table

| Requirement | Correct direction | Main distractor |
|---|---|---|
| Employees need direct CLI inference with SSO | IAM Identity Center permission set with runtime-only actions | IAM users with long-term access keys |
| Lambda must invoke Bedrock without public internet | `bedrock-runtime` interface endpoint with private DNS | NAT gateway |
| Knowledge Base runtime calls must be private | `bedrock-agent-runtime` endpoint | Only a `bedrock` control-plane endpoint |
| Bedrock may read only selected data columns | Lake Formation plus IAM/S3/KMS controls | Prompt instruction |
| Tool can read one table and one secret | Tool-specific role scoped to both resources | Agent administrator role |
| Cross-Region inference must stay in EU | Geographic EU inference profile and allow its destination Regions | Global profile |
| Stored S3 data needs sensitive-data discovery | Macie | Guardrails |
| Runtime text needs PII redaction | Guardrails/Comprehend before invocation | Macie after storage |
| Third-party API token must rotate | Secrets Manager or AgentCore credential provider | Hard-coded environment value |
| All account callers must use an approved guardrail | Bedrock guardrail enforcement policy | Rely on each prompt to mention the policy |
| Auditors need API caller identity | CloudTrail | X-Ray only |
| Security needs payload-level forensic records | Deliberately protected invocation/application logs | CloudTrail assumed to hold full prompts |

## Security and governance

- Enforce identity, IAM, endpoint, KMS, and data-layer policies independently; one control does not replace another.
- Treat prompts, retrieval context, traces, and invocation logs as sensitive data with explicit access, encryption, and retention.
- Derive user and tenant scope from authenticated claims and enforce it again in tools and data stores.
- Use CloudTrail for API audit evidence and deliberately protected application or invocation logs for payload evidence.
- Test both allowed and denied paths, including cross-tenant access, blocked tools, missing KMS grants, and disallowed Regions.

## Cost, latency, and reliability

- Interface endpoints can remove NAT and internet-egress dependencies, but each endpoint has hourly and data-processing charges; create only the endpoint families and Availability Zone coverage the workload needs.
- Private DNS, endpoint health, security groups, and downstream endpoints become availability dependencies. Monitor and test the complete path.
- Excessive payload logging increases storage cost and privacy exposure. Prefer metrics and redacted metadata; retain raw content only when justified.
- Fine-grained authorization and short-lived credentials add small processing overhead but reduce blast radius. Cache only safe policy/configuration data, never authorization decisions beyond their valid lifetime.
- Single-Region processing gives tighter residency control; an approved cross-Region profile can improve capacity resilience but changes policy, audit, and residency requirements.

## Failure modes and troubleshooting

| Symptom | Probable cause | Evidence | Remediation |
|---|---|---|---|
| `AccessDeniedException` invoking model | Missing action, wrong resource ARN, explicit deny, model access, or endpoint policy | CloudTrail and IAM policy simulator | Check every policy layer and exact model/profile ARN |
| Converse fails despite `bedrock:Converse` guess | Underlying InvokeModel permission absent | IAM docs/CloudTrail | Grant scoped runtime invocation permission |
| Cross-Region profile fails in some requests | SCP blocks a destination Region | CloudTrail and profile destination list | Allow all profile destinations or use single Region |
| Private Lambda cannot call Bedrock | Wrong endpoint family, DNS, security group, or endpoint policy | DNS lookup, flow logs, timeout | Correct endpoint, enable private DNS, allow 443 |
| Bedrock is private but S3/Logs time out | Missing endpoints/NAT for dependencies | X-Ray and VPC Flow Logs | Complete the dependency endpoint inventory |
| Athena query sees prohibited column | Lake Formation permission too broad or not enforced | Lake Formation grants and query history | Use governed table and column/row grants |
| S3 access denied after IAM allow | KMS key policy/grant missing | CloudTrail KMS error | Add least-privilege KMS authorization |
| Agent crosses tenants | Tenant ID came from prompt/tool input | Trace and authorization logs | Derive scope from verified identity; enforce in tool/data layer |
| Secret appears in trace | Secret passed as prompt/tool parameter | Trace/log sample | Retrieve credential inside authorized connector; redact |
| Invocation logs expose raw PII | Logging captured original request | Invocation log configuration | Pre-redact, log metadata, data-protection policy |
| Endpoint policy seems to grant access but call is denied | Endpoint policy is not an identity permission grant | IAM evaluation | Add correct identity/resource authorization |

## Common exam traps

- A private subnet is not proof that traffic avoids the public internet.
- A VPC endpoint is not an IAM permission.
- An IAM `Allow` is not enough when a KMS key policy, SCP, boundary, or endpoint policy denies access.
- Use the endpoint family that matches the API: runtime, agent runtime, control plane, or mantle.
- IAM protects AWS resource access; Lake Formation provides finer data permissions.
- Prompt text and metadata filters are not tenant isolation unless the server sets and enforces them.
- Secrets Manager is for secrets; KMS is for encryption keys.
- Macie scans S3; it does not redact a synchronous request.
- Cross-Region inference can move processing outside the source Region.
- “EU geography” is broader than a single EU Region or country.
- CloudTrail tells who called an API; X-Ray tells where request latency occurred.
- Invocation logging can create a new sensitive-data repository.
- Least privilege applies to every action group/tool separately, not just to the host agent.

## Local mock references

- Workforce and runtime least privilege: PE1-Q39; PE2-Q05.
- Private networking and governed data: PE1-Q25, PE1-Q75; PE2-Q35, PE2-Q64.
- Tool boundaries, secrets, and loop control: PE1-Q50; PE2-Q60.
- Central gateway/governance: PE1-Q29, PE1-Q36; PE2-Q32, PE2-Q67, PE2-Q74.
- Privacy, S3 discovery, and retention: PE1-Q05, PE1-Q69; PE2-Q21, PE2-Q29.
- Cross-Region/residency: PE1-Q45, PE1-Q75; PE2-Q09, PE2-Q35.
- Audit and observability: PE1-Q12, PE1-Q22; PE2-Q39, PE2-Q46, PE2-Q69.

## Hands-on validation

Use [Lab 3: Guardrails and private access](../labs/03-guardrails-and-private-access.md) to verify:

1. A workload role can invoke one approved model and cannot manage Bedrock resources.
2. `bedrock-runtime` resolves to private endpoint addresses.
3. Removing the endpoint security-group rule creates a diagnosable timeout.
4. An endpoint policy cannot bypass an IAM explicit deny.
5. A Lake Formation-restricted role cannot query a prohibited column.
6. An SSE-KMS object fails without the correct key authorization.
7. Logs and traces contain correlation metadata but no raw PII or secret.
8. A cross-tenant retrieval test returns no unauthorized chunks.

## Recall questions

1. Why does a VPC endpoint not grant permission to invoke a model?
2. Which endpoint is required for `Converse`, and which for Knowledge Base runtime APIs?
3. What additional network dependencies can break a “private” Lambda architecture?
4. Why are IAM and Lake Formation both required in a governed Athena path?
5. How should a tenant ID reach a retrieval filter?
6. What is the difference between an IAM role, an IAM Identity Center permission set, and an AgentCore workload identity?
7. Why can `s3:GetObject` succeed in policy analysis but fail at runtime for an SSE-KMS object?
8. When is geographic cross-Region inference wrong even though it stays in a broad geography?
9. What should be stored in Secrets Manager rather than KMS?
10. Which evidence would prove who invoked Bedrock, which would show the prompt payload, and which would locate latency?
11. How do you prevent an agent tool from reading an unrelated DynamoDB table?
12. Why is putting an API key in a system prompt unsafe even if the model is instructed not to reveal it?

## Official sources

- [AIP-C01 Domain 3](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain3.html)
- [Security best practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [Bedrock IAM policy examples](https://docs.aws.amazon.com/bedrock/latest/userguide/security_iam_id-based-policy-examples.html)
- [IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html)
- [IAM Identity Center temporary CLI/SDK credentials](https://docs.aws.amazon.com/singlesignon/latest/userguide/howtogetcredentials.html)
- [Bedrock interface VPC endpoints](https://docs.aws.amazon.com/bedrock/latest/userguide/vpc-interface-endpoints.html)
- [Bedrock endpoints](https://docs.aws.amazon.com/bedrock/latest/userguide/endpoints.html)
- [Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html)
- [Lake Formation permissions](https://docs.aws.amazon.com/lake-formation/latest/dg/managing-permissions.html)
- [Bedrock data encryption](https://docs.aws.amazon.com/bedrock/latest/userguide/data-encryption.html)
- [KMS IAM policy best practices](https://docs.aws.amazon.com/kms/latest/developerguide/iam-policies-best-practices.html)
- [S3 security best practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
- [AgentCore Identity](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity.html)
- [Bedrock CloudTrail logging](https://docs.aws.amazon.com/bedrock/latest/userguide/logging-using-cloudtrail.html)
- [Bedrock model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html)
- [What is AWS X-Ray?](https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html)
