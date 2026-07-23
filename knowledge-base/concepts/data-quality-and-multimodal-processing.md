# Data Quality and Multimodal Processing

Status: Verified  
Official tasks: 1.3.1–1.3.4  
Last verified: 2026-07-23

## Why this matters

An FM cannot compensate reliably for malformed, stale, poisoned, or incorrectly formatted input. AIP-C01 tests the complete path from raw enterprise data to validated model requests across text, images, documents, audio, video, and tables.

## Core concepts

### The quality contract

Define checks in five layers:

| Layer | Example checks |
|---|---|
| Transport | supported type, size, checksum, decompression, malware scan |
| Structure | schema, required fields, parse success, dimensions, duration |
| Content | completeness, range, uniqueness, freshness, language, OCR/transcript quality |
| Privacy/safety | PII, secrets, prohibited content, source rights, prompt-injection indicators |
| Model readiness | correct request schema, token limit, modality support, normalized encoding |

Quality is not one score. Publish rule-level results, preserve lineage, quarantine failed records, and block downstream processing when required thresholds fail.

### Immutable zones

Keep raw input immutable, then write separately versioned normalized, validated, curated, and rejected outputs. This makes replay and root-cause analysis possible without silently changing source evidence.

### Service boundaries

| Need | AWS service |
|---|---|
| Serverless rules on tabular/JSON data | AWS Glue Data Quality |
| Custom batch transformation/container | SageMaker Processing |
| General documents, images, audio, and video to structured insights | Bedrock Data Automation |
| OCR, forms, tables, queries, expenses, or identity documents | Amazon Textract |
| Speech-to-text, vocabulary, speaker/channel features, transcript PII controls | Amazon Transcribe |
| Text entities, language, classification, sentiment, or PII | Amazon Comprehend |
| Lightweight validation, normalization, and request assembly | AWS Lambda |
| Durable branching and asynchronous job polling | AWS Step Functions |

Choose from the required output, not merely the input type. A scanned form needing cells and key-value geometry points to Textract; a mixed-media workflow needing business fields defined by a blueprint points to Bedrock Data Automation.

## How it works

### Reference pipeline

```text
source event
  → validate envelope/type/size
  → copy immutable raw artifact
  → parse or transcribe
  → normalize encoding/schema/units
  → evaluate data-quality rules
  → valid / quarantine split
  → redact or tokenize sensitive content
  → enrich with entities and lineage
  → assemble model-specific request
  → invoke
  → validate structured and business output
```

Use Step Functions for asynchronous jobs, retry classification, polling, and terminal-state handling. Use Lambda for bounded transformations, not large media or long-running processing.

### Glue Data Quality

Glue Data Quality uses Data Quality Definition Language (DQDL) rules. It is serverless and can run in Data Catalog and ETL contexts. Publish metrics to CloudWatch and detailed results to supported destinations.

Row-level failure identification is available for ETL rules, enabling valid and quarantine outputs. Current documentation warns that nested/list data types are not evaluated directly: flatten or otherwise normalize them before applying rules.

Example rule families:

- `IsComplete` for required values;
- `IsUnique` for identifiers;
- `ColumnValues` for ranges or enumerations;
- `Freshness` for recency;
- `RowCount` and ratios for pipeline-health assertions.

A passing job with a failed rule is not necessarily a successful data release. The orchestration must inspect the rule outcome and enforce its threshold.

### Bedrock Data Automation

Bedrock Data Automation (BDA) produces standard structured output from documents, images, audio, and video. Projects hold configuration. Blueprints define the desired fields/output for a document or image class. Processing is asynchronous, so start the job, poll or react to completion, validate its output, and handle unsupported or low-quality documents.

BDA is broader than OCR. Use Textract when the explicit requirement is OCR/form/table/layout primitives or one of its specialized analysis APIs. Use BDA when the requirement is managed multimodal insight extraction into a business-oriented schema.

### Audio processing

Amazon Transcribe custom vocabularies improve recognition of domain terms, names, and acronyms. PII redaction produces redacted transcript output for supported settings, but AWS documents that machine-learning redaction is not guaranteed to identify every PII instance and is not a HIPAA de-identification mechanism. Validate high-risk output and layer deterministic controls.

Use Comprehend after transcription when text-level entity, classification, or PII operations are required. Comprehend offers real-time PII location and asynchronous PII redaction; choose based on workflow and supported features.

### Text and documents

Use Comprehend to extract named entities or custom domain entities before normalization, routing, metadata creation, or redaction. Use Textract for OCR and structural document analysis. Preserve page/span provenance so downstream citations can point back to source evidence.

Do not let extracted document instructions become trusted system instructions. Treat all source content as untrusted data and delimit it explicitly when prompting.

### SageMaker Processing

SageMaker Processing runs built-in or custom containers on managed compute with S3 inputs and outputs. It fits large batch transforms, existing Python/Spark logic, feature preparation, and algorithms that exceed Lambda duration or resource constraints. Pin the image and code version and persist processing-job metadata for lineage.

### Model-specific formatting

