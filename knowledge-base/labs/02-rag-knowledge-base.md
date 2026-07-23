# Lab 02 — RAG Knowledge Base, Hybrid Retrieval, and Reranking

Status: Ready to run with account-specific values  
Related exam tasks: 1.4.1–1.4.5, 1.5.1–1.5.6  
Last verified: 2026-07-23

## Objective

Build and verify a small Amazon Bedrock Knowledge Base that:

1. ingests versioned S3 documents and metadata;
2. uses a documented chunking strategy and compatible embedding dimension;
3. proves that an update is not searchable until ingestion completes;
4. compares semantic and hybrid retrieve-only results;
5. applies metadata filters and reranking;
6. exposes citations/source locations without treating relevance scores as factual confidence;
7. diagnoses failures at ingestion, filtering, retrieval, and ranking stages;
8. records cost, security, and cleanup evidence.

The primary track uses a **customer-managed Knowledge Base with OpenSearch Serverless** because it exposes vector-index, hybrid-search, and reranking decisions. If your goal is lowest ownership rather than index control, complete the Managed Knowledge Base comparison at the end.

## Success criteria

The lab is complete only when you can show:

- the embedding output dimension exactly matches the vector-index mapping;
- every source has lineage and filterable version/status metadata;
- at least one exact identifier ranks better with hybrid than semantic-only search;
- an ineligible document is excluded by a metadata filter;
- reranking improves or preserves a defined retrieval metric on the benchmark;
- an updated/deleted source is reflected only after a terminal successful sync;
- `Retrieve` results are inspected independently from generated answers;
- an intentionally injected failure is located at the correct pipeline stage;
- temporary paid vector resources and broad permissions are removed.

Do not declare success from one fluent generated answer.

## Architecture

```text
Versioned test files + sidecar metadata in S3
                │
                ▼
Customer-managed Bedrock Knowledge Base
  parser → chunker → embedding model
                │
                ▼
OpenSearch Serverless vector index
  vector field + filterable text + metadata
                │
                ▼
bedrock-agent-runtime Retrieve
  semantic or hybrid → metadata filter → candidate set → reranker
                │
                ├─ retrieve-only benchmark evidence in local/S3 JSONL
                └─ optional governed generation + claim-level citations

CloudWatch: latency/errors/operational signals
CloudTrail: control-plane/API audit
```

Current product boundary:

- Customer-managed/vector Knowledge Bases support the documented `Retrieve` and `RetrieveAndGenerate` patterns.
- The product named **Managed Knowledge Base** uses `Retrieve` or `AgenticRetrieveStream`; current `RetrieveAndGenerate` documentation explicitly excludes Managed Knowledge Bases.

Record the Knowledge Base type in every lab result. “Bedrock KB” alone is ambiguous.

## Prerequisites

- An AWS account and approved lab Region.
- Temporary credentials from an approved federation method.
- Access to Amazon Bedrock, one supported text embedding model, and one supported reranker.
- An encrypted S3 bucket with lifecycle expiration for test inputs/results.
- An OpenSearch Serverless vector collection/index compatible with Knowledge Bases.
- A Bedrock Knowledge Base execution role scoped to:
  - the lab S3 prefix;
  - the selected embedding model;
  - the lab vector collection/index;
  - the KMS keys needed for those resources.
- A current AWS SDK. Examples use Boto3 clients:
  - `bedrock-agent` for ingestion control;
  - `bedrock-agent-runtime` for retrieval.
- No customer, regulated, secret, or copyrighted production corpus.

### Cost guardrails

Before provisioning:

1. choose a 6–10 document synthetic corpus;
2. cap the benchmark at approximately 20 queries;
3. set a maximum candidate count and reranked count;
4. note OpenSearch Serverless minimum-capacity/cost behavior for your account;
5. set a calendar reminder to delete the collection after the lab;
6. use one embedding configuration rather than repeatedly reingesting blindly.

If an existing approved non-production vector store is available, reuse it only when isolation and cleanup are clear. Do not attach this lab to production indexes.

## Step 1 — Create a diagnostic corpus

Create synthetic documents that reveal different retrieval failures:

| File | Required content | Purpose |
|---|---|---|
| `returns-v1.md` | old return period and code `RET-1042` | stale-version/filter test |
| `returns-v2.md` | current return period and same code | current-version test |
| `warranty.md` | paraphrasable warranty policy | semantic test |
| `product-codes.md` | exact IDs `ZX-4815`, `AB-2207` | lexical/hybrid test |
| `shipping.md` | several headings and a narrow exception clause | chunking test |
| `restricted.md` | synthetic internal-only rule | authorization/filter test |

