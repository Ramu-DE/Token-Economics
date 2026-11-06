---
title: "A/B Model Routing with LiteLLM"
weight: 2
---

Your evaluation data shows that Qwen 3 8B scores 8.1/10 for helpfulness at $0.002 per decision, while Claude 4.5 Sonnet scores 9.5/10 at $0.045 per decision. A/B model routing enables you to automatically route simple cases to Qwen (95% cost savings) while reserving Claude for complex scenarios.

## Hands-On: Understand Current Routing

First, examine how LiteLLM currently routes requests:

### Step 1: Check Current Model Configuration

:::code{language=bash showCopyAction=true}
# View current LiteLLM model configuration
kubectl get configmap litellm-config -n litellm -o yaml | grep -A 20 "model_list:"
:::

**What you will see:**

:::code{language=yaml showCopyAction=false}
# Current configuration (all models equal weight):
model_list:
  - model_name: bedrock/claude-4.5-sonnet
    litellm_params:
      model: bedrock/global.anthropic.claude-sonnet-4-5-20250929-v1:0
      aws_region_name: us-west-2
  
  - model_name: vllm/qwen3-8b-neuron
    litellm_params:
      model: openai/qwen3-8b-neuron
      api_base: http://qwen3-8b-neuron.vllm:8000/v1
      api_key: fake-key
:::

**Current Behavior:**
- Applications explicitly specify which model to use
- No automatic routing or fallback
- No load balancing between similar models

### Step 2: Check Current Request Distribution

Check how requests are currently distributed:

:::code{language=bash showCopyAction=true}
# Query Langfuse for model usage distribution
echo "Open Langfuse and navigate to: Dashboards → Usage → Model Distribution"
echo "Langfuse URL: http://$(kubectl get ingress -n langfuse langfuse -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
:::

**Expected Current Distribution:**
- **Claude 4.5 Sonnet**: ~90% (default for Loan Buddy)
- **Qwen 3 8B**: ~10% (manual testing only)
- **Total Cost**: High (mostly premium model)

## What is A/B Model Routing?

**A/B routing** allows LiteLLM to automatically distribute traffic across multiple models based on configurable weights. It functions as a load balancer that can split traffic 80/20, 50/50, or any ratio you choose.

**Key Capabilities:**

::::tabs

:::tab{label="Weighted Distribution"}
**Traffic Splitting**

Split requests across models by percentage:

```yaml
# Example: 80% Claude, 20% Qwen
model_list:
  - model_name: loan-processing
    litellm_params:
      model: bedrock/global.anthropic.claude-sonnet-4-5-20250929-v1:0
      weight: 8  # 80% traffic
  - model_name: loan-processing
    litellm_params:
      model: openai/qwen3-8b-neuron
      weight: 2  # 20% traffic
```

**Use Cases:**
- **Gradual Rollout**: Start a new model at 10%, increase to 50%, then 100%
- **Cost Optimization**: 80% lower-cost model, 20% premium for validation
- **Performance Testing**: Split traffic to measure real-world performance
:::

:::tab{label="Automatic Fallback"}
**Resilience & Reliability**

Automatically failover when primary model is unavailable:

```yaml
# Fallback cascade:
router_settings:
  fallbacks: [{"vllm/qwen3-8b-neuron": ["bedrock/claude-4.5-sonnet"]}]
  # Try fast, cheap model first → fall back to reliable premium
```

**Fallback Triggers:**
- Model timeout (>30s response time)
- Service unavailable (503 error)
- Rate limit exceeded (429 error)
- Model crash or restart
:::

:::tab{label="Load Balancing"}
**Horizontal Scaling**

Distribute load across multiple replicas:

```yaml
# Balance across 3 Qwen replicas (same model_name = automatic load balancing):
model_list:
  - model_name: vllm/qwen3-8b-neuron
    litellm_params:
      model: openai/qwen3-8b-neuron
      api_base: http://qwen3-8b-neuron-1.vllm:8000/v1
  - model_name: vllm/qwen3-8b-neuron
    litellm_params:
      model: openai/qwen3-8b-neuron
      api_base: http://qwen3-8b-neuron-2.vllm:8000/v1
  - model_name: vllm/qwen3-8b-neuron
    litellm_params:
      model: openai/qwen3-8b-neuron
      api_base: http://qwen3-8b-neuron-3.vllm:8000/v1
```

