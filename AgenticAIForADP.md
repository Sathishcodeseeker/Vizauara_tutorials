# SSE, LangGraph, and Production Security for an AIP Chatbot

This document is a standalone reference for the concepts discussed after introducing Server-Sent Events (SSE) in the AIP/ADP agent architecture. It explains Python asynchronous execution, browser streaming, LangGraph events, distributed messaging, production debugging, and sensitive-data protection.

The examples assume Python 3.11+, FastAPI, LangGraph, a browser chatbot, and governed AIP/ADP data sources.

## 1. The Core Mental Model

For a normal chatbot response:

```text
LLM
 │ produces text chunks
 ▼
FastAPI
 │ sends SSE events
 ▼
Browser chatbot
```

Four different responsibilities are involved:

| Component | Responsibility |
|---|---|
| LLM streaming API | Produces incremental text deltas |
| `asyncio` / `async` and `await` | Lets Python perform non-blocking concurrent I/O |
| SSE | Carries server events to the browser over HTTP |
| Browser code | Renders statuses, sources, tokens, errors, and completion |

The streamed unit is normally a text delta. It is not guaranteed to be exactly one word or one model token.

> `asyncio` controls non-blocking execution; SSE is the delivery channel to the browser.

## 2. `asyncio` Is Not Redis Streams

They operate at different layers:

| `asyncio` | Redis Streams |
|---|---|
| Runs concurrent I/O inside a Python process | Moves and stores events between processes or services |
| Uses coroutines, tasks, `async`, and `await` | Uses an append-only stream, IDs, reads, and consumer groups |
| Does not persist application events | Can retain events for later reading or replay |
| Required for efficient asynchronous FastAPI code | Optional infrastructure for distributed architectures |

Redis Streams cannot replace `asyncio`. A service reading Redis asynchronously still uses an asynchronous Redis client and Python's async execution model.

### Direct chatbot path

Use this for ordinary AIP questions:

```text
Browser → FastAPI → AIP retrieval/MCP → LLM
Browser ←────── SSE answer stream ─────┘
```

Examples:

- “What does AIRAC mean?”
- “Show the runway information for VOBL.”
- “Summarize the cited changes on this amendment page.”

Redis Streams is not required.

### Distributed-worker path

Redis Streams can be introduced when the LLM or graph runs in another service and replayable events are required:

```text
LLM/Agent Worker
      │ XADD events
      ▼
Redis Stream
      │ async read
      ▼
FastAPI
      │ SSE
      ▼
Browser
```

Even here, both the worker and FastAPI still use asynchronous I/O.

Writing every model token as a separate Redis entry usually creates unnecessary network traffic, storage, retention, and latency. If this design is genuinely required, send bounded text chunks or meaningful progress events and trim/expire the stream.

Consumer groups distribute entries among consumers in the same group. They do not automatically broadcast every entry to every browser. Design consumer IDs and replay behavior carefully.

## 3. Recommended AIP Chatbot Usage

```text
Normal chatbot answer       → asyncio + SSE
LLM/search/MCP network I/O  → asyncio
Session and hot-data cache  → Redis key/value cache
Live job-progress cache     → Redis cache, if needed
Durable long-running job    → Azure Service Bus
Enterprise domain events    → Kafka or Azure Event Hubs
Cross-agent collaboration   → A2A
Distributed replayable feed → Redis Streams, only if justified
```

For the initial AIP chatbot, use:

```text
Browser → FastAPI → LangGraph → retrieval/MCP/LLM
Browser ←──────────── SSE ────────────────┘
```

Add a durable broker for complete-amendment analysis or other work that can run for minutes, requires retries, or must survive a service restart.

## 4. What “SSE Is One-Way” Means

One-way applies to the individual SSE connection, not to the entire browser.

```text
SSE connection:  Server ─────────► Browser
```

The browser cannot send messages back through that open SSE response. It can open other HTTP requests at the same time:

```text
SSE:     Server ─────────► Browser   Stream events
POST:    Browser ────────► Server    Submit question
GET:     Browser ────────► Server    Load sources/history
DELETE:  Browser ────────► Server    Cancel a run
```

