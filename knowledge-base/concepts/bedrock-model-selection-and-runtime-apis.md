# Bedrock Model Selection and Runtime APIs

Status: Verified  
Official tasks: 1.1.2, 1.2.1–1.2.4, 1.3.3  
Last verified: 2026-07-23

## Why this matters

AIP-C01 expects a production decision, not recognition of the most capable model. The correct model and runtime must satisfy modality, API, Region, lifecycle, quality, latency, throughput, resilience, security, and cost together.

## Core concepts

### Eliminate before benchmarking

Reject a candidate that fails any hard requirement:

1. input/output modality or language;
2. context and output limits;
3. required tool use, structured output, citations, or streaming;
4. compatible Region and inference type;
5. acceptable lifecycle state and migration horizon;
6. data-residency and provider terms.

Benchmark the remaining models on representative prompts and data. Public leaderboards are only a shortlist signal.

### Control plane versus runtime

| Need | API/service |
|---|---|
| Discover models and basic capabilities | `ListFoundationModels` |
| Inspect one model’s modalities, streaming support, and lifecycle | `GetFoundationModel` |
| Common message interface across supported models | `Converse` / `ConverseStream` |
| Model-native or specialized request body | `InvokeModel` / `InvokeModelWithResponseStream` |
| Estimate request tokens before inference | `CountTokens` |
| Reusable configuration and model/provider routing | AWS AppConfig plus an application adapter |

Bedrock now also offers OpenAI- and Anthropic-compatible APIs through Bedrock Mantle. Use them when client compatibility is an explicit requirement. They do not make `bedrock-runtime` concepts irrelevant to AIP-C01.

### Model access

Current Bedrock documentation states that foundation models are enabled by default. AWS Marketplace models can be enabled on first invocation; the first-use principal may need Marketplace subscription permissions. Anthropic models can require use-case details. Do not memorize the older “manually enable every model” workflow as universal.

## How it works

### Selection workflow

```text
requirements
  → capability and Region filter
  → lifecycle and governance filter
  → representative evaluation
  → load/cost test
  → approved model configuration
  → canary deployment
  → monitor and periodically reevaluate
```

Evaluate task correctness, groundedness, safety, format compliance, latency percentiles, throughput, throttles, and cost per successful task. A cheaper request is not cheaper if failures require retries or human rework.

### Runtime contracts

`Converse` uses a common `messages` structure with roles and typed content blocks. It is the default for portable conversational applications when the selected model supports it.

`InvokeModel` uses the selected provider/model’s native JSON schema. Prefer it for specialized features or modalities unavailable through the common interface.

When a Prompt Management prompt ARN is passed to `Converse`, current API restrictions prohibit supplying `system`, `inferenceConfig`, `toolConfig`, or `additionalModelRequestFields` in that request. Supply declared `promptVariables`; additional messages are appended to the stored prompt.

`ConverseStream` and `InvokeModelWithResponseStream` improve time to first token. Streaming does not guarantee lower full-response latency or lower cost.

`CountTokens` is a preflight estimate for supported requests. It helps prevent context-limit failures and budget requests, but it adds another API call; do not invoke it blindly on every low-risk request.

### Inference options

| Pattern | Best fit | Key trade-off |
|---|---|---|
| On-demand | Bursty or uncertain interactive traffic | Simple, but quotas and throttling still apply |
| Provisioned Throughput | Predictable sustained synchronous demand | Fixed capacity commitment; one option for supported customized models |
| On-demand custom-model deployment | Eligible customized models and supported Regions | Pay per use; eligibility, model, creation date, and Region restrictions apply |
| Batch inference | Large asynchronous offline workloads, including listed custom-model support | S3 JSONL and job latency, but avoids synchronous orchestration |
| Cross-Region inference profile | Capacity resilience or broader throughput | Routing destinations, IAM, SCP, and residency must all be acceptable |
| Intelligent Prompt Routing | Route between supported candidates in one model family | Not arbitrary cross-provider failover |
| SageMaker endpoint | Custom container, arbitrary model, or serving-stack control | Highest operational ownership |

Bedrock batch inference supports model inputs in documented `Converse` or `InvokeModel` formats. Do not build one synchronous Lambda invocation per record when a managed batch job meets the SLA.

### Cross-Region behavior

A geographic inference profile routes within its documented geography, not necessarily within the source Region. A global profile can route worldwide. Every destination must be allowed by IAM and service control policies; one denied destination can make the request fail. Global destinations can change, so reevaluate them.

Cross-Region inference addresses capacity routing. It is not a complete disaster-recovery design: test application dependencies, vector stores, secrets, data residency, and degraded responses too.

### Dynamic switching and graceful degradation

Expose a stable application request/response contract. Put provider-specific serialization, authentication, parameters, error mapping, and response normalization behind adapters. Store approved model IDs, inference settings, routing rules, and kill switches in AppConfig.

Use AppConfig validators, gradual deployment, CloudWatch alarms, and automatic rollback. A configuration flag alone is unsafe if the replacement has a different context limit, schema, or tool behavior.

A practical fallback order is:

```text
primary model
  → same capability through an allowed inference profile
  → evaluated compatible model
  → narrower cached/retrieval-only response
  → queued processing or explicit unavailable response
```

Never silently return a low-quality model answer as if it had the original capability.

### Customized-model lifecycle

For SageMaker-hosted models, version artifacts in SageMaker Model Registry, require approval status, deploy with canary or linear traffic shifting, monitor alarms, and automatically roll back. Large Model Inference containers help serve large models; configure artifact download, container startup, storage, and health-check timeouts for model size.

Use LoRA or other parameter-efficient adapters when they meet quality goals with less training/storage than full fine-tuning. Version the base model, adapter, tokenizer, inference image, and evaluation dataset together. Define retirement and replacement criteria before release.

## AWS services and APIs

- Amazon Bedrock control plane: model discovery and lifecycle metadata.
- Amazon Bedrock Runtime: `Converse`, `ConverseStream`, `InvokeModel`, streaming, and `CountTokens`.
- Inference profiles: geographic/global routing and application profiles for usage attribution.
- Provisioned Throughput and batch inference: capacity patterns.
- AWS AppConfig: runtime configuration, validators, deployment strategies, alarm rollback.
- Amazon SageMaker AI: custom serving, Large Model Inference, Model Registry, deployment guardrails.
- Step Functions: explicit retry, circuit-breaker, fallback, and asynchronous orchestration.
- CloudWatch and CloudTrail: operational signals and control-plane audit.

## Architecture patterns

### Provider-aware adapter

```text
client → stable API → policy/router → provider adapter → Bedrock or SageMaker
                         ↑
                  AppConfig version
```

The adapter owns schema translation and error normalization. The router owns only an approved decision policy.

### Safe release

```text
candidate model/config
  → offline evaluation gate
  → load and cost test
  → canary traffic
  → CloudWatch alarms
  → promote or automatic rollback
```

## Decision table

| Requirement | Prefer | Avoid |
|---|---|---|
| Portable chat across supported Bedrock FMs | `Converse` | Duplicating native payloads without need |
| Specialized provider feature | `InvokeModel` | Assuming `Converse` exposes every native field |
| Chat tokens arrive immediately | `ConverseStream` | Claiming streaming lowers total token cost |
| Offline millions-of-records job | Batch inference | Synchronous fanout from Lambda |
| Steady committed traffic | Provisioned Throughput | Scaling Lambda to solve model-capacity limits |
| Capacity routing within an approved geography | Geographic profile | Treating it as same-Region execution |
| Arbitrary model/provider failover | Adapter + AppConfig | Intelligent Prompt Routing as a universal router |
| Custom serving stack | SageMaker AI | Bedrock when required model/runtime is unsupported |
| Production custom-model release | Registry + approval + guarded rollout | Updating an endpoint in place without rollback |

## Security and governance

- Grant only required model IDs, inference profiles, prompt ARNs, and actions.
- When invoking an inference profile, permit all documented destination Regions and constrain them intentionally.
- Use VPC endpoints/private connectivity where required; remember that networking does not replace IAM.
- Encrypt model artifacts, configuration, and logs; restrict KMS key use.
- Never store secrets in prompts or AppConfig plaintext.
- Log model ID/profile, configuration version, prompt version, token use, latency, and outcome without copying sensitive raw content.
- Review provider use terms and data residency before a model or destination is approved.
- Separate evaluation approval from deployment permission.

## Cost, latency, and reliability

| Driver | Response |
|---|---|
| Large input/context | Retrieve less, prune history, cache stable prefixes, and use `CountTokens` selectively |
| Variable output | Task-sized output limit and explicit stop/schema |
| Bursty demand | On-demand plus quotas, backoff, and concurrency control |
| Predictable saturation | Provisioned Throughput after utilization economics |
| Long offline workload | Batch inference |
| First-token latency | Streaming and an appropriate latency tier |
| Regional capacity | Approved inference profile plus tested application fallback |
| Large SageMaker model startup | LMI, appropriate storage, and increased download/startup timeouts |

Use exponential backoff with jitter only for retryable throttles/transient errors. Cap attempts and make orchestrated steps idempotent. Retrying invalid input wastes cost and increases latency.

## Failure modes and troubleshooting

| Symptom | Likely cause | Evidence | Fix |
|---|---|---|---|
| Validation exception after model switch | Wrong endpoint or native schema | Model/API and request body | Use capability matrix and provider adapter |
| Unsupported parameter ignored/rejected | Feature differs by model | `GetFoundationModel` and model docs | Map only supported parameters |
| Cross-Region request denied | Destination blocked by IAM/SCP | Profile destinations and policy evaluation | Permit all approved destinations or choose another profile |
| Throttles despite Lambda scaling | Bedrock model capacity/quota | Bedrock throttles, token rate, concurrency | Backoff, smooth demand, profile or provisioned capacity |
| High bill with low task success | Long context/output or weak model | Tokens plus evaluation outcomes | Optimize cost per successful task |
| Prompt ARN invocation fails | Disallowed request fields with stored prompt | Converse request | Move configuration into stored prompt/version |
| SageMaker endpoint fails to start | Artifact/storage/startup timeout | endpoint events and container logs | LMI/appropriate instance/storage/timeouts |
| Fallback returns malformed output | Candidate lacks schema/tool parity | contract regression tests | Block promotion or normalize/validate |