| Target | Required contract |
|---|---|
| Bedrock `Converse` / `ConverseStream` | `messages` with `role` and typed content blocks |
| Bedrock `InvokeModel` | selected model/provider’s native JSON request |
| SageMaker `InvokeEndpoint` | body and `ContentType` expected by the deployed container |
| Structured output | supported JSON schema/tool schema plus semantic validation |

For multimodal content, verify each model’s supported formats, size/count limits, Region, and API. A typed `image`, `document`, audio, or video content block is not accepted merely because another model supports it.

### Input enhancement

Use deterministic normalization first:

- canonical character encoding and Unicode;
- dates, units, currencies, phone numbers, and identifiers;
- whitespace and duplicated headers;
- language detection and routing;
- entity extraction and canonical names;
- de-duplication and content hashes;
- source, revision, jurisdiction, and permissions metadata.

Use an FM to reformat ambiguous prose only when the output is validated against a schema and source evidence. Never let a generative rewrite silently replace the immutable original.

## AWS services and APIs

- AWS Glue Data Quality: DQDL rules, job outcomes, row-level results, CloudWatch publishing.
- Bedrock Data Automation: projects, blueprints, asynchronous multimodal processing.
- Amazon Textract: synchronous/asynchronous document detection and analysis.
- Amazon Transcribe: transcription jobs, custom vocabulary, PII redaction.
- Amazon Comprehend: entity and PII detection/redaction.
- SageMaker Processing: managed container jobs and S3 artifacts.
- Lambda: small normalizers, validators, and adapters.
- Step Functions and EventBridge: orchestration and state changes.
- S3, KMS, CloudWatch, and CloudTrail: zones, encryption, evidence, and audit.

## Architecture patterns

### Curated/quarantine split

```text
raw S3
  → parser/normalizer
  → Glue DQ rules
      ├─ pass → curated S3 → embedding/inference
      └─ fail → quarantine S3 → review/remediation
                 ↓
             CloudWatch alarm
```

### Mixed-media packet

```text
document/image → BDA or Textract ┐
audio → Transcribe → Comprehend  ├→ Lambda schema assembler → validated packet
CSV/JSON → Glue DQ               ┘
```

Include source pointers, processing versions, quality outcomes, and redaction state in the packet.

## Decision table

| Scenario | Prefer | Why |
|---|---|---|
| Recurring CSV completeness/range checks | Glue Data Quality | Declarative rules and published quality outcomes |
| Nested JSON rules | Flatten, then Glue DQ | Nested/list types are not evaluated directly |
| Existing heavy Python transform | SageMaker Processing | Managed batch container, not Lambda limits |
| Mixed PDF/JPEG/audio/video extraction | BDA | Unified managed multimodal processing |
| Forms and tables with geometry | Textract | Specialized document structure |
| Domain-heavy call recordings | Transcribe custom vocabulary | Improves recognition before text analysis |
| Locate PII in synchronous text | Comprehend real-time PII | Text-native entity offsets |
| Small request normalization | Lambda | Bounded, event-driven transformation |
| Chat model with images | `Converse` typed blocks if supported | Common message contract |
| Specialized model request | `InvokeModel` native body | Exact provider/model schema |

## Security and governance

- Classify raw data before processing and reject unsupported sensitivity levels.
- Use least-privilege roles per processor and separate raw, curated, and quarantine access.
- Encrypt S3, job outputs, logs, and temporary artifacts with approved KMS keys.
- Use VPC endpoints/private networking where required; restrict outbound container access.
- Redact or tokenize before sending data to a model when the model does not need identity.
- Avoid logging raw documents, transcripts, prompts, or rejected PII.
- Track source consent/licensing, retention, deletion, and residency across derivatives.
- Preserve lineage from output field to artifact/page/time span.
- Treat ML PII detection as probabilistic; use deterministic validation or human review for high-impact flows.

## Cost, latency, and reliability

| Driver | Control |
|---|---|
| Reprocessing unchanged content | Content hashes and idempotent job keys |
| Large media | Early type/size/duration validation and appropriate async services |
| BDA or transcription retries | Poll terminal state, retry only transient errors |
| Expensive semantic reformatting | Normalize deterministically first; invoke FMs only for ambiguity |
| DQ scan cost | Scope columns/rules and run incrementally when valid |
| Lambda timeout/memory pressure | Move heavy batch work to SageMaker Processing |
| Downstream poison records | Enforce quality gate, not merely publish a metric |
| Duplicate outputs | Idempotent writes and job correlation IDs |

## Failure modes and troubleshooting