FastAPI can serve those requests concurrently when the handlers and libraries are properly asynchronous. Avoid blocking calls such as `time.sleep()`, synchronous network clients, or CPU-heavy document processing inside an async endpoint.

### Two common API patterns

#### Pattern A: one streaming POST

```text
POST /chat/stream
Request body: question
Response body: streamed SSE-formatted events
```

The browser normally consumes this with the Fetch API because native `EventSource` does not submit a POST body.

#### Pattern B: create a run, then subscribe

```text
POST   /runs                    → returns run_id
GET    /runs/{run_id}/events   → opens SSE connection
DELETE /runs/{run_id}          → cancels generation
```

This pattern makes cancellation, reconnection, authorization, and run tracking explicit.

For one conversation, either queue new questions or cancel the active run before starting another. Unrelated browser requests can continue without cancelling SSE.

## 5. SSE Event Design

SSE does not search documents or run the LLM. It only transports events produced by the application.

Use structured event types:

| Event | Example purpose |
|---|---|
| `run_started` | The agent run was accepted |
| `status` | A safe user-facing progress update |
| `source` | A cited document/page was selected |
| `tool_started` | An approved tool invocation began |
| `tool_completed` | A tool finished successfully |
| `token` | A final-answer text delta |
| `approval_required` | Human confirmation is required |
| `error` | A sanitized error occurred |
| `done` | The run completed |

Example wire format:

```text
event: status
data: {"stage":"retrieval","message":"Searching AIP documents"}

event: source
data: {"document":"AIP AMDT 08/2026","page":42}

event: token
data: {"text":"The amendment contains"}

event: token
data: {"text":" two candidate runway changes."}

event: done
data: {"status":"completed"}
```

The frontend switches on the event name instead of parsing ordinary answer text to guess what the agent is doing.

Do not stream private chain-of-thought, raw prompts, credentials, authorization tokens, or complete graph state.

## 6. How LangGraph Connects to SSE

SSE is not embedded in a LangGraph node. The layers are:

```text
LangGraph node
     │ emits a graph/custom event
     ▼
LangGraph runtime stream
     │ consumed by
     ▼
FastAPI adapter or LangGraph Agent Server
     │ serialized as SSE
     ▼
Browser
```

### LangGraph streaming modes

Important modes include:

| Mode | Meaning |
|---|---|
| `updates` | State updates after graph steps |
| `messages` | LLM message/token chunks plus metadata |
| `custom` | Application-defined progress emitted by nodes/tools |
| `tasks` | Task start/finish and related runtime events |

Do not expose these modes directly to users without filtering. State and debug events can contain sensitive internal data.

### Emit a safe event inside a node

```python
from langgraph.config import get_stream_writer


async def search_aip_node(state: dict) -> dict:
    writer = get_stream_writer()

    writer({
        "event": "status",
        "stage": "retrieval",
        "message": "Searching authorized AIP documents",
    })

    documents = await search_aip(state["question"])

    writer({
        "event": "sources_found",
        "count": len(documents),
    })

    return {"documents": documents}
```

The node reports domain progress. It does not import FastAPI, know about HTTP, or format SSE frames.

### Translate LangGraph events in FastAPI

```python
import json

from fastapi import FastAPI
from fastapi.sse import EventSourceResponse, ServerSentEvent

app = FastAPI()


@app.post("/chat/stream", response_class=EventSourceResponse)
async def stream_chat(request: ChatRequest):
    async for part in graph.astream(
        {"question": request.question},
        stream_mode=["custom", "messages"],
        version="v2",
    ):
        if part["type"] == "custom":
            payload = part["data"]
            yield ServerSentEvent(
                event=payload["event"],
                data=json.dumps(payload),
            )

        elif part["type"] == "messages":
            message_chunk, metadata = part["data"]
            if message_chunk.content:
                yield ServerSentEvent(
                    event="token",
                    data=json.dumps({"text": message_chunk.content}),
                )

    yield ServerSentEvent(event="done", data='{"status":"completed"}')
```

In production, introduce an explicit mapper/allowlist between LangGraph and SSE. Do not serialize arbitrary `part["data"]` from every stream mode.

