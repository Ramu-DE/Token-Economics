# Pending Modules — Status, Flow & Remaining Steps

Live cluster: `genai-on-eks` (us-west-2). Always start with:
```bash
export AWS_REGION=us-west-2 AWS_DEFAULT_REGION=us-west-2 CLUSTER_NAME=$EKS_CLUSTER_NAME
export AGENT_KEY=<A1 LiteLLM virtual key>
```

```mermaid
flowchart TD
  A1[A1 Gateway + Observability ✅] --> A2[A2 Loan Buddy app ✅]
  A2 --> A3R[A3 Model routing ✅]
  A2 --> A3E[A3 Evaluation ⏳]
  A1 --> A3M[A3 Monitoring & Scaling ❌]
  A2 --> A4S[A4 Security hardening ⚠️]
  A2 --> A4R[A4 RAG pipeline ⚠️]
  A4R --> A4T[A4 Streaming & testing ❌]
  A1 --> A4Y[A4 Ray Serve autoscaling ⏸]
  B1[B1 Kong + Arize setup ⏸] --> B2[B2 Gateway + Observability] --> B3[B3 Strands app] --> B4[B4 Routing/Eval/Cost] --> B5[B5 Production-ready]
```

| File | Module |
|---|---|
| [a3-monitoring-scaling.md](a3-monitoring-scaling.md) | A3 Monitoring & Scaling |
| [a3-evaluation.md](a3-evaluation.md) | A3 Evaluation |
| [a4-security-hardening.md](a4-security-hardening.md) | A4 Security |
| [a4-rag-streaming.md](a4-rag-streaming.md) | A4 RAG + Streaming |
| [a4-ray-serve.md](a4-ray-serve.md) | A4 Ray Serve |
| [track-b.md](track-b.md) | Track B (B1–B5) |
