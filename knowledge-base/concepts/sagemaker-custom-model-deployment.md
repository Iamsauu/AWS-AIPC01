# SageMaker Custom and Open-Weight Model Deployment

Status: Verified  
Official tasks: 2.2; supporting 2.4, 3.2–3.3, 4.1–4.3, and 5.2  
Last verified: 2026-07-23

## Why this matters

Amazon Bedrock is the default managed-FM answer when its model catalog and APIs meet the requirement. Amazon SageMaker AI becomes the stronger answer when the application must host a custom, open-weight, imported, fine-tuned, or specially containerized model and control the serving stack.

The exam does not require advanced model-training theory. It does require you to choose an inference mode, deploy a large model successfully, operate versions safely, tune offline inference, and diagnose endpoint failures.

## Core concepts

### Bedrock versus SageMaker AI

| Requirement | Prefer Amazon Bedrock | Prefer SageMaker AI |
|---|---|---|
| Use a supported provider FM through managed API | Yes | Usually unnecessary |
| Lowest infrastructure management | Yes | No |
| Bedrock Guardrails, Knowledge Bases, Agents, Prompt Management | Native integration | Requires application integration |
| Arbitrary open-weight/custom model or custom container | Not the default | Yes |
| Full control of serving image, framework, GPU, and model server | Limited | Yes |
| Arbitrary custom-container real-time endpoint | No | Yes |
| Offline inference for a Bedrock-supported FM | Bedrock batch inference | Not required |
| Offline inference for a supported Bedrock customized model | Bedrock batch inference where the current model/Region table lists custom-model support | Not required |
| Offline inference for an arbitrary custom model/container | No | SageMaker Batch Transform |
| Governance of custom model versions and approval | Possible surrounding process | SageMaker Model Registry is a direct fit |

Do not select SageMaker merely because the scenario says “machine learning.” Choose it because the model or serving requirements need that control.

### Inference mode decision

| Mode | Use when | Cost/latency character | Main distractor |
|---|---|---|---|
| Real-time endpoint | Interactive, persistent low-latency custom model | Pay while instances run; autoscale within limits | Batch Transform for user-facing chat |
| Asynchronous inference | Requests can queue, payload/processing is longer, result can be retrieved later | Managed queue around an endpoint | Holding a synchronous API request |
| Batch Transform | Offline S3 dataset, no persistent endpoint | Temporary compute for job | Real-time endpoint left idle |
| Serverless inference | Supported model with intermittent traffic and compatible resource needs | No instance management; cold-start/resource constraints | Very large GPU LLM |
| Multi-model endpoint | Many compatible models share serving fleet and tolerate load/unload behavior | Better utilization; possible cold load | One huge hot model needing predictable latency |

Feature support and limits change. Verify current model size, hardware, Region, concurrency, and endpoint compatibility before selecting an inference mode.

## How it works

### Real-time endpoint resource chain

```text
ECR image + S3 model artifacts
        ↓
CreateModel
  - image / model data source
  - execution role
  - environment
  - optional VPC/network isolation
        ↓
CreateEndpointConfig
  - production variant
  - instance type/count
  - storage and timeouts
  - deployment configuration for updates
        ↓
CreateEndpoint
        ↓
InvokeEndpoint
  - exact Content-Type
  - exact container/model request schema
```

An endpoint name is not a model version. A production variant references a SageMaker model, which in turn references an image and model data.

### Container serving contract

A custom inference container must:

- start the model server as the container entry point;
- listen on the expected port and network interface;
- respond successfully to health checks such as `/ping`;
- implement `/invocations`;
- parse the declared `Content-Type`;
- return an appropriate response content type and status;
- load artifacts under the SageMaker model directory contract;
- log enough sanitized startup/inference detail to CloudWatch;
- handle termination and concurrency safely.

The exact request body is defined by the container/model handler, not by SageMaker. A Bedrock Converse payload sent unchanged to `InvokeEndpoint` is usually invalid.

### Why LLM deployment differs from small-model deployment

Large language models introduce:

- weights too large for one GPU;
- additional memory for runtime, activations, and key-value cache;
- long S3 download and initialization;
- large disk/volume requirements;
- tensor/model parallelism across accelerators;
- token-based rather than simple request-based capacity;
- batching/continuous batching opportunities;
- long generation and first-token latency;
- large container images and model artifacts;
- expensive idle GPU capacity.

