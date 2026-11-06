---
title: "Langfuse - LLM Observability"
weight: 2
---

Throughout your interactions with vLLM and Bedrock models, Langfuse has been recording each request — capturing prompts, responses, performance metrics, and costs. This section examines the data it has collected about your AI usage.

## Hands-On: See Your AI Interactions Tracked

Review what Langfuse has captured about your AI usage:

### Step 1: Explore Your Observability Stack

:::code{language=bash showCopyAction=true}
# See the complete observability stack
kubectl get all -n langfuse
:::

You should see the following components:
- **langfuse-clickhouse-0**: Analytics database for performance data
- **langfuse-postgresql-0**: Database for trace metadata
- **langfuse-redis-master-0**: Caching layer
- **langfuse-s3-xxx**: Object storage for large data
- **langfuse-web-xxx**: Main Langfuse application

### Step 2: Explore the Real Configuration

In your VSC IDE, examine the actual configuration:

:::code{language=bash showCopyAction=true}
# Look at the actual Langfuse Helm values
cat /workshop/components/o11y/langfuse/values.template.yaml

# See the complete stack configuration
grep -A 5 "postgresql:\|clickhouse:\|redis:" /workshop/components/o11y/langfuse/values.template.yaml
:::

Notice how Langfuse is pre-configured with your project settings and automatically initialized.

## What is Langfuse?

Langfuse is an open-source LLM observability platform that provides:

- **Tracing**: Every LLM interaction captured with full context
- **Cost tracking**: Token usage and costs across all models
- **Performance analytics**: Latency, throughput, and quality metrics
- **Debugging tools**: Detailed request/response inspection
- **Usage analytics**: Patterns, trends, and optimization insights
- **Multi-database architecture**: PostgreSQL + ClickHouse + Redis for scale

## How Langfuse is Deployed