**Benefits:**
- Higher throughput (3x capacity)
- Better resilience (1 replica down = still 66% capacity)
- Rolling updates (update replicas one at a time)
:::

:::tab{label="Smart Routing Rules"}
**Context-Aware Routing**

Route based on request characteristics:

```python
# Example: Application-level routing based on request characteristics
def route_by_complexity(prompt: str, metadata: dict) -> str:
    """Route based on request complexity."""
    prompt_length = len(prompt.split())
    
    if prompt_length < 100:  # Simple, short requests
        return "loan-processing-simple"  # Qwen
    elif metadata.get("user_tier") == "premium":
        return "bedrock/claude-4.5-sonnet"  # Premium users get Claude
    else:
        return "loan-processing"  # A/B split for standard requests
```

**Routing Factors:**
- Prompt length (token count)
- Request tags/metadata
- User tier (free vs premium)
- Time of day (off-peak = lower-cost models)
- Cost budget constraints
:::

::::

## Hands-On: Configure 80/20 A/B Routing

Set up weighted routing to send 80% of traffic to Claude and 20% to Qwen for cost comparison:

### Step 1: Backup Current Configuration

:::code{language=bash showCopyAction=true}
# Backup existing LiteLLM values
cp /workshop/components/ai-gateway/litellm/values.rendered.yaml \
   /workshop/components/ai-gateway/litellm/values.rendered.yaml.backup

# Verify backup
ls -lh /workshop/components/ai-gateway/litellm/values.rendered.yaml*
:::

### Step 2: Edit LiteLLM Configuration

Open the configuration file:

:::code{language=bash showCopyAction=true}
# Open in editor
code /workshop/components/ai-gateway/litellm/values.rendered.yaml
:::

### Step 3: Add Router Configuration

Add the following configuration to enable 80/20 weighted routing. In LiteLLM, you create multiple deployments with the **same** `model_name` and different `weight` values:

:::code{language=yaml showCopyAction=true}
# All LiteLLM config lives under the top-level `proxy_config:` key
# (this is the chart schema introduced in Module A1). Add the new
# `loan-processing` entries to model_list and the router_settings
# block under the existing `proxy_config:`.
proxy_config:
  model_list:
    # Individual models (keep for direct access)
    - model_name: bedrock/claude-4.5-sonnet
      litellm_params:
        model: bedrock/global.anthropic.claude-sonnet-4-5-20250929-v1:0
        aws_region_name: us-west-2

    - model_name: vllm/qwen3-8b-neuron
      litellm_params:
        model: openai/qwen3-8b-neuron
        api_base: http://qwen3-8b-neuron.vllm:8000/v1
        api_key: fake-key

    # A/B routing group: same model_name, different weights
    - model_name: loan-processing
      litellm_params:
        model: bedrock/global.anthropic.claude-sonnet-4-5-20250929-v1:0
        aws_region_name: us-west-2
        weight: 8    # 80% traffic

    - model_name: loan-processing
      litellm_params:
        model: openai/qwen3-8b-neuron
        api_base: http://qwen3-8b-neuron.vllm:8000/v1
        api_key: fake-key
        weight: 2    # 20% traffic

  router_settings:
    routing_strategy: simple-shuffle
    num_retries: 2
    timeout: 30
    fallbacks: [{"loan-processing": ["bedrock/claude-4.5-sonnet"]}]

  litellm_settings:
    callbacks: ["langfuse"]
    success_callback: ["langfuse"]
    failure_callback: ["langfuse"]

  general_settings:
    master_key: os.environ/PROXY_MASTER_KEY
    store_model_in_db: true
:::

**Configuration Breakdown:**

| Setting | Value | Purpose |
|---------|-------|---------|
| `routing_strategy` | simple-shuffle | Use probability-based distribution |
| `num_retries` | 2 | Retry failed requests up to 2 times |
| `timeout` | 30 | Max 30 seconds per request |
| `weight: 8` | 80% | Claude gets 80% of traffic |
| `weight: 2` | 20% | Qwen gets 20% of traffic |
| `fallbacks` | loan-processing→Claude | If any model in group fails, try Claude |