Instance selection must account for model weights **and** runtime headroom. “The file fits in GPU memory” is not enough.

### Memory planning

Estimate:

```text
weight memory
+ quantization/runtime overhead
+ KV cache (context length × concurrent sequences)
+ activations/workspace
+ framework/serving overhead
+ safety margin
```

If the model does not fit:

- choose an instance with more/larger accelerators;
- use supported tensor parallelism/sharding;
- use a supported lower-precision/quantized representation after quality testing;
- reduce maximum context/concurrency;
- use an LMI container/serving engine designed for the model;
- select a smaller model if it meets quality.

Do not assume endpoint autoscaling solves an individual replica that cannot load the model.

## AWS services and APIs

### Large Model Inference containers

SageMaker Large Model Inference (LMI) containers provide a managed deep-learning container path designed for large-model serving. They use serving technology such as DJL Serving and supported optimized backends.

LMI capabilities can include, depending on image/backend/model:

- tensor parallelism;
- optimized model loading;
- dynamic or continuous batching;
- GPU/accelerator support;
- quantization;
- high-performance serving.

Use a published compatible LMI image and documented serving configuration. Do not assume every model architecture, quantization format, or optimization is supported by every image version.

### Model artifacts: `ModelDataUrl` versus `ModelDataSource`

Traditional model deployment often uses a compressed `model.tar.gz` artifact. For very large artifacts, download plus decompression can increase startup time and require extra temporary storage.

`ModelDataSource` can reference an S3 object or prefix. With an S3 prefix and `CompressionType: None`, SageMaker downloads uncompressed model files into the model directory without requiring one giant tar extraction.

Use this pattern when:

- the model consists of large uncompressed shards/files;
- startup is dominated by archive download/decompression;
- the selected serving image expects that layout.

The execution role needs read permission to the exact S3 model location and KMS key when encrypted.

### Large-model endpoint settings

For a large model stored in S3, understand these production-variant settings:

| Setting | Purpose | Symptom when too small |
|---|---|---|
| `VolumeSizeInGB` | EBS storage used for model artifacts when the instance lacks sufficient local storage | Download/extraction/no-space failure |
| `ModelDataDownloadTimeoutInSeconds` | Time allowed for SageMaker to obtain model data | Deployment fails while artifact is still downloading |
| `ContainerStartupHealthCheckTimeoutInSeconds` | Time allowed for the container to initialize and pass health check | Endpoint marks container unhealthy while weights are still loading |

Increasing timeouts only helps if the process is making progress. It does not fix:

- incompatible image/model;
- insufficient GPU memory;
- missing IAM/KMS access;
- corrupted artifacts;
- wrong entry point;
- `/ping` that never becomes healthy.

### SageMaker Model Registry

Register every releasable custom/fine-tuned model as a version in a model package group. Attach:

- model artifacts/image reference;
- model and prompt/evaluation metadata;
- quality, safety, latency, and cost metrics;
- dataset lineage where required;
- LoRA/base-model version relationship;
- approval status.

Typical lifecycle:

```text
new model package
  → PendingManualApproval
  → automated/human evaluation
      ├─ Approved → deployment pipeline
      └─ Rejected → no promotion
```

Approval status is a governance signal and pipeline trigger. It does not independently prove model quality or deploy the model unless automation is configured.

### LoRA and frequent adapter releases

For parameter-efficient fine-tuning:

- record the immutable base model and exact adapter version;
- register each releasable combination;
- preserve tokenizer, serving image, inference code, and prompt compatibility;
- evaluate the combined model, not the adapter in isolation;
- require approval before production;
- deploy gradually and keep the prior approved version available.

Changing only the adapter can still change safety, formatting, latency, and task behavior.

### Endpoint deployment guardrails

SageMaker deployment guardrails support safer updates to real-time and asynchronous endpoints.

Blue/green modes include:

- all-at-once;
- canary traffic shifting;
- linear traffic shifting.

With canary:

1. Create the green fleet with the new model/configuration.
2. Shift a small percentage of traffic.
3. Observe during a baking period.
4. If configured CloudWatch alarms fire, roll back to the blue fleet.
5. Otherwise continue the shift and retire the old fleet.

Alarm examples:

- invocation 5xx/4xx rate;
- model latency;
- overhead latency;
- timeout rate;
- CPU/GPU/memory utilization;
- invalid response/schema rate as a custom metric;
- safety or task-quality regression as a custom/synthetic signal.

