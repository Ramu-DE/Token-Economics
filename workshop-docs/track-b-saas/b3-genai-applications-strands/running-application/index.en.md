---
title: "Deploy & Run the Loan Buddy Agent"
weight: 30
---

# Deploy Loan Buddy (Strands)

In this section, you deploy the Strands Loan Buddy and observe it process John Michael Doe's loan application: **Strands agent → Kong `/loan-strands` → Bedrock**, with the 3 MCP tools and full tracing in **Arize AX**.

![John's Application Ready for Processing](/static/images/shared/example1.png)

*John's application, ready for processing.*

## Prerequisites Check

Before deploying, confirm the Track B stack platform from Module B2 is ready.

::alert[**Interactive zsh note:** if pasting fails with `command not found: #` or `unknown username`, run `setopt interactive_comments` once — zsh does not treat `#` as a comment by default. The blocks below avoid inline comments to be safe.]{type="info"}

Kong `/loan-strands` route works (expect `401` without a key = key-auth active):

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
curl -s -o /dev/null -w "kong /loan-strands no-key: HTTP %{http_code} (expect 401)\n" \
  -X POST "$KONG_PROXY/loan-strands" -H "Content-Type: application/json" -d '{"messages":[{"role":"user","content":"hi"}]}'
:::

Arize credentials installed:

:::code{language=bash showCopyAction=true}
kubectl get secret arize-credentials -n arize
:::

### Deploy the Kong Bedrock vision route (native multimodal)

Kong's AI Gateway (`ai-proxy` / `ai-proxy-advanced`) translates **text** and **tool-calling** between the OpenAI Chat Completions API and Amazon Bedrock — that is why the Strands agent's reasoning routes cleanly through Kong. It does **not**, however, translate an OpenAI `image_url` block into a Bedrock `image` block, so **OpenAI-format** multimodal requests fail. Kong **does** support Bedrock vision through its **native pass-through** (`llm_format: bedrock`): you send a native Bedrock **Converse** payload (with a real `image` block) and Kong forwards it to Bedrock's `/converse`, signing with the DataPlane's **IAM role** (no keys).

Deploy that native vision route (`/vision`). It is independent of `/loan-strands` and the agent.

**1. KongService** (dummy upstream — the AI plugin intercepts before proxying):

:::code{language=bash showCopyAction=true}
if kubectl get kongservice bedrock-vision-service -n kong >/dev/null 2>&1; then
  echo "✅ KongService 'bedrock-vision-service' already exists — skipping"
else
cat <<EOF | kubectl apply -f -
apiVersion: configuration.konghq.com/v1alpha1
kind: KongService
metadata:
  name: bedrock-vision-service
  namespace: kong
spec:
  name: bedrock-vision-service
  host: localhost
  port: 80
  controlPlaneRef:
    type: konnectNamespacedRef
    konnectNamespacedRef:
      name: kong-aws
EOF
fi
:::

**2. `ai-proxy-advanced` plugin** with `llm_format: bedrock` (native Converse pass-through; DataPlane IAM, no keys):

:::code{language=bash showCopyAction=true}
if kubectl get kongplugin ai-proxy-advanced-bedrock-vision -n kong >/dev/null 2>&1; then
  echo "✅ KongPlugin 'ai-proxy-advanced-bedrock-vision' already exists — skipping"
else
cat <<EOF | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-bedrock-vision
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  llm_format: bedrock
  genai_category: text/generation
  targets:
  - route_type: llm/v1/chat
    auth:
      allow_override: false
    model:
      provider: bedrock
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      options:
        bedrock:
          aws_region: us-east-1
EOF
fi
:::

**3. KongRoute** `/vision` (with the plugin attached):

:::code{language=bash showCopyAction=true}
if kubectl get kongroute bedrock-vision-route -n kong >/dev/null 2>&1; then
  echo "✅ KongRoute 'bedrock-vision-route' already exists — skipping"
else
cat <<EOF | kubectl apply -f -
apiVersion: configuration.konghq.com/v1alpha1
kind: KongRoute
metadata:
  name: bedrock-vision-route
  namespace: kong
  annotations:
    konghq.com/plugins: ai-proxy-advanced-bedrock-vision
spec:
  name: bedrock-vision-route
  paths:
  - /vision
  protocols:
  - http
  - https
  serviceRef:
    type: namespacedRef
    namespacedRef:
      name: bedrock-vision-service
EOF
fi
:::

**4. Confirm it programmed, then test vision** (native Converse `image` block → `/vision/converse`):

:::code{language=bash showCopyAction=true}
kubectl get kongroute bedrock-vision-route -n kong

KONG="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
B64=$(base64 -w0 /workshop/workshops/eks-genai-workshop/static/code/module3/credit-validation/example1.png)
cat > /tmp/converse.json <<JSON
{"messages":[{"role":"user","content":[
  {"text":"What is in this image? Reply in one sentence."},
  {"image":{"format":"png","source":{"bytes":"${B64}"}}}]}],
 "inferenceConfig":{"maxTokens":200}}
JSON
curl -s -X POST "$KONG/vision/converse" -H "Content-Type: application/json" -d @/tmp/converse.json
:::

::::expand{header="Check Output"}

```text
NAME                   PROGRAMMED   AGE
bedrock-vision-route   True         *

{"output":{"message":{"role":"assistant","content":[{"text":"The image shows a completed personal
loan application form for John Michael Doe requesting $8,500 for home improvement, dated
August 21, 2024."}]}},"stopReason":"end_turn","usage":{...}}
```

::::

::alert[**Track B uses a Kong-native image-processor — no LiteLLM.** Track A's image-processor sends OpenAI `image_url` through LiteLLM; Track B ships a small **custom image-processor** (`examples/mcp-server/image-processor/`) whose vision call is a **native Bedrock Converse** request to the Kong `/vision` route above. The other two MCPs (address + employment validators) are **deterministic** — they make no model calls — so they're reused from Track A's image unchanged. Net: **every Track B AI call goes through Kong → Bedrock; LiteLLM is not used.**]{type="success"}

### Deploy the Track B image-processor MCP (Kong-native)

**1. Namespace + S3-enabled service account** (the image-processor reads the uploaded document from S3):

:::code{language=bash showCopyAction=true}
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
BUCKET=$(aws s3 ls | grep -oE 'genai-on-eks-bucket-langfuse-[a-z0-9]+' | head -1)

# Namespace (kubectl apply is idempotent — safe to re-run)
kubectl create namespace workshop --dry-run=client -o yaml | kubectl apply -f -

# S3-enabled service account (copied from langfuse) — create only if missing
if kubectl get sa langfuse -n workshop >/dev/null 2>&1; then
  echo "✅ ServiceAccount 'langfuse' already exists in namespace workshop"
else
  kubectl get sa langfuse -n langfuse -o yaml | sed 's/namespace: langfuse/namespace: workshop/' | kubectl apply -f -
  echo "✅ Created ServiceAccount 'langfuse' in namespace workshop"
fi

# Pod Identity association (S3 access) — create only if missing
if aws eks list-pod-identity-associations --cluster-name genai-on-eks --namespace workshop \
     --query "associations[?serviceAccount=='langfuse'].associationId" --output text 2>/dev/null | grep -q .; then
  echo "✅ Pod Identity association for workshop/langfuse already exists"
else
  aws eks create-pod-identity-association --cluster-name genai-on-eks --namespace workshop \
    --service-account langfuse \
    --role-arn arn:aws:iam::${ACCOUNT}:role/genai-on-eks-us-east-1-langfuse-s3-access
  echo "✅ Created Pod Identity association for workshop/langfuse"
fi
:::

**2. Deploy it** (Deployment + Service; `KONG_VISION_URL` points at the Kong `/vision` route — no LiteLLM key). The image is a **pre-built multi-arch** public image — no build on the Code Editor, no per-participant ECR:

::alert[**Image source:** this pulls `public.ecr.aws/agentic-ai-platforms-on-k8s/mcp-server-image-processor:latest`, published centrally (multi-arch) from `examples/mcp-server/image-processor/` via `examples/build-ecr-images.sh`. If your environment targets a fork whose image is not yet published, publish it first (run `examples/build-ecr-images.sh`) or the pod will `ImagePullBackOff`.]{type="info"}

:::code{language=bash showCopyAction=true}
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-image-processor
  namespace: workshop
spec:
  replicas: 1
  selector:
    matchLabels: { app: mcp-image-processor }
  template:
    metadata:
      labels: { app: mcp-image-processor }
    spec:
      serviceAccountName: langfuse
      containers:
      - name: mcp-image-processor
        image: public.ecr.aws/agentic-ai-platforms-on-k8s/mcp-server-image-processor:latest
        command: ["python", "/app/server.py"]
        ports:
        - containerPort: 8000
        env:
        - { name: KONG_VISION_URL, value: "http://proxy1.kong.svc.cluster.local/vision" }
        - { name: S3_BUCKET_NAME, value: "${BUCKET}" }
        - { name: AWS_REGION, value: "us-east-1" }
---
apiVersion: v1
kind: Service
metadata:
  name: mcp-image-processor
  namespace: workshop
spec:
  selector: { app: mcp-image-processor }
  ports:
  - port: 8000
    targetPort: 8000
EOF
kubectl rollout status deploy/mcp-image-processor -n workshop --timeout=180s
:::

### Deploy the 2 validator MCP servers

The **address** and **employment** validators make no model calls (deterministic), so they're reused from Track A's shared image unchanged — just fill the S3 config:

:::code{language=bash showCopyAction=true}
cd /workshop/workshops/eks-genai-workshop/static/code/module3/credit-validation

# Keep ONLY the address + employment validator Deployment/Service pairs (drop the agent + image-processor)
python3 - "$BUCKET" <<'PY'
import sys
bucket = sys.argv[1]; q = chr(34)
docs = open('agentic-application-deployment.yaml').read().split('\n---')
keep = [d for d in docs if d.strip() and 'loan-buddy-agent' not in d and 'mcp-image-processor' not in d]
lines = ('\n---'.join(keep)).split('\n')
for i in range(len(lines) - 1):
    if 'name: S3_BUCKET_NAME' in lines[i]: lines[i+1] = lines[i+1].replace(q+'<>'+q, q+bucket+q)
    if 'name: AWS_REGION'     in lines[i]: lines[i+1] = lines[i+1].replace(q+'<>'+q, q+'us-east-1'+q)
open('/tmp/mcp-validators.yaml', 'w').write('\n'.join(lines))
print('Wrote /tmp/mcp-validators.yaml')
PY
kubectl apply -f /tmp/mcp-validators.yaml
kubectl rollout status deploy/mcp-address-validator -n workshop --timeout=120s
:::

### Verify the platform is ready

Confirm the Kong route enforces auth, the Arize credentials exist, and the 3 MCP services are up:

:::code{language=bash showCopyAction=true}
kubectl get secret arize-credentials -n arize
kubectl get svc -n workshop | grep -E "mcp-image-processor|mcp-address-validator|mcp-employment-validator"
:::

::::expand{header="Check Output"}

```text
NAME                TYPE     DATA   AGE
arize-credentials   Opaque   3      *h

mcp-address-validator      ClusterIP   172.20.x.x   <none>   8000/TCP   *
mcp-employment-validator   ClusterIP   172.20.x.x   <none>   8000/TCP   *
mcp-image-processor        ClusterIP   172.20.x.x   <none>   8000/TCP   *
```

::::

::alert[**Platform Required**: Before deploying the agent, confirm the Kong `/loan-strands` route returns `HTTP 401` without a key (from [Module B2](/track-b-saas/b2-ai-gateway-observability/)), `arize-credentials` exists in the `arize` namespace (from [B1/B2](/track-b-saas/b1-platform-setup/)), and the 3 `mcp-*` services are `Running` in the `workshop` namespace (deployed above). MCP is framework-agnostic — the Strands agent connects to these servers unchanged.]{type="warning"}

## Step 1: Understanding the Configuration

Before deploying, review what the Strands Loan Buddy needs to connect to the Track B stack platform:

### Required Configuration

1. **Kong route + key** - `KONG_BASE_URL` (`http://<kong-proxy-lb>/loan-strands`) + the consumer key (`apikey: loan-strands-key-123`) from Module B2
2. **Arize credentials** - the `arize-credentials` Secret (Service key + Space ID), mounted into the agent
3. **S3 Bucket** - where to store the uploaded loan document (reuses the langfuse bucket)
4. **AWS Region** - `us-east-1`
5. **Service Account with S3 Access** - EKS Pod Identity granting **S3** (not Bedrock — Bedrock is reached through Kong)

::alert[**Difference from the default Loan Buddy:** the default agent's pod holds Bedrock IAM and a LiteLLM Virtual Key. The Strands agent holds **only S3 IAM** — all Bedrock access goes through Kong's DataPlane (which has the Bedrock IAM), and auth to Kong is the `apikey` header. Fewer secrets in the app.]{type="info"}

## Step 2: Automated Deployment

### Run the Deployment Command

The starter-kit CLI builds the image, wires Kong + Arize + MCP + S3, and deploys into the `strands-agents` namespace:

:::code{language=bash showCopyAction=true}
cd /workshop
./cli strands-agents loan-buddy-agent install
:::

### What the Command Does

::::expand{header="Under the Hood: What Is Being Automated"}

1. **Terraform apply** — creates an **S3 Pod Identity** association for the agent (it needs S3 to store the uploaded image; Bedrock access is via Kong, not the agent). No ECR repo — the agent runs a pre-built multi-arch public image.
2. **Pulls the pre-built agent image** `public.ecr.aws/agentic-ai-platforms-on-k8s/strands-agents-loan-buddy-agent:latest` (published centrally; no build on the Code Editor).
3. **Resolves the Kong proxy LB** hostname → sets `KONG_BASE_URL=http://<lb>/loan-strands`.
4. **Copies the `arize-credentials` Secret** from the `arize` namespace into `strands-agents`.
5. **Renders + applies** the Deployment / Service / ServiceAccount.

**The command does all of this in one step** — the same automation pattern used in production CI/CD pipelines.

::::

Wait for the deployment to be ready:

:::code{language=bash showCopyAction=true}
kubectl rollout status deployment/loan-buddy-agent -n strands-agents
kubectl get pods -n strands-agents -l app=loan-buddy-agent
:::

### Expected Output

:::code{language=text showCopyAction=false}
deployment "loan-buddy-agent" successfully rolled out
NAME                                READY   STATUS    RESTARTS   AGE
loan-buddy-agent-xxxxxxxxx-xxxxx    1/1     Running   0          1m
:::

## Step 3: Monitor the Agent

### Set Up Real-Time Monitoring

Open a second terminal to follow the agent's decision process:

:::code{language=bash showCopyAction=true}
# In a NEW terminal, watch the agent logs
kubectl logs -f deployment/loan-buddy-agent -n strands-agents
:::

**Keep this terminal visible** - the agent's decisions appear in the logs as it processes requests.

### Expose the Application

Back in your main terminal:

:::code{language=bash showCopyAction=true}
# Idempotent — start the port-forward only if 8890 isn't already forwarding (safe to re-run in any module)
pgrep -f "port-forward.*8890:8080" >/dev/null || kubectl port-forward service/loan-buddy-agent 8890:8080 -n strands-agents >/tmp/loanbuddy-pf.log 2>&1 &

# use the same sample application image as Track A's Module A2
cd /workshop/workshops/eks-genai-workshop/static/code/module3/credit-validation/
:::

::alert[**API-Only Service**: Loan Buddy is a FastAPI REST server — no web UI. `http://localhost:8890/` returns `{"detail":"Not Found"}` (normal); Swagger is at `http://localhost:8890/docs`.]{type="info"}

::alert[**If `port-forward` disconnects** (`lost connection to pod`): the cluster autoscaler may consolidate nodes and reschedule the pod. Wait for the pod to be `Running` again (`kubectl get pods -n strands-agents -l app=loan-buddy-agent`) and re-run the `port-forward`. A single processing request takes **60-90 seconds** (multiple LLM turns + 7 tool calls), so keep the connection open until it returns.]{type="info"}

## Step 4: Process John's Loan Application

### Submit John's Application

:::code{language=bash showCopyAction=true}
curl -s --max-time 180 -X POST \
  -H "X-Consumer-ID: loan-strands-agent" \
  -H "X-Consumer-ApiKey: loan-strands-key-123" \
  -F "image_file=@./example1.png" \
  http://localhost:8890/api/process_credit_application_with_upload \
  -o /tmp/loan_strands.json

python3 -c "import json;d=json.load(open('/tmp/loan_strands.json'));print('status:',d['status']);print();print(d.get('credit_assessment') or d.get('message',''))"
:::

::alert[**Allow 60-90 seconds.** The agent runs a multi-step loop (extract → validate address/employment/ownership → decide) with several LLM round-trips through Kong, so the request is expected to take about a minute. `--max-time 180` keeps `curl` from timing out early.]{type="info"}

::alert[**Model note:** Kong pins Claude Sonnet 4.5 and injects it (`allow_override: false`), so the agent never sends a model name — the gateway controls model + cost. Tool-calling works through Kong's `ai-proxy` (OpenAI `tools` ↔ Bedrock `toolConfig` translation is handled at the gateway).]{type="info"}

### Observe the Processing

**In your log terminal, you will see** the agent store the image in S3, load the MCP tools, then drive the loop — extract data → validate address → validate employment → decide:

The logs are verbose (each reasoning cycle prints a LiteLLM call, and the full metrics trace is long). The **key lines** to look for:

```text
INFO:loan-buddy-strands:🔄 Starting credit application processing (consumer=loan-strands-agent)...
INFO:utils:Successfully stored content to s3://genai-on-eks-bucket-langfuse-.../362d6c5f...
INFO:loan-buddy-strands:✅ Image stored in S3 with ID: 362d6c5f...
INFO:loan-buddy-strands:🔧 Loading MCP tools...
INFO:loan-buddy-strands:Available tools: ['extract_credit_application_data', 'validate_document_authenticity', 'validate_address', 'perform_address_fraud_check', 'verify_address_ownership', 'validate_income_employment', 'check_employment_stability']
INFO:loan-buddy-strands:🤖 Processing with Strands agent...

# one Bedrock (Claude Sonnet 4.5) call per reasoning cycle, routed through Kong:
INFO:LiteLLM: LiteLLM completion() model=global.anthropic.claude-sonnet-4-5-20250929-v1:0; provider=openai
... (repeats once per cycle) ...

# the model drives the tool loop, in the order IT decides:
Tool #1: extract_credit_application_data
Tool #2: validate_document_authenticity
Tool #3: validate_income_employment
Tool #4: check_employment_stability
Tool #5: validate_address
Tool #6: perform_address_fraud_check
Tool #7: verify_address_ownership

INFO:loan-buddy-strands:stop_reason=end_turn
INFO:loan-buddy-strands:metrics summary: {'total_cycles': 5, ... 'success_rate': 1.0, 'error_count': 0 ...}
```

::alert[The agent drives the loop itself (model-first): it calls the tools in the order **it** decides — typically extract + authenticate first, then employment/income, then address checks — across ~5 event-loop cycles. The `metrics summary` line reports per-tool call counts and success rates (all `1.0`, `error_count: 0` on a healthy run).]{type="info"}

### Review the Response

The command prints `status: COMPLETED` followed by the full `credit_assessment` — a long, multi-section markdown report. The **key parts** (trimmed; the model's exact wording and some numbers vary per run):

:::code{language=text showCopyAction=false}
status: COMPLETED

# COMPREHENSIVE CREDIT ASSESSMENT REPORT

## Application Summary
**Applicant:** John Michael Doe  |  **Loan:** $8,500 (Home Improvement)  |  **Income:** $75,000/yr

## Verification Results (key points)
- Document authenticity: 72/100 — REVIEW (typed signature; non-standard SSN formatting)
- Employment & income: VERIFIED / PASSED — Tech Solutions Inc, Software Engineer, $75,000 (0% variance)
- Employment stability: STABLE (85/100, 3.5 yrs)
- Address: VALID (100/100, risk LOW) — 123 Main Street, Anytown, CA 90210
- Fraud check: PASSED (fraud score 0, LOW)

## FINAL CREDIT DECISION
### RECOMMENDATION: CONDITIONAL APPROVAL
Strong fundamentals (verified income/employment, 11.3% loan-to-income), but the typed
signature and non-standard SSN formatting lower the document authenticity score, so
conditions are attached: obtain a wet signature, verify the full SSN, request pay stubs.

... (full multi-section report — applicant info, risk analysis, financial analysis,
     conditions, next steps — trimmed here for brevity) ...
:::

::alert[**`CONDITIONAL APPROVAL` is the expected result** with Claude Sonnet 4.5. The model approves on the strong fundamentals (verified income/employment, low 11.3% loan-to-income) but flags the document's typed signature and non-standard SSN formatting as authenticity concerns — so it attaches conditions rather than a clean approval. This is the model reasoning correctly about the sample document, not a fault. The full `credit_assessment` is a multi-section markdown report (applicant info, verification results, risk analysis, decision, next steps).]{type="info"}

## Step 5: Explore the Complete Workflow in Arize AX

### Open Arize AX

1. Open [app.arize.com](https://app.arize.com) and select your Space
2. Navigate to **Tracing Projects** → project **`loan-strands`**
3. Open the **Traces** tab

::alert[**First time you open the project, Arize prompts "Enable Signal for this project?"** **Signal** is Arize AX's automated monitoring agent: on a schedule it reviews your project's traces, **auto-triages** them, **always watches** for regressions, and surfaces **ranked issues with linked evidence and actionable next steps** — so you don't have to manually scan traces. The sweeps run on **Claude Code** (pick the **AI Provider** — leave it on **Arize managed (default)**), and your trace data may be processed by Anthropic as part of each sweep. Click **Enable Signal** to turn it on for `loan-strands` (or **Not now** to skip — you can enable it later from the project's **Signal** tab).]{type="info"}

![Arize AX — "Enable Signal for this project?" dialog on the loan-strands project: Automatic triage, Always watching, Actionable next steps, AI Provider = Arize managed (default), click Enable Signal](/static/images/track-b-saas/b3-genai-applications-strands/arize-enable-signals.png)

### Find John's Application Trace

Look for the most recent trace — the top-level **AGENT** span named **`invoke_agent Strands Agents`** (Status `OK`, latency ~60s; carries the `session.id = loan-buddy` and `tag.tags` attributes we set). Click it to expand the span tree.

### What You Will See in the Trace

**The trace shows** the full nested tree with proper OpenInference span kinds:
- **AGENT** - `invoke_agent Strands Agents` (the root run), with the user prompt as **Input** and the final assessment as **Output**
- **CHAIN** - `execute_event_loop_cycle` spans (one per reasoning cycle, ~4)
- **LLM** - `chat` spans, each a Bedrock (Claude Sonnet 4.5) call through Kong, with prompt/completion **token counts**
- **TOOL** - one `execute_tool ...` span per MCP call (all 7: extract, authenticate, validate address/employment, fraud check, ownership) with the tool **arguments** as Input and the **JSON result** as Output
- **Performance** - latency per span; **token usage** per LLM span; full prompts/arguments/results on the **Input / Output** tab

![Arize AX trace for John's loan application — the invoke_agent Strands Agents span tree (execute_event_loop_cycle CHAIN, chat LLM, execute_tool TOOL spans) with Status OK, latency, token counts, trace evals, and the Input/Output panel](/static/images/track-b-saas/b3-genai-applications-strands/arize-trace-output.png)

This is the Arize-AX equivalent of the Langfuse trace in Track A's Module A2 — **same workflow, different (SaaS) observability backend**.

::alert[**Why the span content is populated:** the agent registers Arize with the `StrandsAgentsToOpenInferenceProcessor`, which converts Strands' native OpenTelemetry `gen_ai.*` spans into the OpenInference layout Arize renders — giving you the AGENT/CHAIN/LLM/TOOL span kinds and the filled Input/Output tabs. Without that processor the spans still arrive, but with no span kind and an empty Input/Output.]{type="info"}

## Step 6: Try Different Scenarios (Optional)

:::code{language=bash showCopyAction=true}
# Process a second application
curl -s --max-time 180 -X POST \
  -H "X-Consumer-ID: loan-strands-agent" \
  -H "X-Consumer-ApiKey: loan-strands-key-123" \
  -F "image_file=@./example2.png" \
  http://localhost:8890/api/process_credit_application_with_upload -o /tmp/loan2.json
python3 -c "import json;d=json.load(open('/tmp/loan2.json'));print('status:',d['status']);print(d.get('credit_assessment') or d.get('message',''))"
:::

## Summary

### What Was Accomplished

You have:
- **Deployed** a Strands-based loan processing application on Kubernetes
- **Processed** John's loan application in under a minute
- **Routed all AI through Kong** (`ai-proxy` → Bedrock) — no LLM keys in the app
- **Traced the full workflow in Arize AX** (AGENT/LLM/TOOL spans)
- **Reused the same 3 MCP servers** as the default stack

### How this compares to the default Loan Buddy

| | Track A · Module A2 (LangChain) | Track B · Module B3 (Strands) |
|---|---|---|
| Deploy | `./deploy-workshop-app.sh` | `./cli strands-agents loan-buddy-agent install` |
| Agent framework | LangChain + LangGraph (`create_react_agent`) | **Strands Agents SDK** |
| LLM access | LiteLLM `sk-` Virtual Key → Bedrock | Kong `apikey` → `ai-proxy` → Bedrock |
| App IAM | Bedrock + S3 | **S3 only** (Bedrock via Kong) |
| Trace | Langfuse `credit_underwriting_agent_with_image_id` | Arize AX project `loan-strands` |
| Tools | 3 MCP servers (SSE) | **same 3 MCP servers** |

## Key Takeaways

You have learned how to:
1. **Build agentic apps** with the Strands SDK
2. **Route all AI traffic through Kong AI Gateway** (SaaS-managed, K8s-native config)
3. **Reuse MCP servers** across agent frameworks (MCP is the interop layer)
4. **Trace with Arize AX** (OpenInference) instead of Langfuse
5. **Deploy on Kubernetes** with least-privilege IAM (S3 only)

::alert[**Keep the agent running — do not uninstall it here.** Modules **B4** (scaling, token economics, evaluation, monitoring) and **B5** (RAG, security, streaming, caching) build directly on this deployed `loan-buddy-agent` and the 3 MCP servers. Cleanup for the entire Track B stack is done **once at the end**, in the [Workshop Summary](/summary/).]{type="warning"}

---

**[Next: Module B4 — Tokenomics & Evaluation →](/track-b-saas/b4-scaling-evaluating/)**