Write 12–20 benchmark queries before indexing. For each, record:

```json
{
  "queryId": "q-001",
  "query": "Which rule applies to product ZX-4815?",
  "expectedDocumentIds": ["product-codes"],
  "expectedExactText": ["ZX-4815"],
  "requiredFilter": {
    "status": "current"
  },
  "mustNotReturn": ["returns-v1"],
  "category": "exact-identifier"
}
```

Include:

- semantic paraphrases;
- exact IDs;
- a question requiring a current-version filter;
- a narrow clause that needs surrounding context;
- one multi-part question;
- one question with no supporting evidence.

The no-evidence case must produce an empty/insufficient-evidence outcome, not a guessed answer.

## Step 2 — Add metadata sidecars

For each S3 source object, add a sidecar beside it:

```text
returns-v2.md
returns-v2.md.metadata.json
```

Current S3 connector documentation limits the sidecar to 10 KB. A representative structure is:

```json
{
  "metadataAttributes": {
    "documentId": {
      "value": {
        "type": "STRING",
        "stringValue": "returns-v2"
      },
      "includeForEmbedding": false
    },
    "domain": {
      "value": {
        "type": "STRING",
        "stringValue": "returns-policy"
      },
      "includeForEmbedding": true
    },
    "status": {
      "value": {
        "type": "STRING",
        "stringValue": "current"
      },
      "includeForEmbedding": false
    },
    "effectiveYear": {
      "value": {
        "type": "NUMBER",
        "numberValue": 2026
      },
      "includeForEmbedding": false
    }
  }
}
```

Use `includeForEmbedding: true` only when the value should affect semantic similarity. IDs, effective dates, ACL-like labels, and statuses usually belong in filterable metadata. Record this decision:

| Field | Filterable | Included in embedding | Reason |
|---|---:|---:|---|
| `documentId` | Yes | No | Lineage and exact filter |
| `domain` | Yes | Yes | Domain words improve semantic routing |
| `status` | Yes | No | Eligibility, not meaning |
| `effectiveYear` | Yes | No | Version filter |
| `accessLevel` | Yes | No | Security-related eligibility |

An `accessLevel` filter in this lab demonstrates retrieval precision; it is not a complete authorization boundary.

## Step 3 — Choose chunking

Start with fixed-size chunking and overlap so results are easy to compare. Record:

```text
Chunking strategy:
Maximum tokens:
Overlap percentage/tokens:
Reason:
Expected failure:
```

Then test a second configuration, preferably hierarchical chunking:

- child chunks optimize precise matching;
- parent chunks preserve surrounding meaning;
- retrieval matches a child and returns the associated parent.

Keep headings with the paragraphs they qualify. Do not use so much overlap that several top results are copies of the same passage.

If the console/API cannot change the chunking configuration after data-source creation, create a separate test data source/Knowledge Base or rebuild deliberately. Do not pretend an in-place change retroactively rechunked existing vectors.

## Step 4 — Select embeddings and define the index

Select a current text embedding model supported for Knowledge Bases in your Region. Record:

```text
Embedding model ID:
Output dimension:
Vector data type:
Input limits:
Supported Region:
Selection date:
```

For Titan Text Embeddings V2, current documentation describes 1,024 dimensions by default and 512 or 256 as alternatives. Choose one and configure the OpenSearch mapping to the exact same dimension.

The vector index also needs the documented fields for:

- vector values;
- chunk/source text;
- Bedrock-managed metadata;
- a filterable text field when the selected hybrid-search configuration requires it.

Use the `faiss` engine for the OpenSearch Serverless Knowledge Base index. Current Bedrock guidance requires `faiss` for metadata filtering; do not reuse an older `nmslib` index and assume feature parity.

Do not copy an index mapping from a different embedding model. Save the created mapping as evidence.

### Safe mismatch test

Do not corrupt the working index. In a separate disposable index definition, deliberately specify the wrong dimension and prove that setup/ingestion/query validation fails. Record the error, then delete that disposable index.

Exam lesson: dimension reduction is a cost/latency hypothesis, not an automatic optimization. Reindex and compare retrieval quality before adopting it.

## Step 5 — Create the customer-managed Knowledge Base

Configure:

1. the selected embedding model;
2. the OpenSearch Serverless collection/index and field mappings;
3. the S3 data source prefix;
4. the chosen chunking strategy;
5. a least-privilege Knowledge Base service role;
6. encryption and network/resource policies.

Record identifiers without secrets:

```text
Region:
Knowledge Base type: customer-managed
Knowledge Base ID:
Data source ID:
Embedding model:
Vector collection/index:
Chunking configuration:
Creation timestamp:
```

