---
title: "Track A: Open-Source Stack with LiteLLM, Langfuse and LangGraph"
weight: 30
---

# Track A — Open-Source Stack

This track builds the complete Agentic AI platform on **self-hosted, open-source** components running in your Amazon EKS cluster:

- **AI Gateway** — [LiteLLM](https://docs.litellm.ai/) (in-cluster) for a unified, OpenAI-compatible API to Amazon Bedrock and your vLLM models
- **Observability** — [Langfuse](https://langfuse.com/docs) (in-cluster) for tracing, sessions, and evaluations
- **Agent framework** — [LangChain / LangGraph](https://langchain-ai.github.io/langgraph/) for the agentic application

You will build the platform (Module A1), create the credit-underwriting agent (A2), scale and evaluate it (A3), and harden it for production (A4).

::alert[**Choosing a track:** Track A uses self-hosted open-source tooling you run and operate yourself. If you would rather use **managed SaaS** control planes (Kong Konnect + Arize AX) with the Strands SDK, see [Track B — SaaS Stack](../track-b-saas/). The gateway and observability layers are fully independent between tracks — pick the one that matches how you want to operate the platform. (Note: Track B's agent reuses the MCP tool servers you deploy here in Module A2, since MCP is framework-agnostic.)]{type="info"}

## What's in this track

| Module | Focus |
|---|---|
| **A1: GenAI Platform Components** | LiteLLM gateway + Langfuse observability, deployed with Helm |
| **A2: Building Agentic AI Applications** | LangGraph agent + MCP tools (credit underwriting) |
| **A3: Scaling and Evaluating the Platform** | LLM-as-a-Judge evaluation, A/B model routing, AMP/AMG monitoring, KEDA autoscaling |
| **A4: Production-Ready Agentic AI Applications** | RAG pipeline, Pod Identity, External Secrets, Bedrock Guardrails, SSE streaming |

## Track comparison

| | Track A (this track) | [Track B](../track-b-saas/) |
|---|---|---|
| AI Gateway | **LiteLLM** (self-hosted) | Kong AI Gateway — Konnect (SaaS) |
| Observability | **Langfuse** (self-hosted) | Arize AX (SaaS) |
| Agent framework | **LangChain / LangGraph** | Strands Agents SDK |
| Model access IAM | app holds Bedrock + gateway key | app holds S3 only; Bedrock via gateway |
| Operating model | you run the control planes | managed SaaS control planes |

---

**[Start with Module A1: GenAI Platform Components →](a1-genai-components/)**
