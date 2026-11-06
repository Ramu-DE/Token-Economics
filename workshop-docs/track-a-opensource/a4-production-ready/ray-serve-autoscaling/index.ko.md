---
title: "Ray Serve Autoscaling for vLLM on Inferentia (KubeRay)"
weight: 4
---

Earlier in A4 you added RAG, security hardening, and streaming to the Loan Buddy agent. This section closes the operations story with the piece that controls **Neuron spend under changing load**: **autoscaling the self-hosted vLLM model with Ray Serve on KubeRay** — the open-source, self-managed control plane (Track A's counterpart is [Track B's Anyscale-managed version](/track-b-saas/b5-production-ready/ray-serve-autoscaling/), same model, same inf2 hardware).

The workshop's self-hosted model (`qwen3-8b-neuron`) runs today as a **fixed single-replica vLLM Deployment** — it holds a whole Inferentia2 device (2 NeuronCores) whether it serves one request or none. In production, inference traffic is bursty, and an idle accelerator is the most expensive thing in the stack. This section deploys the **same model** as a **Ray Serve application** that **scales replicas up and down on live request load**, with **Karpenter** provisioning and reclaiming **inf2** nodes underneath — so capacity follows demand.

::alert[**This runs side-by-side with the existing model — nothing is removed.** You keep the original `vllm/qwen3-8b-neuron` model and add a new autoscaled `vllm/qwen3-8b-neuron-ray` model. Both are reachable through LiteLLM and Open WebUI, so you can compare a fixed deployment against an autoscaled one on the same cluster.]{type="info"}

## 🎯 What You Will Learn

- ✅ Why the Track A stack (open-source) uses **KubeRay** for self-managed Ray autoscaling
- ✅ The **three levels of autoscaling** (Serve replicas → Ray worker pods → inf2 nodes) and how they chain
- ✅ Install the **KubeRay operator** and deploy a **`RayService`** that serves vLLM on **Inferentia2 (Neuron)**
- ✅ Why the Neuron path uses a thin **Ray Serve proxy around the workshop's own `vllm serve`** (not the GPU-only `build_openai_app`), and why Ray is **isolated in a venv**
- ✅ Register the autoscaled model **side-by-side** in LiteLLM and use it from Open WebUI
- ✅ Drive **concurrent load** and watch replicas scale **1 → 2** with an inf2 node provisioned by Karpenter
- ✅ See **Ray + vLLM metrics in Grafana** (via Amazon Managed Prometheus), before and after load

## 🧠 Concept: three levels of autoscaling

"Autoscaling" here spans three chained dimensions:

```mermaid
graph TB
    LOAD[Concurrent requests] --> A
    subgraph "a. Ray Serve REPLICAS"
        A[vLLM replica count<br/>each replica = 1 vLLM engine = 1 inf2 device 2 cores<br/>autoscaling_config: min/max_replicas, target_ongoing_requests]
    end
    A -->|new replica needs free NeuronCores| B
    subgraph "b. Ray WORKER PODS"
        B[Ray cluster worker pods<br/>enableInTreeAutoscaling + workerGroup min/maxReplicas]
    end
    B -->|worker pod Pending, no inf2 node has room| C
    subgraph "c. Kubernetes NODES / EC2"
        C[inf2.xlarge instances<br/>provisioned + reclaimed by Karpenter]
    end
```

- **(a) Ray Serve replicas** is the primary control — the number of vLLM engine replicas. Each replica requests **2 NeuronCores** (`ray_actor_options.resources.neuron_cores: 2`), so on a 2-core `inf2.xlarge` it's **one replica per pod per node** (a clean 1:1:1 mapping that makes all three levels visible).
- **(b)** When Serve wants another replica and no Ray worker has free NeuronCores, the **Ray in-tree autoscaler** (`enableInTreeAutoscaling: true`) adds a worker pod.
- **(c)** That pending worker pod triggers **Karpenter** to launch an `inf2.xlarge` node. Scale-down reverses.

