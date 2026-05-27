---
week: W01
day: Thu
topic_slug: streaming-responses
topic_title: "Streaming responses — InvokeModelWithResponseStream + FastAPI SSE"
parent_overview: W01/pre-session/4-Thursday/1-DailyTopicOverview.md
estimated_minutes: 8
sources:
  - url: https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModelWithResponseStream.html
    retrieved_on: 2026-05-22
    recency_category: hot-tech
  - url: https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://fastapi.tiangolo.com/advanced/custom-response/#streamingresponse
    retrieved_on: 2026-05-22
    recency_category: foundation-stable
  - url: https://github.com/KarsunFDE/content/blob/main/research/bedrock-claude-catalog-20260522.md
    retrieved_on: 2026-05-22
    recency_category: hot-tech
last_verified: 2026-05-27
---

# Streaming responses — InvokeModelWithResponseStream + FastAPI SSE

> The overview names streaming as a load-bearing UX requirement. This reading is the depth: why streaming matters for any user-facing LLM endpoint, the Bedrock `InvokeModelWithResponseStream` chunk shape, the FastAPI `StreamingResponse` + SSE pattern, the Angular consumer side, and the structured-outputs / streaming composition pitfall that Friday's PR has to navigate.

## 1. Learning Objectives

By the end of this reading, the learner can:

- Articulate why streaming is **non-negotiable** for any user-facing LLM endpoint and which UX failure modes it prevents.
- Write a correctly-shaped Bedrock `InvokeModelWithResponseStream` call and parse the chunk-event payload.
- Wire a FastAPI `StreamingResponse` that forwards Bedrock chunks as SSE `data:` frames to the Angular client.
- Explain why **streaming and structured outputs do not compose cleanly** and which W1 simplification this forces.

## 2. Introduction

A 30-second blocking POST to Bedrock is a 30-second hung Angular UI. Users walk away. The fix is streaming: Bedrock emits chunks as the model generates; the Python service forwards each chunk to the browser; the browser renders text as it arrives. The pattern is well-established for chat UIs and is the user-experience floor for production LLM endpoints.

The discipline this reading installs: Bedrock-side streaming, FastAPI-side SSE forwarding, Angular-side `EventSource` consumption, plus one specific composition pitfall the cohort hits Friday when structured outputs enter the picture.

## 3. Core Concepts

### 3.1 Why streaming is non-negotiable for user-facing LLM endpoints

Three UX failure modes that streaming prevents:

- **Perceived hang.** Without streaming, a 10-30s LLM response looks like a frozen UI. Users click reload, file support tickets, or walk away. Time-to-first-token (TTFT) is the real UX metric — Sonnet 4.5/4.6 typical TTFT is 0.5-2s; without streaming the user waits 10-30s for the full response before seeing anything.
- **Cancellation cost.** A non-streaming endpoint pays for the entire response even if the user cancels at second 5. A streaming endpoint can be aborted server-side mid-generation, saving tokens.
- **Progressive UX patterns.** Streaming enables "stop generation" buttons, incremental syntax highlighting, partial output review. These patterns are baseline expectations for modern AI UI.

For internal-only batch endpoints (W2 RAG indexing, W5 nightly evals), streaming is unnecessary. For any endpoint a human will wait on, streaming is required.

### 3.2 Bedrock InvokeModelWithResponseStream chunk shape

The Bedrock streaming API [1] returns an event stream where each chunk arrives as a separate event with a `bytes` payload:

```python
import json, boto3

bedrock = boto3.client("bedrock-runtime", region_name=os.environ["AWS_REGION"])

resp = bedrock.invoke_model_with_response_stream(
    modelId=MODEL_ID,
    body=json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "system": SYSTEM_PROMPT,
        "messages": [{"role": "user", "content": user_request}],
        "max_tokens": 1024,
    }),
)
for event in resp["body"]:
    chunk = json.loads(event["chunk"]["bytes"])
    # Anthropic chunk types: message_start, content_block_start,
    # content_block_delta, content_block_stop, message_delta, message_stop
    if chunk["type"] == "content_block_delta":
        delta_text = chunk["delta"]["text"]
        yield delta_text
    elif chunk["type"] == "message_stop":
        # Usage info arrives here
        usage = chunk.get("amazon-bedrock-invocationMetrics", {})
```

