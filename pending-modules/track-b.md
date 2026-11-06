# Track B (SaaS) — B1–B5  ⏸ needs credentials

```mermaid
flowchart LR
  U[User] --> S[Strands agents<br/>ns strands-agents]
  S --> K[Kong Konnect AI Gateway<br/>ns kong · AI Proxy · routing]
  K --> BR[Claude Sonnet 4.5 Bedrock]
  K --> VL[vLLM qwen3-8b]
  S -. OTel .-> AX[Arize AX<br/>traces · evaluators · cost]
  K -. OTel .-> AX
```

| Module | Content |
|---|---|
| B1 | Konnect control plane + data plane, Arize space/keys |
| B2 | Kong AI Proxy routes, Arize tracing, prompt engineering |
| B3 | Strands loan app (components → running app) |
| B4 | Model routing, Arize evaluation, token cost attribution |
| B5 | Monitoring/scaling, RAG, Ray Serve, security, streaming |

**Blocked:** `KONNECT_TOKEN` and Arize API key/space ID are not set. Export them, then start at `workshop-docs/track-b-saas/b1-platform-setup/`.