::alert[**The scaling metric is request load, not accelerator utilization.** Ray Serve scales on **`target_ongoing_requests`** — the target average in-flight/queued requests per replica. NeuronCore utilization is near-100% whenever *any* request runs, so it's a poor proxy for "am I keeping up." A person typing in Open WebUI (one request at a time) will **never** trigger scale-up — you need the concurrent load generator in Part 6.]{type="warning"}

## ✅ Prerequisites

::alert[**Using zsh (e.g. macOS)?** Run `setopt interactive_comments` once so pasted commands with trailing `# ...` notes don't error. (bash — the workshop Code Editor default — needs nothing.)]{type="info"}

- The Track A stack is deployed and the **`qwen3-8b-neuron`** vLLM model has run at least once (so its weights + compiled Neuron artifacts are cached in the image and the `neuron` Karpenter nodepool is known-good).
- Kubeconfig points at the workshop cluster, and the `neuron` nodepool + EFS cache exist:

:::code{language=bash showCopyAction=true}
kubectl get nodepool neuron
kubectl get pvc huggingface-cache -n vllm
:::

(`kubectl get nodepool neuron` should be `READY`; the nodepool provisions `inf2` on demand.)

## Part 1 — Install the KubeRay operator

The **KubeRay operator** is the open-source controller that turns a `RayService` custom resource into a running Ray cluster (head + workers) and manages zero-downtime updates. It is one-time platform setup and runs on CPU nodes.

:::code{language=bash showCopyAction=true}
helm repo add kuberay https://ray-project.github.io/kuberay-helm/
helm repo update kuberay

helm upgrade --install kuberay-operator kuberay/kuberay-operator \
  --version 1.5.1 \
  --namespace ray-system --create-namespace

kubectl -n ray-system rollout status deploy/kuberay-operator --timeout=180s
:::

Verify the CRDs are registered:

:::code{language=bash showCopyAction=true}
kubectl get crd | grep ray.io
:::

::::expand{header="Check Output"}

```text
rayclusters.ray.io    2026-...
rayjobs.ray.io        2026-...
rayservices.ray.io    2026-...
```

::::

## Part 2 — The serving image (pre-built on public ECR)

Unlike the GPU path (where Ray Serve LLM's built-in `ray.serve.llm:build_openai_app` works), the **Neuron path needs a different design** for two reasons:

1. **`build_openai_app` is GPU-oriented** (it assumes `num_gpus` scheduling and bundles a CUDA vLLM). On Inferentia we must use the workshop's **`optimum-neuron` vLLM** that already knows how to run qwen3-8b on NeuronCores.
2. **Version isolation.** Ray Serve 2.58 pulls a newer `fastapi`/`starlette` than the image's **vLLM 0.10.2** tolerates — co-installing them breaks vLLM's OpenAI server (HTTP 500 on every route). So Ray is installed into its **own virtualenv** (`/opt/rayvenv`), and vLLM keeps the image's original Python + deps.

The workshop therefore uses a **pre-built image** that layers `ray[serve]` (in a venv) onto the workshop's own inf2 vLLM image, published to public ECR:

```text
public.ecr.aws/agentic-ai-platforms-on-k8s/ray-vllm-neuron:latest
```

**How it serves:** a thin Ray Serve deployment (`vllm_serve.py`) launches the workshop's **exact `vllm serve …`** as a pod-local subprocess (127.0.0.1:8100) and exposes an OpenAI-compatible proxy. Each replica requests **2 NeuronCores**, so Ray autoscales replicas 1:1 with inf2 devices. You do not build anything — the `RayService` in Part 3 references this image. The `Dockerfile` and `vllm_serve.py` live in the starter-kit under `components/llm-model/vllm-ray/` (see the [Appendix](#appendix--the-image)).

## Part 3 — Deploy the RayService

This `RayService` runs the **same `qwen3-8b-neuron` model** as a Ray Serve application on **inf2.xlarge**. Key points, called out inline:

- The **head** runs on a CPU node (`--num-cpus=0` for scheduling; it hosts the Serve controller/ingress/autoscaler).
- The **worker** pins to `inf2.xlarge`, tolerates the `aws.amazon.com/neuron` taint, and advertises **`neuron_cores: 2`** as a custom Ray resource so exactly one replica lands per device.
- **`NEURON_RT_VISIBLE_CORES=0,1`** (comma) is what Ray's raylet needs to count 2 cores; `vllm_serve.py` converts it to the **range `0-1`** that optimum-neuron needs before launching vLLM.
- Both pods carry `karpenter.sh/do-not-disrupt: "true"` so Karpenter consolidation doesn't evict the Ray head mid-deploy.
- Probes use **python-urllib** against the raylet healthz (`:52365`) because the Neuron base image has **no `curl`/`wget`**.

:::code{language=bash showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: ray.io/v1
kind: RayService
metadata:
  name: qwen3-ray
  namespace: vllm
spec:
  serveConfigV2: |
    applications:
      - name: qwen3-neuron
        import_path: vllm_serve:app
        route_prefix: "/"
  rayClusterConfig:
    rayVersion: '2.58.0'
    enableInTreeAutoscaling: true
    headGroupSpec:
      rayStartParams: {}
      template:
        metadata:
          annotations:
            karpenter.sh/do-not-disrupt: "true"
        spec:
          enableServiceLinks: false
          nodeSelector:
            kubernetes.io/arch: amd64
          containers:
          - name: ray-head
            image: public.ecr.aws/agentic-ai-platforms-on-k8s/ray-vllm-neuron:latest
            imagePullPolicy: Always
            command: ["/bin/bash","-lc"]
            args:
            - |
              /opt/rayvenv/bin/ray start --head --dashboard-host=0.0.0.0 --dashboard-port=8265 --num-cpus=0 --num-gpus=0 --metrics-export-port=8080 --block --dashboard-agent-listen-port=52365 --no-monitor
            env:
            - { name: PYTHONPATH, value: "/home/ray" }
            - { name: RAY_SERVE_HTTP_HOST, value: "0.0.0.0" }
            resources:
              requests: { cpu: 3, memory: 8Gi }
              limits: { cpu: 4, memory: 12Gi }
            ports:
            - { containerPort: 6379, name: gcs }
            - { containerPort: 8265, name: dashboard }
            - { containerPort: 10001, name: client }
            - { containerPort: 8000, name: serve }
            - { containerPort: 52365, name: dashboard-agent }
            - { containerPort: 8080, name: metrics }
            - { containerPort: 44217, name: as-metrics }
            - { containerPort: 44227, name: dash-metrics }
            livenessProbe:
              exec:
                command: ["bash","-c","python3 -c \"import urllib.request,sys; sys.exit(0 if b'success' in urllib.request.urlopen('http://localhost:52365/api/local_raylet_healthz',timeout=8).read() else 1)\""]
              initialDelaySeconds: 60
              periodSeconds: 15
              timeoutSeconds: 10
              failureThreshold: 10
            readinessProbe:
              exec:
                command: ["bash","-c","/opt/rayvenv/bin/ray health-check --address 127.0.0.1:6379 > /dev/null 2>&1"]
              initialDelaySeconds: 30
              periodSeconds: 15
              timeoutSeconds: 10
              failureThreshold: 10
    workerGroupSpecs:
    - replicas: 1
      minReplicas: 1
      maxReplicas: 2
      groupName: neuron-group
      # Tell the in-tree autoscaler this worker group PROVIDES neuron_cores, so it can plan a scale-up
      # (add a worker pod) to satisfy a pending replica's neuron_cores demand. Without this the cluster
      # serves fine but never scales past 1 worker.
      # ⚠️ Exact quoting matters: the KubeRay autoscaler does `resources_string[1:-1]` before json.loads
      # (it strips an assumed outer quote pair), so the value MUST carry an extra outer quote pair —
      # '"{\"neuron_cores\": 2}"' (the canonical ray-cluster.complete.yaml form). A bare
      # '{"neuron_cores": 2}' makes the autoscaler sidecar crash-loop with `JSONDecodeError: Extra data`.
      rayStartParams:
        resources: '"{\"neuron_cores\": 2}"'
      template:
        metadata:
          annotations:
            karpenter.sh/do-not-disrupt: "true"
        spec:
          enableServiceLinks: false
          nodeSelector:
            kubernetes.io/arch: amd64
            node.kubernetes.io/instance-type: inf2.xlarge
          tolerations:
          - { key: "aws.amazon.com/neuron", operator: "Exists", effect: "NoSchedule" }
          containers:
          - name: ray-worker
            image: public.ecr.aws/agentic-ai-platforms-on-k8s/ray-vllm-neuron:latest
            imagePullPolicy: Always
            command: ["/bin/bash","-lc"]
            args:
            - |
              /opt/rayvenv/bin/ray start --address=${RAY_ADDRESS:-${FQ_RAY_IP}:6379} --resources='{"neuron_cores": 2}' --dashboard-agent-listen-port=52365 --metrics-export-port=8080 --block --num-cpus=2 --num-gpus=0 --no-monitor
            env:
            - { name: PYTHONPATH, value: "/home/ray" }
            - { name: NEURON_RT_NUM_CORES, value: "2" }
            - { name: NEURON_RT_VISIBLE_CORES, value: "0,1" }
            - { name: HF_HOME, value: "/root/.cache/huggingface" }
            - { name: HF_HUB_CACHE, value: "/root/.cache/huggingface/hub" }
            resources:
              requests: { cpu: 3, memory: 12Gi, aws.amazon.com/neuroncore: 2 }
              limits: { memory: 20Gi, aws.amazon.com/neuroncore: 2 }
            livenessProbe:
              exec:
                command: ["bash","-c","python3 -c \"import urllib.request,sys; sys.exit(0 if b'success' in urllib.request.urlopen('http://localhost:52365/api/local_raylet_healthz',timeout=8).read() else 1)\""]
              initialDelaySeconds: 90
              periodSeconds: 20
              timeoutSeconds: 10
              failureThreshold: 15
            readinessProbe:
              exec:
                command: ["bash","-c","python3 -c \"import urllib.request,sys; sys.exit(0 if b'success' in urllib.request.urlopen('http://localhost:52365/api/local_raylet_healthz',timeout=8).read() else 1)\""]
              initialDelaySeconds: 45
              periodSeconds: 20
              timeoutSeconds: 10
              failureThreshold: 15
            volumeMounts:
            - { name: shm, mountPath: /dev/shm }
          volumes:
          - name: shm
            emptyDir: { medium: Memory, sizeLimit: 8Gi }
EOF
:::

Watch it come up — the head starts first, then Karpenter provisions an `inf2.xlarge` node for the worker, then Ray Serve loads the model:

:::code{language=bash showCopyAction=true}
kubectl get rayservice qwen3-ray -n vllm -w
:::

Then confirm both pods are `Running` before testing:

:::code{language=bash showCopyAction=true}
kubectl get pods -n vllm -l ray.io/is-ray-node=yes -o wide
:::

::::expand{header="Check Output"}

```text
NAME                                        READY   STATUS    RESTARTS   AGE
qwen3-ray-xxxxx-head-xxxxx                  2/2     Running   0          14m
qwen3-ray-xxxxx-neuron-group-worker-xxxxx   1/1     Running   0          12m
```

::::

::alert[**First bring-up is slow: the image is ~18 GB** (it bakes the model + compiled Neuron artifacts), so the initial pull on a fresh inf2 node takes **~10 minutes**, then the vLLM subprocess loads qwen3-8b across both NeuronCores. Watch the worker with `kubectl logs -n vllm <worker> -c ray-worker`. Only proceed once the head is `2/2` and the worker is `1/1`.]{type="info"}

## Part 4 — Verify the model is serving

KubeRay creates a Serve service `qwen3-ray-serve-svc` on port 8000. Confirm health from three angles — the KubeRay CR, the Ray **cluster**, and the Ray **Serve** app:

:::code{language=bash showCopyAction=true}
HEAD=$(kubectl get pods -n vllm -l ray.io/group=headgroup -o jsonpath='{.items[-1].metadata.name}')
kubectl exec -n vllm "$HEAD" -c ray-head -- /opt/rayvenv/bin/ray status
kubectl exec -n vllm "$HEAD" -c ray-head -- /opt/rayvenv/bin/serve status
:::

::::expand{header="Check Output"}

```text
# ray status
======== Autoscaler status: 2026-... ========
Node status
---------------------------------------------------------------
Active:
 1 headgroup
 1 neuron-group
Resources
---------------------------------------------------------------
Total Usage:
 1.0/2.0 CPU
 2.0/2.0 neuron_cores
 0B/18.49GiB memory

# serve status
applications:
  qwen3-neuron:
    status: RUNNING
    deployments:
      qwen3-neuron:
        status: HEALTHY
        status_trigger: CONFIG_UPDATE_COMPLETED
        replica_states:
          RUNNING: 1
```

::::

Now port-forward the Serve HTTP proxy and hit the OpenAI-compatible endpoints. Port-forward to the **head pod** (the Serve proxy listens on port 8000 there) rather than the ClusterIP service — a service port-forward can silently drop with `lost connection to pod`, which then makes `curl` return an empty body and `json.tool` fail with `Expecting value: line 1 column 1 (char 0)`:

:::code{language=bash showCopyAction=true}
# Resolve the head pod and port-forward to it (stable), replacing any stale forward
HEAD=$(kubectl get pods -n vllm -l ray.io/group=headgroup -o jsonpath='{.items[-1].metadata.name}')
# Idempotent — start the port-forward only if 8000 isn't already forwarding (safe to re-run)
pgrep -f "port-forward.*8000:8000" >/dev/null || kubectl port-forward -n vllm "pod/$HEAD" 8000:8000 >/workshop/ray_pf.log 2>&1 &

# Wait until the endpoint actually answers before curling (avoids the empty-response / "Expecting value" error)
for i in $(seq 1 30); do
  curl -sf http://localhost:8000/v1/models >/dev/null 2>&1 && { echo "Serve endpoint is up"; break; }
  echo "waiting for the port-forward / Serve endpoint..."; sleep 3
done

curl -s http://localhost:8000/v1/models | python3 -m json.tool

curl -s http://localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3-8b-neuron-ray","messages":[{"role":"user","content":"Reply with exactly: ray neuron online"}],"max_tokens":20}' \
  | python3 -m json.tool
:::

::alert[**If you see `Expecting value: line 1 column 1 (char 0)`**, `curl` got an empty response — the port-forward isn't running (or dropped) in *this* shell. Re-run the block above (it re-resolves the head pod and waits for the endpoint). Keep the port-forward's terminal open; if the head pod restarts, its name changes, so re-run the `HEAD=…` + `port-forward` lines to re-point it.]{type="warning"}

::::expand{header="Check Output"}

```json
// GET /v1/models
{
  "object": "list",
  "data": [
    {
      "id": "qwen3-8b-neuron-ray",
      "object": "model",
      "owned_by": "vllm",
      "root": "/root/.cache/neuron/Qwen/Qwen3-8B",
      "max_model_len": 8192
    }
  ]
}

// POST /v1/chat/completions
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "model": "qwen3-8b-neuron-ray",
  "choices": [
    { "index": 0, "message": { "role": "assistant", "content": "ray neuron online" }, "finish_reason": "stop" }
  ]
}
```

::::

::alert[**qwen3-8b is a reasoning model.** With the `--reasoning-parser=qwen3` flag it may emit a `<think>…</think>` block before the final answer; the `content`/`reasoning_content` split shows up in the JSON. That is expected.]{type="info"}

## Part 4b — The Ray Dashboard (built-in)

Ray ships a web dashboard showing the cluster, the Serve applications, and per-replica detail — the fastest way to *see* the autoscaling you configured. Port-forward it:

:::code{language=bash showCopyAction=true}
# Idempotent — start the dashboard port-forward only if 8265 isn't already forwarding (safe to re-run)
pgrep -f "port-forward.*8265:8265" >/dev/null || kubectl port-forward -n vllm svc/qwen3-ray-head-svc 8265:8265 >/workshop/ray_dash.log 2>&1 &
:::

Open **http://localhost:8265**. Three useful views:

**Serve** — the `qwen3-neuron` application, its deployment, and one HTTP Proxy per Ray node. A new replica appears here as `STARTING` → `RUNNING` under load:

![Ray Dashboard — Serve view: the qwen3-neuron application, its deployment/replicas, and one HTTP Proxy per Ray node](/static/images/track-a-opensource/a4-production-ready/ray-dashboard-serve.png)

**Cluster** — the Ray nodes (head + neuron workers) and their resource allocation, including `neuron_cores` per node:

![Ray Dashboard — Cluster view: head + neuron-group worker nodes with CPU / memory / neuron_cores allocation](/static/images/track-a-opensource/a4-production-ready/ray-dashboard-cluster.png)

**Actors / Logs** — the `ServeReplica:qwen3-neuron` actors and their vLLM subprocess logs, viewable without `kubectl exec`:

![Ray Dashboard — Actors view: the ServeReplica:qwen3-neuron actor(s) with state and per-actor logs](/static/images/track-a-opensource/a4-production-ready/ray-dashboard-actors.png)

::alert[The dashboard reflects **logical** Ray state. For **physical NeuronCore utilization / token-timing / throughput over time**, use Grafana (Part 7).]{type="info"}

## Part 5 — Register side-by-side in LiteLLM

The existing model stays untouched; add the autoscaled one so both appear in Open WebUI. The RayService endpoint is OpenAI-compatible, so LiteLLM points at it exactly like the plain vLLM model — just a different in-cluster service name.

For a durable change, add this to the LiteLLM chart values (`components/ai-gateway/litellm/values.template.yaml`) and `helm upgrade`. For a **quick live test**, patch the running config and restart:

:::code{language=bash showCopyAction=true}
kubectl get configmap litellm-config -n litellm -o jsonpath='{.data.config\.yaml}' > /workshop/litellm-config.yaml
cat >> /workshop/litellm-config.yaml <<'EOF'
- litellm_params:
    api_base: http://qwen3-ray-serve-svc.vllm:8000/v1
    api_key: fake-key
    model: openai/qwen3-8b-neuron-ray
  model_name: vllm/qwen3-8b-neuron-ray
EOF
kubectl create configmap litellm-config -n litellm \
  --from-file=config.yaml=/workshop/litellm-config.yaml --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout restart deploy litellm -n litellm
kubectl rollout status  deploy litellm -n litellm --timeout=120s
:::

Confirm both the fixed and the autoscaled model are served. Query LiteLLM through an in-cluster **port-forward** (no reliance on the `$DOMAIN` ingress host — if `$DOMAIN` is unset the URL collapses to `https://litellm./...` and `curl` returns an empty body, which then fails as `Expecting value: line 1 column 1`):

:::code{language=bash showCopyAction=true}
# Idempotent port-forward to LiteLLM (svc litellm:4000); safe to re-run
pgrep -f "port-forward.*4000:4000" >/dev/null || kubectl port-forward -n litellm svc/litellm 4000:4000 >/workshop/litellm_pf.log 2>&1 &
# Wait until LiteLLM answers before querying (avoids the empty-response error)
for i in $(seq 1 20); do curl -sf http://localhost:4000/v1/models -H "Authorization: Bearer sk-1234" >/dev/null 2>&1 && break; sleep 2; done

curl -s http://localhost:4000/v1/models -H "Authorization: Bearer sk-1234" \
  | python3 -c 'import sys,json; print([m["id"] for m in json.load(sys.stdin)["data"] if "qwen3-8b" in m["id"]])'
:::

::::expand{header="Check Output"}

```text
# both the fixed model and the autoscaled Ray model appear (fixed-model name varies by deployment)
['vllm/qwen3-8b-neuron', 'vllm/qwen3-8b-neuron-ray']
```

::::

::alert[**If you prefer the ingress URL** (`https://litellm.$DOMAIN/...`), first confirm `$DOMAIN` is exported in this shell (`echo $DOMAIN`) — it is set during infra setup but not always present in a fresh terminal. The port-forward above avoids that dependency entirely.]{type="info"}

### Try it in Open WebUI

Open WebUI is already deployed and wired to LiteLLM, so the autoscaled Ray model appears as a selectable model — no extra setup. Get the URL and the seeded admin login:

:::code{language=bash showCopyAction=true}
# Open WebUI URL (internet-facing ALB)
echo "http://$(kubectl get ingress openwebui -n openwebui -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

# Seeded admin login — prints the email + password in your own terminal
POD=$(kubectl get pods -n openwebui -o name | grep -iv redis | head -1)
kubectl exec -n openwebui "${POD#pod/}" -- printenv WEBUI_ADMIN_EMAIL WEBUI_ADMIN_PASSWORD
:::

1. Open the URL and **sign in** with the admin email + password from above.
2. In the **model selector** (top-left), choose **`vllm/qwen3-8b-neuron-ray`**.
3. Send a prompt and watch it respond — the request flows **Open WebUI → LiteLLM → Ray Serve → vLLM on Neuron**. Switch to the fixed-replica **`vllm/qwen3-8b-neuron`** to compare the two **side-by-side**.

In the model selector, `vllm/qwen3-8b-neuron-ray` appears alongside the other models (`bedrock/claude-4.5-sonnet`, the fixed `vllm/deepseek-r1-qwen3-8b-neuron`, etc.):

![Open WebUI model selector — vllm/qwen3-8b-neuron-ray listed among the available models](/static/images/track-a-opensource/a4-production-ready/webUI-ray-1.png)

Select it and chat — the response is served by the autoscaled Ray Serve deployment on Neuron:

![Open WebUI chat using vllm/qwen3-8b-neuron-ray — the model answers a "what is k8s?" prompt, served via Ray Serve → vLLM on Neuron](/static/images/track-a-opensource/a4-production-ready/webUI-ray-2.png)

## Part 6 — Watch it autoscale (1 → 2 → 1)

A single chat client never generates enough concurrency to cross `target_ongoing_requests`. Generate **sustained concurrent load** and watch the replica count climb and a second inf2 node appear.

In one terminal, watch the moving parts (Ctrl+C to stop):

:::code{language=bash showCopyAction=true}
while true; do clear; kubectl get pods -n vllm -l ray.io/group=neuron-group -o wide; echo; kubectl get nodeclaims | grep -E "NAME|neuron"; sleep 5; done
:::

**Before the load** — one worker pod (`RUNNING`) on its own inf2 node (a second inf2 node here belongs to the fixed-replica model, not Ray):

![Ray autoscale — before the load: a single neuron-group worker pod Running on one inf2 node](/static/images/track-a-opensource/a4-production-ready/ray-pods-before-scale.png)

In another terminal, keep the Part 4 port-forward running and drive load (well above `target × replicas` for longer than the upscale delay):

:::code{language=bash showCopyAction=true}
cat <<'EOF' > /workshop/ray-load.sh
#!/usr/bin/env bash
set -euo pipefail
DURATION="${1:-420}"; CONCURRENCY="${2:-12}"
ENDPOINT="http://localhost:8000/v1/chat/completions"
PROMPT="Explain Kubernetes horizontal pod autoscaling in about 300 words."
deadline=$(( SECONDS + DURATION ))
while [ "${SECONDS}" -lt "${deadline}" ]; do
  for _ in $(seq 1 "${CONCURRENCY}"); do
    curl -s -o /dev/null --max-time 180 "${ENDPOINT}" \
      -H 'Content-Type: application/json' \
      -d "{\"model\":\"qwen3-8b-neuron-ray\",\"messages\":[{\"role\":\"user\",\"content\":\"${PROMPT}\"}],\"max_tokens\":400}" &
  done
  wait
done
EOF
chmod +x /workshop/ray-load.sh
/workshop/ray-load.sh 420 12
:::

**What to observe:**
- Within ~30s of sustained load, Ray Serve requests a **2nd replica**. On a 2-core `inf2.xlarge` there are no free NeuronCores, so the Ray autoscaler adds a **2nd worker pod** (`Pending`).
- **Karpenter** provisions a **2nd `inf2.xlarge` node**; the worker schedules, the model loads, and the deployment runs **2 replicas**. `serve status` shows `RUNNING: 2` and `ray status` shows `4.0/4.0 neuron_cores` (2 nodes × 2 cores).
- When load stops, after `downscale_delay_s` (default 600s) Serve drops back toward `min_replicas: 1`; the emptied worker pod is removed and **Karpenter reclaims the extra node**.

**Scale-up in progress** — a **2nd worker pod appears** (`Init:0/1`) and **Karpenter adds a 2nd inf2 (spot) node** (`neuron-fx4g4`); the pod stays `Init` while the 18 GB image pulls:

![Ray autoscale — during scale-up: a 2nd neuron-group worker (Init:0/1) on a newly-provisioned inf2 spot node](/static/images/track-a-opensource/a4-production-ready/ray-pods-after-scale-1.png)

**Scale-up complete** — **both workers `Running`** on their own inf2 nodes (3 inf2 nodes total: 2 Ray workers + the fixed model's node):

![Ray autoscale — after scale-up: both neuron-group workers Running, each on its own inf2 node](/static/images/track-a-opensource/a4-production-ready/ray-pods-after-scale-2.png)

Confirm the same thing from the control plane — the pods, then the Ray **cluster** and **Serve** status:

:::code{language=bash showCopyAction=true}
kubectl get pod -n vllm | grep qwen3-ray

HEAD=$(kubectl get pods -n vllm -l ray.io/group=headgroup -o jsonpath='{.items[-1].metadata.name}')
kubectl exec -n vllm "$HEAD" -c ray-head -- /opt/rayvenv/bin/ray status
kubectl exec -n vllm "$HEAD" -c ray-head -- /opt/rayvenv/bin/serve status
:::

::::expand{header="Check Output — 2 workers up (4 NeuronCores), Serve finishing the 2nd replica"}

```text
$ kubectl get pod -n vllm | grep qwen3-ray
qwen3-ray-mf9cd-head-dxdpx                  2/2   Running   0   173m
qwen3-ray-mf9cd-neuron-group-worker-rjwl5   1/1   Running   0   149m
qwen3-ray-mf9cd-neuron-group-worker-xc7gn   1/1   Running   0   14m     # 2nd worker, just finished the image pull

# ray status  — 2 workers + head, all 4 NeuronCores now in the cluster
======== Autoscaler status: 2026-... ========
Node status
---------------------------------------------------------------
Active:
 2 neuron-group
 1 headgroup
Recent failures:
 neuron-group: NodeTerminated (instance_id: ...worker-jq2wc)   # a prior spot node was reclaimed earlier — expected on spot
Resources
---------------------------------------------------------------
Total Usage:
 2.0/4.0 CPU
 4.0/4.0 neuron_cores          # 2 workers × 2 cores
Pending Demands:
 (no resource demands)

# serve status — 3 HEALTHY proxies (head + 2 workers); the 2nd replica is still loading vLLM
proxies:
  e3097bf5...: HEALTHY
  9e1a335d...: HEALTHY
  d4766bb0...: HEALTHY
applications:
  qwen3-neuron:
    status: RUNNING
    deployments:
      qwen3-neuron:
        status: UPSCALING
        status_trigger: AUTOSCALING
        replica_states:
          RUNNING: 1
          STARTING: 1     # becomes RUNNING: 2 once vLLM finishes loading qwen3-8b on the new worker's cores
        message: "... 1 replicas that have taken more than 30s to initialize ..."
```

Note the `ray status` **`Recent failures: NodeTerminated`** line — that's a normal record of an earlier **spot** inf2 node being reclaimed and replaced; the Ray in-tree autoscaler simply provisions a fresh worker (which is why you may see a worker name change across the run).

::::

::::expand{header="Check Output — validated on inf2 (1 → 2 scale-up under load)"}

**During upscale** — `serve status` reports `UPSCALING` / `AUTOSCALING` while the 2nd replica starts, and `ray status` shows the pending NeuronCore demand the autoscaler is planning against:

```text
# serve status  (deployment while scaling)
        status: UPSCALING
        status_trigger: AUTOSCALING
        replica_states:
          RUNNING: 1
          STARTING: 1

# ray status  (the autoscaler's scale-up plan — needs the rayStartParams.resources fix)
 2.0/2.0 neuron_cores
Pending Demands:
 {'CPU': 1.0, 'neuron_cores': 2.0}: 1+ pending tasks/actors
```

**After the 2nd worker pulls the image, lands on a fresh `inf2.xlarge`, and loads the model** — two replicas serving, 3 HEALTHY proxies (head + 2 workers), and the full 4 NeuronCores allocated:

```text
# serve status
proxies:
  e3097bf5...: HEALTHY
  1d737b21...: HEALTHY
  9e1a335d...: HEALTHY
applications:
  qwen3-neuron:
    status: RUNNING
    deployments:
      qwen3-neuron:
        status: HEALTHY
        status_trigger: UPSCALE_COMPLETED
        replica_states:
          RUNNING: 2

# ray status
Total Usage:
 2.0/4.0 CPU
 4.0/4.0 neuron_cores      # 2 workers × 2 cores

# workers + nodes
$ kubectl get pods -n vllm -l ray.io/group=neuron-group
qwen3-ray-...-neuron-group-worker-jq2wc   1/1   Running
qwen3-ray-...-neuron-group-worker-rjwl5   1/1   Running
$ kubectl get nodes -l node.kubernetes.io/instance-type=inf2.xlarge   # 3 inf2 nodes (head is on a non-inf2 node)
```

::::

**Why the 2nd replica takes several minutes — watch the new worker's events.** The delay is almost entirely the **~18 GB image pull** on the freshly-provisioned inf2 node. Watch it directly:

:::code{language=bash showCopyAction=true}
# the newest neuron-group worker (the one being added)
NEWW=$(kubectl get pods -n vllm -l ray.io/group=neuron-group --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
kubectl describe pod "$NEWW" -n vllm | sed -n '/^Events:/,$p'
:::

::::expand{header="Check Output — the events that explain the wait (validated on inf2)"}

```text
Events:
  Type     Reason            Age    From                   Message
  ----     ------            ----   ----                   -------
  Warning  FailedScheduling  15m    default-scheduler      0/6 nodes are available: 2 Insufficient aws.amazon.com/neuroncore ...
  Normal   Nominated         15m    eks-auto-mode/compute  Pod should schedule on: nodeclaim/neuron-fx4g4      # Karpenter launches a new inf2 (spot) node
  Normal   Scheduled         15m    default-scheduler      Successfully assigned .../neuron-group-worker-xc7gn to i-06d12...
  Normal   Pulling           15m    kubelet                Pulling image "public.ecr.aws/.../ray-vllm-neuron:..."
  Normal   Pulled            5m     kubelet                Successfully pulled image "..." in 10m28.134s (10m28s including waiting). Image size: 17948094925 bytes   # ← the ~10-min wait: 17.9 GB
  Normal   Created           5m     kubelet                Created container: wait-gcs-ready
  Normal   Started           5m     kubelet                Started container wait-gcs-ready
  Normal   Pulled            5m     kubelet                Successfully pulled image "..." in 121ms   # ray-worker container reuses the just-pulled image (cached)
  Normal   Started           5m     kubelet                Started container ray-worker
```

The tell-tale line is **`Successfully pulled image … in 10m28s … Image size: 17948094925 bytes`** — the fresh node had to download the ~18 GB image (model weights + compiled Neuron artifacts baked in). Once pulled, `wait-gcs-ready` connects to the head in seconds and the vLLM subprocess loads qwen3-8b onto the NeuronCores (~2-3 min) before the replica flips `STARTING → RUNNING`.

::::

::alert[**Neuron scale-up is slower than GPU** because the ~18 GB image must be pulled on each fresh inf2 node (~10 min) before the 2nd replica can load. For a snappier demo, pre-warm a second inf2 node (or accept the first-pull latency once — subsequent scale events on a warm node are fast). This is the trade-off of baking the model+artifacts into the image (which in return makes the *model load itself* fast, with no runtime recompile).]{type="warning"}

::alert[**All three levels chained:** request load → Ray **Serve** adds a replica (`AUTOSCALING`) → the Ray **in-tree autoscaler** adds a worker pod (needs `enableInTreeAutoscaling`) → **Karpenter** provisions a 2nd `inf2.xlarge` — then reverses on idle. Because each replica needs a whole device (`neuron_cores: 2`), the mapping is a clean 1 replica : 1 pod : 1 node.]{type="success"}

## Part 7 — Ray + vLLM metrics in Grafana (via AMP)

The Ray Dashboard shows *logical* state; for **throughput, token-timing, queue depth, and NeuronCore utilization over time** use Grafana. Ray Serve publishes Ray metrics on the pods' `:8080` metrics port. This section is **fully self-contained** — it stands up the whole **Amazon Managed Prometheus (AMP) → in-cluster Grafana** pipeline (an AMP workspace, an AWS-managed scraper that already includes the Ray job, and Grafana with the AMP data source).

::alert[**Every step here is idempotent and re-run safe.** If you already stood up this stack in the [A3 → Infrastructure Monitoring](../../a3-scaling-evaluating/monitoring-scaling/) module, running these again just **reuses** the existing workspace/scraper/Grafana (each step checks-then-creates). So you can run this module standalone or after A3 — either works.]{type="success"}

### Step 1 — Create the AMP workspace (idempotent)

:::code{language=bash showCopyAction=true}
export AWS_REGION=$(aws configure get region || echo us-east-1)
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export CLUSTER_NAME=$(aws eks list-clusters --query 'clusters[0]' --output text)

# Reuse the workspace with this alias if it already exists, else create it
export AMP_WS_ID=$(aws amp list-workspaces --alias genai-platform-metrics \
  --query 'workspaces[0].workspaceId' --output text --region "$AWS_REGION" 2>/dev/null)
if [ -z "$AMP_WS_ID" ] || [ "$AMP_WS_ID" = "None" ]; then
  export AMP_WS_ID=$(aws amp create-workspace --alias genai-platform-metrics \
    --query workspaceId --output text --region "$AWS_REGION")
fi
aws amp wait workspace-active --workspace-id "$AMP_WS_ID" --region "$AWS_REGION"
export AMP_ENDPOINT=$(aws amp describe-workspace --workspace-id "$AMP_WS_ID" \
  --query 'workspace.prometheusEndpoint' --output text --region "$AWS_REGION")
echo "AMP workspace: $AMP_WS_ID"
echo "AMP endpoint:  $AMP_ENDPOINT"
:::

### Step 2 — Create the AMP managed scraper with the Ray job (idempotent)

First deploy the **`neuron-monitor`** DaemonSet — it exposes **NeuronCore hardware metrics** (utilization, memory, latency) in Prometheus format on `:8000` from every inf2 node, so Grafana can show physical accelerator usage alongside the Ray/vLLM request metrics (idempotent — `kubectl apply` is safe to re-run):

:::code{language=bash showCopyAction=true}
cd /workshop/workshops/eks-genai-workshop/static/code/module4/monitoring
kubectl apply -f neuron-monitor.yaml
kubectl rollout status daemonset/neuron-monitor -n kube-system --timeout=180s
:::

Now use the **AMP managed scraper** — AWS runs it (no ADOT/collector pods, no scrape IAM to manage). The scrape config has three jobs: **`ray`** (head + workers on `:8080`), **`vllm`** (`:8000`), and **`neuron-monitor`** (NeuronCore hardware metrics on `:8000` in `kube-system`):

:::code{language=bash showCopyAction=true}
cat > /workshop/ray-scraper-config.yaml <<'EOF'
global:
  scrape_interval: 30s
scrape_configs:
  - job_name: ray                       # Ray Serve + cluster metrics (head + workers)
    kubernetes_sd_configs:
      - role: pod
        namespaces: {names: [vllm]}
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_ray_io_node_type]
        action: keep
        regex: (head|worker)
      - source_labels: [__meta_kubernetes_pod_ip]
        target_label: __address__
        replacement: "$1:8080"
    metrics_path: /metrics
  - job_name: vllm                       # vLLM engine metrics
    kubernetes_sd_configs:
      - role: pod
        namespaces: {names: [vllm]}
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_ip]
        target_label: __address__
        replacement: "$1:8000"
    metrics_path: /metrics
  - job_name: neuron-monitor            # NeuronCore hardware metrics (utilization, memory, latency)
    kubernetes_sd_configs:
      - role: pod
        namespaces: {names: [kube-system]}
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: neuron-monitor
      - source_labels: [__meta_kubernetes_pod_ip]
        target_label: __address__
        replacement: "$1:8000"
    metrics_path: /metrics
EOF

CLUSTER_ARN=$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query 'cluster.arn' --output text)
SUBNETS=$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query 'cluster.resourcesVpcConfig.subnetIds[:2]' --output text | tr '\t' ',')
SG=$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' --output text)

# Create the scraper if it doesn't exist; if it already exists, UPDATE its scrape config
# (so re-running with a changed config — e.g. after adding the neuron-monitor job — actually
# applies the new config instead of silently keeping the old one).
EXISTING=$(aws amp list-scrapers --region "$AWS_REGION" --query 'scrapers[?alias==`genai-ray-vllm`].scraperId' --output text)
if [ -z "$EXISTING" ] || [ "$EXISTING" = "None" ]; then
  aws amp create-scraper --alias genai-ray-vllm \
    --source "eksConfiguration={clusterArn=$CLUSTER_ARN,subnetIds=[$SUBNETS],securityGroupIds=[$SG]}" \
    --destination "ampConfiguration={workspaceArn=arn:aws:aps:${AWS_REGION}:${AWS_ACCOUNT_ID}:workspace/${AMP_WS_ID}}" \
    --scrape-configuration "configurationBlob=$(base64 -i /workshop/ray-scraper-config.yaml)" \
    --region "$AWS_REGION"
else
  echo "scraper genai-ray-vllm exists ($EXISTING) — updating its scrape config"
  aws amp update-scraper --scraper-id "$EXISTING" \
    --scrape-configuration "configurationBlob=$(base64 -i /workshop/ray-scraper-config.yaml)" \
    --region "$AWS_REGION"
fi

# Poll until ACTIVE (first creation takes 5-10 min). A poll loop is used instead of
# `aws amp wait scraper-active` because that waiter caps its attempts and errors with
# "Waiter ScraperActive failed: Max attempts exceeded" before AMP finishes provisioning.
SID=$(aws amp list-scrapers --region "$AWS_REGION" --query 'scrapers[?alias==`genai-ray-vllm`].scraperId' --output text)
echo "Waiting for scraper $SID to become ACTIVE (5-10 min on first creation)..."
until [ "$(aws amp describe-scraper --scraper-id "$SID" --region "$AWS_REGION" --query 'scraper.status.statusCode' --output text 2>/dev/null)" = "ACTIVE" ]; do
  echo "  status: $(aws amp describe-scraper --scraper-id "$SID" --region "$AWS_REGION" --query 'scraper.status.statusCode' --output text 2>/dev/null) — waiting..."
  sleep 20
done
echo "✅ managed scraper ACTIVE — Ray/vLLM metrics now flow to AMP"
:::

::alert[**Managed scraper = zero in-cluster footprint.** AWS runs the scraper and handles scaling/HA; you only provide the scrape-config YAML. The scraper needs cluster access, which `aws amp create-scraper` sets up automatically for the EKS source. Metrics appear in AMP within ~30s of `ACTIVE`.]{type="info"}

### Step 3 — Deploy in-cluster Grafana with the AMP data source (idempotent)

The manifest provisions Grafana + the AMP data source (SigV4 via Pod Identity) in one apply. `kubectl apply`, the IAM `create-role`, and the Pod Identity association are all safe to re-run:

:::code{language=bash showCopyAction=true}
cd /workshop/workshops/eks-genai-workshop/static/code/module4/monitoring

# Grafana queries AMP → needs AMP *query* permission via Pod Identity (create-or-ignore)
aws iam create-role --role-name grafana-amp-reader \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"pods.eks.amazonaws.com"},"Action":["sts:AssumeRole","sts:TagSession"]}]}' 2>/dev/null || true
aws iam attach-role-policy --role-name grafana-amp-reader \
  --policy-arn arn:aws:iam::aws:policy/AmazonPrometheusQueryAccess