**Critical detail.** Usage info (input + output tokens, stop reason) arrives in the final `message_stop` or `message_delta` event — not in the headers. The cost-telemetry pattern from `8-cost-structure-of-llms.md` has to consume from the stream's end, not from the response object.

### 3.3 FastAPI StreamingResponse + SSE

FastAPI's `StreamingResponse` [3] takes a generator and emits a chunked HTTP response. SSE (Server-Sent Events) is a thin convention on top: each frame is `data: <payload>\n\n`, with optional `event:` and `id:` headers.

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.post("/draft-solicitation/stream")
async def draft_stream(request: DraftRequest):
    def event_generator():
        total_input_tokens = 0
        total_output_tokens = 0
        for delta in bedrock_stream(request.user_request):
            if isinstance(delta, str):
                yield f"data: {json.dumps({'text': delta})}\n\n"
            else:                                          # usage event from §3.2
                total_input_tokens = delta["input_tokens"]
                total_output_tokens = delta["output_tokens"]
        log_usage(
            feature="draft-solicitation", tenant=request.tenant,
            model=MODEL_ID,
            input_tokens=total_input_tokens,
            output_tokens=total_output_tokens,
        )
        yield "data: [DONE]\n\n"
    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

The `media_type="text/event-stream"` header is what makes browsers consume this as SSE. The `[DONE]` terminator is the convention OpenAI established and most clients (including Anthropic SDKs and Angular `EventSource` consumers) expect.

### 3.4 Angular EventSource consumer