Do not put access keys, Secrets Manager values, raw IAM session data, or sensitive source text in the evidence file.

## Step 6 — Start ingestion and prove completion

Start an ingestion job:

```python
import boto3

REGION = "REGION"
KB_ID = "KNOWLEDGE_BASE_ID"
DATA_SOURCE_ID = "DATA_SOURCE_ID"

agent = boto3.client("bedrock-agent", region_name=REGION)

started = agent.start_ingestion_job(
    knowledgeBaseId=KB_ID,
    dataSourceId=DATA_SOURCE_ID,
    description="AIP-C01 RAG lab initial corpus"
)

INGESTION_JOB_ID = started["ingestionJob"]["ingestionJobId"]
```

Poll with `GetIngestionJob` using bounded backoff until a terminal state. Do not mark synchronization successful from the `StartIngestionJob` response.

```python
status = agent.get_ingestion_job(
    knowledgeBaseId=KB_ID,
    dataSourceId=DATA_SOURCE_ID,
    ingestionJobId=INGESTION_JOB_ID
)["ingestionJob"]

print({
    "status": status["status"],
    "statistics": status.get("statistics", {}),
    "failureReasons": status.get("failureReasons", [])
})
```

Pass only when status is `COMPLETE` and the statistics match the intended changes. Keep:

- job ID;
- start/end timestamps;
- final status;
- scanned/new/modified/deleted/failed statistics available in the response;
- sanitized failure reasons.

## Step 7 — Build a retrieve-only harness

Use `Retrieve` before any generation:

```python
runtime = boto3.client("bedrock-agent-runtime", region_name=REGION)

def retrieve(query, search_type="SEMANTIC", number_of_results=10, filter_=None):
    vector = {
        "numberOfResults": number_of_results,
        "overrideSearchType": search_type
    }
    if filter_ is not None:
        vector["filter"] = filter_

    return runtime.retrieve(
        knowledgeBaseId=KB_ID,
        retrievalQuery={"text": query},
        retrievalConfiguration={
            "vectorSearchConfiguration": vector
        }
    )
```

For every result record:

- rank;
- source/document ID and location;
- chunk text hash or sanitized excerpt;
- metadata;
- relevance score;
- request ID;
- client latency.

The score orders relevance; do not label it “confidence.”

### Retrieve-only metrics

Calculate at least:

- expected-source hit at k;
- reciprocal rank of the first expected source;
- context coverage for reference facts;
- forbidden/stale-source rate;
- duplicate-chunk rate;
- p50/p95 client latency.

Use the same benchmark, filters, and candidate count when comparing configurations.

## Step 8 — Compare semantic and hybrid search

Run every query with:

```text
overrideSearchType = SEMANTIC
overrideSearchType = HYBRID
```

Hybrid search depends on a supported store and compatible text-field configuration. Do not claim it is enabled merely because a request accepted the word `HYBRID`; inspect results and the current feature documentation.

Expected result:

- semantic should perform well on paraphrases;
- hybrid should improve exact identifiers such as `ZX-4815` while preserving conceptual matches.

Record a comparison:

| Query category | Semantic MRR/hit@k | Hybrid MRR/hit@k | Winner | Explanation |
|---|---:|---:|---|---|
| Paraphrase |  |  |  |  |
| Exact identifier |  |  |  |  |
| Mixed |  |  |  |  |

Select hybrid only if the measured workload benefits.

## Step 9 — Add metadata filters

Filter to current documents:

```python
current_filter = {
    "equals": {
        "key": "status",
        "value": "current"
    }
}
```

Then combine constraints using the documented logical-filter structure, for example current status plus an allowed domain. Verify:

- `returns-v1` is absent;
- `returns-v2` is eligible;
- a misspelled/missing metadata key fails closed in application logic;
- the client cannot supply an arbitrary tenant/access scope.

Do not ask the FM to “ignore outdated documents” after retrieval. Exclude them before generation.

## Step 10 — Add reranking

Retrieve a measured candidate set, then rerank to a smaller evidence set using a reranking model supported in the Region.

Representative retrieval configuration:

```python
reranked = runtime.retrieve(
    knowledgeBaseId=KB_ID,
    retrievalQuery={"text": "QUERY"},
    retrievalConfiguration={
        "vectorSearchConfiguration": {
            "numberOfResults": 12,
            "overrideSearchType": "HYBRID",
            "filter": current_filter,
            "rerankingConfiguration": {
                "type": "BEDROCK_RERANKING_MODEL",
                "bedrockRerankingConfiguration": {
                    "modelConfiguration": {
                        "modelArn": "RERANKING_MODEL_ARN"
                    },
                    "numberOfRerankedResults": 5
                }
            }
        }
    }
)
```