## 7. Does FastAPI Have to Be Written Explicitly?

There are two choices.

### Self-managed application

```text
Node → LangGraph stream → Your FastAPI endpoint → SSE → Browser
```

You implement the adapter. This is often suitable for an enterprise ADP system because the existing backend can enforce Entra ID authentication, ADP authorization, audit policy, rate limits, data filtering, and a stable frontend contract.

### LangGraph Agent Server

```text
Node → LangGraph Agent Server → streaming API/SSE → Browser or SDK
```

Agent Server supplies run and streaming endpoints. During local development, `langgraph dev` starts an in-memory Agent Server. Managed or standalone deployment options can provide the production server.

Therefore:

> A node emits events, but a web server still exposes them. FastAPI code is optional only when another server layer, such as LangGraph Agent Server, provides that responsibility.

## 8. AIP Agent Execution Example

User request:

> Compare the VOBL runway information in AIP Amendment 08/2026 with the effective ADP dataset.

Possible graph:

```text
classify_request
       │
       ▼
retrieve_aip_sources
       │ custom: searching authorized documents
       │ custom: sources found
       ▼
get_effective_aerodata
       │ custom: reading effective dataset
       ▼
validate_candidates
       │ custom: running deterministic validation
       ▼
generate_answer
       │ messages: final-answer text chunks
       ▼
verify_citations
       │
       ▼
complete
```

FastAPI converts only approved events:

```text
custom status       → SSE status
safe source record  → SSE source
LLM message chunk   → SSE token
sanitized exception → SSE error
graph completion    → SSE done
```

The graph nodes perform retrieval, tool orchestration, and validation. SSE merely communicates progress and results.

## 9. Agent Latency Design

Agent latency is not automatically acceptable. An agent is normally slower than an ordinary API or a simple RAG request because each planning step, retrieval operation, tool invocation, validation, retry, and model call can add another network round trip.

```text
Total completion latency ≈
    classification
  + document retrieval
  + model planning
  + sequential tool calls
  + deterministic validation
  + final model generation
```

For example, the elapsed time accumulates when every operation is sequential:

```text
Search documents: 2 seconds
Database lookup:   1 second
Validation:        3 seconds
Model calls:       5 seconds
Total:            11+ seconds
```

These numbers are illustrative. Production targets must be established and measured against the selected models, infrastructure, datasets, and network.

### Perceived latency versus completion latency

**Perceived latency** is how long the user waits before seeing useful feedback:

```text
Request accepted
Searching documents...
Validating candidate changes...
```

SSE improves perceived responsiveness by exposing safe progress and partial output. It does not reduce the underlying execution time.

**Completion latency** is the time required for the entire workflow to finish. Redis and SSE cannot turn a several-minute amendment analysis into a fast computation; they only communicate its state.

### Route by request complexity

| AIP request | Execution path |
|---|---|
| “What does AIRAC mean?” | Direct RAG without an agent loop |
| “Find VOBL runway information” | RAG plus, at most, a focused lookup tool |
| “Compare this amendment with effective data” | Bounded agent workflow |
| “Analyze the complete amendment” | Durable background job through Service Bus |
| “Publish approved changes” | Deterministic workflow with explicit human approval |

The fastest agent step is the unnecessary step that is not executed. Do not send every question through the most complex graph.

### Control latency deliberately

- Classify requests and route simple questions around the agent loop.
- Ingest and index AIP documents before query time.
- Limit graph iterations, model calls, and tool calls.
- Use async database, HTTP, MCP, search, and model clients.
- Run independent nodes concurrently.
- Cache only versioned, permission-safe, non-sensitive results.
- Use smaller or faster models for narrow routing tasks when evaluation proves them sufficient.
- Apply per-node and whole-run timeouts with bounded retries.
- Use deterministic code instead of another model call for rules and validation.
- Stream safe progress immediately.
- Move work that exceeds the interactive latency budget to a durable background workflow.

Independent operations can often use fan-out/fan-in execution:

```text
Sequential
──────────
Search AIP → Read database → Find dependencies → Combine

Parallel
────────
              ┌─ Search AIP ──────────┐
Start ────────┼─ Read database ───────┼─ Combine
              └─ Find dependencies ───┘
```

Parallel execution reduces elapsed time from approximately the sum of independent operations toward the duration of the slowest branch. LangGraph supports parallel graph branches, but only independent operations should run concurrently.

### Measure instead of assuming

Track latency separately for each request class:

```text
request acceptance
time to first status
retrieval duration
each tool duration
time to first answer token
total run duration
queue waiting time
```

Monitor percentiles such as p50, p95, and p99 rather than relying on averages. Use trace IDs and sanitized timing data so performance can be diagnosed without exposing prompts or retrieved content.

> Agent latency is acceptable only when the workflow is classified correctly, bounded, observable, and moved to background processing when necessary. In ADP, correctness is more important than raw speed, but the Workbench must provide immediate and truthful progress feedback.

## 10. PII and Confidential-Data Risk

Production observability can become a data-leak path:

```text
User prompt
   ↓
LangGraph state
   ↓
Retrieved documents and tool results
   ↓
LLM input/output
   ↓
Trace or application log
   ↓
Support engineer
```

Potentially sensitive fields include:

- Names, email addresses, employee IDs, and session identifiers.
- User questions and full conversation history.
- Retrieved document text.
- Tool arguments and results.
- Graph state and checkpoints.
- Model responses and generated artifacts.
- Proprietary aeronautical-production information that is confidential even when it is not PII.

### Safe support model

| Access level | Visible information | Typical audience |
|---|---|---|
| Operational | Trace ID, node/tool name, status, timing, error code | Production support |
| Redacted | Masked inputs/outputs and selected source identifiers | Authorized AI support |
| Raw | Original sensitive evidence | Exceptional break-glass investigators |

Raw access should be exceptional, approved, time-limited, justified, and audited.

Support should begin with data such as:

```json
{
  "trace_id": "abc-123",
  "node": "retrieve_aip_sources",
  "status": "failed",
  "error_code": "SEARCH_TIMEOUT",
  "document_ids": ["AMDT-08-2026"],
  "duration_ms": 4200
}
```

Do not log raw prompts and documents by default.

## 11. Protecting SSE Output

After sensitive text reaches the browser, it cannot be recalled.

```text
LLM output → output policy/PII check → SSE → Browser
```

For low-risk text, an incremental filter may inspect bounded chunks before sending them. For high-risk workflows, buffer the complete answer, validate it, and only then return it. This sacrifices the token-by-token experience but gives stronger disclosure protection.

Never display raw internal exceptions through SSE. Return a stable error code and trace ID:

```text
event: error
data: {"code":"ADP_SEARCH_FAILED","trace_id":"abc-123"}
```

## 12. LangSmith Sensitive-Data Configuration

### Safest production baseline

Configure these values in the application runtime environment and restart/redeploy the application:

```env
LANGSMITH_TRACING=true
LANGSMITH_PROJECT=adp-production-sanitized
LANGSMITH_HIDE_INPUTS=true
LANGSMITH_HIDE_OUTPUTS=true
LANGSMITH_HIDE_METADATA=true
```

This preserves run structure and operational signals while hiding trace payloads and metadata.

### Disable tracing for a sensitive operation

```python
import langsmith as ls


with ls.tracing_context(enabled=False):
    result = await graph.ainvoke(sensitive_input)
```

Use this for zero-retention users, restricted documents, PII-heavy requests, or operations whose content must never be traced.

### Selectively anonymize inputs and outputs

Use this only when the organization has approved storing redacted content:

```python
from langsmith import Client
from langsmith.anonymizer import create_anonymizer


anonymizer = create_anonymizer([
    {
        "pattern": r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}",
        "replace": "<EMAIL>",
    },
    {
        "pattern": r"\b\d{10}\b",
        "replace": "<PHONE_NUMBER>",
    },
])

safe_client = Client(
    anonymizer=anonymizer,
    hide_metadata=True,
)
```

Apply the client to the graph execution:

```python
import langsmith as ls


with ls.tracing_context(client=safe_client):
    result = await graph.ainvoke(input_data)
```

