---
title: "Virtual Key Shortcut (Advanced Pathway)"
weight: 20
---

This page is the **Advanced pathway shortcut**. It extracts only the LiteLLM Virtual Key step required by Module A2, so experienced attendees can skip the shared foundation (Interacting with Models) and Module A1 and proceed directly to agentic applications.

::alert[**When to use this page**: you already understand vLLM, LiteLLM, and Langfuse on Kubernetes, and you want to reach Module A2 / A3 / A4 within a ~2-hour event slot. If this is your first time with the stack, follow the [Beginner pathway (Interacting with Models → A1 → A2)](../) instead.]{type="info"}

## Prerequisites

Before starting this shortcut, verify the workshop environment is ready. The CloudFormation stack provisions vLLM, LiteLLM, Langfuse, and Open WebUI automatically.

:::code{language=bash showCopyAction=true}
kubectl get pods -A | grep -E "vllm|litellm|langfuse|openwebui"
:::

All pods should be in the `Running` state. If any are still pending, wait for provisioning to complete — the `VerifyPodReadiness` step in CloudFormation typically takes 10-15 minutes after stack creation.

## Step 1: Access the LiteLLM UI

Run the command below — it prints the Admin UI URL along with the login username and password, ready to copy:

:::code{language=bash showCopyAction=true}
source /workshop/.env
echo "LiteLLM Admin UI: http://$(kubectl get ingress -n litellm litellm -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')/ui"
echo "Username: $LITELLM_UI_USERNAME"
echo "Password: $LITELLM_UI_PASSWORD"
:::

Open the printed **Admin UI** URL in your browser and log in with the username and password shown.

## Step 2: Create the Agent Virtual Key

1. In the LiteLLM sidebar, click **"Virtual Keys"**.
2. Click **"+ Create New Key"**.
3. Configure:
   - **Owned By**: You
   - **Key Name**: `agent-key`
   - **Models**: select all three:
     - `bedrock/claude-4.5-sonnet`
     - `vllm/qwen3-8b-neuron`
     - `vllm/deepseek-r1-qwen3-8b-neuron`
   - **Key Type**: Default
4. Click **"Create Key"**.

::alert[**Copy the key now**: LiteLLM displays the `sk-...` secret **only once**. If you close the dialog without copying, you must create a new key.]{type="warning"}

## Step 3: Export AGENT_KEY for Module A2

Save the key as an environment variable so Module A2's Loan Buddy deployment picks it up automatically.

:::code{language=bash showCopyAction=true}
# Replace sk-... with the key you just copied
export AGENT_KEY="sk-your-copied-key"

# Persist it to the workshop env file so later terminals inherit it
grep -v '^AGENT_KEY=' /workshop/.env > /tmp/.env.new && mv /tmp/.env.new /workshop/.env
echo "AGENT_KEY=${AGENT_KEY}" >> /workshop/.env
:::

## Step 4: Verify Access

Verify that the key authenticates and routes correctly through LiteLLM:

:::code{language=bash showCopyAction=true}
LITELLM_URL="http://$(kubectl get ingress -n litellm litellm -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
curl -sS -X POST "${LITELLM_URL}/v1/chat/completions" \
  -H "Authorization: Bearer ${AGENT_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"model":"bedrock/claude-4.5-sonnet","messages":[{"role":"user","content":"Reply with just OK"}]}' \
  | head -c 500
:::

Expected: a JSON response containing `"content":"OK"` (or similar) from Claude Sonnet 4.5.

## Next Step

You now have the artifact Module A2 requires — the `AGENT_KEY` environment variable. Proceed directly to Module A2 and start with the "Running the Application" section.

---

**[Continue to Module A2: Building GenAI Applications →](/track-a-opensource/a2-genai-applications/)**
