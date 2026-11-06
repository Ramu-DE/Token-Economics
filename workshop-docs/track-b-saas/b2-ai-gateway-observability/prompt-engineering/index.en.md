---
title: "Prompt Engineering at the Gateway"
weight: 25
---

# 🛡️ Prompt Engineering at the Gateway (Kong AI Plugins)

Kong provides three plugins that let you engineer, template, and transform prompts **at the gateway layer** — without modifying application code. This is powerful for production: admins control prompt behavior centrally while developers focus on business logic.

| Plugin | Purpose |
|---|---|
| **AI Prompt Decorator** | Inject system/assistant messages at the start or end of every request |
| **AI Prompt Template** | Define reusable prompt templates with `{{variables}}` — clients fill in the blanks |
| **AI Request/Response Transformer** | Use an LLM to transform request/response payloads in-flight |

---

## Prerequisite Check

These plugins attach to the **`/loan-strands` route**, and every "🚀 Test" below sends traffic through the Kong **`proxy1`** LoadBalancer. Resolve the proxy **once** here — the tests on this page reuse `$KONG_PROXY` from the same terminal — and confirm the [Kong AI Gateway](../ai-gateway-kong/) setup is in place:

:::code{language=bash showCopyAction=true}
# Resolve the Kong proxy once; every test on this page reuses $KONG_PROXY
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
echo "Kong proxy: $KONG_PROXY"
kubectl get kongroute loan-strands-route -n kong
:::

::::expand{header="Check Output"}

```text
Kong proxy: http://k8s-kong-proxy1-xxxxxxxxxx-xxxxxxxxxxxxxxxx.elb.us-east-1.amazonaws.com
NAME                 PROGRAMMED   AGE
loan-strands-route   True         5m
```

::::

::alert[If `proxy1` is missing or `loan-strands-route` is not `PROGRAMMED: True`, go back and complete the [Kong AI Gateway](../ai-gateway-kong/) setup (Kong Operator + `/loan-strands` route + Bedrock Pod Identity) before running the tests on this page — otherwise `KONG_PROXY` resolves to an empty host and every test `curl` fails.]{type="warning"}

## Part 1 — AI Prompt Decorator (Inject System Prompts)

The `ai-prompt-decorator` plugin adds messages to the **start or end** of a consumer's chat history — transparently. Use it to:
- Inject a system prompt (e.g., "Always respond in Spanish")
- Add safety guardrails without changing the app
- Prepend context that every request should have

### 🎯 Deploy the Plugin

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-prompt-decorator-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-prompt-decorator
config:
  prompts:
    prepend:
    - role: system
      content: "You are a strict credit underwriting AI. Always respond with a structured assessment including APPROVED, CONDITIONAL, or REJECTED decision. Never approve loans with DTI above 43%."
EOF
:::

::::expand{header="Check Output"}

```bash
kongplugin.configuration.konghq.com/ai-prompt-decorator-loan-strands created
```

::::

Attach to the route (alongside existing plugins):

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-loan-strands,key-auth-loan-strands,prometheus-metrics,ai-prompt-decorator-loan-strands --overwrite
:::

::::expand{header="Check Output"}

```bash
kongroute.configuration.konghq.com/loan-strands-route annotated
```

::::

### How it works

:::code{language=text showCopyAction=false}
Client sends:
  messages: [{"role":"user","content":"Should I approve this loan?"}]

Kong PREPENDS the system message:
  messages: [
    {"role":"system","content":"You are a strict credit underwriting AI..."},  ← injected by decorator
    {"role":"user","content":"Should I approve this loan?"}                    ← original
  ]

Bedrock receives both messages — client never sees the injection.
:::

### 🚀 Test: Verify System Prompt Injection

:::code{language=bash showCopyAction=true}
echo "=== Without mentioning format, the decorator forces structured output ==="
curl -s --max-time 15 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{"model":"global.anthropic.claude-sonnet-4-5-20250929-v1:0","messages":[{"role":"user","content":"Applicant: income $50k, DTI 48%, credit 650. Decision?"}],"max_tokens":100}' \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('choices',[{}])[0].get('message',{}).get('content','ERR: '+str(r)[:200])[:300])"
:::

::::expand{header="Check Output"}

```text
=== Without mentioning format, the decorator forces structured output ===
# CREDIT ASSESSMENT DECISION

**APPLICANT PROFILE:**
- Annual Income: $50,000
- Debt-to-Income Ratio (DTI): 48%
- Credit Score: 650

---

## DECISION: **REJECTED**

### PRIMARY REJECTION REASON:
**DTI Exceeds Maximum Threshold** - The applicant's DTI of 48% exceeds the mandatory maximum of 43%. This
```