**What This Does:**
- Creates a virtual model group called "loan-processing"
- When applications request "loan-processing", LiteLLM automatically routes 80/20 based on weights
- Individual models (bedrock/claude-4.5-sonnet, vllm/qwen3-8b-neuron) still accessible for testing
- If any deployment fails, requests automatically fall back to Claude

### Step 4: Apply Configuration Changes

Deploy the updated configuration:

:::code{language=bash showCopyAction=true}
# Apply the new configuration
helm upgrade --install litellm oci://ghcr.io/berriai/litellm-helm \
  --namespace litellm \
  -f /workshop/components/ai-gateway/litellm/values.rendered.yaml \
  --wait

# Verify rollout completed
kubectl rollout status deployment/litellm -n litellm

# Check logs for router initialization
kubectl logs -n litellm deployment/litellm --tail=50 | grep -i "router"
:::

After the rollout completes, the router init logs should look similar to the following (exact wording and order vary by LiteLLM version):

::::expand{header="Check Output"}

```yaml
# Router initialization (illustrative):
INFO: Router initialized with model group: loan-processing
INFO: Model group 'loan-processing' has 2 deployments with weights ~[0.8, 0.2]
INFO: Fallbacks configured for loan-processing
INFO: LiteLLM proxy started successfully on port 4000
```

::::

::alert[**Deployment Time**: Configuration changes take ~30-60 seconds to apply. Wait for "rollout status" to show "successfully rolled out" before testing.]{type="info"}

## Hands-On: Test A/B Routing

Verify the 80/20 distribution is working:

### Step 1: Point the Agent at the Model Group

The 80/20 split lives entirely in the LiteLLM gateway you just configured — the agent code does not need to change its routing logic. The agent already sends every request to the gateway (using `GATEWAY_MODEL_ACCESS_KEY`); you only need it to request the new `loan-processing` model **name** instead of a single model.

The model name the agent uses is set in code (`llm_model` in `credit-underwriting-agent.py`), not in a YAML config file. To switch the agent over, update that one line and redeploy:

:::code{language=bash showCopyAction=true}
# The model name is hardcoded in the agent source, not a config file
grep -n 'llm_model' \
  /workshop/apps/loan-buddy/credit-validation/credit-underwriting-agent.py
:::

:::code{language=python showCopyAction=true}
# Change from:
llm_model = "bedrock/claude-4.5-sonnet"

# To (uses the gateway 80/20 routing automatically):
llm_model = "loan-processing"
:::

Redeploy Loan Buddy to pick up the change:

:::code{language=bash showCopyAction=true}
# Restart the agent pods
kubectl rollout restart deployment/loan-buddy-agent -n workshop

# Wait for restart
kubectl rollout status deployment/loan-buddy-agent -n workshop
:::

::alert[**Prefer to leave the agent untouched?** Because the routing is done by the gateway, you can also keep the agent pointed at any model name and test the 80/20 split directly against the gateway with `curl` (next step) — no redeploy required.]{type="info"}

### Step 2: Send Test Requests

Test the routing directly through LiteLLM to see the 80/20 distribution:

:::code{language=bash showCopyAction=true}
# Port-forward LiteLLM service
kubectl port-forward svc/litellm 4000:4000 -n litellm &

# Get the gateway API key (same key the agents use)
source /workshop/.env
export LITELLM_KEY=$AGENT_KEY

# Send 20 test requests
for i in {1..20}; do
  curl -X POST http://localhost:4000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $LITELLM_KEY" \
    -d "{
      \"model\": \"loan-processing\",
      \"messages\": [{
        \"role\": \"user\",
        \"content\": \"Evaluate loan application: Test User $i, income \$75000, credit 720, requesting \$250000\"
      }]
    }" &
  
  # Small delay between requests
  sleep 0.5
done

# Wait for all background jobs
wait

echo "Sent 20 test requests to loan-processing model group"
:::

### Step 3: Verify Distribution in Langfuse

Open Langfuse and check the model distribution:

:::code{language=bash showCopyAction=true}
# Open Langfuse
echo "Langfuse URL: http://$(kubectl get ingress -n langfuse langfuse -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
echo "Navigate to: Dashboards → Create custom dashboard"
:::

Create a quick distribution widget:

**Widget Configuration:**
- **View**: Traces
- **Metric**: Count
- **Breakdown**: Model Name
- **Date Range**: Past 1 hour
- **Chart Type**: Donut Chart

**Expected Results:**

:::code{language=yaml showCopyAction=false}
# Model distribution from 20 requests:
bedrock/global.anthropic.claude-sonnet-4-5: 16 requests (80%)
openai/qwen3-8b-neuron: 4 requests (20%)

# Your exact numbers may vary slightly due to randomness
# but should be close to 80/20 split
:::

::alert[**Statistical Note**: With only 20 requests, you might see 15/5 or 17/3 splits due to random variation. Run 100+ requests to see the distribution converge to exactly 80/20.]{type="info"}

### Step 4: Test Fallback Behavior

Verify that requests automatically failover when Qwen is unavailable:

:::code{language=bash showCopyAction=true}
# Temporarily scale Qwen to 0 replicas (simulate failure)
kubectl scale deployment qwen3-8b-neuron -n vllm --replicas=0

# Wait for pods to terminate
sleep 10

# Send test request (should automatically use Claude via fallback)
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $LITELLM_KEY" \
  -d '{
    "model": "loan-processing",
    "messages": [{
      "role": "user",
      "content": "Evaluate loan: Fallback Test, income $75000, credit 720, requesting $250000"
    }]
  }'

# Check which model handled it in Langfuse
echo "Check Langfuse - trace should show Claude (fallback activated)"

# Restore Qwen
kubectl scale deployment qwen3-8b-neuron -n vllm --replicas=1
:::

**What You Will See in Langfuse:**

In the Langfuse trace for this request, the fallback metadata looks like:

**Trace Metadata:**
- **Requested Model**: loan-processing
- **Attempted Model**: vllm/qwen3-8b-neuron
- **Fallback Reason**: "Connection refused - target service unavailable"
- **Actual Model**: bedrock/global.anthropic.claude-sonnet-4-5
- **Fallback Time**: 2.1s (immediate detection + Claude response)

## Hands-On: Analyze Cost vs Quality Trade-offs

Now compare the A/B results to make data-driven routing decisions:

### Step 1: Create Cost Comparison Dashboard

In Langfuse, create a comprehensive A/B comparison dashboard:

**Dashboard Name**: `A/B Test: Claude vs Qwen (80/20)`

Add these widgets:

**Widget 1: Cost Per Model**

:::code{language=yaml showCopyAction=false}
View: Traces
Metric: Total Cost (Sum)
Breakdown: Model Name
Chart: Vertical Bar
Date Range: Past 1 hour
:::

**Widget 2: Average Latency Per Model**

:::code{language=yaml showCopyAction=false}
View: Traces
Metric: Latency (Average)
Breakdown: Model Name
Chart: Horizontal Bar
Date Range: Past 1 hour
:::

**Widget 3: Quality Scores by Model**

:::code{language=yaml showCopyAction=false}
View: Scores
Metric: Score Value (Average)
Breakdown: Model Name + Score Name
Chart: Grouped Bar
Date Range: Past 1 hour
Filter: Score Name in [loan-accuracy, loan-helpfulness, loan-safety]
:::

### Step 2: View Comparative Results

**Analysis Results (from 100 requests):**

| Metric | Claude 4.5 Sonnet (80 req) | Qwen 3 8B (20 req) | Difference |
|--------|----------------------------|---------------------|------------|
| **Total Cost** | $3.60 | $0.04 | Claude 90x more expensive |
| **Avg Cost/Request** | $0.045 | $0.002 | $0.043 savings per Qwen request |
| **Avg Latency** | 3.2s | 5.8s | Qwen 2.6s slower |
| **Accuracy Score** | 9.2/10 | 8.3/10 | -0.9 points |
| **Helpfulness Score** | 9.5/10 | 8.1/10 | -1.4 points |
| **Safety Score** | 10.0/10 | 9.7/10 | -0.3 points |

**Business Impact Calculation:**