Infrastructure alarms alone cannot detect fluent but incorrect output.

### Autoscaling

Autoscale a real-time endpoint based on workload and serving behavior. Candidate signals include invocations per instance or appropriate concurrency/utilization metrics for the endpoint architecture.

Plan for:

- scale-out delay;
- model download/startup duration;
- GPU quota and capacity;
- minimum warm capacity;
- maximum cost;
- cooldown;
- token length and generation time, not only request count.

If loading a 70 GB model takes many minutes, reactive scaling may be too slow for a sudden spike. Maintain adequate warm capacity or use a different architecture.

### SageMaker Batch Transform

Use Batch Transform for custom-model inference over S3 data when a persistent endpoint is unnecessary.

Important parameters:

- `SplitType: Line` for line-delimited records that can be split;
- `BatchStrategy: MultiRecord` to send several records per container invocation;
- `MaxPayloadInMB` to bound each request payload;
- `MaxConcurrentTransforms` for concurrent requests per instance;
- `InvocationsTimeoutInSeconds` for model processing time;
- `AssembleWith: Line` when line-oriented output is required;
- instance type and count;
- S3 input/output locations.

Current documented constraint: `MaxPayloadInMB` cannot exceed 100 MB, and when `MaxConcurrentTransforms` is specified, `MaxConcurrentTransforms × MaxPayloadInMB` must not exceed 100 MB.

#### GPU underutilization pattern

Symptom:

- millions of small JSON Lines records;
- one record per container request;
- low GPU utilization;
- memory headroom.

Remediation:

1. Confirm the container accepts newline-delimited mini-batches and returns one result per record.
2. Set `SplitType: Line`.
3. Use `BatchStrategy: MultiRecord`.
4. Increase `MaxPayloadInMB` gradually.
5. Tune `MaxConcurrentTransforms`.
6. Observe GPU utilization, latency, timeouts, and output alignment.

More concurrency is not always better. It can exhaust GPU memory or create longer per-request latency.

### Bedrock batch inference versus SageMaker Batch Transform

| Requirement | Bedrock batch inference | SageMaker Batch Transform |
|---|---|---|
| Model is supported managed Bedrock FM | Strong fit | Usually unnecessary |
| Arbitrary custom/open-weight container | No | Strong fit |
| Input/output in S3 | Yes | Yes |
| Tune container batching/concurrency | No serving-container control | Yes |
| Manage compute image/instances | No | Yes |
| Tool calling/multi-turn interaction | Not a normal batch fit | Not a normal per-record Batch Transform fit |

### Inference invocation contract

`InvokeEndpoint` requires:

- endpoint name;
- body matching the model handler;
- correct `ContentType`;
- optional `Accept`;
- optional targeting fields only if the endpoint configuration supports them.

The application adapter should convert its canonical request into the endpoint schema and normalize the response. Log schema version, model version, endpoint/variant, and safe correlation metadata.

## Architecture patterns

### Pattern 1 — Interactive open-weight LLM

```text
client
  → API Gateway/service
  → request adapter and safety checks
  → SageMaker real-time endpoint
      → LMI container
      → uncompressed S3 model shards
      → GPU production variant
  → deterministic output validation
  → client
```

Surrounding Bedrock-native features do not automatically apply to a SageMaker endpoint. Implement safety, prompt management, evaluation, and logging explicitly where needed.

### Pattern 2 — Governed LoRA release

```text
base model + adapter candidate
  → package/register version
  → evaluation dataset
  → quality/safety/latency/cost gate
  → PendingManualApproval
  → Approved
  → update endpoint with canary guardrail
  → bake alarms and synthetic tests
  → promote or rollback
```

The release manifest must pin base model, adapter, tokenizer, container, inference code, parameters, and evaluation dataset.

### Pattern 3 — Offline custom-model inference

```text
S3 JSONL shards
  → Batch Transform
      → Line splitting
      → MultiRecord mini-batches
      → tuned concurrency/payload
  → S3 output
  → validation/reconciliation
```

Use multiple reasonably sized input objects so SageMaker can distribute them across instances. One large unsplittable object can leave additional instances idle.

### Pattern 4 — Hybrid Bedrock and SageMaker

```text
stable application API
  → router
      ├─ routine managed task → Bedrock
      └─ domain-specific custom task → SageMaker endpoint
  → normalize and validate
```