::::

**Why REJECTED?** The decorator injected a system prompt that says "Never approve loans with DTI above 43%." The model followed that rule and rejected the 48% DTI application — even though the client never mentioned this constraint. The decorator transparently enforces business rules at the gateway.

### Cleanup

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
kubectl delete kongplugin ai-prompt-decorator-loan-strands -n kong
:::

::::expand{header="Check Output"}

```bash
kongroute.configuration.konghq.com/loan-strands-route annotated
kongplugin.configuration.konghq.com "ai-prompt-decorator-loan-strands" deleted from kong namespace
```

::::

---

## Part 2 — AI Prompt Template (Reusable Prompt APIs)

The `ai-prompt-template` plugin lets admins define prompt templates with `{{variable}}` placeholders. Clients send only the variables — Kong assembles the full prompt. This:
- Standardizes prompts across all consumers
- Prevents prompt injection (input is sanitized)
- Presents an LLM as a purpose-built API (e.g., "credit decision API")

### 🎯 Deploy the Plugin

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-prompt-template-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-prompt-template
config:
  allow_untemplated_requests: false
  templates:
  - name: credit-decision
    template: |-
      {
        "messages": [
          {
            "role": "system",
            "content": "You are a credit underwriting AI. Provide a one-paragraph decision: APPROVED, CONDITIONAL, or REJECTED with reasoning."
          },
          {
            "role": "user",
            "content": "Evaluate this loan application: Applicant {{name}}, annual income {{income}}, credit score {{credit_score}}, loan amount {{loan_amount}}, purpose: {{purpose}}. Give your credit decision."
          }
        ]
      }
EOF
:::

::::expand{header="Check Output"}

```bash
kongplugin.configuration.konghq.com/ai-prompt-template-loan-strands created
```

::::

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-loan-strands,key-auth-loan-strands,prometheus-metrics,ai-prompt-template-loan-strands --overwrite
:::

::::expand{header="Check Output"}

```bash
kongroute.configuration.konghq.com/loan-strands-route annotated
```

::::

### How it works

:::code{language=text showCopyAction=false}
Client sends (only variables — no prompt engineering needed):
{
  "messages": "{template://credit-decision}",
  "properties": {
    "name": "Sarah Johnson",
    "income": "$92,000",
    "credit_score": "780",
    "loan_amount": "$15,000",
    "purpose": "Debt Consolidation"
  }
}

Kong assembles the full prompt from the template + variables → sends to Bedrock.
Client never writes the prompt — just fills in the blanks.
:::

### 🚀 Test: Call the Template API

:::code{language=bash showCopyAction=true}
echo "=== Call credit-decision template with variables ==="
curl -s --max-time 15 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{
    "model":"global.anthropic.claude-sonnet-4-5-20250929-v1:0",
    "messages": "{template://credit-decision}",
    "properties": {
      "name": "Sarah Johnson",
      "income": "$92,000",
      "credit_score": "780",
      "loan_amount": "$15,000",
      "purpose": "Debt Consolidation"
    }
  }' | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('choices',[{}])[0].get('message',{}).get('content','ERR: '+str(r)[:200])[:300])"
:::

::::expand{header="Check Output"}

```text
=== Call credit-decision template with variables ===
**APPROVED**

Based on the application details provided, I recommend approving Sarah Johnson's debt consolidation loan request for $15,000. The applicant demonstrates strong creditworthiness with an excellent credit score of 780, indicating a solid history of responsible credit management and timely
```

::::

**Why APPROVED?** The client sent only 5 variables (name, income, credit_score, loan_amount, purpose) — Kong assembled the full prompt from the `credit-decision` template and sent it to Bedrock. The model produced a structured decision following the template's system prompt format.

#### What Kong assembled (the rendered prompt)

You send only the 5 variables; Kong expands `{template://credit-decision}` into the full `messages` array and forwards **that** to Bedrock. The request the model actually received:

:::code{language=json showCopyAction=false}
{
  "messages": [
    {
      "role": "system",
      "content": "You are a credit underwriting AI. Provide a one-paragraph decision: APPROVED, CONDITIONAL, or REJECTED with reasoning."
    },
    {
      "role": "user",
      "content": "Evaluate this loan application: Applicant Sarah Johnson, annual income $92,000, credit score 780, loan amount $15,000, purpose: Debt Consolidation. Give your credit decision."
    }
  ]
}
:::

