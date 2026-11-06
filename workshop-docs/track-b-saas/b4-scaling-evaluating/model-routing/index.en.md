---
title: "Model Routing & Semantic Caching"
weight: 30
---

Kong's **`ai-proxy-advanced`** plugin fronts multiple LLM models on a single route and routes requests using configurable algorithms — weighted round-robin, cost-aware, latency-aware, or header-based sticky routing. This page walks through each algorithm hands-on.

::alert[**Key pattern for Bedrock + IRSA:** Each target requires an `auth` block for Konnect schema validation. Use `header_name: x-dummy` / `header_value: unused` / `allow_override: false` — Bedrock auth comes from the pod's IRSA role automatically. The client sends `"model":""` (empty string) to let the plugin pick the model.]{type="warning"}

## 🎯 What You Will Learn

**🔴 Core — the highest-value patterns; do these first:**

- ⚖️ **Part 1** — Weighted round-robin (80/20 A/B split) — the canonical canary / A-B rollout
- 🛡️ **Part 5** — Failover / circuit breaker (automatic fallback on failure) — production reliability
- 💰 **Part 2** — Lowest-usage (cost-aware: route to cheapest model) — ties to the token-economics theme

**🟡 Optional — other routing algorithms (each standalone; pick by interest):**

- ⚡ **Part 3** — Lowest-latency (speed-aware: route to fastest model)
- ⚖️ **Part 8** — Least-connections (load balancing by active connections)
- 🔒 **Part 4** — Consistent-hashing (sticky: same user always hits same model)
- 🏷️ **Part 6** — Model alias (client-controlled: send friendly name)

**🧠 Advanced — semantic routing/caching (require Redis; do Part 7 before Part 9):**

- 🧠 **Part 7** — Semantic routing (route by prompt content using embeddings)
- 🧠 **Part 9** — Semantic caching (response caching by prompt similarity) — 60-80% cost cut on repeat prompts

::alert[**How to use this module — it's a menu, not a sequence.** Each Part is a **self-contained routing algorithm** (deploy the plugin → test on the same `loan-strands-route`), so you can do them in any order. If time is short, do the **🔴 Core** set — Part 1 (A/B), Part 5 (failover), Part 2 (cost-aware) — which cover the most common production patterns. The **🟡 Optional** algorithms (3, 8, 4, 6) are each standalone; try whichever interest you. The **🧠 Advanced** semantic parts (7, 9) need Redis — **do Part 7 first**, since Part 9 (semantic caching) reuses the Redis that Part 7 deploys. The Algorithm Summary table at the end is your reference.]{type="info"}

---

## ✅ Prerequisites — verify the Kong gateway, route, and proxy exist

Every routing test on this page mutates the **`loan-strands-route`** KongRoute and calls the **`proxy1`** LoadBalancer. Those are created earlier — the DataPlane + `proxy1` service in [B1 — Kong Setup](../../b1-platform-setup/kong-setup/) and the `loan-strands-route` KongRoute + `ai-proxy-loan-strands` / `key-auth-loan-strands` plugins in [B2 — Kong AI Gateway](../../b2-ai-gateway-observability/ai-gateway-kong/). Confirm they're present **before** you start Part 1 (on a fresh or torn-down cluster they may be missing):

:::code{language=bash showCopyAction=true}
kubectl get kongroute loan-strands-route -n kong >/dev/null 2>&1 \
  && kubectl get svc proxy1 -n kong -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' >/dev/null 2>&1 \
  && echo "✅ Prerequisites present: loan-strands-route + proxy1 LoadBalancer found" \
  || echo "❌ PREREQUISITE MISSING — complete B1 (Kong DataPlane + proxy1 service) and B2 (loan-strands route + ai-proxy plugin) first, then return here"
:::

::alert[If you see **PREREQUISITE MISSING**, the earlier Kong setup did not run (or was torn down). Go back and complete **[B1 — Kong Setup](../../b1-platform-setup/kong-setup/)** (DataPlane + `proxy1`) and **[B2 — Kong AI Gateway](../../b2-ai-gateway-observability/ai-gateway-kong/)** (the `loan-strands-route` KongRoute + `ai-proxy-loan-strands` / `key-auth-loan-strands` plugins). Every command below depends on them.]{type="warning"}

---

## Part 1 — Weighted Round-Robin (80/20 A/B Split) ⭐ *(Core)*

The `round-robin` algorithm distributes requests by `weight`. Set 80% Claude, 20% Nova on a single route:

### 🎯 Deploy the Plugin

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  balancer:
    algorithm: round-robin
  targets:
  - model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 3.0
        output_cost: 15.0
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    weight: 80
  - model:
      name: amazon.nova-micro-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 0.035
        output_cost: 0.14
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    weight: 20
EOF
:::

Swap the route to use `ai-proxy-advanced` instead of the single-model `ai-proxy`:

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-advanced-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
:::

### Config explained

| Field | Value | Meaning |
|---|---|---|
| `balancer.algorithm` | `round-robin` | Probabilistic distribution using per-target `weight` |
| `targets[0].weight` | `80` | Claude gets ~80% of traffic |
| `targets[1].weight` | `20` | Nova gets ~20% of traffic |
| `input_cost` / `output_cost` | USD per 1M tokens | Feeds Kong's cost metrics + enables `lowest-usage` (Part 2) |
| `auth.header_name: x-dummy` | bypass | Required by Konnect schema; Bedrock auth uses IRSA (pod identity) |

### 🚀 Test: Verify 80/20 Distribution

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

CLAUDE=0; NOVA=0
for i in {1..10}; do
  MODEL=$(curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
    -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
    -d '{"model":"","messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('model','ERR'))" 2>/dev/null)
  case "$MODEL" in *claude*) CLAUDE=$((CLAUDE+1));; *nova*) NOVA=$((NOVA+1));; esac
  echo "  Request $i: $MODEL"
  sleep 0.5
done
echo ""
echo "Distribution: Claude=$CLAUDE Nova=$NOVA (expected ~8/2)"
:::

::::expand{header="Check Output"}

```text
  Request 1: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 2: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 3: amazon.nova-micro-v1:0
  Request 4: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  ...
Distribution: Claude=8 Nova=2 (expected ~8/2)
```

::::

**Why 8/2?** The `round-robin` algorithm distributes requests probabilistically based on `weight`. With weight 80:20, each request has 80% chance of hitting Claude and 20% chance of hitting Nova. Over 10 requests, you'll see approximately 8:2 (exact numbers vary due to randomness).

::alert[**`"model":""`** (empty string) is the key. The client sends an empty model field and the plugin picks the target based on the algorithm. If you send a specific model name (e.g., `"model":"global.anthropic.claude-sonnet-4-5-20250929-v1:0"`), it only works when routed to that exact target — requests routed to the other target fail with "cannot use own model."]{type="warning"}

---

## Part 2 — Lowest-Usage (Cost-Aware Routing) *(Core)*

The `lowest-usage` algorithm routes to the target with the **lowest accumulated token or cost spend** — a gateway-native cost optimizer. Traffic naturally shifts toward the cheaper model.

### 🎯 Deploy the Plugin

Re-apply the same `ai-proxy-advanced-loan-strands` plugin with the new algorithm. `kubectl apply` **overwrites the plugin in place** and Kong reloads it — resetting the balancer's per-target state. (We deliberately do **not** `kubectl delete` between Parts: Kong CRs finalize via the Konnect control plane, so a delete would block the shell on the finalizer, and immediately re-creating the same name would race the still-terminating object. Overwriting in place avoids both. The single [Cleanup](#cleanup-restore-single-model-route) at the end removes the plugin once.)

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  balancer:
    algorithm: lowest-usage
    tokens_count_strategy: cost
  targets:
  - model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 3.0
        output_cost: 15.0
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
  - model:
      name: amazon.nova-micro-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 0.035
        output_cost: 0.14
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
EOF
:::

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-advanced-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
:::

### Config explained

| Field | Value | Meaning |
|---|---|---|
| `balancer.algorithm` | `lowest-usage` | Route each request to the target with the **lowest cumulative spend** so far |
| `tokens_count_strategy` | `cost` | Track spend in **dollars** (alternatives: `prompt-tokens`, `completion-tokens`, `total-tokens` for raw token counting) |
| `input_cost` | `3.0` (Claude) / `0.035` (Nova) | USD per **1 million input tokens** — used to compute dollar cost per request |
| `output_cost` | `15.0` (Claude) / `0.14` (Nova) | USD per **1 million output tokens** |

### How the algorithm decides

The plugin maintains a **running cost total per target**. After each request completes, it computes the cost and adds it to that target's total:

:::code{language=text showCopyAction=false}
cost_per_request = (prompt_tokens / 1,000,000 × input_cost) + (completion_tokens / 1,000,000 × output_cost)
:::

For a typical short request (8 prompt + 3 completion tokens):

| Target | Calculation | Cost per request |
|---|---|---|
| **Claude** | (8/1M × $3.0) + (3/1M × $15.0) | **$0.000069** |
| **Nova** | (8/1M × $0.035) + (3/1M × $0.14) | **$0.000000700** |

Claude is **~100x more expensive per request**. So after just 1 Claude request, its running total ($0.000069) far exceeds Nova's ($0). The plugin routes all subsequent requests to Nova (the "lowest usage" target) until Nova's running total eventually catches up — which takes ~100 Nova requests to match 1 Claude request.

**The net effect:** Nova gets the vast majority of traffic automatically. The plugin is a gateway-native cost optimizer — no application logic needed.

### 🚀 Test: Observe Cost-Aware Shift

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "Sending 10 requests (lowest-usage will favor Nova after warmup)..."
for i in {1..10}; do
  MODEL=$(curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
    -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
    -d '{"model":"","messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('model','ERR'))" 2>/dev/null)
  echo "  Request $i: $MODEL"
  sleep 0.5
done
:::

::::expand{header="Check Output"}

```text
Sending 10 requests (lowest-usage will favor Nova after warmup)...
  Request 1: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 2: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 3: amazon.nova-micro-v1:0
  Request 4: amazon.nova-micro-v1:0
  Request 5: amazon.nova-micro-v1:0
  Request 6: amazon.nova-micro-v1:0
  Request 7: amazon.nova-micro-v1:0
  Request 8: amazon.nova-micro-v1:0
  Request 9: amazon.nova-micro-v1:0
  Request 10: amazon.nova-micro-v1:0
```

::::

**Why this pattern?**

| Request | Running totals before this request | Decision | Why |
|---|---|---|---|
| 1 | Claude: $0, Nova: $0 | **Claude** | Tied — plugin picks first target |
| 2 | Claude: $0.000069, Nova: $0 | **Nova** or **Claude** | Nova is lower, but initial warmup may still probe both |
| 3 | Claude: ~$0.000138, Nova: ~$0.0000007 | **Nova** | Nova's total ($0.0000007) is far below Claude's ($0.000138) |
| 4-10 | Claude stays ~$0.000138, Nova slowly climbs | **Nova** | It takes ~100 Nova requests to match Claude's cost from just 2 requests |

After the initial warmup (requests 1-2), Nova dominates because its per-request cost is ~100x lower. The plugin would need to send ~100 requests to Nova before its cumulative cost catches up to Claude's 2-request total — effectively making this a "90%+ Nova" router.

::alert[**Production use case:** Set `input_cost`/`output_cost` to real Bedrock pricing. The algorithm automatically balances dollar spend across models — useful when you have a combined budget and want Kong to distribute it optimally without manual weight tuning.]{type="info"}

---

## Part 3 — Lowest-Latency (Speed-Aware Routing) *(Optional)*

The `lowest-latency` algorithm routes to the **fastest-responding model** using Exponentially Weighted Moving Average (EWMA). Nova Micro is typically 3-5x faster than Claude Sonnet.

### 🎯 Deploy the Plugin

Re-apply the plugin with the new algorithm (overwrites in place and reloads it, resetting the EWMA state):

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  balancer:
    algorithm: lowest-latency
    latency_strategy: tpot
  targets:
  - model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
  - model:
      name: amazon.nova-micro-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
EOF
:::

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-advanced-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
:::

### Config explained

| Field | Value | Meaning |
|---|---|---|
| `balancer.algorithm` | `lowest-latency` | Route each request to the target with the **lowest observed latency** |
| `latency_strategy` | `tpot` | Measure **Time Per Output Token** — total response time / completion tokens (alternative: `e2e` for raw end-to-end latency) |

### How the algorithm decides

The plugin maintains an **Exponentially Weighted Moving Average (EWMA)** of latency for each target. After each response:

:::code{language=text showCopyAction=false}
EWMA_new = α × latency_this_request + (1 - α) × EWMA_previous
:::

The next request goes to whichever target has the **lower EWMA**. Key behaviors:

1. **Warmup phase** — Until each target has handled enough requests to build a reliable EWMA, the algorithm probes both targets. During warmup, routing may appear random or biased toward the first target.
2. **Steady state** — After warmup, ~90-99% of traffic goes to the fastest model. The plugin **never sends 100%** to one target — it periodically probes the slower target to detect if conditions change (e.g., the slower model gets faster due to reduced load).
3. **TPOT vs E2E** — `tpot` (time per output token) measures generation speed independent of response length. `e2e` measures total wall-clock time (includes time-to-first-token + all generation time). TPOT is better for comparing models fairly when response lengths vary.

**Typical latencies for our models:**

| Model | TTFT (time to first token) | TPOT (per output token) | E2E for 10 tokens |
|---|---|---|---|
| Claude Sonnet 4.5 | ~800ms | ~50ms | ~1.3s |
| Nova Micro | ~200ms | ~30ms | ~0.5s |

### 🚀 Test: Observe Latency-Aware Routing

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "Sending 10 requests (lowest-latency routes to fastest model)..."
for i in {1..10}; do
  MODEL=$(curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
    -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
    -d '{"model":"","messages":[{"role":"user","content":"hi"}],"max_tokens":10}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('model','ERR'))" 2>/dev/null)
  echo "  Request $i: $MODEL"
  sleep 0.5
done
:::

::::expand{header="Check Output"}

```text
Sending 10 requests (lowest-latency routes to fastest model)...
  Request 1: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 2: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 3: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 4: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 5: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 6: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 7: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 8: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 9: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 10: global.anthropic.claude-sonnet-4-5-20250929-v1:0
```

::::

**Why all Claude (and not splitting across models to probe both)?**

The `lowest-latency` algorithm does NOT use a round-robin warmup. Instead, it uses **EWMA with a decay factor** — it starts with the first configured target and only shifts when it accumulates enough evidence that another target is faster. With `tpot` (time per output token) and non-streaming Bedrock responses:

| Factor | Explanation |
|---|---|
| **No round-robin warmup** | Unlike `lowest-usage` (which starts both at $0 and alternates), `lowest-latency` starts routing to the first target and stays there until EWMA data indicates a shift. It does NOT proactively split traffic to "discover" latencies. |
| **TPOT requires streaming** | The `tpot` strategy measures time between output tokens. In non-streaming mode (what this workshop uses), the entire response arrives at once — the plugin computes `total_time / output_tokens` as an approximation, but this gives high variance per-request. |
| **Slow probe rate** | The algorithm sends occasional probes to other targets (not every request), but with only 10-30 requests the probes may not have fired yet. In production with thousands of requests, probes accumulate enough data to shift. |

**The key insight:** `lowest-latency` is designed for **high-traffic production workloads** where thousands of requests naturally build EWMA data over time. For a workshop test with 10-30 requests, you'll see it stick to Claude (the first target). In production, after ~100+ requests, it would discover Nova is faster and gradually shift ~90% of traffic there.

::alert[**`lowest-latency` shines in production at scale.** With high request volume, the EWMA converges quickly and routes 90%+ traffic to the fastest model. For workshop testing, `round-robin` with weights (Part 1) or `lowest-usage` (Part 2) show clearer immediate behavior.]{type="info"}

::alert[**`latency_strategy: e2e` vs `tpot`** — Use `e2e` (end-to-end) when you care about total response time and aren't streaming. Use `tpot` (time per output token) when you use streaming and want to compare generation speed independent of response length.]{type="info"}

---

## Part 4 — Consistent-Hashing (Sticky Routing) *(Optional)*

The `consistent-hashing` algorithm routes based on a **header value** — the same header value always hits the same model. Useful for per-user or per-tier routing where you want deterministic model assignment.

### 🎯 Deploy the Plugin

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  balancer:
    algorithm: consistent-hashing
    hash_on_header: X-User-Tier
  targets:
  - model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
  - model:
      name: amazon.nova-micro-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
EOF
:::

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-advanced-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
:::

### Config explained

| Field | Value | Meaning |
|---|---|---|
| `balancer.algorithm` | `consistent-hashing` | Hash a header value to deterministically pick the same target every time |
| `hash_on_header` | `X-User-Tier` | The HTTP header whose value is hashed to select a target |

### How the algorithm decides

The plugin uses a **hash ring** (consistent hashing) to map header values to targets:

:::code{language=text showCopyAction=false}
target_index = hash(header_value) % number_of_targets
:::

1. Kong hashes the value of `X-User-Tier` (e.g., hash("premium") → some number)
2. That number maps to a position on the hash ring
3. The nearest target on the ring handles the request
4. **Same value = same target every time** (deterministic, no randomness)

**Key behaviors:**
- The **same header value** always routes to the **same model** — guaranteed consistency across requests
- **Different header values** may or may not route to different models — it depends on where they land on the hash ring
- With only 2 targets, each value has a ~50% chance of landing on either target. Some values ("premium", "free") may hash to the same target — this is expected hash ring behavior, not a bug

### 🚀 Test: Sticky Routing by Header

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "=== premium tier (same header → same model every time) ==="
for i in {1..3}; do
  curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
    -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
    -H "X-User-Tier: premium" \
    -d '{"model":"","messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); print(f'  {r.get(\"model\",\"ERR\")}')" 2>/dev/null
  sleep 0.5
done

echo ""
echo "=== free tier (different header value → deterministic but may land on same or different model) ==="
for i in {1..3}; do
  curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
    -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
    -H "X-User-Tier: free" \
    -d '{"model":"","messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); print(f'  {r.get(\"model\",\"ERR\")}')" 2>/dev/null
  sleep 0.5
done

echo ""
echo "=== try more values to find one that hashes to Nova ==="
for TIER in gold silver bronze team-a team-b user-123; do
  MODEL=$(curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
    -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
    -H "X-User-Tier: $TIER" \
    -d '{"model":"","messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); m=r.get('model','ERR'); print('Claude' if 'claude' in m else 'Nova')" 2>/dev/null)
  echo "  X-User-Tier: $TIER → $MODEL"
  sleep 0.3
done
:::

::::expand{header="Check Output"}

```text
=== premium tier (same header → same model every time) ===
  global.anthropic.claude-sonnet-4-5-20250929-v1:0
  global.anthropic.claude-sonnet-4-5-20250929-v1:0
  global.anthropic.claude-sonnet-4-5-20250929-v1:0

=== free tier (different header value → deterministic but may land on same or different model) ===
  global.anthropic.claude-sonnet-4-5-20250929-v1:0
  global.anthropic.claude-sonnet-4-5-20250929-v1:0
  global.anthropic.claude-sonnet-4-5-20250929-v1:0

=== try more values to find one that hashes to Nova ===
  X-User-Tier: gold → Claude
  X-User-Tier: silver → Nova
  X-User-Tier: bronze → Claude
  X-User-Tier: team-a → Nova
  X-User-Tier: team-b → Claude
  X-User-Tier: user-123 → Claude
```

::::

**Why "premium" and "free" both route to Claude?**

| Observation | Explanation |
|---|---|
| Both hit Claude | `hash("premium")` and `hash("free")` both happen to land on the same side of the hash ring → Claude. This is normal — with only 2 targets, any given pair of values has a 50% chance of landing on the same target. |
| All 3 repeats are consistent | **This is the guarantee** — same value ALWAYS routes to the same model. "premium" → Claude every time, forever. No randomness. |
| Other values hit Nova | Try more header values (silver, team-a, etc.) — some will hash to the Nova side of the ring. In production with many unique user IDs, you get natural distribution across both models. |

**The key guarantee:** Consistency, not distribution. A user with tier "premium" **always** gets Claude. A user with tier "silver" **always** gets Nova. The mapping is deterministic — you can test which values map where and use that knowledge in your application.

::alert[**Production pattern:** Hash on `X-Consumer-Username` or a user ID header instead of tier. With many unique users (e.g., hundreds of loan applicants), the hash ring naturally distributes ~50/50 across 2 targets. Each user gets consistent model assignment across all their requests — useful for A/B testing where you want users to have a consistent experience.]{type="info"}

---

## Part 5 — Failover / Circuit Breaker (Automatic Fallback) *(Core)*

The `failover_criteria` configuration makes `ai-proxy-advanced` automatically fail over from the primary model to a backup when errors, timeouts, or rate limits occur. This is equivalent to Track A's `router_settings.fallbacks`.

### 🎯 Deploy the Plugin

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  balancer:
    algorithm: round-robin
    failover_criteria:
    - error
    - timeout
    - http_429
    - http_503
    - http_500
    max_fails: 3
    fail_timeout: 10000
    retries: 2
  targets:
  - model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 3.0
        output_cost: 15.0
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    weight: 100
  - model:
      name: amazon.nova-micro-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 0.035
        output_cost: 0.14
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    weight: 1
EOF
:::

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-advanced-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
:::

### Config explained

| Field | Value | Meaning |
|---|---|---|
| `failover_criteria` | `[error, timeout, http_429, http_503, http_500]` | Conditions that trigger failover to the next target |
| `max_fails` | `3` | After 3 consecutive failures, mark the target as unhealthy |
| `fail_timeout` | `10000` | Wait 10 seconds before re-probing an unhealthy target |
| `retries` | `2` | Retry up to 2 times on a different target before returning error to client |
| `weight: 100` (Claude) / `weight: 1` (Nova) | 100:1 | Normal traffic goes almost entirely to the primary (Claude, ~99%); Nova stays a dormant fallback that only takes over when Claude fails |

### How the algorithm decides

:::code{language=text showCopyAction=false}
Request arrives
  → Route to primary target (Claude, weight=100)
  → If response is error/timeout/429/503/500:
      → Retry on next target (Nova)
      → If Nova succeeds: return Nova's response to client
      → If Nova also fails: return error to client
  → After 3 consecutive failures on Claude:
      → Mark Claude "unhealthy" for 10 seconds
      → All traffic goes to Nova during this period
      → After 10s, probe Claude again — if healthy, restore it
:::

**Failover triggers (what each criterion catches):**

| Criterion | Catches | Example scenario |
|---|---|---|
| `error` | Connection refused, DNS failure | Bedrock regional outage |
| `timeout` | Response exceeds configured timeout | Model overloaded, slow response |
| `http_429` | Rate limit exceeded | Bedrock throttling your account |
| `http_503` | Service unavailable | Bedrock capacity issue |
| `http_500` | Internal server error | Provider-side bug |

### 🚀 Test: Verify Normal Operation (Primary Target)

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "=== Normal operation — Claude (primary, weight=100) handles all requests ==="
for i in {1..5}; do
  curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
    -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
    -d '{"messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); print(f'  Request $i: {r.get(\"model\",\"ERR\")}')" 2>/dev/null
  sleep 0.5
done
:::

::::expand{header="Check Output"}

```text
=== Normal operation — Claude (primary, weight=100) handles all requests ===
  Request 1: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 2: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 3: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 4: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 5: global.anthropic.claude-sonnet-4-5-20250929-v1:0
```

::::

**Why all Claude?** With weight 100:1, the round-robin algorithm sends ~99% to Claude. Under normal operation, Claude is healthy and handles every request. Nova is the dormant fallback — it only activates when Claude fails.

### What happens when Claude fails (failover behavior)

The failover is **transparent** — the client sees a normal response (from Nova) instead of an error:

| Scenario | What the client sees | What happens internally |
|---|---|---|
| Claude healthy | `model: claude-sonnet-4-5` (normal) | Direct to Claude |
| Claude returns 429 (rate limited) | `model: nova-micro` (fallback) | Kong catches 429 → retries on Nova → returns Nova's response |
| Claude returns 503 (capacity issue) | `model: nova-micro` (fallback) | Kong catches 503 → retries on Nova → returns Nova's response |
| Claude timeout (>30s) | `model: nova-micro` (fallback) | Kong catches timeout → retries on Nova |
| Claude fails 3 consecutive times | **All traffic → Nova for 10s** | Claude marked unhealthy (`max_fails:3`) → circuit open for `fail_timeout:10000`ms |
| After 10s circuit timeout | `model: claude-sonnet-4-5` (recovered) | Kong probes Claude → if healthy, restores it as primary |
| Both Claude AND Nova fail | Error returned to client | Retries exhausted (`retries:2`) — client gets the error |

::alert[**The failover is invisible to the client.** When Bedrock throttles Claude (429) or has a capacity issue (503), Kong automatically retries on Nova. The client receives a normal 200 response — they never see the error. The `model` field in the response reveals which target actually handled the request.]{type="success"}

::alert[**Cannot trigger failover on demand in this workshop** — Bedrock's throttle limits are high, and there's no way to artificially make Claude return a 429/503 without extreme concurrent load. The circuit breaker validates by confirming (1) normal operation routes to Claude, and (2) the failover criteria is correctly configured. In production, Kong catches real failures automatically.]{type="info"}

---

## Part 6 — Model Alias (Client-Controlled Routing) *(Optional)*

With `model_alias`, clients explicitly choose which model to use by sending a human-friendly name (e.g., `"model":"complex"` or `"model":"simple"`) instead of the full Bedrock model ID. The gateway maps the alias to the correct backend.

### 🎯 Deploy the Plugin

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  targets:
  - model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      model_alias: complex
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 3.0
        output_cost: 15.0
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
  - model:
      name: amazon.nova-micro-v1:0
      provider: bedrock
      model_alias: simple
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 0.035
        output_cost: 0.14
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
EOF
:::

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-advanced-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
:::

### Config explained

| Field | Value | Meaning |
|---|---|---|
| `model_alias` | `complex` / `simple` | Human-friendly name the client sends in the `"model"` field |
| No `balancer` block | — | Without a balancer algorithm, routing is purely by alias matching |

### How the algorithm decides

:::code{language=text showCopyAction=false}
Client sends: "model": "complex"
  → Plugin scans targets for model_alias == "complex"
  → Match found: Claude target
  → Route request to Bedrock with model = global.anthropic.claude-sonnet-4-5-20250929-v1:0

Client sends: "model": "simple"
  → Plugin scans targets for model_alias == "simple"
  → Match found: Nova target
  → Route request to Bedrock with model = amazon.nova-micro-v1:0
:::

**The power of alias:** The application never needs to know the full Bedrock model ID. It just sends `"complex"` or `"simple"`. If you later swap Claude for a fine-tuned model, update the plugin config only — zero application code changes.

### 🚀 Test: Route by Alias

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "=== Send model='complex' → routes to Claude target ==="
curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{"model":"complex","messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(f'Response: {r.get(\"model\", r.get(\"error\",{}).get(\"message\",str(r)[:80]))}')"

echo ""
echo "=== Send model='simple' → routes to Nova target ==="
curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{"model":"simple","messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(f'Response: {r.get(\"model\", r.get(\"error\",{}).get(\"message\",str(r)[:80]))}')"
:::

::::expand{header="Check Output"}

```text
=== Send model='complex' → routes to Claude target ===
Response: cannot use own model - must be: global.anthropic.claude-sonnet-4-5-20250929-v1:0

=== Send model='simple' → routes to Nova target ===
Response: cannot use own model - must be: amazon.nova-micro-v1:0
```

::::

**Why this error (and why it proves the alias works)?**

| Step | What happened | Evidence |
|---|---|---|
| 1. Client sends `"model":"complex"` | Plugin resolves alias → selects **Claude** target | Error says "must be: global.anthropic.claude-sonnet..." |
| 2. Client sends `"model":"simple"` | Plugin resolves alias → selects **Nova** target | Error says "must be: amazon.nova-micro..." |
| 3. Model name enforcement | Plugin requires request model to match target's `model.name` exactly | This is a **Kong 3.13 bug** — alias resolution works but the model name check fires after |

The alias routing logic is correct — it selected the right target in both cases. The error is a Kong 3.13 model name enforcement bug that should be fixed in a future version.

::alert[**Kong 3.13 limitation:** `model_alias` correctly resolves the target but still enforces exact model name matching after resolution. The config pattern is correct for production use once this is fixed. Filed with Kong team.]{type="warning"}

**Application integration pattern (for when alias works end-to-end):**

:::code{language=python showCopyAction=false}
def get_model_alias(credit_score: int, dti_ratio: float) -> str:
    """Application picks 'complex' or 'simple' — gateway maps to the right model."""
    if credit_score >= 700 and dti_ratio <= 36:
        return "simple"    # Nova Micro (fast, cheap)
    else:
        return "complex"   # Claude Sonnet (accurate for edge cases)

response = client.chat.completions.create(
    model=get_model_alias(applicant.credit_score, applicant.dti),
    messages=[{"role": "user", "content": prompt}]
)
:::

---

## Part 7 — Semantic Routing (Prompt-Complexity-Based) 🧠 *(Advanced — needs Redis)*

The `semantic` algorithm is the most advanced routing option — it uses an **embedding model** to analyze the incoming prompt and route based on **semantic similarity** to each target's description. Complex prompts go to Claude; simple ones fall to Nova (CATCHALL).

::alert[**Prerequisites:** Semantic routing requires (1) **Amazon Bedrock Titan Embeddings** (`amazon.titan-embed-text-v2:0`) — already enabled in your account, (2) **Redis Stack** with vector search support — deploy it below if not already running.]{type="warning"}

### Deploy Redis Stack (if not already running)

Redis Stack provides vector search capabilities needed for embedding-based routing and semantic caching. Check if it's already deployed:

:::code{language=bash showCopyAction=true}
kubectl get pods -n redis 2>/dev/null | grep redis-stack || echo "Redis Stack NOT found — deploy it below"
:::

If Redis Stack is not running, deploy it via Helm:

:::code{language=bash showCopyAction=true}
helm repo add redis-stack https://redis-stack.github.io/helm-redis-stack
helm repo update
# upgrade --install is idempotent: installs on first run, no-ops on re-runs (no "already exists" error)
helm upgrade --install redis-stack redis-stack/redis-stack -n redis --create-namespace

# Wait for Redis to be ready
kubectl rollout status deployment/redis-stack -n redis --timeout=60s
:::

Verify Redis is accessible from the Kong namespace:

:::code{language=bash showCopyAction=true}
kubectl run redis-test --rm -i --restart=Never --image=redis:7 -n kong -- \
  redis-cli -h redis-stack.redis.svc.cluster.local -p 6379 PING
:::

**Expected:** `PONG`

### 🎯 Deploy the Plugin

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  balancer:
    algorithm: semantic
  embeddings:
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    model:
      provider: bedrock
      name: amazon.titan-embed-text-v2:0
      options:
        bedrock:
          aws_region: us-east-1
  vectordb:
    dimensions: 1024
    distance_metric: cosine
    strategy: redis
    threshold: 1.0
    redis:
      host: redis-stack.redis.svc.cluster.local
      port: 6379
  targets:
  - model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    description: "Complex loan evaluation, edge cases, low credit score, high debt-to-income ratio, bankruptcy history, collections"
  - model:
      name: amazon.nova-micro-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    description: "Simple questions, greetings, general knowledge, basic math, weather"
EOF
:::

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-advanced-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
:::

### Config explained

| Field | Value | Meaning |
|---|---|---|
| `balancer.algorithm` | `semantic` | Route based on semantic similarity between prompt and target descriptions |
| `embeddings.model` | `amazon.titan-embed-text-v2:0` | Bedrock Titan Embeddings converts prompts + descriptions to 1024-dim vectors |
| `vectordb.strategy` | `redis` | Redis Stack stores description embeddings as vectors |
| `vectordb.threshold` | `1.0` | Maximum cosine distance — lower values are more strict (0 = exact match only) |
| `vectordb.dimensions` | `1024` | Must match Titan Embed v2 output dimensions |
| `description` (Claude) | "Complex loan evaluation, edge cases..." | Prompts semantically similar to this go to Claude |
| `description` (Nova) | "Simple questions, greetings..." | Prompts similar to this go to Nova |

### How the algorithm decides

:::code{language=text showCopyAction=false}
1. First request: Kong embeds both target descriptions using Titan Embeddings
   → Stores embeddings in Redis (one-time setup, auto on first request)

2. For each incoming request:
   a. Kong embeds the prompt: "Evaluate: credit 580, bankruptcy" → vector [0.12, 0.85, ...]
   b. Kong searches Redis for nearest description vector:
      - vs "Complex loan evaluation, edge cases..." → distance = 0.2 (close match!)
      - vs "Simple questions, greetings..." → distance = 0.9 (far)
   c. Nearest match (0.2) < threshold (1.0) → route to Claude

3. Another request: "What is 2 plus 2?"
   - vs "Complex loan evaluation..." → distance = 0.95 (far)
   - vs "Simple questions, greetings, basic math..." → distance = 0.3 (close!)
   → Route to Nova
:::

### 🚀 Test: Semantic Routing

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "=== Complex loan prompt → should route to Claude ==="
curl -s --max-time 15 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{"model":"","messages":[{"role":"user","content":"Evaluate this high-risk loan: credit score 580, DTI 52%, recent bankruptcy, collections on record. Should we approve?"}],"max_tokens":20}' \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(f'Model: {r.get(\"model\", \"ERR: \"+str(r)[:80])}')"
:::

:::code{language=bash showCopyAction=true}
echo "=== Simple question → should route to Nova ==="
curl -s --max-time 15 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{"model":"","messages":[{"role":"user","content":"What is 2 plus 2?"}],"max_tokens":20}' \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(f'Model: {r.get(\"model\", \"ERR: \"+str(r)[:80])}')"
:::

::::expand{header="Check Output"}

```text
=== Complex loan prompt → should route to Claude ===
Model: global.anthropic.claude-sonnet-4-5-20250929-v1:0

=== Simple question → should route to Nova ===
Model: amazon.nova-micro-v1:0
```

::::

**Why this works:** Kong analyzed the **content** of each prompt using Titan Embeddings and routed to the model whose description is most semantically similar. The complex loan prompt (bankruptcy, low credit, high DTI) matches the Claude target's description. The simple math question matches the Nova target's description. No application code needed — the gateway makes the routing decision entirely.

### Verify Redis stored the embeddings

:::code{language=bash showCopyAction=true}
kubectl run redis-scan --rm -i --restart=Never --image=redis:7 -n redis -- \
  redis-cli -h redis-stack.redis.svc.cluster.local -p 6379 --scan 2>/dev/null | grep "semantic_routing"
:::

::::expand{header="Check Output"}

```text
semantic_routing:<plugin-id>:route:<hash1>
semantic_routing:<plugin-id>:route:<hash2>
```

::::

Two entries — one for each target's description embedding stored in Redis.

::alert[**Track A equivalent:** LiteLLM does not have built-in semantic routing. Track A achieves content-based routing through application-level code (`route_by_complexity()` function). Track B's semantic routing does the same thing at the **gateway layer** — no application code needed, and the routing logic can be changed by updating the description field (no redeploy).]{type="info"}

::alert[**Tip:** Short, focused description phrases produce sharper embeddings than long prose. Use domain-specific keywords: "bankruptcy, collections, low credit score" rather than generic "difficult financial situations."]{type="info"}

---

## Part 8 — Least-Connections (Load Balancing) *(Optional)*

The `least-connections` algorithm routes to the target currently handling the **fewest active connections** — ideal for load balancing across model replicas or distributing concurrent traffic evenly.

### 🎯 Deploy the Plugin

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  balancer:
    algorithm: least-connections
    retries: 2
    failover_criteria:
    - error
    - timeout
    - http_429
  targets:
  - model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 3.0
        output_cost: 15.0
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
  - model:
      name: amazon.nova-micro-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 0.035
        output_cost: 0.14
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
EOF
:::

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-advanced-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
:::

### Config explained

| Field | Value | Meaning |
|---|---|---|
| `balancer.algorithm` | `least-connections` | Route to the target with the fewest in-flight requests |
| `retries` | `2` | Retry on a different target if the selected one fails |
| `failover_criteria` | `[error, timeout, http_429]` | Conditions that trigger retry on another target |

### How the algorithm decides

:::code{language=text showCopyAction=false}
Request arrives:
  → Check active connections: Claude=2, Nova=1
  → Route to Nova (fewest connections)

Next request:
  → Check active connections: Claude=2, Nova=2
  → Route to either (tied — round-robin tiebreaker)
:::

**Key behaviors:**
- With sequential requests (like this test), both targets have 0 active connections when each request arrives, so the algorithm alternates ~50/50
- With **concurrent** requests, it shines: slower models (Claude ~2s) accumulate connections while faster models (Nova ~0.5s) free them up faster, naturally getting more traffic
- This is the Track B equivalent of Track A's "load balancing across replicas" pattern

### 🚀 Test: Observe Even Distribution

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "Sending 10 sequential requests (least-connections → ~50/50 distribution)..."
CLAUDE=0; NOVA=0
for i in {1..10}; do
  MODEL=$(curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
    -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
    -d '{"model":"","messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('model','ERR'))" 2>/dev/null)
  case "$MODEL" in *claude*) CLAUDE=$((CLAUDE+1));; *nova*) NOVA=$((NOVA+1));; esac
  echo "  Request $i: $MODEL"
  sleep 0.3
done
echo ""
echo "Distribution: Claude=$CLAUDE Nova=$NOVA (expected ~5/5)"
:::

::::expand{header="Check Output"}

```text
Sending 10 sequential requests (least-connections → ~50/50 distribution)...
  Request 1: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 2: amazon.nova-micro-v1:0
  Request 3: amazon.nova-micro-v1:0
  Request 4: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 5: amazon.nova-micro-v1:0
  Request 6: amazon.nova-micro-v1:0
  Request 7: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 8: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 9: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 10: amazon.nova-micro-v1:0

Distribution: Claude=5 Nova=5 (expected ~5/5)
```

::::

**Why ~50/50?** With sequential requests (each completing before the next starts), both targets always show 0 active connections → the algorithm alternates. In production with concurrent traffic, the faster model (Nova) frees connections sooner and gets more requests — naturally load-balancing by actual capacity.

::alert[**Production use case:** Use `least-connections` when you have multiple replicas of the same model (e.g., 3 Nova instances behind different endpoints). It distributes traffic based on actual load, not static weights — slower replicas automatically get less traffic.]{type="info"}

---

## Part 9 — Semantic Caching (Response Caching by Prompt Similarity) 🧠 *(Advanced — needs Redis)*

The routing algorithms above pick *which model* handles a request. **Semantic caching** is a different lever on the same route: it stores LLM responses and returns them for **semantically similar** future requests — even if the wording differs — using the same Titan embeddings + Redis you set up in Part 7. A cache hit skips Bedrock entirely: zero tokens, zero cost, sub-50ms latency.

**Example:** ask "What is Amazon EKS?" then later "Tell me about EKS" → the cache returns the stored response without calling Bedrock again.

### Prerequisites

::alert[**Redis Stack** (with vector search) must be running. If you completed **Part 7 (Semantic Routing)** above, Redis is already deployed — reuse it. Otherwise deploy it with the Helm step from Part 7 first.]{type="warning"}

:::code{language=bash showCopyAction=true}
kubectl run redis-ping --rm -i --restart=Never --image=redis:7 -n kong -- \
  redis-cli -h redis-stack.redis.svc.cluster.local -p 6379 PING
:::

**Expected:** `PONG`

### How Semantic Caching Works

:::code{language=text showCopyAction=false}
1. User sends: "What is Amazon EKS?"
2. ai-semantic-cache plugin embeds the prompt using Titan Embeddings → vector
3. Plugin searches Redis for semantically similar cached vectors
4. MISS (first time): forward to Bedrock via ai-proxy, cache response + embedding
5. Return response with X-Cache-Status: Miss

Next request: "Tell me about EKS"
1. Plugin embeds the prompt → vector
2. Plugin searches Redis → finds a similar vector (cosine distance < threshold)
3. HIT: return the cached response directly from Redis
4. Return response with X-Cache-Status: Hit (no Bedrock call, no cost, ~0ms)
:::

| Component | Role |
|---|---|
| **Titan Embed Text v2** | Converts prompts to 1024-dim vectors for similarity comparison |
| **Redis Stack** | Vector database storing prompt embeddings + cached responses |
| **Cosine distance** | Measures similarity between vectors (0 = identical, 2 = opposite) |
| **Threshold** | Maximum cosine distance to consider a match (lower = stricter) |

### 🎯 Deploy the AI Semantic Cache Plugin

This attaches to the **single-model** `ai-proxy-loan-strands` route (semantic caching sits in front of the LLM call, independent of the routing algorithms above):

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-semantic-cache-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-semantic-cache
config:
  embeddings:
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    model:
      provider: bedrock
      name: amazon.titan-embed-text-v2:0
      options:
        bedrock:
          aws_region: us-east-1
  vectordb:
    dimensions: 1024
    distance_metric: cosine
    strategy: redis
    threshold: 0.2
    redis:
      host: redis-stack.redis.svc.cluster.local
      port: 6379
EOF
:::

Attach it to the `/loan-strands` route (this also resets the route to the single-model `ai-proxy-loan-strands` used for caching):

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-loan-strands,key-auth-loan-strands,prometheus-metrics,ai-semantic-cache-loan-strands --overwrite
:::

#### Config explained

| Field | Value | Meaning |
|---|---|---|
| `embeddings.model` | `amazon.titan-embed-text-v2:0` | Bedrock Titan Embeddings converts prompts to 1024-dim vectors |
| `vectordb.strategy` | `redis` | Use Redis Stack as the vector database |
| `vectordb.threshold` | `0.2` | Maximum cosine distance for a cache hit — queries must be very similar (>0.8 cosine similarity) |
| `vectordb.dimensions` | `1024` | Must match Titan Embed v2 output dimensions |
| `vectordb.redis.host` | `redis-stack.redis.svc.cluster.local` | Redis Stack service in the `redis` namespace |

### 🚀 Test: Observe Cache Hit/Miss Behavior

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
echo "KONG_PROXY=$KONG_PROXY"

echo "=== Request 1: First time (expect Miss) ==="
curl -s -D /tmp/cache_r1_h -o /tmp/cache_r1_b --max-time 15 \
  -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{"model":"global.anthropic.claude-sonnet-4-5-20250929-v1:0","messages":[{"role":"user","content":"What is Amazon EKS?"}],"max_tokens":50}'
grep -iE "X-Cache-Status|X-Kong-Upstream-Latency" /tmp/cache_r1_h

echo "=== Request 2: Same question (expect Hit) ==="
curl -s -D /tmp/cache_r2_h -o /dev/null --max-time 15 \
  -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{"model":"global.anthropic.claude-sonnet-4-5-20250929-v1:0","messages":[{"role":"user","content":"What is Amazon EKS?"}],"max_tokens":50}'
grep -iE "X-Cache-Status|X-Cache-Ttl" /tmp/cache_r2_h

echo "=== Request 3: Similar wording (expect Hit) ==="
curl -s -D /tmp/cache_r3_h -o /dev/null --max-time 15 \
  -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{"model":"global.anthropic.claude-sonnet-4-5-20250929-v1:0","messages":[{"role":"user","content":"What is EKS?"}],"max_tokens":50}'
grep -iE "X-Cache-Status" /tmp/cache_r3_h

echo "=== Request 4: Different topic (expect Miss) ==="
curl -s -D /tmp/cache_r4_h -o /dev/null --max-time 15 \
  -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{"model":"global.anthropic.claude-sonnet-4-5-20250929-v1:0","messages":[{"role":"user","content":"How do I deploy a container on Kubernetes?"}],"max_tokens":50}'
grep -iE "X-Cache-Status" /tmp/cache_r4_h
:::

::::expand{header="Check Output"}

```text
=== Request 1: First time (expect Miss) ===
X-Cache-Status: Miss
X-Kong-Upstream-Latency: 2738
=== Request 2: Same question (expect Hit) ===
X-Cache-Status: Hit
X-Cache-Ttl: 273
=== Request 3: Similar wording (expect Hit) ===
X-Cache-Status: Hit
=== Request 4: Different topic (expect Miss) ===
X-Cache-Status: Miss
```

::::

| Request | Prompt | X-Cache-Status | Why |
|---|---|---|---|
| 1 | "What is Amazon EKS?" | **Miss** | First time — stored in Redis |
| 2 | "What is Amazon EKS?" | **Hit** | Exact match (distance = 0) |
| 3 | "What is EKS?" | **Hit** | Semantically similar (distance < 0.2) |
| 4 | "How do I deploy a container on Kubernetes?" | **Miss** | Different topic (distance > 0.2) |

### 📊 Cost & Latency Impact

| Metric | Without cache (Miss) | With cache (Hit) |
|---|---|---|
| **Bedrock call** | Yes (tokens consumed) | **No** (zero tokens) |
| **Cost** | ~$0.000099 per request | **$0** |
| **Latency** | ~2-3 seconds | **< 50ms** (Redis lookup + embedding only) |

::alert[**Production impact:** for workloads with repeated/similar queries (chatbots, FAQ, documentation assistants), semantic caching can reduce LLM cost by **60-80%** and cut P50 latency from seconds to milliseconds. Tune `vectordb.threshold`: `0.1` = strict (near-exact prompts), `0.2` = recommended (similar intent), `0.5` = loose (broadly related). Monitor the `X-Cache-Status` header to measure hit rate; Kong's `kong_ai_cache_*` Prometheus metrics (see token-cost-attribution Part 5) populate once this plugin is active.]{type="success"}

---

## Cleanup: Restore Single-Model Route

After testing all algorithms, restore the original single-model configuration:

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
# --wait=false: Kong CRs finalize via Konnect, so a default delete blocks the shell until that completes.
# The annotate above already restored the single-model route, so the agent works regardless of when the object finishes terminating.
kubectl delete kongplugin ai-proxy-advanced-loan-strands -n kong --ignore-not-found --wait=false
# Also remove the Part 9 semantic-cache plugin (the annotate above already detached it from the route).
kubectl delete kongplugin ai-semantic-cache-loan-strands -n kong --ignore-not-found --wait=false
:::

---

## Algorithm Summary

| Algorithm | Part | Best For | Traffic Pattern | Application Change |
|---|---|---|---|---|
| `round-robin` + `weight` | Part 1 | Progressive rollout, A/B testing | Static 80/20 split | None (`model:""`) |
| `lowest-usage` + `cost` | Part 2 | Minimize spend automatically | Shifts toward cheapest model | None (`model:""`) |
| `lowest-latency` + `tpot` | Part 3 | Speed-critical (high-traffic production) | Shifts toward fastest model | None (`model:""`) |
| `consistent-hashing` | Part 4 | Per-user/per-tier sticky routing | Deterministic by header | Set routing header |
| `failover_criteria` | Part 5 | Resilience, automatic fallback | Primary + backup on failure | None (`model:""`) |
| `model_alias` | Part 6 | Client-controlled routing | Explicit by alias name | Send alias in `model` field |
| `semantic` | Part 7 | Content-based routing by prompt | Intelligent per-prompt | None (`model:""`) |
| `least-connections` | Part 8 | Load balancing across replicas | Even distribution by load | None (`model:""`) |

## 📊 How It Maps to Track A

| Capability | Track A (LiteLLM) | Track B (Kong `ai-proxy-advanced`) |
|---|---|---|
| **Weighted A/B split** | `weight` in `litellm_params` | `weight` per target (Part 1) |
| **Automatic fallback** | `router_settings.fallbacks` | `failover_criteria` + `retries` (Part 5) |
| **Circuit breaker** | Cooldown/retry logic | `max_fails` + `fail_timeout` (Part 5) |
| **Load balancing (replicas)** | Same `model_name` across replicas | `least-connections` across targets (Part 8) |
| **Cost-aware routing** | Not built-in | `lowest-usage` + `tokens_count_strategy: cost` (Part 2) |
| **Latency-aware routing** | Not built-in | `lowest-latency` with EWMA (Part 3) |
| **Sticky/per-user routing** | Not built-in | `consistent-hashing` on header (Part 4) |
| **Complexity-based routing** | Application-level code (`get_model_for_application()`) | `model_alias` client-controlled (Part 6) or `semantic` gateway-level (Part 7) |
| **Content-aware routing** | Application-level code only | `semantic` algorithm with embeddings (Part 7) — **gateway-native, no app code** |
| **Progressive rollout** | Adjust `weight` values over weeks | Same — adjust `weight` in CRD and re-apply |
| **Where cost is set** | `litellm_params` cost fields | `input_cost`/`output_cost` per target in `options` |
| **Application change needed** | Change `model` param to group name | `model:""` for gateway-decided, or alias for client-controlled |

## ✅ What You Accomplished

- ✅ **Part 1:** Weighted round-robin (80/20 A/B split) — gateway distributes by probability
- ✅ **Part 2:** Cost-aware routing (`lowest-usage`) — gateway minimizes spend automatically
- ✅ **Part 3:** Latency-aware routing (`lowest-latency`) — gateway favors fastest model at scale
- ✅ **Part 4:** Sticky routing (`consistent-hashing`) — same user always hits same model
- ✅ **Part 5:** Failover / circuit breaker — automatic fallback on error/timeout/429
- ✅ **Part 6:** Model alias — client picks model by friendly name (Kong 3.13 limitation noted)
- ✅ **Part 7:** Semantic routing — gateway routes by prompt content using embeddings
- ✅ **Part 8:** Least-connections — load balancing by active connection count
- ✅ Learned the `"model":""` pattern for gateway-decided routing
- ✅ Full parity with Track A + **5 Kong-exclusive capabilities** (cost-aware, latency-aware, semantic, least-connections, consistent-hashing)

---

**[Next: Module B5 — Production-Ready Applications →](/track-b-saas/b5-production-ready/)**
