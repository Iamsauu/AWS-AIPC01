# Agents, Tools, MCP, and Amazon Bedrock AgentCore

Status: Verified  
Official tasks: 2.1, 2.5; supporting 3.1, 3.2, 4.3, and 5.1–5.2  
Last verified: 2026-07-23

## Why this matters

Agent questions test whether you can combine probabilistic reasoning with deterministic control. The high-scoring answer is rarely “give the model more autonomy.” It is usually a bounded agent with typed tools, explicit state, least privilege, failure controls, observable execution, and human approval for consequential actions.

## Core concepts

### Agent versus workflow versus chat

| System | Control source | Best use | Main risk |
|---|---|---|---|
| Chat with tools | Model chooses among a small tool set | Interactive lookup and simple action | Ambiguous tool selection or parameters |
| Agent loop | Model repeatedly plans, acts, and observes | Open-ended multi-step tasks | Loops, excess cost, unsafe actions |
| Deterministic workflow | Code or state machine defines allowed paths | Regulated, auditable, repeatable processes | Less flexible for novel requests |
| Hybrid | Model proposes; workflow validates and controls | Most production agentic use cases | More design work, but safer and testable |

Use the least autonomous design that meets the requirement. If the valid states and branches are known, a deterministic workflow is often better than an open-ended agent.

### The controlled agent loop

```text
goal + policy + authorized context
            ↓
      model selects next step
       ↙        ↓        ↘
 final answer  tool call  clarification
       ↓        ↓             ↓
   validate  authorize       user input
              validate         ↓
              execute          └── loop
              observe
              update state
              increment counter
                 ↓
          stop/risk/circuit check
                 └── loop or safe terminal state
```

Required production controls:

- maximum iterations and maximum tool calls;
- end-to-end and per-tool timeouts;
- token and cost budget;
- only approved tools;
- schema validation before execution;
- authorization based on the authenticated actor, not model output;
- idempotency for side effects;
- bounded retries for retryable failures;
- unsafe-action branch and human approval;
- circuit breaker for unhealthy dependencies;
- trace, metrics, and an explicit final status.

### Reasoning without exposing hidden chain of thought

The exam guide uses terms such as ReAct and chain-of-thought approaches. Interpret this as implementing structured problem decomposition and reason-act-observe control. Do not require or log provider-internal chain of thought.

Useful auditable evidence includes:

- selected action and concise reason code;
- tool name, sanitized arguments, result status, and latency;
- retrieved source identifiers;
- workflow state transitions;
- approval and policy decisions;
- final answer and evaluation result.

An Amazon Bedrock agent trace is orchestration evidence. It is not the model provider's private reasoning.

## How it works

### State, short-term memory, and long-term memory

These are related but not interchangeable:

| Data | Meaning | Suitable store |
|---|---|---|
| Workflow state | Current deterministic step, retry count, approval state, risk flag | Step Functions plus DynamoDB/S3 as needed |
| Application session | User/session binding, slots, recent turns, job state, TTL | DynamoDB |
| Runtime session context | Ephemeral process/filesystem/context for an AgentCore Runtime session | AgentCore Runtime isolated session |
| Short-term agent memory | Raw events and conversation continuity within a session | AgentCore Memory short-term memory |
| Long-term agent memory | Extracted preferences, summaries, semantic facts, episodes | AgentCore Memory strategies |
| Authoritative business record | Order, claim, payment, entitlement, consent | Source business system; never agent memory |

#### AgentCore Memory identity model

- `actorId` identifies the person, agent, or system whose memory is isolated.
- `sessionId` identifies one conversation or interaction stream.
- Use a different session ID for each browser tab/chat when their context must not mix.
- Reuse the actor ID across sessions when long-term preferences should carry forward.
- Store both user and assistant events needed by the configured strategy.
- If an integration buffers writes, explicitly close/flush the memory session so a short chat does not end before the batch threshold.

The application must authenticate the caller and map the caller to allowed actor/session identifiers. AgentCore Runtime does not make an arbitrary client-supplied session ID an authorization credential.

#### When DynamoDB is still correct

Use DynamoDB instead of, or alongside, AgentCore Memory when the question asks for:

- deterministic slots such as location, vehicle type, or service type;
- job status and result lookup;
- a fixed 30-day TTL history;
- a circuit-breaker record;
- approval status;
- custom conditional writes and access patterns;
- an auditable application system of record.