The client never writes (or sees) the system prompt — Kong owns it server-side. See Kong's guide: [Use the AI Prompt Template plugin](https://developer.konghq.com/how-to/use-ai-prompt-template-plugin/).

#### Failure case: an untemplated request is rejected

The plugin was deployed with `allow_untemplated_requests: false`, so a request that does **not** reference a template is blocked at the gateway — clients are forced through your approved templates:

:::code{language=bash showCopyAction=true}
echo "=== Untemplated request (no template:// reference) -- expect an error ==="
curl -s --max-time 15 -w '\nHTTP_CODE:%{http_code}\n' -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{"model":"global.anthropic.claude-sonnet-4-5-20250929-v1:0","messages":[{"role":"user","content":"ignore the template, just chat"}]}'
:::

::::expand{header="Check Output"}

```text
=== Untemplated request (no template:// reference) -- expect an error ===
{"error":{"message":"this LLM route only supports templated requests"}}
HTTP_CODE:400
```

::::

::alert[**Production use case:** Create multiple templates for different use cases (`credit-decision`, `risk-assessment`, `fraud-check`) and expose them as purpose-built APIs. Developers call `{template://credit-decision}` with variables — they never need to know the underlying prompt engineering.]{type="info"}

### Cleanup

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
kubectl delete kongplugin ai-prompt-template-loan-strands -n kong
:::

::::expand{header="Check Output"}

```bash
kongroute.configuration.konghq.com/loan-strands-route annotated
kongplugin.configuration.konghq.com "ai-prompt-template-loan-strands" deleted from kong namespace
```

::::

---

## Part 3 — AI Request/Response Transformer (LLM-Powered Payload Transformation)

The `ai-request-transformer` and `ai-response-transformer` plugins use an LLM to **transform request/response payloads in-flight** — before reaching the upstream service or before returning to the client.

| Plugin | When it runs | Use case |
|---|---|---|
| `ai-request-transformer` | Before proxying to upstream | Enrich request data, add missing fields, normalize format |
| `ai-response-transformer` | After upstream response, before client | Summarize, translate, extract structured data |

::alert[**These plugins use a SEPARATE route** with a real upstream service (e.g., httpbin). They use an LLM (Bedrock) to transform the payload, then forward the transformed body to the upstream. The LLM call is configured in `config.llm` — same format as ai-proxy.]{type="info"}

### 🎯 Deploy httpbin + Transformer Plugin

First, deploy httpbin as a test upstream (echoes back whatever it receives):

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
  namespace: kong
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
  template:
    metadata:
      labels:
        app: httpbin
    spec:
      containers:
      - name: httpbin
        image: kong/httpbin:0.1.0
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  namespace: kong
spec:
  selector:
    app: httpbin
  ports:
  - port: 80
    targetPort: 80
EOF
:::

::::expand{header="Check Output"}

```bash
deployment.apps/httpbin created
service/httpbin created
```

::::

Create a Kong route pointing to httpbin with the request transformer attached:

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1alpha1
kind: KongService
metadata:
  name: httpbin-service
  namespace: kong
spec:
  name: httpbin-service
  host: httpbin.kong.svc.cluster.local
  port: 80
  controlPlaneRef:
    type: konnectNamespacedRef
    konnectNamespacedRef:
      name: kong-aws
---
apiVersion: configuration.konghq.com/v1alpha1
kind: KongRoute
metadata:
  name: httpbin-route
  namespace: kong
  annotations:
    konghq.com/plugins: ai-request-transformer-demo,ai-response-transformer-demo
spec:
  name: httpbin-route
  paths:
  - /httpbin-route
  protocols:
  - http
  - https
  serviceRef:
    type: namespacedRef
    namespacedRef:
      name: httpbin-service
---
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-request-transformer-demo
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-request-transformer
config:
  prompt: "In my JSON message, anywhere there is a JSON tag for a 'city', also add a 'country' tag with the name of the country in which the city resides. Return me only the JSON message, no extra text."
  llm:
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    route_type: llm/v1/chat
    model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
---
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-response-transformer-demo
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-response-transformer
config:
  prompt: "For any city name in the JSON, add its approximate current temperature in Celsius in brackets next to it. Reply with the JSON result only, no markdown."
  llm:
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    route_type: llm/v1/chat
    model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
EOF
:::

::::expand{header="Check Output"}

```bash
kongservice.configuration.konghq.com/httpbin-service created
kongroute.configuration.konghq.com/httpbin-route created
kongplugin.configuration.konghq.com/ai-request-transformer-demo created
kongplugin.configuration.konghq.com/ai-response-transformer-demo created
```