## Common exam traps

1. `Converse` is portable only across models that support the required feature.
2. `InvokeModel` is not “better”; it is model-native.
3. Streaming reduces perceived latency, not necessarily full generation time.
4. More Lambda concurrency cannot create Bedrock model capacity.
5. Geographic inference is not same-Region processing.
6. Global inference improves routing scope but can conflict with residency.
7. Intelligent Prompt Routing is not arbitrary provider switching.
8. Public benchmarks cannot replace a representative evaluation.
9. Provisioned Throughput is not the default for bursty demand.
10. A model ID in AppConfig is not sufficient without schema/capability adaptation.
11. Model versioning without evaluation, approval, and rollback is not lifecycle management.
12. Older questions that say all model access requires manual enablement are stale.

## Local mock references

Mocks are practice evidence, not current product authority.

| Focus | Questions |
|---|---|
| Batch and capacity | PE1-Q18, Q60 |
| AppConfig routing | PE1-Q21; PE2-Q38 |
| Cross-Region/residency | PE1-Q45; PE2-Q9, Q35 |
| Model evaluation/release | PE1-Q59; PE2-Q19, Q55 |
| Intelligent routing | PE1-Q71 |
| Provider adapters | PE2-Q17, Q38 |
| API/capability choice | PE2-Q31, Q56, Q61 |
| LoRA/Registry/guardrails | PE1-Q62 |
| Large SageMaker models | PE1-Q54; PE2-Q25 |

## Hands-on validation

1. Build a capability matrix from `ListFoundationModels` and `GetFoundationModel`.
2. Send equivalent chat requests with `Converse` and a model-native request with `InvokeModel`.
3. Stream a response and measure both first-token and full-response latency.
4. Run `CountTokens`, then prove the same request stays within the model limit.
5. Put two compatible model configurations in AppConfig; validate, canary, alarm, and roll back one.
6. Inspect every destination in a geographic and global inference profile and test a deliberately restrictive policy.
7. Submit a small S3 JSONL batch inference job and verify terminal status/output.
8. Register a model version and simulate a canary rollback from an alarm.

## Recall questions

1. Which hard constraints should eliminate a model before benchmarking?
2. When does `Converse` fail to provide the portability a scenario needs?
3. When is `InvokeModel` the correct choice?
4. What does `CountTokens` prove, and what does it not prove?
5. Why can an inference-profile call fail when the source Region is allowed?
6. What residency guarantee does a geographic profile provide?
7. Why is Intelligent Prompt Routing not arbitrary provider failover?
8. Which extra layers turn AppConfig into a safe switching design?
9. When does Provisioned Throughput beat on-demand?
10. Why is batch inference better than synchronous Lambda fanout for an offline job?
11. Which artifacts must be versioned with a LoRA adapter?
12. What signals should automatically roll back a custom-model canary?

## Official sources

- [Bedrock models and feature compatibility](https://docs.aws.amazon.com/bedrock/latest/userguide/models.html)
- [Get model information](https://docs.aws.amazon.com/bedrock/latest/userguide/models-get-info.html)
- [Model lifecycle](https://docs.aws.amazon.com/bedrock/latest/userguide/model-lifecycle.html)
- [Model access](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html)
- [Converse APIs](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference.html)
- [Inference APIs](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-api.html)
- [Bedrock Runtime and Mantle endpoints](https://docs.aws.amazon.com/bedrock/latest/userguide/endpoints.html)
- [CountTokens](https://docs.aws.amazon.com/bedrock/latest/userguide/count-tokens.html)
- [Geographic cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/geographic-cross-region-inference.html)
- [Global cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/global-cross-region-inference.html)
- [Inference-profile support](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-support.html)
- [Provisioned Throughput](https://docs.aws.amazon.com/bedrock/latest/userguide/prov-throughput.html)
- [Create a batch inference job](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference-create.html)
- [Batch inference model and custom-model support](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference-supported.html)
- [On-demand custom-model deployment](https://docs.aws.amazon.com/bedrock/latest/userguide/deploy-custom-model-on-demand.html)
- [Intelligent Prompt Routing](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-routing.html)
- [AppConfig deployment strategies](https://docs.aws.amazon.com/appconfig/latest/userguide/appconfig-creating-deployment-strategy.html)
- [AppConfig deployment monitoring and rollback](https://docs.aws.amazon.com/appconfig/latest/userguide/monitoring-deployments.html)
- [SageMaker Model Registry](https://docs.aws.amazon.com/sagemaker/latest/dg/model-registry.html)
- [SageMaker deployment guardrails](https://docs.aws.amazon.com/sagemaker/latest/dg/deployment-guardrails.html)
- [SageMaker Large Model Inference](https://docs.aws.amazon.com/sagemaker/latest/dg/large-model-inference.html)