:::code{language=yaml showCopyAction=false}
# Current 80/20 split results:
Daily volume: 1,000 applications
Claude (80%): 800 req × $0.045 = $36.00
Qwen (20%): 200 req × $0.002 = $0.40
Daily cost: $36.40

# Compare to 100% Claude:
1,000 req × $0.045 = $45.00/day
Savings: $8.60/day = $258/month (19% reduction)

# Compare to 100% Qwen:
1,000 req × $0.002 = $2.00/day
Savings: $34.40/day = $1,032/month (95% reduction)
BUT: Quality drops 1-1.4 points on key metrics
:::

### Step 3: Implement Complexity-Based Routing

LiteLLM does not have built-in "quality-aware" routing, but you can implement intelligent routing at the application level by defining multiple model groups and routing to them based on request characteristics.

Update your LiteLLM configuration with separate model groups for simple and complex cases:

:::code{language=yaml showCopyAction=true}
model_list:
  # Simple cases: Qwen only (cheap, fast)
  - model_name: loan-processing-simple
    litellm_params:
      model: openai/qwen3-8b-neuron
      api_base: http://qwen3-8b-neuron.vllm:8000/v1
      api_key: fake-key
  
  # Complex cases: Claude only (premium, accurate)
  - model_name: loan-processing-complex
    litellm_params:
      model: bedrock/global.anthropic.claude-sonnet-4-5-20250929-v1:0
      aws_region_name: us-west-2
  
  # General A/B testing group (80/20 split)
  - model_name: loan-processing
    litellm_params:
      model: bedrock/global.anthropic.claude-sonnet-4-5-20250929-v1:0
      aws_region_name: us-west-2
      weight: 8
  - model_name: loan-processing
    litellm_params:
      model: openai/qwen3-8b-neuron
      api_base: http://qwen3-8b-neuron.vllm:8000/v1
      api_key: fake-key
      weight: 2

router_settings:
  routing_strategy: simple-shuffle
  fallbacks: [
    {"loan-processing-simple": ["loan-processing-complex"]},
    {"loan-processing": ["bedrock/claude-4.5-sonnet"]}
  ]
:::

Then implement application-level routing logic in your Loan Buddy agent:

:::code{language=python showCopyAction=true}
def get_model_for_application(credit_score: int, dti_ratio: float) -> str:
    """Route to appropriate model based on application complexity."""
    if credit_score >= 700 and dti_ratio <= 36:
        # High credit score + low DTI = simple case
        return "loan-processing-simple"   # Qwen: fast + cheap
    else:
        # Low credit or high DTI = edge case needing careful analysis
        return "loan-processing-complex"  # Claude: accurate for edge cases

# Usage in agent:
model_name = get_model_for_application(
    credit_score=applicant.credit_score,
    dti_ratio=applicant.debt_to_income_ratio
)

response = litellm_client.chat.completions.create(
    model=model_name,
    messages=[{"role": "user", "content": prompt}]
)
:::

**Expected Optimization:**

:::code{language=yaml showCopyAction=false}
# Intelligent routing results (assuming typical distribution):
70% of applications are simple (credit ≥700, DTI ≤36) → 100% Qwen
30% of applications are complex → 100% Claude

New cost breakdown:
- Simple (700/day): 700 Qwen × $0.002 = $1.40
- Complex (300/day): 300 Claude × $0.045 = $13.50
- Total: $14.90/day vs $45/day baseline = 67% cost reduction
- Quality maintained: Complex cases still get best model
:::

## Hands-On: Implement Progressive Rollout

Use A/B routing for safe model deployment:

### Step 1: Gradual Traffic Shift Strategy

**Scenario**: You want to roll out a new fine-tuned Qwen model. Use progressive rollout to minimize risk:

:::code{language=yaml showCopyAction=true}
# Week 1: Canary (5% new model)
model_list:
  - model_name: loan-processing
    litellm_params:
      model: openai/qwen3-8b-neuron-v2  # New fine-tuned model
      api_base: http://qwen3-8b-neuron-v2.vllm:8000/v1
      weight: 1  # 5% traffic
  - model_name: loan-processing
    litellm_params:
      model: bedrock/global.anthropic.claude-sonnet-4-5-20250929-v1:0
      weight: 19  # 95% traffic