Complete hiding and selective anonymization are alternative policies. If inputs and outputs are completely hidden, there is no payload for the anonymizer to retain in redacted form.

Regex masking is not sufficient for every type of PII. For broader detection, consider an approved PII detection system such as Microsoft Presidio and validate its recall against organization-specific data.

### Important scope limitation

LangSmith settings protect LangSmith traces. They do not automatically sanitize:

- FastAPI or Uvicorn logs.
- Azure Application Insights or OpenTelemetry exports.
- Container/platform logs.
- Database audit tables.
- Service Bus dead-letter messages.
- Redis values or streams.
- Browser telemetry and analytics.

Each sink requires its own allowlist, redaction, retention, and access policy.

## 13. Verification Checklist

Before enabling production traffic:

- [ ] Send synthetic email, phone, employee ID, and confidential-marker values.
- [ ] Inspect parent and child LangGraph/LangSmith runs.
- [ ] Inspect tool inputs, outputs, errors, metadata, and checkpoints.
- [ ] Inspect FastAPI, platform, Application Insights, and broker logs.
- [ ] Verify the browser receives only allowlisted SSE event fields.
- [ ] Verify exceptions are sanitized.
- [ ] Verify support roles cannot view raw content.
- [ ] Exercise and audit the break-glass process.
- [ ] Confirm retention and deletion policies.
- [ ] Test cancellation and browser disconnect cleanup.
- [ ] Test that high-risk outputs are validated before disclosure.
- [ ] Define interactive and background latency budgets by request type.
- [ ] Record time to first status, time to first token, and total duration.
- [ ] Verify graph iteration, retry, timeout, and tool-call limits.
- [ ] Confirm independent retrieval and analysis nodes run concurrently where safe.

## 14. Interview Summary

> For the AIP chatbot, I use asynchronous Python for non-blocking model, retrieval, MCP, and network calls. FastAPI exposes an SSE channel that carries allowlisted progress, citation, token, error, and completion events to the browser. LangGraph nodes emit domain progress through custom events, while the runtime automatically exposes node updates and model-message chunks. FastAPI—or LangGraph Agent Server—translates those events into an HTTP stream. I route simple questions through direct RAG, bound interactive agent workflows, execute independent nodes concurrently, and move complete-amendment processing to durable background jobs. Redis Streams is not a substitute for `asyncio` or SSE; it is optional infrastructure for replayable events between distributed services. In production, raw graph state is never sent to the browser or general support tooling. LangSmith inputs, outputs, and metadata are hidden or anonymized, sensitive executions can disable tracing, and all other telemetry sinks receive the same data-minimization treatment.

## 15. References and Learning Resources

- [Python `asyncio` documentation](https://docs.python.org/3/library/asyncio.html)
- [FastAPI concurrency and `async`/`await`](https://fastapi.tiangolo.com/async/)
- [FastAPI Server-Sent Events](https://fastapi.tiangolo.com/tutorial/server-sent-events/)
- [MDN: Using Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [Redis Streams documentation](https://redis.io/docs/latest/develop/data-types/streams/)
- [LangGraph streaming](https://docs.langchain.com/oss/python/langgraph/streaming)
- [LangGraph Graph API and parallel execution](https://docs.langchain.com/oss/python/langgraph/use-graph-api)
- [LangGraph local Agent Server](https://docs.langchain.com/oss/python/langgraph/local-server)
- [LangGraph Agent Server architecture](https://docs.langchain.com/langsmith/agent-server)
- [LangSmith: Prevent logging sensitive data](https://docs.langchain.com/langsmith/mask-inputs-outputs)
- [LangSmith conditional tracing](https://docs.langchain.com/langsmith/conditional-tracing)
- [Video: Server-Sent Events with Next.js and FastAPI](https://www.youtube.com/watch?v=Yfj3jfKL_AQ)
- [Video: LLM-style streaming with FastAPI and SSE](https://www.youtube.com/watch?v=hOAAg1WaZh8)
- [Video: SSE in FastAPI for real-time notifications](https://www.youtube.com/watch?v=tOyUQuZ6bhg)