Use this only if evaluation proves the custom model adds required capability or quality. The router must record which path served the request.

## Decision table

| Requirement | Choose | Avoid |
|---|---|---|
| Managed provider FM with least operations | Bedrock | Custom SageMaker GPU stack |
| Custom 70 GB open-weight interactive LLM | SageMaker real-time + LMI | Lambda or SageMaker serverless by assumption |
| Nightly custom-model classification | Batch Transform | Always-on real-time endpoint |
| Nightly Bedrock-supported FM generation | Bedrock batch inference | One Lambda model call per record |
| Large artifacts spend too long downloading/untarring | `ModelDataSource`, uncompressed files, adequate volume/timeouts | Only increasing instance count |
| Container initializes after health deadline | Increase startup health-check timeout after confirming progress | Infinite timeout masking incompatible image |
| Predictable managed-FM peak | Bedrock Provisioned Throughput | SageMaker unless custom hosting is required |
| Frequent governed LoRA versions | Model Registry + approval + endpoint guardrails | Overwriting one S3 key/model name |
| Low GPU utilization on small batch records | MultiRecord, payload/concurrency tuning | More instances while each remains underfed |
| New endpoint version may regress | Canary/linear guardrail + alarms | All-at-once with manual observation |

## Security and governance

### IAM and artifacts

- Give the SageMaker execution role only required S3, ECR, KMS, CloudWatch, and related access.
- Scope the application caller to the intended endpoint.
- Keep model artifacts in private S3 locations with encryption and versioning.
- Scan and sign/approve container images according to organizational policy.
- Pin immutable image digests and model versions for reproducibility.
- Do not place secrets in environment variables when a runtime credential provider or Secrets Manager pattern is appropriate.
- Separate build/training, registry approval, deployment, and invocation roles.

### Network controls

Depending on requirements:

- place the model container in a VPC configuration;
- use private subnets and endpoints for supported dependencies;
- use network isolation when the model container must not make outbound network calls;
- ensure model data and image access still follow supported SageMaker service behavior;
- protect the public application API separately.

Network isolation and VPC attachment solve different concerns. Verify the serving container’s required dependencies before blocking network access.

### Data handling

- Do not log raw prompts/responses containing regulated data.
- Set retention for endpoint logs and captured data.
- Apply application-level input/output safety controls to SageMaker-hosted models.
- Validate structured outputs deterministically.
- Encrypt in transit and at rest.
- Record model/version, prompt/config version, request correlation, and evaluation lineage.

### Model governance

Model Registry plus approval should answer:

- What exact model artifact and image were deployed?
- Which base model and adapter created this version?
- Which dataset and metrics justified approval?
- Who approved it and when?
- Which endpoint/variant received it?
- Which alarms or evaluations triggered rollback?

## Cost, latency, and reliability

### Cost drivers

- instance/GPU type and count;
- minimum warm replicas;
- idle endpoint time;
- blue/green overlap during deployment;
- model artifact storage and transfer;
- autoscaling headroom;
- Batch Transform instance count and duration;
- failed/repeated endpoint creation;
- oversized context/output and low batch efficiency.

Reduce cost by:

- proving a smaller or quantized model meets quality;
- using Batch Transform for offline work;
- improving batching and accelerator utilization;
- right-sizing minimum/maximum capacity;
- deleting unused test endpoints;
- shortening rollout overlap without weakening the bake gate;
- routing routine tasks to Bedrock/smaller models when evaluated.

### Latency

Track:

- container/model loading time;
- queueing;
- model latency;
- overhead latency;
- time to first token if the serving stack supports streaming;
- per-token generation time;
- p50/p95/p99;
- cold/new-replica behavior;
- input length, output length, and concurrency.

### Reliability

- Keep immutable prior versions available for rollback.
- Use health checks that reflect actual readiness.
- Use endpoint deployment guardrails and CloudWatch alarms.
- Test model load in the target instance type before production.
- Validate artifact checksum/layout and container compatibility.
- Capacity-plan for accelerator quotas and startup delay.
- Make API callers use bounded retries only for transient endpoint errors.
- Do not retry an invalid payload.

## Failure modes and troubleshooting

### Diagnostic sequence