# Apply Grafana with the AMP endpoint + region substituted in.
sed -e "s#__AMP_ENDPOINT__#${AMP_ENDPOINT}#" -e "s#__REGION__#${AWS_REGION}#" \
  grafana.yaml | kubectl apply -f -

aws eks create-pod-identity-association --cluster-name "$CLUSTER_NAME" \
  --namespace grafana --service-account grafana \
  --role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/grafana-amp-reader 2>/dev/null || true
kubectl rollout restart deployment/grafana -n grafana
kubectl rollout status  deployment/grafana -n grafana --timeout=180s
:::

Confirm the Ray targets are being scraped — after the scraper is `ACTIVE` (~30s), a Grafana **Explore** query for `up{job="ray"}` shows the head + worker `:8080` targets at `up=1`.

### Step 4 — Import the Ray dashboards

Ray auto-generates its Grafana dashboard JSONs inside the head pod. Port-forward Grafana, then copy + import them wired to the AMP data source:

:::code{language=bash showCopyAction=true}
kubectl scale deploy grafana -n grafana --replicas=1
kubectl rollout status deploy grafana -n grafana --timeout=180s

# Start the Grafana port-forward (idempotent) — same pattern as B5's monitoring module
pgrep -f "port-forward.*3000:3000" >/dev/null || kubectl port-forward -n grafana svc/grafana 3000:3000 >/workshop/grafana-pf.log 2>&1 &
echo "Grafana: http://localhost:3000 (admin/admin)"