# Week 2: Increase to 25% if quality maintained (weight: 5 vs 15)
# Week 3: Increase to 75% if quality maintained (weight: 15 vs 5)
# Week 4: Full rollout at 100% if successful (weight: 20 vs 0)
:::

### Step 2: Monitor Rollout Health

Create a rollout health dashboard:

**Key Metrics to Watch:**
- **Error Rate**: Should stay below 1%
- **P95 Latency**: Should not increase more than 20%
- **Quality Scores**: Should not drop more than 0.5 points
- **Cost**: Track savings realization
- **Rollback Count**: Monitor fallback activations

**Rollout Decision Rules:**

| Metric Status | Action |
|---------------|--------|
| All metrics healthy for 24h | Increase traffic by 25% |
| Any metric degraded | Hold current percentage, investigate |
| Any metric failing | Roll back to previous percentage |

## Production Patterns for A/B Routing

**Pattern 1: Time-Based Routing**

:::code{language=python showCopyAction=true}
# Use cheaper models during low-stakes hours (application-level logic):
from datetime import datetime
import pytz

def get_model_by_time() -> str:
    """Route to appropriate model based on time of day."""
    et = pytz.timezone('US/Eastern')
    current_hour = datetime.now(et).hour
    
    if 9 <= current_hour < 17:  # Business hours (9am-5pm ET)
        return "bedrock/claude-4.5-sonnet"  # Premium model
    else:  # Off hours
        return "loan-processing"  # 80/20 split (mostly Qwen)
:::

**Pattern 2: User-Tier Based Routing**

:::code{language=python showCopyAction=true}
# Premium customers get premium models (application-level logic):
def get_model_by_tier(user_tier: str) -> str:
    """Route to appropriate model based on user subscription tier."""
    if user_tier == "premium":
        return "bedrock/claude-4.5-sonnet"  # 100% Claude for paying customers
    elif user_tier == "free":
        return "loan-processing-simple"  # Qwen for free tier (cost control)
    else:  # standard tier
        return "loan-processing"  # 80/20 A/B split
:::

**Pattern 3: Load-Based Routing**

:::code{language=yaml showCopyAction=true}
# LiteLLM automatically load-balances across multiple replicas with same model_name:
model_list:
  # Deploy 3 Qwen replicas for horizontal scaling
  - model_name: vllm/qwen3-8b-neuron
    litellm_params:
      model: openai/qwen3-8b-neuron
      api_base: http://qwen3-8b-neuron-1.vllm:8000/v1
      api_key: fake-key
  
  - model_name: vllm/qwen3-8b-neuron
    litellm_params:
      model: openai/qwen3-8b-neuron
      api_base: http://qwen3-8b-neuron-2.vllm:8000/v1
      api_key: fake-key
  
  - model_name: vllm/qwen3-8b-neuron
    litellm_params:
      model: openai/qwen3-8b-neuron
      api_base: http://qwen3-8b-neuron-3.vllm:8000/v1
      api_key: fake-key

router_settings:
  routing_strategy: simple-shuffle  # Round-robin across replicas
:::

## Key Benefits of A/B Model Routing

The exercises in this section demonstrated:

- **Cost optimization**: 67% cost reduction through complexity-based routing
- **Quality maintenance**: Complex cases still use premium models
- **Resilience**: Automatic failover avoids downtime
- **Safe rollouts**: Progressive traffic shifting minimizes deployment risk
- **Data-driven decisions**: Real metrics guide model selection
- **Flexibility**: Adjust routing strategy based on business needs

## What's Next?

You have now implemented A/B model routing. Your platform can:
- Automatically distribute traffic 80/20 across models
- Fail over immediately when models are unavailable
- Compare cost vs quality trade-offs with real data
- Roll out new models progressively with safety controls

The next step is to monitor the infrastructure health underneath these models. The following section adds production-grade monitoring with Amazon Managed Prometheus and Grafana to track vLLM performance, Neuron utilization, and auto-scale based on real-time metrics.

---

**[Next: Infrastructure Monitoring and Auto-Scaling →](../monitoring-scaling/)**