```text
CreateEndpoint/UpdateEndpoint failed
  → DescribeEndpoint failure reason
  → CloudWatch endpoint/container startup logs
  → IAM/KMS/ECR/S3 access
  → disk/volume availability
  → model download progress and timeout
  → container entry point and /ping
  → GPU memory and model-server logs
  → serving image/model compatibility
```

### Troubleshooting matrix

| Symptom | Check first | Probable cause | Remediation |
|---|---|---|---|
| Endpoint fails before model download completes | logs and elapsed download time | `ModelDataDownloadTimeoutInSeconds` too low | Increase after confirming access and progress |
| Health check fails while weights load | startup logs and `/ping` readiness | startup timeout too low | Increase `ContainerStartupHealthCheckTimeoutInSeconds` |
| “No space left” | artifact size, compressed expansion, volume | EBS/local storage insufficient | Increase `VolumeSizeInGB`; use uncompressed source/layout |
| CUDA out-of-memory at startup | GPU logs and shard config | model/runtime does not fit replica | Larger/more GPUs, supported parallelism/quantization, smaller model |
| OOM only under concurrency | context/output/concurrent sequences | KV cache/workspace growth | Reduce concurrency/context, tune batching, add capacity |
| Container never becomes healthy | process/port/`/ping` logs | wrong entry point or server contract | Fix container contract; timeout is not the cure |
| Access denied downloading model | execution role, bucket/key/KMS policy | missing S3/KMS permission | Scope and grant required access |
| Image pull failure | ECR URI, permissions, architecture | wrong image/tag/role/platform | Correct immutable image and permissions |
| `InvokeEndpoint` JSON parse error | Content-Type and request body | body does not match handler schema | Model-aware serializer and exact content type |
| Output format changes across release | release manifest and tests | tokenizer/adapter/handler/prompt mismatch | Pin all components; schema regression gate |
| Canary automatically rolls back | alarm history and variant metrics | errors/latency/custom quality threshold | Diagnose candidate; do not disable alarm to force release |
| Autoscaling cannot absorb spike | scale-out/startup timing | model replicas take too long to load | More warm capacity or different serving strategy |
| Batch GPU utilization low | records/request, concurrency, input sharding | SingleRecord and underfilled GPU | MultiRecord, tune payload/concurrency, more input files |
| Batch job times out | per-mini-batch time and timeout | payload/concurrency too high or timeout low | Start smaller; tune payload, concurrency, invocation timeout |
| Extra Batch Transform instances idle | input object count and split | one unsplittable S3 object | line split or multiple input objects |
| Batch output cannot align to input | split/assembly/container output | response does not preserve one result per record | Correct handler and `AssembleWith`/record protocol |

## Common exam traps

1. **SageMaker is not automatically better for a custom prompt.** Prompt customization is not the same as custom-model hosting.
2. **Lambda cannot host a huge GPU LLM.**
3. **Provisioned Throughput is a Bedrock capacity feature, not a SageMaker endpoint mode.**
4. **More endpoint instances do not fix a model that cannot fit on one replica.**
5. **A startup timeout does not fix GPU OOM or an incompatible container.**
6. **A model artifact download timeout and a container health-check timeout control different phases.**
7. **Compressed artifacts can need extra time and disk.**
8. **`ModelDataSource` with `CompressionType: None` is useful for uncompressed large-model files; it is not a quality optimization.**
9. **Endpoint autoscaling reacts after signals and may be slower than large-model startup.**
10. **Batch Transform does not require a persistent endpoint.**
11. **`MultiRecord` is correct only if the container can parse and return aligned multi-record data.**
12. **Adding instances cannot parallelize one unsplittable object across them.**
13. **Model Registry approval is not an evaluation by itself.**
14. **Canary infrastructure health does not prove semantic quality.**
15. **A LoRA adapter release must pin and evaluate its base model and serving stack.**
16. **SageMaker endpoints do not automatically gain Bedrock Guardrails or Converse semantics.**

## Local mock references

- **PE1-Q47:** Bedrock and SageMaker calls require separate request schemas and correct content types.
- **PE1-Q54:** increase model-download and startup-health timeouts when logs prove the model is still loading.
- **PE1-Q62:** register LoRA versions, require approval, use endpoint deployment guardrails and alarm rollback.
- **PE2-Q14:** Batch Transform `MultiRecord`, `MaxConcurrentTransforms`, and `MaxPayloadInMB`.
- **PE2-Q25:** 70 GB open-weight LLM with LMI, uncompressed `ModelDataSource`, volume, download timeout, and health timeout.

