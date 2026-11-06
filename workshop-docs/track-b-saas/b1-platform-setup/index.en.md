---
title: "Module B1: Platform Setup — Kong Konnect & Arize AX"
weight: 10
---

Before building and deploying GenAI applications with the **Track B Stack**, this section walks you through setting up the two SaaS platforms that replace LiteLLM and Langfuse: **Kong AI Gateway** (via Kong Konnect) and **Arize AX** (observability). Unlike the default stack -- where LiteLLM and Langfuse are pre-deployed in-cluster -- the Track B stack uses managed SaaS control planes that you register and connect to your Amazon EKS cluster.

::alert[This is the **Track B Stack** setup. It runs in parallel with the default LiteLLM + Langfuse stack and does not modify it. The shared components (vLLM, Open WebUI, Qdrant, TEI) remain pre-configured -- only the gateway and observability layers change.]{type="info"}

## What You Will Set Up

| Layer | Default stack (pre-configured) | Track B stack (you set up here) |
|---|---|---|
| AI Gateway | LiteLLM (in-cluster pod) | **Kong AI Gateway** -- Konnect (SaaS) control plane + in-cluster DataPlane |
| Observability | Langfuse (in-cluster) | **Arize AX** (SaaS) -- Space + API key |
| LLM provider | Amazon Bedrock | Amazon Bedrock *(unchanged)* |
| Self-hosted models | vLLM Qwen | vLLM Qwen *(unchanged)* |

## Why SaaS Control Planes

Production teams frequently standardize on **managed** gateway and observability layers to avoid operating databases and control-plane infrastructure themselves. This pathway shows how the *same* platform is built when the gateway (Kong Konnect) and observability (Arize AX) are SaaS:

- **Kong Konnect** hosts the gateway configuration, analytics, and AI Manager UI. You run only a stateless **DataPlane** in your cluster (managed by the Kong Gateway Operator).
- **Arize AX** hosts traces, spans, sessions, and evaluations. Your agents export **OpenInference** spans to it -- nothing to run in-cluster.

## Setup Sections

Complete both setup pages **before** starting Module B2 (where you configure and interact with the Kong and Arize UIs):

#### 1. [Kong AI Gateway Setup →](kong-setup/)
Register a Kong Konnect subscription, create a Personal Access Token, install the Kong Gateway Operator, and deploy a Control Plane + DataPlane connected to your EKS cluster with Bedrock access.

#### 2. [Arize AX Setup →](arize-setup/)
Create an Arize AX account, set up a Space, generate a Service API key, and install the credentials into your cluster so agents can export traces.

## Prerequisites

You need everything from the default workshop's [Infrastructure Setup](../../introduction/infra-setup/) (EKS Auto Mode cluster, vLLM, Open WebUI, kubectl access), **plus**:

- A **Kong Konnect** account (free-tier / Plus trial) -- set up on the next page
- An **Arize AX** account (free tier) -- set up on the Arize page
- The workshop Code Editor terminal with `kubectl`, `helm`, and `aws` CLI available

## Success Checklist

Before proceeding to Module B2, confirm:

- **Kong Konnect** account created and control plane `kong-aws` visible in Gateway Manager
- **Kong DataPlane** (`dataplane1`) running in the `kong` namespace and connected (healthy)
- **Bedrock access** confirmed for the DataPlane (Pod Identity → IAM role)
- **Arize AX** account created, Space set up, Service API key generated
- **`arize-credentials`** Secret present in the cluster

## Ready for Module B2

Once both setups are complete, you have:

- Kong Konnect control plane + in-cluster DataPlane connected to Amazon Bedrock
- Arize AX Space + credentials ready for agent tracing
- The same shared model layer (vLLM, Open WebUI) as the default stack

You are ready to configure the Kong Bedrock route and Arize observability.

---

**[Begin: Kong AI Gateway Setup →](kong-setup/)**