On the Angular side, the consumer is a few lines (in concept — the actual training-project Angular code lands in today's PR):

```typescript
const es = new EventSource('/api/draft-solicitation/stream', { /* ... */ });
es.onmessage = (evt) => {
  if (evt.data === '[DONE]') { es.close(); return; }
  const chunk = JSON.parse(evt.data);
  this.draftText += chunk.text;     // incremental render
};
es.onerror = (err) => {
  es.close();
  this.handleStreamError(err);
};
```

**Known constraint:** `EventSource` does not support POST natively — the request is GET-only. For POST-shaped streaming endpoints, the Angular side uses `fetch` with a streaming `ReadableStream` consumer. The training-project's `acquire-gov` frontend handles this via a small wrapper; the wrapper exists today as scaffolding, the cohort wires it.

### 3.5 The streaming + structured outputs composition pitfall

Friday's structured-output PR forces a decision. Streaming yields text deltas as they generate; structured outputs constrain the full response to a schema. The two compose awkwardly:

- **Streaming-with-partial-JSON.** The Python service buffers the streamed deltas, attempts incremental JSON parsing, surfaces valid fields as they complete. Complex; partial-JSON parsing is its own engineering surface. Friday's PR does **not** do this.
- **Non-streaming for the structured path.** Endpoints that produce structured outputs are non-streaming; endpoints that produce free-text are streaming. Simple; matches the UX expectation (you wait briefly for "the form to fill in", not for "the conversation to flow"). **Friday's PR uses this.**

The cohort revisits structured-output streaming in W3 when agentic patterns make it relevant. W1 stays simple — streaming for free-text, non-streaming for structured.

## 4. Generic Implementation

The full FastAPI streaming endpoint, structured-output and retry kept separate per their own files:

```python
@app.post("/draft-solicitation/stream")
async def draft_stream(request: DraftRequest):
    def event_generator():
        try:
            for delta_or_usage in bedrock_stream(request.user_request):
                if isinstance(delta_or_usage, str):
                    yield f"data: {json.dumps({'text': delta_or_usage})}\n\n"
                else:
                    log_usage(**delta_or_usage)
            yield "data: [DONE]\n\n"
        except APIStatusError as e:
            yield f"data: {json.dumps({'error': str(e)})}\n\n"
            yield "data: [DONE]\n\n"
    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

Retry-with-jitter wraps `bedrock_stream` per `7-retry-logic-and-strategies.md`; cost telemetry happens on usage events per `8-cost-structure-of-llms.md`; structured-output handling is on a separate non-streaming endpoint per §3.5.

## 5. Real-world Patterns

**Anthropic SDK + Bedrock streaming alignment.** The Anthropic Python SDK and the Bedrock streaming API expose nearly identical chunk shapes — Anthropic's direct API stream and Bedrock's `InvokeModelWithResponseStream` are designed to be interchangeable from the consumer's perspective [1][4]. Code written against one ports to the other with minimal change. The cohort benefits: SDK examples found online are reliable references for Bedrock streaming behaviour.

**Chat UI defaults.** Every major commercial chat UI (Claude, ChatGPT, Gemini, Mistral Chat) streams by default. The expectation is industry-wide. A federal-context UI that does not stream looks dated by 2026 standards.

**Cancellation patterns in production.** Production deployments expose a "stop generation" UI control that closes the SSE connection and aborts the upstream Bedrock call. The token savings on cancelled long generations are non-trivial — high-volume deployments observe 5-15% of generations cancelled mid-stream [4]. The cohort sees this in W5 as a cost-management pattern.

## 6. Best Practices

- **Stream every user-facing LLM endpoint.** TTFT is the UX metric; without streaming, the perceived response time is the full generation time.
- **Aggregate usage info at stream end, not from the response object.** The streaming API returns usage in the terminal event, not in HTTP headers [1].
- **Use SSE with the `[DONE]` terminator convention.** It's the industry default; SDKs and browser consumers expect it.
- **Decide streaming-vs-structured-output explicitly per endpoint.** Don't try to stream a structured-output endpoint in W1; the partial-JSON parsing problem deserves its own scope (W3+).
- **Implement client-side cancellation from the first PR.** A "stop" button on the UI side closes the `EventSource`; the server detects closed-connection mid-stream and aborts the upstream Bedrock call.

## 7. Hands-on Exercise

**(8 minutes, paper.)** For the `POST /draft-solicitation/stream` endpoint you'll wire today:

- (a) Sketch the sequence of events from "user clicks 'Draft'" to "Angular renders the final paragraph" — six steps, max one sentence each.
- (b) Identify the **two points** at which usage info is captured for cost telemetry and explain why both points are needed.
- (c) Name the **specific failure mode** that the `[DONE]` terminator prevents.
- (d) Defend the W1 decision to keep structured-output endpoints non-streaming against the pushback "but everything else streams; users expect it".

**What good looks like.** Sequence: click → POST → Bedrock invoke_model_with_response_stream → chunks arrive → FastAPI re-emits as SSE → Angular EventSource consumes and renders. Cost capture: from the final `message_stop` event server-side, and (separately) from any partial-completion if cancelled — both are needed because cancellation skips the final event. The `[DONE]` terminator prevents a client-side hang waiting for the next chunk that will never arrive. The structured-output defence is the §3.5 composition pitfall: partial-JSON parsing is a complex engineering surface that does not belong in W1's scope.

## 8. Key Takeaways

- Can you name the three UX failure modes that streaming prevents and articulate TTFT as the real UX metric?
- Can you write a Bedrock `InvokeModelWithResponseStream` call from memory and identify which event type carries usage info?
- Can you implement a FastAPI `StreamingResponse` + SSE generator that forwards Bedrock chunks and captures usage at stream end?
- Can you defend the W1 decision to keep streaming and structured-outputs on separate endpoints and explain the W3+ composition follow-on?

## Sources

1. [InvokeModelWithResponseStream — Bedrock Runtime API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModelWithResponseStream.html) — retrieved 2026-05-22 via WebFetch (fallback when /web-research unavailable)
2. [Supported foundation models in Amazon Bedrock — AWS](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html) — retrieved 2026-05-26 via /web-research
3. [FastAPI StreamingResponse documentation](https://fastapi.tiangolo.com/advanced/custom-response/#streamingresponse) — retrieved 2026-05-22 via WebFetch (fallback)
4. [Bedrock Claude catalog research brief (`fde-10-week/research/bedrock-claude-catalog-20260522.md`)](https://github.com/KarsunFDE/content/blob/main/research/bedrock-claude-catalog-20260522.md) — last_verified 2026-05-22

> [!instructor-review]
> Sources 1, 3 retrieved via WebFetch fallback rather than /web-research. Recommend a fresh /web-research pass on these two URLs before Thursday hand-off to confirm current API surface — particularly the SSE consumer-side conventions if AWS or FastAPI have updated their reference patterns in the last 3 months.

Last verified: 2026-05-27