| Symptom | Likely cause | Evidence | Corrective action |
|---|---|---|---|
| Glue job succeeds but bad data proceeds | Rule outcome was not enforced | DQ results and orchestration branch | Fail/quarantine on threshold |
| Rules skip/fail nested fields | Struct/list not flattened | input schema | Normalize to supported columns |
| Document fields drift | Blueprint/parser version or document layout changed | sample output by version | Version blueprint, regression test, route exceptions |
| Transcript misses product terms | Vocabulary absent or poor audio | confidence and sampled transcript | Custom vocabulary and audio-quality remediation |
| PII appears in output | Probabilistic detector missed it | raw/redacted comparison | Layer rules/review; fail closed for high risk |
| Multimodal request rejected | Unsupported format/size/model/API | capability and validation exception | Convert within policy or select compatible model |
| SageMaker request returns 415/schema error | Wrong `ContentType` or body | endpoint container contract | Use exact serializer |
| Lambda times out | Media/job too large or synchronous polling | duration/memory logs | Async service plus Step Functions |
| Model-ready JSON parses but is wrong | Schema checked shape only | business-rule validation | Add semantic validation/reference checks |
| Quarantine leaks into retrieval | Prefix/role boundaries weak | S3 paths, IAM, ingestion source | Separate buckets/prefix policies and explicit sources |

## Common exam traps

1. A successful ETL job does not mean every DQ rule passed.
2. Glue DQ does not directly evaluate nested/list types; flatten first.
3. Lambda is not the default for heavy media or long batch transforms.
4. BDA is not simply Textract under a new name; choose by output requirement.
5. Transcribe PII redaction is probabilistic and not HIPAA de-identification.
6. Comprehend extracts/analyzes text; it does not transcribe audio.
7. `Converse` uses typed message blocks; `InvokeModel` is model-specific.
8. Correct JSON syntax is not correct business meaning.
9. An FM rewrite should not overwrite original evidence.
10. One multimodal model’s limits do not generalize to every Bedrock model.
11. Metrics without an enforced gate do not protect downstream systems.

## Local mock references

Mocks illustrate scenario patterns; AWS documentation controls current behavior.

| Focus | Questions |
|---|---|
| Glue DQ, quarantine, CloudWatch | PE1-Q1; PE2-Q51 |
| BDA blueprints and asynchronous processing | PE1-Q37, Q72; PE2-Q72 |
| Comprehend entities and normalization | PE1-Q38 |
| API-specific request formatting | PE1-Q47 |
| Shared multimodal embedding space | PE1-Q65 |
| Transcribe vocabulary/PII plus Processing | PE2-Q72 |

Mock caveat: PE1-Q65 names a particular embedding answer. Learn the capability—one semantic space across required modalities—and verify the current supported model. Current AWS documentation also describes Nova Multimodal Embeddings.

## Hands-on validation

1. Write DQDL rules for completeness, uniqueness, range, and freshness.
2. Prove a nested field is not evaluated as intended, flatten it, and rerun.
3. Split row-level valid/quarantine data and alarm on a required rule.
4. Create a BDA blueprint, process a document or image, and inspect the async result.
5. Extract a form with Textract and compare its geometry/table output with BDA’s business fields.
6. Transcribe domain audio with and without a custom vocabulary; sample PII-redacted output.
7. Run a custom SageMaker Processing container with S3 input/output.
8. Assemble the same evidence for `Converse` and for one native `InvokeModel` request.
9. Force a malformed/oversized modality and verify it is rejected before paid inference.

## Recall questions

1. Which quality layers must run before model invocation?
2. Why retain immutable raw data after normalization?
3. What does Glue DQ publish, and what must orchestration still enforce?
4. What must happen before applying DQ rules to nested lists?
5. When is BDA preferable to Textract?
6. When is Textract preferable to BDA?
7. Why does custom vocabulary precede Comprehend analysis?
8. Why is Transcribe PII redaction insufficient as the only high-risk control?
9. When should a Lambda transform move to SageMaker Processing?
10. How do `Converse` and `InvokeModel` formatting differ?
11. What must be validated after a response conforms to a JSON schema?
12. Which lineage fields allow an extracted answer to be audited?

## Official sources

- [AWS Glue Data Quality](https://docs.aws.amazon.com/glue/latest/dg/glue-data-quality.html)
- [Getting started with Glue Data Quality](https://docs.aws.amazon.com/glue/latest/dg/data-quality-getting-started.html)
- [How Bedrock Data Automation works](https://docs.aws.amazon.com/bedrock/latest/userguide/bda-how-it-works.html)
- [BDA blueprints](https://docs.aws.amazon.com/bedrock/latest/userguide/bda-blueprints-console.html)
- [Amazon Textract](https://docs.aws.amazon.com/textract/latest/dg/what-is.html)
- [Transcribe custom vocabularies](https://docs.aws.amazon.com/transcribe/latest/dg/custom-vocabulary.html)
- [Transcribe PII redaction](https://docs.aws.amazon.com/transcribe/latest/dg/pii-redaction.html)
- [Comprehend PII](https://docs.aws.amazon.com/comprehend/latest/dg/how-pii.html)
- [SageMaker Processing](https://docs.aws.amazon.com/sagemaker/latest/dg/processing-job.html)
- [Bedrock Converse APIs](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference.html)
- [Bedrock inference APIs](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-api.html)
- [Nova Multimodal Embeddings](https://docs.aws.amazon.com/nova/latest/nova2-userguide/embeddings.html)
