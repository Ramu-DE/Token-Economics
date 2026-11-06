# A3 — Evaluation (Langfuse LLM-as-a-Judge)  ⏳ UI steps

```mermaid
flowchart LR
  APP[Loan Buddy agent] -->|traces| LF[Langfuse]
  GW[LiteLLM] -->|traces| LF
  LF --> EV[Evaluator template<br/>judge model via LiteLLM]
  EV --> SC[Scores on traces]
  SC --> DB[Langfuse dashboard]
```

**Ready:** A2 traces already in Langfuse (`litellm-acompletion` traces).
**Remaining (Langfuse UI):** add LLM connection (LiteLLM URL + key) → create evaluator from template → target traces from Loan Buddy → run on new traces → review scores.