### Tool lifecycle

1. **Discover/select:** expose only currently authorized tools; descriptions must distinguish them.
2. **Propose:** the model returns a tool name and structured arguments.
3. **Validate:** enforce JSON schema and business rules.
4. **Authorize:** evaluate the authenticated actor, resource, action, and policy.
5. **Approve:** pause if the action is high impact.
6. **Execute:** use a least-privilege adapter and an idempotency key.
7. **Normalize:** return a stable result/error envelope.
8. **Observe:** log sanitized metadata, trace, latency, and dependency result.
9. **Continue or stop:** update state, budget, risk, and circuit status.

### Tool schema design

Example shape:

```json
{
  "name": "get_order_status",
  "description": "Read the current status for one order the authenticated user is authorized to view.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "orderId": {
        "type": "string",
        "pattern": "^[A-Z0-9-]{6,40}$"
      }
    },
    "required": ["orderId"],
    "additionalProperties": false
  }
}
```

A useful output envelope:

```json
{
  "ok": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "orderId is required",
    "retryable": false,
    "missingFields": ["orderId"]
  },
  "correlationId": "safe-correlation-id"
}
```

Return a structured, actionable error so the agent can ask for missing information or choose a valid fallback. Never return a stack trace, secret, raw dependency response, or internal authorization detail.

### Idempotency

For a side-effecting tool:

- derive the idempotency key from a stable business operation ID;
- store/check completion atomically;
- pass the key to an external API when it supports idempotency;
- distinguish “already completed” from “failed before commit”;
- do not blindly retry ambiguous timeouts without checking outcome;
- make compensation a defined business operation, not an FM improvisation.

## AWS services and APIs

### AWS Step Functions as the deterministic harness

Use a **Standard** workflow for long-lived and auditable agents.

| State/feature | Agent use |
|---|---|
| `Task` | Invoke model, Lambda tool, service API, or callback |
| `Choice` | Route on action, risk, iteration count, or approval result |
| `Parallel` | Run independent specialists/tools concurrently |
| `Wait` | Controlled delay; not a human callback by itself |
| `Succeed` | Validated terminal success |
| `Fail` | Unsafe, budget-exceeded, or nonrecoverable terminal state |
| `Retry` | Bounded transient-error retry |
| `Catch` | Fallback, compensation, or safe failure path |
| Callback task token | Pause for an external approval/result without polling |
| Execution history | Auditable path and diagnostic evidence |

#### Retry is not a circuit breaker

- **Retry:** repeats one failed task under a bounded policy.
- **Stop condition:** prevents the current loop from exceeding its budget.
- **Circuit breaker:** remembers a dependency is unhealthy and rejects/skips calls for a cooldown, often across workflow executions.
- **Fallback:** uses a safe alternative after defined failures.

A serverless circuit breaker can use a DynamoDB item with state, failure count, opened time, and TTL/cooldown. Check it before invoking the dependency. Permit a controlled probe before closing the circuit.

### Human-in-the-loop callback

Use a callback task token:

```text
draft → store pending record → notify reviewer
      → wait with task token
      → reviewer API validates identity and decision
      → SendTaskSuccess / SendTaskFailure
      → persist final edit/rating → continue
```

Security requirements:

- keep callback completion in the workflow account;
- never expose a task token in public logs or analytics;
- bind the approval to reviewer identity, object, action, and expiry;
- use one-time completion semantics;
- define timeout and rejection paths;
- persist the business status separately from the token.

### Amazon Bedrock Agents

Bedrock Agents can orchestrate a model, Knowledge Bases, and action groups. For transparency:

- call `InvokeAgent`;
- set `enableTrace` to `true` when trace evidence is required;
- process the final response from the response chunks;
- persist trace events separately;
- show citations/source attribution separately from orchestration trace.

Action groups backed by Lambda still require validation, authorization, idempotency, least privilege, and structured errors.

### Model Context Protocol (MCP)

MCP defines a portable client/server protocol for capabilities such as:

- `tools/list` and `tools/call`;
- prompts;
- resources;
- session-aware interactions in supported transports.

MCP is useful when several agent frameworks or clients should reuse one stable tool/retrieval contract. It hides backend changes—for example, a `vector_search` tool can route internally to OpenSearch or Aurora PostgreSQL.

MCP does **not** provide:

- IAM authorization by itself;
- tool safety or input validation;
- durable execution;
- transactional semantics;
- retries or idempotency;
- a hosting runtime;
- business approval.

### MCP hosting decision

| Need | Host/pattern | Why |
|---|---|---|
| Small stateless lookup | Lambda, often converted/exposed through AgentCore Gateway | Serverless and low operations |
| Native libraries or CPU-heavy document processing | ECS with Fargate | Container flexibility and longer execution |
| Stateful streamable-HTTP, progress, elicitation, sampling | AgentCore Runtime MCP server | Session-aware, agent-purpose-built runtime |
| Existing tool APIs should become MCP without rewriting | AgentCore Gateway with Lambda/OpenAPI/Smithy target | Managed protocol translation and governance |
| Many MCP servers should appear as one tool catalog | AgentCore Gateway MCP aggregation | Consolidated discovery and optional semantic tool search |

Current AgentCore Runtime MCP contract:

- bind to `0.0.0.0`;
- listen on port `8000`;
- expose `/mcp`;
- use streamable HTTP;
- stateless mode is preferred for basic tools;
- stateful mode supports capabilities requiring session continuity;
- retain and resend the MCP session ID as required.

Do not memorize this contract without rechecking before implementation; it is product detail that can change.

### Amazon Bedrock AgentCore

AgentCore is modular. Select only the services the requirement needs.

#### Runtime

Use Runtime for framework-agnostic, serverless agent or tool hosting with isolated sessions and streaming support. It supports agents built with frameworks such as Strands Agents and supports protocols including MCP.

Important distinctions:

- runtime compute/session state is ephemeral;
- long-term durable context belongs in AgentCore Memory or another store;
- each user/conversation needs a correctly managed runtime session ID;
- the application remains responsible for mapping users to sessions;
- use separate sessions to prevent cross-conversation contamination.

#### Gateway

Gateway provides a managed, secured entry point to tools and other agentic targets. It can:

- convert Lambda, OpenAPI, and Smithy definitions into MCP-compatible tools;
- aggregate MCP target capabilities;
- front other agents or HTTP endpoints;
- provide authentication/credential handling;
- list/call tools and expose prompts/resources where supported;
- apply centralized observability and control.

For a Lambda target:

- define each tool in `toolSchema`;
- validate the event against the relevant `inputSchema`;
- expect a target-qualified/tool-prefixed name where documented;
- normalize or strip the prefix before dispatching to an existing unprefixed handler;
- never dispatch an unvalidated arbitrary handler name.

For managed Knowledge Base targets, bind the Knowledge Base and retrieval settings in the target. Do not let every client pass an unrestricted Knowledge Base ID at runtime.

#### Memory

Use short-term raw events plus configured long-term strategies such as user preference, semantic memory, summarization, or episodic memory. Scope data correctly by actor and session.

#### Evaluations and observability

Evaluation asks whether the agent completes tasks correctly and uses tools effectively—not merely whether its final prose sounds fluent.

Useful measures:

- task completion/correctness;
- correct tool selection;
- argument validity;
- number of repeated tool calls;
- tool success and latency;
- number of model/tool cycles;
- human escalation rate;
- safety-policy compliance;
- token use, latency, and cost per completed task.

For caller-initiated evaluation, the current AgentCore Evaluations API can score supplied OpenTelemetry session spans with an evaluator and reference inputs. A mock scenario uses a built-in correctness evaluator and an expected response. Always verify current evaluator IDs and input schema in the API documentation.

### Strands Agents

Strands Agents is an open-source SDK for building model-driven agents with:

- custom function tools;
- MCP tools;
- structured output;
- conversation managers;
- hooks and OpenTelemetry observability;
- agents-as-tools;
- multi-agent graphs, swarms, and workflows.

Use:

- a **Graph** when dependencies and routes should be explicit and largely deterministic;
- a **Workflow** when agents/tasks follow known dependencies;
- a **Swarm** when collaboration/handoffs are more dynamic.

Bound graph steps, concurrency, node timeouts, and cyclic paths. Tool descriptions and schemas are still part of the safety boundary.

### AWS Agent Squad terminology warning

The AIP-C01 blueprint explicitly names **AWS Agent Squad** as an example multi-agent framework. The project began under AWS Labs, but its repository now says it moved to the `2FastLabs` organization. For the exam, know the blueprint concept: classify/route requests and coordinate specialized agents. For real implementation, verify current ownership, maintenance, security posture, and support expectations before adoption.