The Langfuse deployment uses the [official Langfuse Helm Chart](https://langfuse.com/self-hosting/deployment/kubernetes-helm) with a full observability stack. The full values.template.yaml file can be found at `/workshop/components/o11y/langfuse/values.template.yaml`. The following tabs describe each section of the Helm configuration:

:::::tabs

::::tab{label="Resources & Architecture"}
**Main Application Resources**

Langfuse requires adequate resources for processing and analyzing LLM traces:

:::code{language=yaml showCopyAction=true}
langfuse:
  resources:
    requests:
      cpu: 1 
      memory: 2Gi
    limits:
      memory: 2Gi
:::

**Multi-Database Architecture**

Langfuse uses a multi-database setup:

:::code{language=yaml showCopyAction=true}
postgresql:  # Metadata and user data
  auth:
    username: admin
    password: password123
  primary:
    resources: 
      requests:
        cpu: 125m
        memory: 256Mi

clickhouse:  # Analytics and time-series data
  auth:
    password: password123
  resources:
    requests:
      cpu: 1    
      memory: 3Gi

redis:  # Caching and sessions
  auth:
    password: password123
  primary:
    resources: 
      requests:
        cpu: 125m
        memory: 256Mi

s3:  # Object storage for large traces
  auth:
    rootPassword: password123
  resources:
    requests:
      cpu: 250m    
      memory: 256Mi
:::

**Why This Architecture:**
- **PostgreSQL**: Fast queries for recent traces and user management
- **ClickHouse**: Optimized for analytics and large-scale aggregations
- **Redis**: High-speed caching for UI responsiveness
- **S3**: Cost-effective storage for historical trace data
::::

::::tab{label="Authentication & Initialization"}
**Automatic Project Setup**

Langfuse is pre-configured with your workshop project and user account:

:::code{language=yaml showCopyAction=true}
langfuse:
  additionalEnv:
    - name: LANGFUSE_INIT_ORG_ID
      value: my-org
    - name: LANGFUSE_INIT_PROJECT_ID
      value: my-project
    - name: LANGFUSE_INIT_PROJECT_PUBLIC_KEY
      value: {{{LANGFUSE_PUBLIC_KEY}}}
    - name: LANGFUSE_INIT_PROJECT_SECRET_KEY
      value: {{{LANGFUSE_SECRET_KEY}}}
    - name: LANGFUSE_INIT_USER_EMAIL
      value: {{{LANGFUSE_USERNAME}}}
    - name: LANGFUSE_INIT_USER_PASSWORD
      value: {{{LANGFUSE_PASSWORD}}}
:::

**Initialization Features:**
- **Organization**: Automatically creates "my-org" organization
- **Project**: Sets up "my-project" with API keys
- **User Account**: Creates admin user with workshop credentials
- **API Keys**: Generates keys for LiteLLM integration
- **Zero Setup**: Everything ready to use immediately
::::

::::tab{label="Security Configuration"}
**Authentication & Security**

Langfuse security configuration:

:::code{language=yaml showCopyAction=true}
langfuse:
  salt:
    value: salt
  nextauth:
    secret:
      value: secret
:::

**Database Security**

All databases are configured with authentication:

:::code{language=yaml showCopyAction=true}
postgresql:
  auth:
    username: admin
    password: password123

clickhouse:
  auth:
    password: password123

redis:
  auth:
    password: password123

s3:
  auth:
    rootPassword: password123
:::

**Security Features:**
- **Password Protection**: All database components secured
- **NextAuth Integration**: Secure session management
- **Salt Configuration**: Password hashing security
- **API Key Management**: Secure key generation and storage
::::

::::tab{label="Networking & Ingress"}
**Load Balancer Configuration**

Langfuse uses AWS Application Load Balancer for external access:

:::code{language=yaml showCopyAction=true}
langfuse:
  ingress:
    enabled: true
    annotations:
      alb.ingress.kubernetes.io/target-type: ip
      {{#if DOMAIN}}
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
      {{/if}}
    hosts:
      - paths:
          - path: /
            pathType: Prefix
        {{#if DOMAIN}}
        host: langfuse.{{{DOMAIN}}}
        {{/if}}
:::

**Networking Features:**
- **ALB Integration**: Uses AWS Application Load Balancer
- **IP Target Type**: Direct pod networking for better performance
- **Conditional HTTPS**: HTTPS enabled when domain is configured
- **Path-Based Routing**: All traffic routed to Langfuse web interface
- **Dynamic Configuration**: Adapts based on domain availability
::::

::::tab{label="Deployment Commands"}
**Helm Deployment Process**

::alert[**Warning**: These commands have already been executed in your workshop environment. **Do not run these commands** — they will interfere with your existing setup.]{type="warning"}

The Langfuse deployment uses these Helm commands:

:::code{language=bash showCopyAction=true}
# Add the Langfuse Helm repository
helm repo add langfuse https://langfuse.github.io/langfuse-k8s
helm repo update

# Deploy Langfuse with custom values
helm upgrade --install langfuse langfuse/langfuse \
  --namespace langfuse \
  --create-namespace \
  -f values.rendered.yaml

# Check deployment status
kubectl rollout status deployment/langfuse-web -n langfuse
:::

**Deployment Process:**
1. **Template Rendering**: `values.template.yaml` → `values.rendered.yaml`
2. **Helm Installation**: Chart deployed with rendered values
3. **Database Initialization**: PostgreSQL, ClickHouse, Redis, S3 setup
4. **Project Setup**: Organization and project automatically created
5. **Integration Ready**: API keys configured for LiteLLM connection

**What Happens During Deployment:**
- Multi-database stack deployed (PostgreSQL, ClickHouse, Redis, S3)
- Langfuse web application started
- Organization and project automatically initialized
- User account created with workshop credentials
- Ingress/LoadBalancer exposes the service
- LiteLLM integration configured automatically
::::

:::::

---



## Explore the Langfuse Interface

Use the Langfuse interface to view your AI interactions, metrics, and detailed traces:

### Step 1: Access Langfuse

Get the URL for your Langfuse instance:

:::code{language=bash showCopyAction=true}
# Get Langfuse URL
echo "Langfuse URL: http://$(kubectl get ingress -n langfuse langfuse -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
:::

Open the URL in your browser. You'll see the Langfuse login page:

![Langfuse Login Page](/static/images/track-a-opensource/a1-genai-components/langfuse-login.png)

**Login Process:**
- **Email**: `admin@example.com`
- **Password**: `Pass@123`
- **Click "Sign in"** to access the interface

### Step 2: Navigate to Your Project

After logging in, you will see the organizations page:

![Langfuse Project Selection](/static/images/track-a-opensource/a1-genai-components/go-to-project.png)

**What to do:**
- **Click "Go to project"** button for the "Provisioned Project"
- This takes you to your workshop project with all your AI interaction data

### Step 3: Explore Your AI Usage Dashboard

You'll now see the main Langfuse dashboard:

![Langfuse Main Dashboard](/static/images/track-a-opensource/a1-genai-components/langfuse.png)

**Key Metrics You'll See:**
- **71 Total traces tracked** - Every AI interaction from your workshop
- **$0.10957 Total cost** - Your actual workshop spending
- **Model Breakdown**:
  - `global.anthropic.claude-sonnet-4-5`: 20.63K tokens, $0.106486
  - `openai.gpt-oss-20b-1:0`: 7.52K tokens, $0.001084
- **Usage Timeline**: Chart showing your interaction patterns over time

**What This Tells You:**
- Most of your cost comes from Claude 4.5 Sonnet
- GPT-OSS-20B has a lower per-token cost
- Self-hosted models have no per-token costs
- Usage patterns are visible over time

### Step 4: Browse Your Traces

Click on **"Tracing"** in the left sidebar to see all your interactions:

![Langfuse Traces List](/static/images/track-a-opensource/a1-genai-components/traces.png)

**What you're seeing:**
- **Complete trace history** - Every AI conversation you have had
- **Timestamps** - When each interaction occurred
- **Input/Output preview** - Snippets of your conversations
- **Search and filter** capabilities for finding specific interactions

**Your Actual Conversations:**
You can see traces of questions like:
- "Did snakes or sea snakes come first?"
- "Where did the word dragon come from?"
- "Generate 1-3 broad tags..."

### Step 5: Examine Individual Traces

Click on any trace to see detailed information:

![Individual Trace View](/static/images/track-a-opensource/a1-genai-components/trace.png)

**Trace Details:**
- **Trace ID**: Unique identifier for this interaction
- **Latency**: 5.91s response time
- **Total Cost**: $0.000459 for this specific interaction
- **Token Usage**: 63 input → 1,514 output tokens (1,577 total)
- **Full Conversation**: Complete user question and AI response

**Click the expand button** (orange icon in top right) to see more details.

### Step 6: Deep Dive into Trace Analytics

The expanded view shows the full trace information:

![Detailed Trace Analysis](/static/images/track-a-opensource/a1-genai-components/specific-trace.png)

**Advanced Metrics:**
- **Performance Details**: Environment, latency breakdown
- **Cost Analysis**: Exact cost calculation
- **Token Breakdown**: Input vs output token usage
- **Full Context**: Complete conversation with formatting

### Step 7: Analyze Generation Details

Click on the generation section to see the deepest level of detail:

![Generation Details](/static/images/track-a-opensource/a1-genai-components/generation.png)

**Generation Analytics:**
- **Latency**: 5.91s total, 5.90s time to first token
- **Model**: openai.gpt-oss-20b-1:0 (the exact model used)
- **Cost**: $0.000459 for this generation
- **Tokens**: 63 prompt → 1,514 completion tokens
- **Additional Parameters**: Stream settings, region, retry configuration

**What You Learn:**
- Exact model performance characteristics
- Cost breakdown per interaction
- Token efficiency patterns
- Response quality and speed trade-offs

### Step 8: Watch New Traces Appear

Now go back to your Open WebUI tab and send a message to any model. Watch the trace appear in Langfuse in real-time.

### Step 9: Create Custom Dashboards

Langfuse provides dashboard capabilities for visualizing your AI metrics. Explore the built-in dashboards and create your own custom analytics.

Navigate to **"Dashboards"** in the left sidebar:

![Dashboards Navigation](/static/images/track-a-opensource/a1-genai-components/dashboard.png)

**What you will see:**
- **Pre-built Dashboards** - Three default dashboards ready to use:
  - **Langfuse Cost Dashboard** - Review your LLM costs
  - **Langfuse Latency Dashboard** - Monitor latency metrics for performance optimization
  - **Langfuse Usage Management** - Track usage metrics across traces
- **"+ New dashboard"** button in the top right

::alert[**Dashboards list is empty?** The three pre-built dashboards and curated evaluator templates are seeded by the Langfuse `langfuse-worker` pod on first boot. If the worker was still initializing when Langfuse first became reachable, the list may appear empty for a few minutes. Restart the worker once and refresh\: `kubectl rollout restart deployment/langfuse-worker -n langfuse && kubectl rollout status deployment/langfuse-worker -n langfuse`.]{type="info"}

### Explore the Cost Dashboard

Click on **"Langfuse Cost Dashboard"** to see the cost analytics:

![Cost Dashboard Full View](/static/images/track-a-opensource/a1-genai-components/cost-dashboard-long.png)

**Dashboard Widgets:**
- **Total Count Traces**: 27 traces tracked
- **Total Count Observations**: 28 observations recorded
- **Total costs**: Line chart showing spending over time ($0.038 total)
- **Cost by Model Name**: Bar chart comparing model costs
- **Cost by Environment**: Donut chart showing environment distribution
- **Top 20 Users by Cost**: User spending breakdown
- **Top 20 Use Cases**: Cost analysis by trace and observation

### Filter for Workshop Activity

To see just your workshop activity, adjust the time filter:

1. Click the **"Past 7 days"** dropdown
2. Select **"Past 1 hour"**
3. Watch the dashboard update with your recent activity:

![Filtered Cost Dashboard](/static/images/track-a-opensource/a1-genai-components/cost-dashboard.png)

**What changes:**
- **Time range**: Now showing "Sep 06, 25 : 11:53 - Sep 06, 25 : 12:53"
- **Activity spikes**: Your recent model interactions clearly visible
- **Real-time costs**: Exact spending for your workshop session

### Create Your Own Dashboard

Build a custom latency-focused dashboard. Click **"+ New dashboard"** to start:

![Back to Dashboards](/static/images/track-a-opensource/a1-genai-components/back-dashboard.png)

In the creation dialog:

![Create Dashboard Dialog](/static/images/track-a-opensource/a1-genai-components/create-dashboard.png)

**Configure your dashboard:**
- **Dashboard Name**: Enter "Latency Dashboard"
- **Description**: Add a description (optional)
- Click **"Create"** to generate your dashboard

### Build Your First Widget

Your new dashboard starts empty:

![Empty Dashboard](/static/images/track-a-opensource/a1-genai-components/latency-dashboard.png)

**Add your first widget:**
1. Click **"+ Add Widget"** button
2. In the popup, select **"Create New Widget"**

### Configure Widget for Model Latency Comparison

Design a widget to compare average latency across models:

![Widget Configuration](/static/images/track-a-opensource/a1-genai-components/final-widget.png)

**Widget Settings:**

**Data Selection:**
- **View**: Select "Observations"
- **Metric**: Choose "Latency"
- **Aggregation**: Select "Avg"

**Visualization:**
- **Name**: "Avg Latency by Provided Model Name (Observations)"
- **Description**: "Shows the avg latency of Observations by provided model name"
- **Chart Type**: "Vertical Bar Chart"
- **Date Range**: "Past 3 hours"
- **Breakdown Dimension**: "Provided Model Name"

Click **"Save Widget"** to add it to your dashboard.

### View Your Custom Dashboard

Your dashboard now displays model latency comparisons:

![Final Dashboard with Widget](/static/images/track-a-opensource/a1-genai-components/final-dashboard.png)

**What your widget shows:**
- Because of the reasoning/thinking feature of the locally hosted Qwen models, their average latency is much higher than the models served by Bedrock.

::alert[**Challenge**: Create additional widgets to track metrics that matter to you. Consider adding widgets for token usage per model, success rates, or cost per hour.]{type="success"}

### Step 10: Advanced Features

Langfuse also supports advanced evaluation capabilities like:

**LLM-as-a-Judge**: Langfuse can automatically evaluate response quality using AI models as judges. This feature allows you to:

- **Automated Scoring**: Use AI models to rate response quality, accuracy, and helpfulness
- **Consistency Evaluation**: Compare responses across different models objectively
- **Quality Benchmarking**: Track improvement over time with standardized metrics
- **Continuous Evaluation**: Automatically assess every interaction for quality patterns

**Use Cases:**
- **Model Comparison**: Which model gives better responses for specific tasks?
- **Prompt Optimization**: Which prompts generate higher-quality responses?
- **Quality Monitoring**: Detect when model performance degrades
- **A/B Testing**: Compare different model configurations objectively

[Learn more about LLM-as-a-Judge →](https://langfuse.com/docs/evaluation/evaluation-methods/llm-as-a-judge)


## How Langfuse Records Traces

The following diagram shows what occurs each time you send a message:

```mermaid
sequenceDiagram
    participant User
    participant OpenWebUI
    participant LiteLLM
    participant Model
    participant Langfuse
    
    User->>OpenWebUI: Send message
    OpenWebUI->>LiteLLM: API request
    LiteLLM->>Model: Route to model
    LiteLLM->>Langfuse: Start trace
    Model-->>LiteLLM: Response
    LiteLLM->>Langfuse: Complete trace with metrics
    LiteLLM-->>OpenWebUI: Response
    OpenWebUI-->>User: Display response
    
    Note over Langfuse: Stores: prompt, response,<br/>tokens, cost, latency
```


## How Applications Connect to Langfuse

Langfuse supports multiple integration paths, giving you flexibility in how you instrument your applications:

| Integration Method | How It Works | Used In This Workshop |
|---|---|---|
| **LiteLLM Callbacks** | LiteLLM automatically sends traces for every LLM request that passes through the gateway. Zero application code changes needed. | Interacting with Models & Module A1 - all Open WebUI conversations are traced automatically |
| **Langfuse SDK** | Applications import the Langfuse `CallbackHandler` directly into their code for fine-grained, application-level tracing of agent workflows and tool calls. | Module A2 - Loan Buddy agent uses `langfuse.langchain.CallbackHandler` |
| **OpenTelemetry (OTEL)** | Standard OTEL protocol sends traces to Langfuse's OTEL-compatible endpoint (`/api/public/otel/v1/traces`). Useful for polyglot environments or existing OTEL instrumentation. | Pre-configured in Module A2 deployment for extensibility |

**This workshop uses a dual-layer tracing approach:**
1. **Gateway layer** (LiteLLM Callbacks) captures every LLM API call with token counts, costs, and latency
2. **Application layer** (Langfuse SDK) captures the higher-level agent workflow, tool invocations, and decision chains

This combination gives you both the low-level model performance data and the high-level application behavior in a single Langfuse dashboard.

## Key Insights You Can Discover

- **Model Performance**: Compare response times between your models
- **Cost Analysis**: See exactly how much each interaction cost
- **Usage Patterns**: Understand your AI interaction habits
- **Quality Tracking**: Identify which models gave better responses
- **Token Efficiency**: See which prompts were most effective


## What's Next?

Now that you have explored both LiteLLM (your API gateway) and Langfuse (your observability platform), you have a complete GenAI platform running on EKS.

In the next module, this platform foundation is used to build AI applications with agents, memory, and reasoning capabilities.

---

**[Continue to Module A2: Building GenAI Applications →](/track-a-opensource/a2-genai-applications/)**