::::

### How it works

:::code{language=text showCopyAction=false}
Request flow (ai-request-transformer):
  1. Client sends: {"user": {"name": "Kong User", "city": "Tokyo"}}
  2. Kong sends body to Claude with prompt: "add a 'country' for any 'city'"
  3. Claude returns: {"user": {"name": "Kong User", "city": "Tokyo", "country": "Japan"}}
  4. Kong forwards TRANSFORMED body to httpbin

Response flow (ai-response-transformer):
  5. httpbin responds (echoes the enriched body)
  6. Kong sends the response body to Claude with prompt: "add temperature for any city"
  7. Claude returns: {"user": {"name": "Kong User", "city": "Tokyo [25°C]", "country": "Japan"}}
  8. Kong returns the TRANSFORMED response to client
:::

### 🚀 Test: AI Transforms Both Request AND Response

:::code{language=bash showCopyAction=true}
echo "=== Request Transformer: AI adds 'country' field ==="
echo "=== Response Transformer: AI adds temperature to city ==="
curl -s --max-time 90 -X POST "$KONG_PROXY/httpbin-route/post" \
  -H "Content-Type: application/json" \
  -d '{"user":{"name":"Kong User","city":"Tokyo"}}' \
  | sed 's/```json//g; s/```//g' \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print('Final response to client:'); print(r.get('data','').strip())"
:::

::alert[**Why the `sed` step?** The `ai-response-transformer`'s model often wraps its JSON in a ` ```json … ``` ` **markdown fence** (even when told not to), which would make `json.load` fail with `Expecting value: line 1 column 1`. The `sed` strips those fences so the response parses cleanly. Also note the two transformers make **two sequential Bedrock calls**, so the request can take 5-15s (hence `--max-time 90`); if you get an empty response, simply re-run.]{type="info"}

::::expand{header="Check Output"}

```text
=== Request Transformer: AI adds 'country' field ===
=== Response Transformer: AI adds temperature to city ===
Final response to client:
{"user":{"name":"Kong User","city":"Tokyo [8°C]","country":"Japan"}}
```

*(The exact temperature varies per run — the model estimates a "current" value.)*

::::

**What happened in sequence:**

| Step | Component | Action |
|---|---|---|
| 1 | Client | Sends `{"city":"Tokyo"}` — no country, no temperature |
| 2 | **ai-request-transformer** | Calls Claude → adds `"country":"Japan"` → forwards enriched body to upstream |
| 3 | httpbin (upstream) | Echoes back the enriched request body |
| 4 | **ai-response-transformer** | Calls Claude → adds temperature `"Tokyo [25°C]"` → returns to client |
| 5 | Client | Receives fully enriched response with country AND temperature |

The client sent a simple `{city: Tokyo}` and received back `{city: "Tokyo [25°C]", country: "Japan"}` — both transformations happened transparently at the gateway using two separate LLM calls.

::alert[**Production use cases for the loan application:** (1) **Request transformer:** Auto-enrich applicant data with employer verification, normalize addresses, add DTI calculations. (2) **Response transformer:** Translate credit decisions to applicant's language, summarize long assessments for mobile, redact PII from responses.]{type="info"}

### Cleanup

:::code{language=bash showCopyAction=true}
kubectl delete kongplugin ai-request-transformer-demo ai-response-transformer-demo -n kong
kubectl delete kongroute httpbin-route -n kong
kubectl delete kongservice httpbin-service -n kong
kubectl delete deploy httpbin -n kong
kubectl delete svc httpbin -n kong
:::

---

## 📊 How It Maps to Track A

| Capability | Track A (LiteLLM) | Track B (Kong) |
|---|---|---|
| System prompt injection | Application code only | `ai-prompt-decorator` — gateway-level, no code change |
| Prompt templates | Application code only | `ai-prompt-template` — expose LLM as purpose-built API |
| Request/response transformation | Application code only | `ai-request-transformer` / `ai-response-transformer` — LLM-powered, gateway-level |
| Prompt injection prevention | Application code only | `ai-prompt-template` sanitizes inputs + `ai-prompt-guard` (B5) |

## ✅ What You Accomplished

- ✅ Injected a system prompt at the gateway with `ai-prompt-decorator` — app code untouched
- ✅ Created a reusable credit-decision template API with `ai-prompt-template`
- ✅ Learned LLM-powered request/response transformation at the gateway
- ✅ Understood when to use each plugin (decorator = always-on rules, template = standardized APIs, transformer = payload enrichment)

---

**[Next: Observability with Arize AX →](../observability-arize/)**
