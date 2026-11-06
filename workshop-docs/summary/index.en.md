---
title: "Workshop Summary and Next Steps"
weight: 50
---

# Workshop Summary and Next Steps

You have completed the **Architect and deploy advanced Agentic AI platforms on Amazon EKS** workshop. Over the course of this workshop, you built a complete Agentic AI platform on Amazon EKS and used it to create a business application. This hands-on experience provides practical skills for deploying and scaling Agentic AI solutions in production environments.

## What You Have Accomplished

Over the course of this workshop, you have:

### Interacting with Models _(shared foundation)_
**Deployed and interacted with real LLMs on EKS**
- Set up Open WebUI as a ChatGPT-style interface
- Explored vLLM serving Qwen 3 8B and DeepSeek R1 Qwen 3 8B on AWS Neuron chips
- Integrated Claude Sonnet 4.5 via AWS Bedrock
- Compared self-hosted vs. managed model deployment patterns
- Understood the Kubernetes resources required for LLM hosting

### Track A · Module A1: GenAI Platform Components
**Built a unified Agentic AI platform with comprehensive observability**
- Configured LiteLLM as a unified API gateway routing requests across models
- Implemented Langfuse for complete observability and tracing
- Added new models to the platform through Helm configuration
- Explored cost tracking, performance monitoring, and usage analytics
- Understood how the platform components integrate

### Track A · Module A2: Building Agentic AI Applications
**Created a loan processing application**
- Deployed "Loan Buddy" - an AI agent that automates loan reviews
- Implemented LangGraph workflows for complex decision-making
- Integrated Model Context Protocol (MCP) servers for external tools:
  - Address validation
  - Employment verification
  - Image processing and data extraction
- Observed complete workflow execution in Langfuse
- Reduced a manual review process from 2-3 hours to approximately 2 minutes

### Track A · Module A3: Scaling and Evaluating the Platform
**Verified quality and scaled the platform for production traffic**
- Configured LLM-as-a-Judge evaluation in Langfuse to automatically score agent outputs
- Set up A/B model routing in LiteLLM (Claude vs Qwen weighted comparison)
- Deployed Amazon Managed Prometheus (AMP) and Amazon Managed Grafana (AMG) for infrastructure monitoring
- Built Agentic AI-specific dashboards (token throughput, inference latency, Neuron utilization)
- Configured HPA auto-scaling and observed Karpenter node provisioning under load

### Track A · Module A4: Production-Ready Applications
**Prepared Loan Buddy for enterprise requirements**
- Deployed Milvus vector database and built a RAG pipeline with Bedrock Titan Embeddings
- Enhanced Loan Buddy to make policy-aware decisions grounded in actual documents
- Evaluated RAG quality using Ragas metrics (faithfulness, relevancy, precision, recall)
- Replaced hardcoded credentials with External Secrets Operator + AWS Secrets Manager
- Configured EKS Pod Identity for least-privilege IAM access
- Added Bedrock Guardrails for content safety and PII anonymization
- Implemented SSE streaming for real-time agent responses
- Validated outputs with Pydantic schemas for compliance
- Completed a full production readiness checklist

::alert[The accomplishments above are the **Track A (Open-Source Stack)** path. If you took **Track B (SaaS Stack)**, see below.]{type="info"}

### Track B: SaaS Stack (Kong Konnect + Arize AX + Strands)
**Built the same platform and agent on managed SaaS control planes** _(if you took Track B)_
- Registered a **Kong Konnect** control plane and ran a stateless **DataPlane** in-cluster (Kong Operator 2.x)
- Configured the Amazon Bedrock `ai-proxy` route with **native Kubernetes CRDs** (`KongService`/`KongRoute`/`KongPlugin`/`KongConsumer`) — no decK, GitOps-friendly
- Set up **Arize AX** (SaaS) and exported **OpenInference** traces (AGENT/CHAIN/LLM/TOOL spans)
- Built the Loan Buddy agent with the **Strands Agents SDK**, calling Bedrock **through Kong** (the application holds S3 IAM only — no Bedrock key)
- Reused the **same 3 MCP tool servers** as Track A, demonstrating that MCP is framework-agnostic

## Key Technologies Covered

### Infrastructure & Platforms
- **Amazon EKS Auto Mode**: Fully managed Kubernetes with automatic scaling
- **AWS Neuron**: Purpose-built chips for cost-effective AI inference
- **Amazon EFS**: Shared storage for model caching and persistence
- **Helm Charts**: Declarative deployment and configuration management

### Agentic AI Stack
- **vLLM**: High-performance inference server with continuous batching
- **LiteLLM**: Unified gateway supporting multiple LLM providers
- **Langfuse**: Comprehensive observability for LLM applications
- **Open WebUI**: Chat interface for model interaction
- **Milvus**: Cloud-native vector database for RAG pipelines
- **Amazon Managed Prometheus/Grafana**: Infrastructure-level monitoring