# Wait until Grafana answers before importing the dashboards (the import below calls its API)
GRAFANA="http://admin:admin@localhost:3000"
for i in $(seq 1 30); do curl -sf "$GRAFANA/api/health" >/dev/null 2>&1 && break; sleep 2; done

DS_UID=$(curl -s "$GRAFANA/api/datasources" | python3 -c 'import sys,json; print(next((d["uid"] for d in json.load(sys.stdin) if d["type"]=="prometheus"), ""))')

# Import the committed Ray dashboards from ray-dashboards/ (wired to the AMP data source).
# Each JSON is read from the repo, has ${datasource} swapped for the AMP UID, and is POSTed
# to Grafana — no pod exec, no /tmp temp files.
for j in serve_llm serve serve_deployment default; do
  python3 - "ray-dashboards/${j}_grafana_dashboard.json" "$DS_UID" <<'PY' \
    | curl -s -H "Content-Type: application/json" -d @- "$GRAFANA/api/dashboards/db" >/dev/null && echo "imported: ${j}"
import sys, json
dash = json.loads(open(sys.argv[1]).read().replace("${datasource}", sys.argv[2]))
dash.pop("id", None); dash["uid"] = None
print(json.dumps({"dashboard": dash, "overwrite": True, "folderId": 0}))
PY
done
:::