Do not confuse this open-source framework with the managed multi-agent collaboration feature of Amazon Bedrock Agents.

## Architecture patterns

### Pattern 1 — Auditable reason-act loop

```text
API Gateway
  → request validator / Lambda
  → Step Functions Standard
      → load state and circuit
      → Bedrock model chooses an allowed action
      → Choice
          → read-only Lambda tool
          → approval callback
          → safe final response
      → validate and persist result
```

Use for regulated decisions, strict stop conditions, several-day sessions, and reviewable execution history.

### Pattern 2 — Portable tools

```text
Strands or another MCP client
  → AgentCore Gateway
      ├─ Lambda target: small stateless tools
      ├─ OpenAPI/Smithy target: existing APIs
      └─ MCP target: containerized complex tools
```

Use when multiple clients need the same governed tool catalog.

### Pattern 3 — Specialized multi-agent system

```text
request
  → router/supervisor
      ├─ triage agent
      ├─ policy agent
      └─ runbook agent
  → deterministic aggregation and validation
  → final response
```

Each specialist should have a narrow role and tool allowlist. Limit delegation depth and total budget. Compare its success/cost/latency against a single-agent baseline.

### Pattern 4 — Memory-aware tutoring/support

```text
authenticated actor
  → new session per tab/chat
  → Strands agent on AgentCore Runtime
  → AgentCore Memory
      ├─ short-term: session turns
      └─ long-term: actor preferences
  → explicit close/flush at chat end
```

Do not store authoritative grades, medical facts, financial balances, or entitlements as learned preference memory.

## Decision table

| Requirement | Choose | Avoid |
|---|---|---|
| Known branches, audit history, long approval | Step Functions Standard hybrid | Unbounded FM-controlled loop |
| Simple managed Bedrock agent with KB/action groups | Bedrock Agents | Rebuilding orchestration without a requirement |
| Custom framework and runtime isolation | AgentCore Runtime | Treating Lambda local memory as durable |
| Stable portable tool contract | MCP | Provider-specific tool adapters in every client |
| Existing APIs converted to MCP | AgentCore Gateway | Rewriting every API as a custom MCP server |
| Custom slots/status/TTL | DynamoDB | Treating extracted memory as system of record |
| Preferences across sessions | AgentCore Memory | Appending every prior turn to every prompt |
| Two independent specialists | Parallel execution plus deterministic merge | Sequential calls when latency is the priority |
| Consequential write | Approval + idempotent tool adapter | Direct model-to-database credentials |
| Dependency repeatedly fails | Bounded retry + circuit breaker + fallback | Infinite retry or repeated model replanning |

## Security and governance

### Required controls

- Give each tool adapter its own least-privilege IAM role.
- Scope secrets to the exact secret ARN; do not put credentials in tool schemas or prompts.
- Authorize from authenticated identity and authoritative policy.
- Pass only the minimum context required by a tool.
- Treat tool output as untrusted input before putting it back into a model prompt.
- Validate URLs, paths, SQL, commands, and resource identifiers against allowlists.
- Apply Guardrails and deterministic checks where required.
- Separate read tools from write tools.
- Require human approval for high-impact actions.
- Store only sanitized trace fields; never raw secrets or regulated content.
- Encrypt state and memory, configure retention, and remove stale sessions.

### Prompt injection through tool output

Retrieved documents and tool results can contain instructions such as “ignore prior rules.” Mark them as data, not instructions. Keep system policy separate, restrict the tool catalog, validate tool calls after the model chooses them, and do not let retrieved text expand IAM permissions.

## Cost, latency, and reliability

### Cost drivers

- every model turn in the loop;
- repeated context and tool descriptions;
- tool errors that trigger replanning;
- multi-agent handoffs;
- parallel model calls;
- durable workflow state transitions;
- Runtime/session duration and memory operations;
- evaluation frequency.

Reduce cost by simplifying the loop, shortening tool descriptions without losing precision, summarizing history, limiting visible tools, using smaller models for routing, caching stable prompt prefixes where supported, and stopping early when evidence is sufficient.

### Reliability budget

If the external API must respond within 10 seconds, the sum of model, tool, retry, validation, and transport budgets must fit under 10 seconds. Do not give each nested component a 10-second timeout.

