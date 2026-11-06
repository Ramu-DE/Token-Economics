---
title: "Amazon Bedrock - Managed AI Services"
weight: 23
---

Amazon Bedrock is a fully managed service that provides access to high-performing foundation models from leading AI companies through a single API. In this section, you will configure access to Claude Sonnet 4.5 and learn how to use Bedrock models for the GenAI platforms you configure on EKS.

## What is Amazon Bedrock?

Amazon Bedrock offers:

- **Instant Access**: No infrastructure to manage or models to deploy
- **Foundation Models**: Claude (Anthropic), Llama (Meta), Mistral, Amazon Titan, and more
- **Pay-Per-Use**: Only pay for tokens processed, no idle costs
- **Security**: Data privacy, compliance, and governance built-in
- **Performance**: Optimized infrastructure with consistent low latency
- **Additional Features**: Fine-tuning, RAG, agents, and guardrails

## Why Use Bedrock for This Workshop?

In the previous section, you used self-hosted models on the workshop infrastructure. Due to hardware constraints (inf2.xlarge instances), the self-hosted models have limited performance. For the remaining modules, Bedrock is recommended:

1. **No Hardware Constraints**: Not limited by workshop instance sizes
2. **Consistent Performance**: Reliable response times for hands-on exercises  
3. **Latest Models**: Access to Claude Sonnet 4.5 with advanced capabilities
4. **Focus on Learning**: Spend time on concepts, not waiting for responses

::alert[**Workshop Suggestion**: For the remaining modules, we recommend using Bedrock (Claude Sonnet 4.5). Both self-hosted and managed approaches are appropriate in different scenarios.]{type="info"}

## Using Bedrock Models in Open WebUI

Once model access is enabled, you can interact with Bedrock models via Open WebUI through LiteLLM:

### Step 1: Access Open WebUI

1. Use the OpenWebUI URL from your previous commands
2. Log in with your credentials

### Step 2: Select Bedrock Model

1. Click the model dropdown at the top of the chat
2. Look for **"claude-4.5-sonnet"** (it should appear automatically)
3. Select it as your active model

### Step 3: Test Claude Sonnet 4.5

Test Claude's capabilities with some practical examples.

#### Test 1: Advanced Technical Reasoning

Ask Claude this question to observe its technical depth:

:::code{language=markdown showCopyAction=true}
Explain the concept of Kubernetes operators and provide a simple example of when you would create a custom operator.
:::

**Expected Response**: Claude provides a comprehensive explanation with clear structure:

![Claude's Kubernetes Operators Response](/static/images/module-interacting-with-models/claude-kubernetes-operators-response.png)

**What to Notice**:
- **Structured Response**: Clear headings and logical flow
- **Technical Depth**: Explains control loops, CRDs, and practical examples
- **Real-World Context**: Provides concrete use cases

#### Test 2: Code Generation (Optional)

Try this for code generation capabilities:

:::code{language=markdown showCopyAction=true}
Write a Python class that implements a simple LRU cache with get and put methods. Explain the time complexity of each operation.
:::

#### Test 3: Long-Form Analysis (Optional)

For complex analysis tasks:

:::code{language=markdown showCopyAction=true}
Analyze the benefits and challenges of running AI workloads on Kubernetes. Consider aspects like scalability, resource management, and operational complexity.
:::

**Key Observation**: Claude provides detailed, well-structured responses that are well-suited for the learning exercises in the upcoming modules.

## Model Comparison: Exploring Different Approaches

Open WebUI includes a feature that allows you to compare responses from different models side-by-side. Use it to understand how different models approach the same problem and to choose an appropriate model for specific tasks.

### How to Compare Models

1. **Select your first model** (e.g., Claude Sonnet 4.5)
2. **Click the "+" button** next to the model name to add a second model
3. **Choose a second model** (e.g., vLLM Deepseek)
4. **Click the "+" button** next to the model name to add a third model
5. **Choose a third model** (e.g., vLLM Qwen 3)
6. **Ask the same question** to all three models simultaneously

### Example Comparison

Try this mathematical question with all three models:

**Test Question**: "What is the square root of 144 divided by 29 multiplied by pi"

![Model Comparison: Side-by-Side Results](/static/images/module-interacting-with-models/model-comparison-claude-vs-llama.png)

::alert[**/no_think**: Some reasoning models (like Qwen) spend extra time on a "thinking" step before responding. If this takes too long (6 minutes in this example), add `/no_think` to your prompt to skip the reasoning step and get a direct answer.]{type="info"}

### What You Will Observe

**Different Response Styles**: Each model has its own approach to problem-solving:
- Some models provide step-by-step explanations
- Others give direct, concise answers
- Response formatting and detail levels vary

**Learning Opportunity**: This comparison feature helps you:
- Understand model characteristics
- Choose appropriate models for different use cases
- Evaluate response quality and style preferences
- Learn from different problem-solving approaches

## Advanced Features: RAG in Open WebUI

Open WebUI includes basic RAG (Retrieval Augmented Generation) capabilities. The following demonstration shows how RAG changes model responses by providing specific context from your documents.

### Step 1: Create a Test Document

First, create a sample document in your VSC IDE terminal:

:::code{language=bash showCopyAction=true}
# Create the test file in your VSC IDE terminal
cat > super-secret.txt << 'EOF'
Super Secret Document

This document contains classified information about the secret project.
Only authorized personnel are allowed to access this data.

Project Codename: Nightfall
Objective: Develop an untraceable communication device.
Status: In progress

Remember, this information is top secret and must not be shared with anyone.
EOF

code super-secret.txt
:::

### Step 2: Download to Your Local Device

After creating the file in VSC IDE:

1. **Locate the file** in your VSC IDE file explorer
2. **Right-click** on `super-secret.txt`
3. **Select "Download"** to save it to your local machine
4. **Remember the download location** - you will need to upload it to OpenWebUI

::alert[**Why Download?** OpenWebUI runs in your browser and needs files from your local machine, not from the remote VSC IDE environment.]{type="info"}

### Step 3: Create Knowledge Base in OpenWebUI

Now set up the knowledge base in OpenWebUI:

1. **Access Workspace**: Click on **"Workspace"** and then **"Knowledge"**

![Knowledge Tab](/static/images/module-interacting-with-models/openwebui-knowledge-workspace.png)

2. **Create New Knowledge Base**: Click the **"+"** button on the right side

3. **Fill out the form**:
   - **Name**: `super-secret-info`
   - **Description**: `Store secret info in knowledge base`
   - **Visibility**: Select **"Private"** 

![Create Knowledge Base Form](/static/images/module-interacting-with-models/openwebui-create-knowledge-base.png)

4. **Create and Upload**:
   - Click **"Create Knowledge"**
   - Click on the **+**
   - Click **"Upload files"**
   - Select your downloaded `super-secret.txt` file
   - Wait for the upload to complete

### Step 4: Test WITHOUT RAG (Baseline)

First, observe how Claude responds without access to the document:

**Ask this question**: "What is the objective of project nightfall?"

![Claude Without RAG Context](/static/images/module-interacting-with-models/claude-without-rag-context.png)

**Notice**: Claude gives a generic response about a social media platform because it has no knowledge of the document content.

### Step 5: Test WITH RAG

Now observe the effect of RAG in action:

1. **Start a new chat**
2. **Type `#`** in the message box - this activates knowledge base selection

![Knowledge Base Selection](/static/images/module-interacting-with-models/openwebui-select-knowledge-base.png)

3. **Select your knowledge base**: Click on **"super-secret-info"**
4. **Ask the exact same question**: "What is the objective of project nightfall?"

![Claude With RAG Context](/static/images/module-interacting-with-models/claude-with-rag-context.png)

Now Claude correctly references the document and provides the exact information: "Develop an untraceable communication device."

### The Effect of RAG

This demonstrates the fundamental capability of RAG:

- **Without RAG**: Claude uses only its training data, producing a generic, incorrect response
- **With RAG**: Claude uses your specific document as context, producing an accurate, relevant response

The same model and the same question produce different answers depending on the available context.

### What You Just Experienced

This is OpenWebUI's built-in RAG capability. You have seen how:

- Documents can be uploaded and indexed
- AI models can retrieve relevant information from your documents
- Responses become contextually accurate and specific to your data

::alert[**Simple RAG**: This is an introductory example. OpenWebUI provides basic RAG functionality. There are many different approaches to implementing more advanced RAG capabilities.]{type="success"}

## Key Takeaways

- **Instant Deployment**: No infrastructure to manage, models ready immediately
- **Managed Service Benefits**: Optimized infrastructure without operational overhead
- **Advanced Models**: Access to Claude 4.5 with 200K context window
- **Pay-Per-Use**: Cost model suited for variable workloads
- **Additional Features**: Built-in security, compliance, and governance

## Module Summary

You have now used three different approaches to model deployment:

1. **Open WebUI**: Unified chat interface for all models
2. **vLLM**: Self-hosted models with full control but hardware constraints
3. **Bedrock**: Managed service with consistent performance

### Suggested Approach for Remaining Modules

**Use Bedrock (Claude Sonnet 4.5)** for the remaining modules:
- Consistent performance for hands-on exercises
- Advanced capabilities for complex learning scenarios
- No infrastructure management overhead
- Reliable responses for interactive learning

### What You Have Learned

- How to deploy and configure LLM infrastructure on Kubernetes
- The trade-offs between self-hosted and managed models
- How to integrate multiple model providers through a unified API
- The importance of hardware selection for AI workloads
- Cost and performance optimization strategies

---

## Choose Your Track

You have completed the shared foundation. Now pick the stack you want to build the Agentic AI platform on — the two tracks are parallel, choose one:

- **[Track A — Open-Source Stack](/track-a-opensource/)** — LiteLLM + Langfuse + LangGraph, self-hosted in your cluster. The full end-to-end journey (platform → apps → scaling → production).
- **[Track B — SaaS Stack](/track-b-saas/)** — Kong Konnect + Arize AX + Strands, managed SaaS control planes. Builds the same platform + agent.

---

**[Back to Module Overview](../) | [Start Track A →](/track-a-opensource/) | [Start Track B →](/track-b-saas/)**
