# A3 — Monitoring & Scaling  ❌ blocked

```mermaid
flowchart LR
  VL[vLLM pods<br/>/metrics] --> ADOT[ADOT collector]
  NM[neuron-monitor DaemonSet] --> ADOT
  ADOT -->|remote_write SigV4| AMP[(Amazon Managed Prometheus<br/>us-west-2)]
  AMP --> G[Grafana dashboards]
  AMP --> KEDA[KEDA ScaledObject<br/>prometheus trigger]
  KEDA -->|scale replicas| VL
  VL -->|pending pods| KP[Karpenter / Auto Mode<br/>neuron NodePool → inf2]
  LHPA[HPA] --> LL[LiteLLM]
```

**Done:** `neuron-ls` verified (inf2.xlarge).
**Blocked:** `workshops/eks-genai-workshop/static/code/module4/monitoring/` (neuron-monitor.yaml, adot-collector.yaml) is not in the starter kit.
**Cleanup needed:** AMP workspace `ws-584a362c-2859-4eaf-b32e-459cbad9a2a0` was created in **us-east-1** (wrong region) — delete and recreate in us-west-2:
```bash
aws amp delete-workspace --workspace-id ws-584a362c-2859-4eaf-b32e-459cbad9a2a0 --region us-east-1
aws amp create-workspace --alias genai-workshop --region us-west-2
```
**Remaining (once module4 code available):** neuron-monitor → ADOT → verify AMP query → Grafana → KEDA + IAM role → ScaledObject → LiteLLM HPA → NodePool check → load test → alerts.
**Gotcha:** `pkill -f "port-forward.*vllm"` can kill your own shell — kill the port-forward PID instead.