### Application Frameworks
- **LangChain**: Building blocks for LLM applications
- **LangGraph**: Workflow orchestration for AI agents
- **Model Context Protocol (MCP)**: Standard for tool integration
- **Ragas**: RAG evaluation framework (faithfulness, relevancy, precision)
- **Pydantic**: Data validation and structured output schemas

### Security & Production
- **EKS Pod Identity**: Fine-grained IAM for Kubernetes pods
- **External Secrets Operator**: Managed credential synchronization
- **Bedrock Guardrails**: Content safety and PII protection

## Business Outcomes

The "Loan Buddy" application demonstrated measurable business value:

- **Speed**: Reduced processing time from 2-3 hours to 2 minutes (99% improvement)
- **Consistency**: Standardized decision-making across all applications
- **Cost Savings**: Automated manual review processes
- **Compliance**: Complete audit trails for every decision
- **Experience**: Improved service for both employees and customers

The same pattern applies to other business processes, including:
- Document processing and extraction
- Customer service automation
- Quality assurance workflows
- Compliance monitoring

## Next Steps

### Immediate Actions

#### 1. Deploy in Your Own Environment
Use the **[AWS GenAI on EKS Starter Kit](https://github.com/aws-samples/sample-genai-on-eks-starter-kit)** to:
- Deploy the same platform in your AWS account
- Customize configurations for your use cases
- Scale based on your requirements
#### 2. Explore Advanced Serving with llm-d
The **[AWS GenAI on EKS Starter Kit](https://github.com/aws-samples/sample-genai-on-eks-starter-kit)** includes support for **llm-d**, an open-source framework for disaggregated LLM serving on Kubernetes. Disaggregated serving separates the prefill (prompt processing) and decode (token generation) phases into independently scalable components, enabling significant improvements in throughput and latency at scale. This represents the next step in optimizing LLM inference beyond the patterns covered in this workshop.

### Learning Resources

#### Official Documentation
- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Amazon Bedrock Documentation](https://docs.aws.amazon.com/bedrock/)
- [LangChain Documentation](https://python.langchain.com/)
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/)
- [vLLM Documentation](https://docs.vllm.ai/)
- [LiteLLM Documentation](https://docs.litellm.ai/)
- [Langfuse Documentation](https://langfuse.com/docs)
- [Milvus Documentation](https://milvus.io/docs)
- [Ragas Documentation](https://docs.ragas.io/)

## Cleanup — Track B (SaaS Stack)

If you took **Track B**, tear down its resources here — **only after finishing B3–B5**, since every one of those modules depends on the Kong `/loan-strands` route and consumer.

::alert[Run this **at the end of Track B**, not earlier. Deleting these CRDs mid-track breaks the Strands agent (B3), token/cost attribution and model routing (B4), and RAG/caching/streaming (B5).]{type="warning"}

**1. Uninstall the Strands Loan Buddy agent (B3):**

:::code{language=bash showCopyAction=true}
cd /workshop && ./cli strands-agents loan-buddy-agent uninstall
:::

**2. Remove the Kong `/loan-strands` route CRDs (B2)** — leaves the Kong Operator + control plane intact:

:::code{language=bash showCopyAction=true}
kubectl delete kongroute loan-strands-route -n kong
kubectl delete kongservice loan-strands-bedrock -n kong
kubectl delete kongplugin ai-proxy-loan-strands key-auth-loan-strands -n kong
kubectl delete kongcredentialapikey loan-strands-apikey -n kong
kubectl delete kongconsumer loan-strands-agent -n kong
:::

::alert[If you also added the second model route from B2 (Amazon Nova), delete it too: `kubectl delete kongroute loan-strands-nova-route -n kong && kubectl delete kongservice loan-strands-nova -n kong && kubectl delete kongplugin ai-proxy-nova -n kong`. The Kong DataPlane, Konnect control plane, and Arize AX Space can be removed from the Kong Konnect and Arize consoles when you are done.]{type="info"}

## Conclusion

You have completed the Agentic AI on Amazon EKS workshop. Over the course of this workshop, you have:

- Built a complete Agentic AI platform from scratch
- Deployed real LLMs on specialized hardware
- Created a unified API gateway with observability
- Developed a business application driven by an AI agent
- Evaluated quality and scaled for production traffic
- Built a RAG pipeline with Milvus and Ragas evaluation
- Hardened security with Pod Identity, External Secrets, and Guardrails
- Gained hands-on experience with current AI technologies

The skills and patterns you have learned are directly applicable to real-world scenarios, whether you are building customer service bots, document processors, or complex AI agents.

---

*Thank you for participating in this workshop. For questions, feedback, or support, engage with the AWS community or refer to the [AWS GenAI on EKS Starter Kit](https://github.com/aws-samples/sample-genai-on-eks-starter-kit) repository.*