Related deployment-selection questions:

- **PE1-Q18:** Bedrock batch for a Bedrock-managed FM, not per-record Lambda.
- **PE1-Q26, Q60:** Bedrock Provisioned Throughput for predictable managed-FM demand.
- **PE1-Q59:** candidate model evaluation plus canary release.
- **PE1-Q68, Q71:** smaller-model routing/cascading can avoid unnecessary custom GPU infrastructure.

## Hands-on validation

1. Draw `CreateModel → CreateEndpointConfig → CreateEndpoint → InvokeEndpoint` and label every artifact, role, schema, and log source.
2. Calculate whether a hypothetical model fits after weights, KV cache, runtime overhead, concurrency, and safety margin.
3. Create a startup checklist that distinguishes storage, download timeout, health timeout, and GPU memory.
4. Define a Model Registry package for a LoRA release with base model, adapter, tokenizer, container digest, metrics, and approval.
5. Design a canary endpoint update with alarm and semantic-quality signals.
6. Tune a hypothetical JSONL Batch Transform workload from `SingleRecord` to `MultiRecord`, explaining each metric and limit.
7. Explain why the same application request needs different serializers for Bedrock Converse, Bedrock InvokeModel, and SageMaker InvokeEndpoint.

## Recall questions

1. Which requirement makes SageMaker preferable to Bedrock?
2. When should a custom model use real-time, asynchronous, or Batch Transform inference?
3. Why does a large model need memory beyond its weight file size?
4. What problem does an LMI container solve?
5. When is uncompressed `ModelDataSource` useful?
6. How do `VolumeSizeInGB`, model download timeout, and startup health timeout differ?
7. Why can increasing the health timeout hide rather than solve a problem?
8. What endpoints must a custom serving container implement?
9. Why must `InvokeEndpoint` use the container's own schema?
10. What artifacts must a LoRA release pin?
11. What does Model Registry approval mean?
12. How do canary and linear traffic shifting differ?
13. Which alarms should trigger endpoint rollback?
14. Why might autoscaling be too slow for a large LLM spike?
15. What causes low Batch Transform GPU utilization?
16. What must be true before selecting `MultiRecord`?
17. Why can one input object leave Batch Transform instances idle?
18. How do Bedrock batch inference and SageMaker Batch Transform differ?
19. Which safety controls must be added around a SageMaker-hosted FM?
20. Which evidence distinguishes timeout, disk, IAM, and GPU-memory startup failures?

## Official sources

- [AIP-C01 Domain 2](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain2.html)
- [AWS decision guide: Amazon Bedrock or Amazon SageMaker AI](https://docs.aws.amazon.com/pdfs/decision-guides/latest/bedrock-or-sagemaker/bedrock-or-sagemaker.pdf)
- [Bedrock on-demand custom-model deployment](https://docs.aws.amazon.com/bedrock/latest/userguide/deploy-custom-model-on-demand.html)
- [Bedrock batch inference model and custom-model support](https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference-supported.html)
- [SageMaker large-model endpoint parameters](https://docs.aws.amazon.com/sagemaker/latest/dg/large-model-inference-hosting.html)
- [Deploy models with DJL Serving](https://docs.aws.amazon.com/sagemaker/latest/dg/deploy-models-frameworks-djl-serving.html)
- [SageMaker `S3ModelDataSource` API](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_S3ModelDataSource.html)
- [SageMaker Batch Transform](https://docs.aws.amazon.com/sagemaker/latest/dg/batch-transform.html)
- [Batch Transform troubleshooting](https://docs.aws.amazon.com/sagemaker/latest/dg/batch-transform-errors.html)
- [Custom inference code with Batch Transform](https://docs.aws.amazon.com/sagemaker/latest/dg/your-algorithms-batch-code.html)
- [SageMaker Model Registry model versions](https://docs.aws.amazon.com/sagemaker/latest/dg/model-registry-models.html)
- [Update Model Registry approval status](https://docs.aws.amazon.com/sagemaker/latest/dg/model-registry-approve.html)
- [SageMaker deployment guardrails](https://docs.aws.amazon.com/sagemaker/latest/dg/deployment-guardrails.html)
- [Deployment guardrail alarms and rollback](https://docs.aws.amazon.com/sagemaker/latest/dg/deployment-guardrails-configuration.html)
