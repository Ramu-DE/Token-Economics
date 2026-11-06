---
title: "Streaming Responses and Production Testing"
weight: 30
---

# 🌊 Streaming Responses and Production Testing

A loan officer shouldn't stare at a blank screen for 60 seconds while the agent reasons. Streaming delivers the assessment as it's generated. This page adds SSE streaming with the **Strands SDK**, Pydantic output validation, and a production-readiness checklist for the SaaS stack.

::alert[**This is the SaaS-stack counterpart to [Track A's streaming & testing page](/track-a-opensource/a4-production-ready/streaming-testing/).** The UX rationale and Pydantic validation are stack-independent; the delta is that streaming is implemented with the **Strands SDK** and flows **through Kong** (which passes streaming responses through unchanged).]{type="info"}

## 🛠️ Hands-On: Enable Streaming with Strands

### The UX difference (shared with Track A)

Without streaming, the loan officer waits for the entire multi-step assessment. With streaming, tokens appear as they're generated. See [Track A's UX comparison](/track-a-opensource/a4-production-ready/streaming-testing/) — it applies identically.

### Delta: Strands async streaming

Track A streams from a LangGraph app. In Track B, the Strands SDK exposes **`stream_async()`** on the agent, yielding events as the agent loop runs. Expose it from the FastAPI app as an SSE endpoint:

:::code{language=python showCopyAction=true}
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

@app.post("/api/process_credit_application_stream")
async def process_stream(image_file: UploadFile = File(...)):
    # ... store image, load MCP tools, build agent (as in B3) ...

    async def event_generator():
        async for event in agent.stream_async(user_prompt):
            # Strands yields text deltas + tool events; forward as SSE
            if "data" in event:
                yield f"data: {event['data']}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(event_generator(), media_type="text/event-stream")
:::

::alert[**Streaming works through Kong unchanged.** Kong's `ai-proxy` proxies the streaming Bedrock response back to the agent; the agent re-streams to the client via SSE. Kong even labels streamed calls with `request_mode="stream"` on its AI metrics (v3.12+), so your B4 dashboards distinguish streamed vs one-shot traffic. Confirm streaming is enabled on the route (it is by default for `llm/v1/chat`).]{type="info"}

### Streaming + tracing

Strands emits the same OpenInference spans for streamed runs, so **Arize still captures the full trace** (AGENT/LLM/TOOL spans, tokens) — streaming changes the client delivery, not the observability.

## 📐 Structured Output Validation with Pydantic (shared)

The Pydantic credit-decision schema and validation logic are **stack-independent** — follow **[Track A's Pydantic section](/track-a-opensource/a4-production-ready/streaming-testing/)**. The only difference is where it sits in the agent: validate the Strands agent's final `result` against the schema before returning it.

:::code{language=python showCopyAction=true}
# Validate the Strands agent's final output against the shared schema
from pydantic import ValidationError

result = agent(user_prompt)
try:
    decision = CreditDecision.model_validate_json(str(result))   # same schema as Track A
except ValidationError as e:
    logger.error("Output failed schema validation: %s", e)
    # re-prompt or return a safe error
:::

## ✅ Production Readiness Checklist (Track B)

Verify all SaaS-stack production requirements before go-live:

| Area | Requirement | Where |
|---|---|---|
| **Gateway** | `/loan-strands` route live; `key-auth` enforced; model pinned (`allow_override:false`) | B2 |
| **Cost control** | Kong AI metrics on; per-Consumer attribution; `ai-rate-limiting-advanced` budget set | B4 |
| **Observability** | Arize receiving AGENT/LLM/TOOL spans; cost configured (before ingest); evaluators running | B2, B4 |
| **RAG** | Milvus deployed; policy docs ingested; retrieval tool traced; Ragas evaluated | B5 RAG |
| **Security** | Agent Pod Identity = **S3 only**; secrets via ESO; `ai-prompt-guard` on route; Bedrock Guardrails applied | B5 Security |
| **Streaming** | SSE endpoint via `stream_async`; validated with Pydantic | this page |
| **Monitoring** | Kong AI metrics → AMP → Grafana; KEDA autoscaling | B4 monitoring |
| **Multi-arch** | agent image built for amd64 + arm64 (mixed-node cluster) | B3 |

## 🎊 What You've Built — Track B Complete

Across Modules B1-B5, you built a complete production-grade agentic AI platform on the **Kong + Arize SaaS stack**:

- ✅ **Kong AI Gateway** (Konnect + DataPlane, CRD-configured) fronting Amazon Bedrock — no LLM keys in the app
- ✅ **Arize AX** full-trace observability with evaluation and cost
- ✅ **Strands** Loan Buddy agent with MCP tools, RAG, and streaming
- ✅ **Per-tenant token/cost attribution AND edge enforcement** — the SaaS-stack superpower
- ✅ **Production hardening** — least-privilege IAM, external secrets, gateway + model content safety

::alert[**The two-track story:** you've now seen the same agentic AI platform built two ways — [Track A](/track-a-opensource/) (self-hosted open source) and Track B (managed SaaS). Customers can compare operating models and choose what fits their team.]{type="success"}

---

**[Next: Infrastructure Monitoring and Auto-Scaling →](../monitoring-scaling/)**
