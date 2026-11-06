---
title: "At AWS Event"
weight: 14
---

Your GenAI infrastructure has been pre-deployed by the event organizers. This section walks you through connecting to your workshop environment and verifying that everything is ready.

## What's Already Done

::alert[The infrastructure deployment is already complete. You can focus on the workshop content.]{type="success"}

Your workshop environment includes:
- **EKS cluster** with Auto Mode enabled
- **GenAI platform** components (vLLM, LiteLLM, Langfuse, Open WebUI)
- **Pre-loaded models** (Qwen 3 8B, DeepSeek R1 Qwen 3 8B)
- **Specialized hardware** (Neuron nodes for inference)
- **Observability stack** configured and running

## Step 1: Access Workshop Studio

Workshop Studio provides your isolated AWS environment for this event.

### 1.1 Connect to Workshop Studio

Navigate to the event URL provided by your instructor. If an access code was provided, enter your access code at this link:

```
https://catalog.workshops.aws/join
```

You will see the Workshop Studio landing page:

![Workshop Studio Event Page](/static/images/introduction/workshopstudio-event1.jpg)

### 1.2 Sign In with Email OTP

Choose **Email One-Time Password (OTP)** to proceed with passwordless authentication:

1. Enter your email address
2. Choose **Send passcode**
3. Check your email for subject: **"Your one-time passcode"**
4. Enter the code
5. Choose **Sign in**

::alert[**Tip**: The passcode expires in 10 minutes. If it expires, request a new one.]{type="info"}

### 1.3 Join the Event

After signing in, you will see the event access page:

1. The **Event access code** should be pre-filled. Choose **Next**.
2. Select **"I agree with the Terms and Conditions"**
   ![Workshop Studio Event Page](/static/images/introduction/workshopstudio-event3.jpg)
3. Choose **Join event**

You are now in your workshop environment.

## Step 2: Access Your AWS Account

### 2.1 Open AWS Console

In the Workshop Studio left sidebar, locate and choose:

**Open AWS Console**
![Open AWS Console link in Workshop Studio sidebar](/static/images/introduction/open-aws-console.png)

This opens the AWS Management Console in a new tab with temporary credentials already configured.

::alert[**Important**: Use this specific console link. Do not use your personal AWS account for this workshop.]{type="warning"}

## Step 3: Access Your Development Environment

### 3.1 Open Visual Studio Code (VSC) IDE

Access your cloud-based development environment through Workshop Studio:

1. **Navigate to the Event Page**:
   - In your Workshop Studio interface, look at the **top left corner**
   - Choose the **event title box** that shows "Architect and deploy advanced Agentic AI platforms on Amazon EKS event" (the title may differ)
   - This takes you to the main Event page with all the workshop details

2. **Find the Event Outputs Section**:
   - Scroll down on the Event page to locate the **"Event Outputs (1)"** section
   - You should see a table with output values from your workshop environment

![Workshop Studio Event Outputs](/static/images/introduction/url.png)

3. **Locate the VSC IDE URL**:
   - In the outputs table, find the row with **Key** = "URL"
   - This URL (highlighted in orange) points to your cloud-based Visual Studio Code environment
   - The URL will look similar to: `https://d3fbc6117q00r.cloudfront.net/...`

4. **Open Your Development Environment**:
   - Choose the **URL link** in the Value column
   - This opens a new browser tab with a full Visual Studio Code IDE
   - Wait a moment for the environment to load completely

5. **What You Get**:
   Your cloud-based VSC IDE includes:
   - **Full Visual Studio Code interface** with syntax highlighting and extensions
   - **Integrated terminal** with bash access
   - **kubectl pre-configured** and connected to your EKS cluster
   - **AWS CLI ready to use** with temporary credentials
   - **Code editor** for all workshop exercises and modifications

::alert[**Important**: Do not close this window. You will need to return to your development environment multiple times during the workshop.]{type="info"}

### 3.2 Verify kubectl Access

In the VSC terminal, verify your connection to the EKS cluster:

:::code{language=bash showCopyAction=true}
# Check cluster access
kubectl cluster-info

# Expected output:
# Kubernetes control plane is running at https://...
:::

## Setup Complete

You now have access to:

- **AWS Console** with temporary credentials
- **VSC IDE** with terminal and kubectl configured
- **EKS cluster** connection verified

## What's Next?

Now that you have access to your development environment, the next step is to explore the GenAI infrastructure that has been deployed for you and verify that everything is working correctly.

---

**[Continue to Infrastructure Overview →](/introduction/infra-setup/)**
