# A4 — RAG Pipeline + Streaming & Testing  ⚠️ partial

```mermaid
flowchart LR
  DOCS[Loan policy docs] --> ING[ingest_policies.py] --> EMB[Titan Embeddings v2<br/>Bedrock] --> MV[(Milvus standalone<br/>ns milvus)]
  Q[Applicant query] --> V2[Loan Buddy v2 agent]
  V2 -->|similarity search| MV
  V2 --> GW[LiteLLM] --> LLM[Claude / qwen3-8b]
  V2 -->|SSE stream| U[Client]
  V2 -. traces .-> LF[Langfuse] --> RG[Ragas eval]
```

| Step | Status |
|---|---|
| Titan v2 model access | ✅ |
| Milvus install (etcd, minio, standalone) | ✅ Running |
| Ingest policies / similarity search | ❌ `ingest_policies.py` missing (module5) |
| Deploy `loan-buddy-agent-v2` | ❌ `deploy-rag-agent.sh` missing |
| Ragas eval | ❌ depends on v2 |
| Streaming agent deploy / `production_readiness_check.py` | ❌ missing (module5) |

**Unblock:** supply `workshops/eks-genai-workshop/static/code/module5/`; then run ingest → deploy v2 → patch SA (security step 4) → guardrail tests → streaming → readiness check.