For a multi-step task, track:

- end-to-end deadline;
- remaining time;
- cycle and tool-call count;
- token/cost spend;
- dependency circuit state;
- current risk state.

### Safe retry table

| Failure | Retry? | Notes |
|---|---|---|
| Throttling/transient service error | Yes, bounded backoff with jitter | Honor service guidance and deadline |
| Tool validation error | No unchanged retry | Ask for missing/corrected input |
| Access denied | No | Fix policy or deny safely |
| Unsupported action | No | Route to safe alternative |
| Ambiguous timeout after side effect | Check idempotency/outcome first | Blind retry can duplicate the action |
| Dependency circuit open | No immediate call | Fail fast or use approved fallback |
| Model output schema invalid | Limited repair attempt | Then fail safely; measure rate |

## Failure modes and troubleshooting

| Symptom | Inspect | Probable cause | Remediation |
|---|---|---|---|
| Missing/malformed tool arguments | Tool schema, model trace, Lambda validation log | Weak description/schema | Required typed fields; examples/enums; structured validation response |
| Same tool called repeatedly | Trace, cycle count, error code | Error gives no actionable signal or no stop rule | Structured errors, clarification, maximum cycles |
| Cross-user context appears | actor/session mapping, runtime header | Session ID reused or trusted from client | Backend-owned mapping; unique per conversation |
| Short chat memory is lost | buffered events and session close | Buffer never flushed | Explicit close/flush and verify memory event |
| Runtime loses state after stop | runtime lifecycle, persistence design | Ephemeral compute treated as durable | AgentCore Memory, session storage, DynamoDB, or S3 as appropriate |
| Gateway invokes wrong handler | qualified tool name | Dispatcher expects unprefixed name | Validate and normalize documented target prefix |
| MCP progress/elicitation fails | transport and server mode | Stateless server used for stateful feature | Stateful streamable HTTP and session continuity |
| Tool duplicates external record | message retry and idempotency store | No stable idempotency key | Conditional record plus downstream idempotency key |
| Human approval never resumes | callback token/account/expiry | Token routed incorrectly or expired | Same-account callback endpoint, durable status, timeout path |
| Agent ignores safety branch | orchestration path and IAM | FM is sole controller | Deterministic Choice/Fail and scoped tool role |
| Circuit never closes | DynamoDB expiry/probe logic | Bad TTL or no half-open probe | Explicit cooldown and controlled probe |
| Multi-agent result is slower/worse | per-agent traces and benchmark | Coordination overhead outweighs specialization | Simplify or change routing/delegation |
| Trace leaks sensitive data | events and log fields | Raw prompt/tool output logging | Redact/minimize; store safe metadata only |

## Common exam traps

1. **“Autonomous” does not mean unbounded.** Maximum cycles and explicit terminal states remain required.
2. **MCP is not a compute service.** Select the hosting environment separately.
3. **AgentCore Runtime session state is not long-term memory.**
4. **AgentCore Memory is not an authoritative transaction database.**
5. **A tool schema improves structure but does not replace runtime validation.**
6. **The model must not choose its own IAM policy.**
7. **A retry counter does not protect every execution from an unhealthy shared dependency; use a circuit breaker.**
8. **A `Wait` state is not the same as a callback.**
9. **SNS notification does not capture reviewer approval state.**
10. **A trace is not citation evidence and is not hidden chain of thought.**
11. **Multi-agent collaboration is not automatically more accurate.**
12. **A Lambda function can be a lightweight tool, but CPU-heavy/native/long-running work often fits Fargate or AgentCore Runtime better.**
13. **Agent Squad in the exam guide is now a lifecycle-sensitive open-source reference; do not assume AWS-managed service support.**

## Local mock references

