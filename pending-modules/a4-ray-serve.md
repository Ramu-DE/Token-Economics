# A4 — Ray Serve Autoscaling  ⏸ awaiting approval (cost)

```mermaid
flowchart LR
  OP[KubeRay operator] --> RS[RayService]
  RS --> HEAD[Ray head]
  HEAD -->|in-tree autoscaler| W[Neuron workers<br/>neuron NodePool]
  W -->|Karpenter| INF2[inf2 nodes 1→2 under load]
  GW[LiteLLM] -->|model qwen3-8b-neuron-ray| RS
  GW --> VL[existing vLLM qwen3-8b (unchanged)]
```

Not applied: provisions additional inf2 nodes. Existing vLLM stays. Registers `qwen3-8b-neuron-ray` in LiteLLM. Apply only after confirming cost.