Recheck the current API schema and selected reranker support before executing; service fields and model availability can evolve.

Compare the same queries:

| Metric | Hybrid baseline | Hybrid + rerank | Pass condition |
|---|---:|---:|---|
| Expected-source hit@5 |  |  | no regression |
| Mean reciprocal rank |  |  | improvement or justified parity |
| Duplicate rate |  |  | no increase |
| p95 latency |  |  | within budget |
| Cost/query |  |  | within budget |

Reranking cannot restore a relevant document that a filter or candidate cutoff removed.

## Step 11 — Test chunking and multi-part queries

For the shipping exception:

1. inspect the exact retrieved child/parent content;
2. compare fixed-size versus hierarchical behavior;
3. verify the narrow exception and its heading/context are both present;
4. record duplicate and irrelevant context.

For the multi-part query:

- baseline with one retrieval query;
- transform into bounded subqueries;
- retrieve per subquery;
- merge, deduplicate, rerank;
- cap subquery count, token budget, and elapsed time.

If using supported Knowledge Base query decomposition, record the orchestration configuration. If comparing a Managed Knowledge Base, use `AgenticRetrieveStream` for its documented multi-step retrieval path.

## Step 12 — Validate synchronization

Run these changes one at a time:

1. add a new synthetic document;
2. modify `returns-v2.md`;
3. delete `returns-v1.md`.

Before synchronization, prove the index remains unchanged. Start one ingestion job, poll terminal status, then prove:

- the new document is retrievable;
- the modified fact replaces the old indexed version;
- the deleted document is no longer returned;
- ingestion statistics reflect the intended change.

For an event-driven design, coalesce bursty S3 events before starting sync. One full ingestion job per object event can create waste and race conditions.

## Step 13 — Optional grounded generation

Only after retrieve-only thresholds pass:

1. pass the bounded top evidence into a governed prompt; or
2. use `RetrieveAndGenerate` for this supported customer-managed Knowledge Base configuration.

Require:

- a response for supported evidence;
- an explicit insufficient-evidence result for the no-answer query;
- source locations/citations;
- claim-level checks that each citation supports its statement.

Do not convert the retrieval score into a fabricated confidence percentage.

## Step 14 — Compare a Managed Knowledge Base

If available in the lab Region, create a small Managed Knowledge Base using the same synthetic source or document subset.

Compare:

| Dimension | Customer-managed lab KB | Managed KB |
|---|---|---|
| Datastore/index setup | You configure | Bedrock manages |
| Mapping/dimension control | Greater | Service-managed |
| Source connectors/ACL features | Configuration-dependent | Managed connector feature set |
| Direct retrieval | `Retrieve` | `Retrieve` |
| Combined/multi-step path | `RetrieveAndGenerate` where supported | `AgenticRetrieveStream` |
| Operational effort | Higher | Lower |

If testing ACL-aware retrieval, authenticate the test identity in the application and send only verified `userContext`. Missing context for an ACL-enabled source should not become unrestricted access. Do not use Web Crawler as the ACL test source.

## Failure injection

Inject only synthetic, reversible failures:

| Injection | Expected evidence | Correct diagnosis |
|---|---|---|
| Wrong dimension in disposable index | setup/ingestion/query validation error | embedding/index contract |
| Omit one metadata sidecar | result lacks intended filter field | metadata ingestion |
| Filter on wrong status value | zero or wrong eligible results | filter configuration |
| Search exact ID with semantic only | lower exact-match rank | retrieval strategy |
| Candidate count too small | reranker never sees expected source | candidate recall |
| Duplicate paragraphs with high overlap | duplicate top chunks | chunking/ingestion |
| Modify S3 file without sync | stale retrieval result | freshness/ingestion |
| Start sync but do not poll | false “success” state | orchestration |
| Delete source before successful sync | old result remains | index freshness |
| No-evidence query | empty/irrelevant candidates | require refusal, not hallucination |
| Managed ACL request with missing context | no eligible results | identity context, not open access |

For each failure, record:

```text
Observed symptom:
Pipeline stage:
Request/job ID:
Evidence:
Root cause:
Fix:
Regression test:
```

## Validation evidence

Create one sanitized lab report containing:

- date, Region, Knowledge Base type and IDs;
- source manifest and content hashes;
- sidecar schema and metadata policy;
- embedding model, dimension, and vector data type;
- vector-store/index mapping;
- chunking configurations;
- ingestion job IDs, terminal status, and statistics;
- benchmark version/hash;
- per-query semantic, hybrid, filter, and reranked results;
- hit@k, MRR, coverage, duplicate rate, and latency;
- failure-injection evidence and root cause;
- optional generation citations and unsupported-claim rate;
- IAM/KMS/network review;
- cost estimate/observed spend;
- cleanup proof.

Example summary:

```json
{
  "benchmarkVersion": "rag-lab-v1",
  "knowledgeBaseType": "customer-managed",
  "embeddingDimension": 512,
  "ingestionStatus": "COMPLETE",
  "semanticHitAt5": 0.75,
  "hybridHitAt5": 0.92,
  "rerankedMrr": 0.88,
  "staleSourceRate": 0.0,
  "cleanupComplete": true
}
```

The values above illustrate a shape, not universal pass thresholds. Set thresholds before running the benchmark.

## Security cautions

- Use synthetic data and a dedicated prefix/index.
- Authenticate before retrieval and compute allowed filters server-side.
- Never accept tenant, ACL, or `userContext` claims directly from an untrusted client.
- Use least-privilege roles for source, embeddings, vector store, reranker, and KMS.
- Restrict direct ingestion and synchronization to trusted principals.
- Treat indexed source text as untrusted prompt data.
- Do not log full chunks or query text if they may contain sensitive values.
- Verify deletion in source, index, caches, and retained evidence.
- Private networking reduces exposure but does not replace IAM/resource policies.

## Cost and reliability cautions

- OpenSearch Serverless can incur continuing capacity cost while idle; delete it promptly.
- Smaller embeddings and binary vectors can reduce storage only when supported and quality-tested.
- Semantic chunking, reranking, decomposition, and generation add separate paid steps.
- Large overlap multiplies vectors, ingestion cost, and duplicate context.
- Batch source events into controlled ingestion jobs.
- Bound retries, candidate count, subqueries, reranked results, and generation tokens.
- Monitor the whole path; an available FM does not help when the vector store or source sync is unavailable.

## Cleanup

Perform cleanup in dependency-safe order:

1. export the sanitized evidence required by your study plan;
2. delete test Knowledge Base data sources/Knowledge Bases as appropriate;
3. delete disposable and lab OpenSearch indexes/collection after confirming no shared use;
4. remove test S3 objects and sidecars, or let the approved lifecycle expire them;
5. delete test EventBridge rules, Lambdas, Step Functions, and schedules if created;
6. remove lab IAM roles/policies and KMS grants that are no longer needed;
7. remove temporary secrets and local credential material;
8. verify the billing console/cost view no longer shows an unintended active vector resource.

Do not delete a shared collection, bucket, role, or key. Resolve exact ownership first.

## Exam lessons

1. RAG updates knowledge; fine-tuning changes behavior.
2. Chunking, embeddings, metadata, search, and reranking solve different failures.
3. Hierarchical chunking matches precise children and returns broader parents.
4. Embedding dimension must match the index.
5. Hybrid search combines semantic and exact-term retrieval.
6. Reranking can reorder only retrieved candidates.
7. Metadata filtering improves eligibility/precision but does not replace authorization.
8. `StartIngestionJob` does not prove the index is current; inspect terminal status/statistics.
9. `Retrieve` isolates retrieval from generation.
10. Relevance scores are not factual confidence.
11. Managed and customer-managed Knowledge Bases have different ownership and API boundaries.
12. More shards, candidates, chunks, and subqueries are not automatically better.
13. Lowest operational overhead favors Managed Knowledge Bases only when their control/features satisfy the requirement.
14. A stable MCP/API tool must enforce scope and never expose raw vector-store credentials.

## Official sources

- [Knowledge Bases for Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)
- [Managed versus customer-managed Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-build-managed.html)
- [Build a customer-managed Knowledge Base](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-build.html)
- [Supported vector stores and setup](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-setup.html)
- [Chunking options](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-chunking.html)
- [S3 data-source connector and metadata](https://docs.aws.amazon.com/bedrock/latest/userguide/s3-data-source-connector.html)
- [Knowledge Base retrieval](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-how-retrieval.html)
- [Configure and test retrieval](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-config.html)
- [`Retrieve` API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_Retrieve.html)
- [`RetrieveAndGenerate` API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_RetrieveAndGenerate.html)
- [Synchronize a Managed Knowledge Base](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-sync.html)
- [Direct ingestion](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-direct-ingestion.html)
- [ACL-aware retrieval](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-retrieve-acl.html)
- [Titan Text Embeddings](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html)
- [OpenSearch shard strategy](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/bp-sharding.html)
- [Aurora PostgreSQL vector search](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.VectorDB.html)