- **PE1-Q02:** Lambda for simple stateless MCP; ECS/Fargate for CPU-heavy native tool.
- **PE1-Q08:** Step Functions reason-act loop with Choice, retries, and execution history.
- **PE1-Q19:** human approval, edited output, and rating.
- **PE1-Q20:** stable MCP retrieval interface over OpenSearch/Aurora.
- **PE1-Q22:** agent trace and source attribution.
- **PE1-Q48:** Strands typed tool plus Lambda validation and structured error.
- **PE1-Q50:** max failures, DynamoDB circuit breaker, timeout, and least privilege.
- **PE1-Q53:** Agent Squad routing, Strands specialists, AgentCore Runtime and Memory.
- **PE1-Q66:** agent evaluation and repeated tool-call analysis.
- **PE1-Q67:** adversarial agent tests and runtime defense in depth.
- **PE2-Q02:** stateful AgentCore Runtime MCP contract.
- **PE2-Q11:** `InvokeAgent` trace separated from final response.
- **PE2-Q34, Q40:** callback approval and several-day auditable workflow.
- **PE2-Q41:** synchronous AgentCore correctness evaluation from OpenTelemetry spans.
- **PE2-Q48:** Gateway Lambda `toolSchema` and target-qualified tool name.
- **PE2-Q60:** bounded cycles, risk `Fail`, circuit breaker, and per-tool IAM.
- **PE2-Q65:** actor/session scoping and explicit memory-session close.
- **PE2-Q70:** Gateway managed Knowledge Base connector with bound retrieval settings.
- **PE2-Q71:** DynamoDB application session and deterministic clarification workflow.

## Hands-on validation

1. Define a read tool and a write tool with strict schemas and structured errors.
2. Write an ASL design containing a model Task, tool Task, `Choice`, `Retry`, `Catch`, max-cycle `Fail`, and callback approval.
3. Create a circuit-breaker item design with atomic failure updates, cooldown, and half-open probe.
4. Demonstrate how authenticated actor, AgentCore `actorId`, Runtime session ID, and application session ID relate without treating any one as authorization by itself.
5. Design one MCP server contract and explain deployment on Lambda/Gateway, Fargate, and AgentCore Runtime.
6. Define an agent evaluation dataset with expected final result, required/forbidden tools, maximum cycles, and safety outcome.

## Recall questions

1. What six controls bound an agent loop?
2. Which agent data belongs in DynamoDB rather than AgentCore Memory?
3. How should two browser tabs for one learner map to actor and session IDs?
4. Why must tool output be treated as untrusted prompt content?
5. What changes make a side-effecting tool safe to retry?
6. How do `Retry`, `Catch`, stop conditions, and circuit breakers differ?
7. Why is a callback task token appropriate for five-day human review?
8. What does MCP standardize?
9. Which requirements imply a stateful rather than stateless MCP server?
10. When is Fargate a better MCP host than Lambda?
11. What does AgentCore Gateway add to existing Lambda/OpenAPI tools?
12. How should a dispatcher handle Gateway-qualified tool names?
13. What is the distinction between AgentCore Runtime sessions and AgentCore Memory?
14. What evidence can an agent trace provide without exposing hidden chain of thought?
15. When should a graph, workflow, or swarm be used in Strands?
16. Why can a multi-agent system be worse than one agent?
17. Which metrics reveal a broken agent loop?
18. What is the current caveat around the “AWS Agent Squad” name?

## Official sources

- [AIP-C01 Domain 2](https://docs.aws.amazon.com/aws-certification/latest/ai-professional-01/ai-professional-01-domain2.html)
- [Amazon Bedrock AgentCore developer guide](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/)
- [Host agents or tools with AgentCore Runtime](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agents-tools-runtime.html)
- [AgentCore Runtime MCP servers](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-mcp.html)
- [AgentCore Runtime isolated sessions](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-sessions.html)
- [AgentCore Gateway](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway.html)
- [Use an AgentCore Gateway](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-using.html)
- [AgentCore Memory API session model](https://docs.aws.amazon.com/bedrock-agentcore/latest/APIReference/API_ListSessions.html)
- [AgentCore evaluation input spans](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/understanding-input-spans.html)
- [Strands Agents tools](https://strandsagents.com/docs/user-guide/concepts/tools/)
- [Strands multi-agent patterns](https://strandsagents.com/docs/user-guide/concepts/multi-agent/multi-agent-patterns/)
- [Strands graph pattern](https://strandsagents.com/docs/user-guide/concepts/multi-agent/graph/)
- [Step Functions overview and human-in-the-loop pattern](https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html)
- [Step Functions error handling](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-error-handling.html)
- [Step Functions callback task-token example](https://docs.aws.amazon.com/step-functions/latest/dg/callback-task-sample-sqs.html)
- [Agent Squad current repository and lifecycle notice](https://github.com/2FastLabs/agent-squad)
- [Amazon Bedrock multi-agent collaboration](https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-announces-general-availability-of-multi-agent-collaboration/)
