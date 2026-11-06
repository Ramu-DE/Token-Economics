---
title: "Getting Started"
chapter: true
weight: 12
---

Before deploying Large Language Models and building agents, ensure you have access to the workshop environment. This workshop can be completed in two ways, depending on how you are participating.

## Choose Your Path

::::tabs

:::tab{label="AWS Event"}
### Participating in an AWS Event

**Your infrastructure is pre-deployed and ready.** 

The event organizers have already set up:
- EKS cluster with all required components
- GenAI platform stack (vLLM, LiteLLM, Langfuse, Open WebUI)
- Model storage and caching
- Networking and security configurations

[**Continue with AWS Event Setup →**](/introduction/getting-started/at-aws-event/)
:::

:::tab{label="Own Account"}
### Using Your Own AWS Account

Running this workshop independently in your personal or company AWS account provides:
- Full control over the environment
- Ability to keep resources after the workshop
- Opportunity to customize configurations
- Deeper understanding of the infrastructure

**You will need to**:
- Have an AWS account with appropriate permissions
- Deploy the workshop infrastructure
- Configure access to the EKS cluster
- Verify all components are running

::alert[**Cost Warning**: This workshop uses GPU and Neuron instances which incur charges. Remember to clean up resources after completion.]{type="warning"}

::alert[**Own Account Setup**: The self-guided setup guide for running this workshop in your own AWS account is coming soon. For now, this workshop is designed to be run at AWS Events with pre-provisioned infrastructure.]{type="info"}
:::

::::

---

**Select your path above to continue.**