> The Ray dashboard JSONs are committed under `ray-dashboards/` (snapshotted from Ray's auto-generated dashboards), so the import reads repo files directly — no need to `kubectl exec` into the head pod.

::alert[**Opening Grafana.** Use the Code Editor's built-in port forwarding — open **`http://localhost:3000`** directly (in the Code Editor's Simple Browser, or click the auto-forwarded-port notification / the **PORTS** panel globe icon). VS Code intercepts `localhost:3000` and proxies it to the port-forward. Log in with **admin / admin**.

**Do not** use the CloudFront `…/ports/3000/` path for Grafana — that sub-path proxy makes Grafana redirect to `localhost` and fail to load. (This is the same access pattern as [B5 → Infrastructure Monitoring](/track-b-saas/b5-production-ready/monitoring-scaling/).)]{type="info"}

### Step 5 — Read it, before and after load

Open the **Serve** and **Serve LLM** dashboards (set the range to **Last 30 minutes**), then drive load as in Part 6 and watch:

- **Before load (idle):** request rate `0`, healthy replicas `1`, queue depth `0`.
- **Under load:** request rate rises, router queue depth climbs, in-flight saturates at `max_ongoing_requests` (8), **Healthy replicas `1 → 2`**, and `neuron_cores` allocated goes `2 → 4`.

::alert[**Neuron hardware metrics:** the **`neuron-monitor`** DaemonSet you deployed in Step 2 (the Neuron equivalent of DCGM) exports physical **NeuronCore utilization / memory / latency**, and the scraper's `neuron-monitor` job pulls those into AMP alongside the Ray/vLLM metrics — so you can chart accelerator utilization next to request-level throughput. Explore them in Grafana with `neuroncore_utilization_ratio` (or import the AWS Neuron Grafana dashboard). The Ray/vLLM Serve dashboards above cover request-level throughput, token timing, and replica counts.]{type="info"}

## Troubleshooting — issues found while bringing this up (and their fixes)

All baked into the manifest/image above, but worth knowing when adapting it:

| Symptom | Root cause | Fix (already applied above) |
|---|---|---|
| Every route returns **HTTP 500** (`'_IncludedRouter' object has no attribute 'path'`) | `ray[serve]==2.58` pulled `starlette 1.x`/`fastapi 0.141`, which breaks vLLM 0.10.2's OpenAI server | **Isolate Ray in a venv** (`/opt/rayvenv`); vLLM keeps the image's original `starlette`/`fastapi` |
| Worker raylet crashes: `Attempting to start raylet with 2 neuron_cores, but NEURON_RT_VISIBLE_CORES contains ['0-1']` | Ray parses `NEURON_RT_VISIBLE_CORES` as a **comma list** (`0,1` → 2 cores); optimum-neuron needs the **range** (`0-1`) | Set container env `0,1` (Ray) and have `vllm_serve.py` **convert to `0-1`** for the vLLM subprocess env |
| Autoscaler sidecar **crash-loops**: `JSONDecodeError: Extra data: line 1 column 15` while reading the worker group | The KubeRay autoscaler does `resources_string[1:-1]` (strips an assumed outer quote pair) **before** `json.loads`, so a bare `'{"neuron_cores": 2}'` becomes invalid JSON | Wrap the value in an **extra outer quote pair**: `rayStartParams.resources: '"{\"neuron_cores\": 2}"'` (canonical `ray-cluster.complete.yaml` form) — serving works either way, but scale-up needs the healthy autoscaler |
| vLLM exits `rc=1`: `tensor parallelism degree (2) is higher than the number of available Neuron cores (1)` | `NEURON_RT_VISIBLE_CORES=0,1` (comma) made optimum-neuron count **1** core | vLLM subprocess must get the **range `0-1`** (the conversion above) |
| Head/worker probe fails: `wget: command not found` / `curl: command not found` | The Neuron base image has neither `wget` nor `curl` | Use **python-urllib** probes against the raylet healthz (`:52365`) |
| `curl` to `localhost:8000` returns empty → `json.tool`: `Expecting value: line 1 column 1 (char 0)` | No live port-forward on `:8000` (not started in this shell, or a `svc/...serve-svc` forward dropped with `lost connection to pod`) | Port-forward to the **head pod** (`pod/$HEAD`, not the service) and **wait for `/v1/models` to answer** before curling — see Part 4 |
| Ray import fails: `No module named 'jinja2'` | `ray[serve]` needs `jinja2`, not pulled by default in the venv | Add `jinja2` to the venv `pip install` |
| `python3 -m venv` can't create the venv (no `ensurepip`) | The base image's Python lacks `ensurepip` | Seed the venv with **`virtualenv`** instead |
| In-cluster (Kaniko) build evicted: `node.kubernetes.io/disk-pressure` | The ~18 GB base overflows ephemeral storage during snapshot | Give the build pod `ephemeral-storage: 60Gi` + `--single-snapshot` |
| New Ray head evicted `Underutilized` mid-deploy | Karpenter consolidation disrupts the head node | `karpenter.sh/do-not-disrupt: "true"` on head + worker |

## Cleanup

Remove the autoscaled model and its inf2 node (the original `qwen3-8b-neuron` deployment is unaffected):

:::code{language=bash showCopyAction=true}
kubectl delete rayservice qwen3-ray -n vllm --wait=false
pkill -f "port-forward.*8000:8000" 2>/dev/null || true
pkill -f "port-forward.*8265:8265" 2>/dev/null || true
:::

Karpenter reclaims the inf2 worker node once the pods are gone. Remove the LiteLLM `model_list` entry you added in Part 5 to drop it from Open WebUI. Leave the KubeRay operator installed if you plan to revisit.

## Appendix — the image

The image layers `ray[serve]` (in an isolated venv) onto the workshop's inf2 vLLM image + the `vllm_serve.py` proxy. Both live in the starter-kit under `components/llm-model/vllm-ray/`; maintainers rebuild + push with `build-and-push.sh <registry-alias>` (amd64-only — the Neuron base has no arm64).

`Dockerfile`:

:::code{language=docker showCopyAction=true}
FROM public.ecr.aws/agentic-ai-platforms-on-k8s/vllm-neuron:qwen3-8b-optimum-neuron
USER root
# Base Python has no ensurepip -> seed the venv with virtualenv. Ray goes in its OWN venv so it
# doesn't upgrade the fastapi/starlette that vLLM 0.10.2 depends on.
RUN python3 -m pip install --no-cache-dir virtualenv && \
    virtualenv /opt/rayvenv && \
    /opt/rayvenv/bin/pip install --no-cache-dir 'ray[serve]==2.58.0' 'httpx>=0.27' jinja2
ENV PATH=/opt/rayvenv/bin:$PATH
COPY vllm_serve.py /home/ray/vllm_serve.py
RUN PYTHONPATH=/home/ray /opt/rayvenv/bin/python -c "import ray; from ray import serve; import vllm_serve; print('rayvenv OK')"
:::

`vllm_serve.py` is a `@serve.deployment` + `@serve.ingress` OpenAI proxy: each replica launches the workshop's exact `vllm serve /root/.cache/neuron/Qwen/Qwen3-8B --tensor-parallel-size 2 --max-num-seqs 2 --max-model-len 8192 …` as a subprocess (converting `NEURON_RT_VISIBLE_CORES` to the range form first) and forwards `/v1/*` to it. Full source in the starter-kit.

---

**Track A complete.** You now run the self-hosted Neuron model with **demand-driven autoscaling** on KubeRay — replicas and inf2 nodes follow live request load instead of holding an accelerator 24/7.
