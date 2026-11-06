---
title: "Deploy and Test Your Application"
weight: 30
---

# Deploy Loan Buddy

In this section, you deploy Loan Buddy and observe it process John Michael Doe's loan application in real time.

![John's Application Ready for Processing](/static/images/shared/example1.png) 

*John's application is ready for AI-powered processing.*

## Prerequisites Check

Before deploying, confirm that your GenAI platform from Interacting with Models & Module A1 is ready:

:::code{language=bash showCopyAction=true}
# Check your platform components are running
kubectl get pods -n litellm
kubectl get pods -n langfuse

# Verify LiteLLM gateway is accessible
echo "LiteLLM URL: http://$(kubectl get ingress -n litellm litellm -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

# Check Langfuse observability
echo "Langfuse URL: http://$(kubectl get ingress -n langfuse langfuse -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
:::

::alert[**Platform Required**: All pods should show "Running" status. If any components are missing, please complete Interacting with Models & Module A1 first.]{type="warning"}

## Step 1: Understanding the Configuration

Before deploying, review what Loan Buddy needs to connect to your GenAI platform:

### Required Configuration

1. **S3 Bucket** - Where to store uploaded loan documents
   - Automatically created by Terraform when Langfuse was deployed
   - Retrieved from: `terraform output langfuse_bucket_name`

2. **AWS Region** - Where your resources are located
   - Configured in your `.env` file as `REGION`
   - Used for S3 and other AWS service calls

3. **LiteLLM API Key** - Your Virtual Key from Module A1
   - Allows the agent to call LLM models through the gateway
   - You created this in Module A1, Step 9

4. **Langfuse Credentials** - For observability and tracing
   - Public and secret keys from your `.env` file
   - Enables complete audit trails of AI decisions

5. **Service Account with S3 Access** - Kubernetes RBAC + AWS IAM
   - Copies the Langfuse service account to workshop namespace
   - Creates Pod Identity Association for S3 permissions
   - Allows pods to securely access S3 without hardcoded credentials

### Verify Your Credentials

First, confirm you have your Virtual Key from Module A1:

:::code{language=bash showCopyAction=true}
# Check if you have your AGENT_KEY from Module A1
if [ -z "$AGENT_KEY" ]; then
  echo "AGENT_KEY not found. Please set it from Module A1"
  echo "Run: export AGENT_KEY='your-sk-key-from-module-2'"
else
  echo "AGENT_KEY found: ${AGENT_KEY:0:10}..."
fi
:::

::alert[**Missing AGENT_KEY?** Go back to Module A1, Step 3 of the Virtual Key section and copy your key. Then run: `export AGENT_KEY="sk-your-actual-key"`]{type="warning"}

## Step 2: Automated Deployment

### Run the Deployment Script

An automated deployment script handles all the configuration:

:::code{language=bash showCopyAction=true}
# Navigate to the credit validation directory
cd /workshop/workshops/eks-genai-workshop/static/code/module3/credit-validation

# Make the script executable
chmod +x deploy-workshop-app.sh

# Run the deployment script
./deploy-workshop-app.sh
:::

### What the Script Does

The script automates the following steps:

1. **Validates Prerequisites** - Checks kubectl and cluster connectivity
2. **Retrieves S3 Bucket** - Gets bucket name from Terraform output
3. **Gets AWS Region** - Loads from .env file
4. **Configures Credentials** - Sets up LiteLLM and Langfuse keys
5. **Creates Workshop Namespace** - Isolated environment for the application
6. **Sets Up Service Account** - Copies Langfuse SA with S3 permissions
7. **Creates Pod Identity Association** - Grants S3 access to workshop pods
8. **Configures Deployment** - Replaces all placeholders in YAML
9. **Deploys Application** - Applies the configured Kubernetes manifests
10. **Waits for Ready** - Ensures all pods are running

::alert[**Learning Opportunity**: The script creates a file called `agentic-application-deployment.configured.yaml` with all values filled in. Review it to see how the configuration works.]{type="info"}

::::expand{header="Under the Hood: What Is Being Automated"}

**Without the script, you would need to:**

1. **Get S3 Bucket Name:**
   ```bash
   cd /workshop/components/o11y/langfuse
   terraform output langfuse_bucket_name
   ```

2. **Get AWS Region:**
   ```bash
   # The .env file is auto-generated during infrastructure setup
   # and contains REGION, CLUSTER_NAME, and other workshop variables
   source /workshop/.env
   echo $REGION
   ```

3. **Manually Edit YAML File:**
   - Replace `<>` with S3 bucket name (2 places)
   - Replace `<>` with AWS region (2 places)
   - Replace Langfuse keys (2 places)
   - Replace LiteLLM keys (2 places)

4. **Create Service Account:**
   ```bash
   kubectl get serviceaccount langfuse -n langfuse -o yaml | \
     sed 's/namespace: langfuse/namespace: workshop/' | \
     kubectl apply -f -
   ```

5. **Create Pod Identity Association:**
   ```bash
   aws eks create-pod-identity-association \
     --cluster-name genai-on-eks \
     --namespace workshop \
     --service-account langfuse \
     --role-arn <IAM_ROLE_ARN>
   ```

6. **Deploy Application:**
   ```bash
   kubectl apply -f agentic-application-deployment.yaml
   ```

**The script does all of this in one command.** This is the same automation pattern used in production CI/CD pipelines.

::::

### Expected Output

You should see output similar to:

```
  Loan Buddy Workshop Deployment

Step 1: Checking prerequisites...
kubectl found
Connected to Kubernetes cluster

Step 2: Retrieving S3 bucket name from Terraform...
S3 Bucket: genai-on-eks-bucket-langfuse-xxxxx

Step 3: Getting AWS region...
AWS Region: us-west-2

Step 4: Getting LiteLLM API key...
LiteLLM API Key: sk-xxxxx...

Step 5: Getting Langfuse credentials...
Langfuse Public Key: lf_pk_xxxxx...
Langfuse Secret Key: lf_sk_xxxxx...

Step 6: Creating workshop namespace...
Created namespace 'workshop'

Step 7: Setting up service account for S3 access...
Service account 'langfuse' created in workshop namespace

Step 7b: Creating Pod Identity Association for S3 access...
Pod Identity Association created for workshop namespace

Step 8: Configuring deployment file...
Deployment file configured

Step 9: Deploying Loan Buddy application...
Application deployed

Step 10: Waiting for deployments to be ready...
All deployments ready

Step 11: Verifying deployment...
NAME                                     READY   STATUS    RESTARTS   AGE
loan-buddy-agent-xxx                     1/1     Running   0          1m
mcp-address-validator-xxx                1/1     Running   0          1m
mcp-employment-validator-xxx             1/1     Running   0          1m
mcp-image-processor-xxx                  1/1     Running   0          1m

  Deployment Complete!
```

## Step 3: Watch the AI in Action

### Set Up Real-Time Monitoring

Open a second terminal to watch the agent's decision process:

:::code{language=bash showCopyAction=true}
# In a NEW terminal, watch the agent logs
kubectl logs -f deployment/loan-buddy-agent -n workshop
:::

**Keep this terminal visible** - you will see the AI making decisions in real time.

### Expose the Application

Back in your main terminal:

:::code{language=bash showCopyAction=true}
# Make Loan Buddy accessible
kubectl port-forward service/loan-buddy-agent 8888:8080 -n workshop &

# Navigate to the test data directory
cd /workshop/workshops/eks-genai-workshop/static/code/module3/credit-validation/
:::

::alert[**API-Only Service**: Loan Buddy is a REST API server built with FastAPI. It does not have a web UI -- all interactions happen through API calls (e.g., `curl`). If you open `http://localhost:8888` in a browser, you will see `{"detail":"Not Found"}`, which is normal. You can explore the API documentation at `http://localhost:8888/docs` (Swagger UI).]{type="info"}

::alert[**Why port 8888?** The workshop Code Editor server already uses port 8080. To avoid conflicts, Loan Buddy is forwarded to local port 8888 while the container still listens on 8080 internally.]{type="info"}

## Step 4: Process John's Loan Application

### Submit John's Application

::alert[This command uses `jq` for JSON formatting, which is pre-installed in the workshop environment.]{type="info"}

:::code{language=bash showCopyAction=true}
# Process John Michael Doe's loan application
curl -X POST -F "image_file=@./example1.png" \
  http://localhost:8888/api/process_credit_application_with_upload \
  | (data=$(cat); echo "$data" | jq -C 'del(.credit_assessment) + {"credit_assessment": "(formatted below)"}'; echo -e "\n\033[1;36m═══ CREDIT ASSESSMENT ═══\033[0m\n"; echo "$data" | jq -r '.credit_assessment' | sed 's/\\n/\n/g')
:::

### Observe the Processing

**In your log terminal, you will see:**

```
Starting credit application processing...
Processing uploaded loan application image...
Image stored in S3 with ID: a7b9c2d4e5f6...
Loading MCP tools...
Processing with agent...

[Agent]: I'll help you process this credit application. Let me start by extracting the information from the document.

[Image Processor MCP]: Extracting data from loan application...
Found: John Michael Doe, DOB: 03/15/1985, SSN: ***-**-7890
Income: $75,000, Employer: Tech Solutions Inc
Address: 123 Main Street, Anytown, CA 90210

[Agent]: Now validating the address information...

[Address Validator MCP]: Checking 123 Main Street, Anytown, CA 90210
Valid residential address - Single family home
Owner occupied - 4 years at address
Risk score: 15 (Low risk)

[Agent]: Verifying employment and income...

[Employment Validator MCP]: Verifying Tech Solutions Inc
Employment confirmed - Software Engineer
Income verified - $75,000 annual
Employment duration - 3.5 years
Stability score: 85 (High stability)

[Agent]: Calculating loan metrics...
- Loan amount: $8,500
- Monthly payment: $275
- Monthly income: $6,250
- Debt-to-income ratio: 4.4% (Excellent)

FINAL DECISION: APPROVED
- Low risk profile
- Stable employment
- Excellent debt-to-income ratio
- Home ownership indicates stability
```

### Review the Response

The curl command returns a JSON response with the complete assessment:

```json
{
  "status": "COMPLETED",
  "image_id": "a7b9c2d4e5f6...",
  "credit_assessment": "APPROVED - John Michael Doe's application for $8,500 home improvement loan is approved. Verified employment at Tech Solutions Inc ($75,000/year), confirmed residential address ownership, excellent debt-to-income ratio of 4.4%.",
  "processing_note": "Complete audit trail available in Langfuse"
}
```

## Step 5: Explore the Complete Workflow in Langfuse

### Open Langfuse Dashboard

1. **Get your Langfuse URL** (from the prerequisites check)
2. **Open it in your browser**
3. **Navigate to "Traces"** in the left sidebar

### Find John's Application Trace

Look for traces with:
- **Name**: `credit_underwriting_agent_with_image_id`
- **Recent timestamp** (just processed)

### What You Will See in the Trace

![Langfuse Trace Example](/static/images/track-a-opensource/a2-genai-applications/LoanBuddy-Observability.png)
*Complete visibility into the AI's decision-making process*

**The trace shows:**
- **Complete workflow** - Every step from upload to decision
- **All tool calls** - Image extraction, address validation, employment verification
- **AI reasoning** - Why the loan was approved
- **Performance metrics** - Processing time, latency for each step
- **Cost tracking** - Tokens used, estimated cost
- **Full audit trail** - Complete record for compliance

### Analyze the Metrics

In Langfuse, click on the trace to see:
- **Total tokens**: ~2,500 tokens used
- **Processing time**: 45-60 seconds total
- **Tool calls**: 3 MCP servers invoked
- **Decision path**: Complete reasoning chain

## Step 6: Try Different Scenarios (Optional)

### Test with the Second Application

If you have another test application:

:::code{language=bash showCopyAction=true}
# Process another application
curl -X POST -F "image_file=@./example2.png" \
  http://localhost:8888/api/process_credit_application_with_upload
:::

### Experiment with Edge Cases

Try modifying the MCP server mock data to see different outcomes:
- What if employment verification fails?
- What if the address is invalid?
- What if the income does not match?

## Summary

### What Was Accomplished

You have:
- **Deployed** a complete AI loan processing system on Kubernetes
- **Processed** John's loan application in under a minute
- **Automated** 2-3 hours of manual work
- **Created** a complete audit trail in Langfuse
- **Integrated** with your GenAI platform from Interacting with Models & Module A1

### The Business Impact

Recall Sarah from the beginning of this module.

**Before Loan Buddy:**
- 30 minutes manually entering John's data
- 45 minutes calling Tech Solutions Inc
- 20 minutes verifying the address
- 30 minutes calculating ratios and documenting
- **Total**: 2+ hours

**After Loan Buddy:**
- Complete processing: 47 seconds
- All verifications: Automated
- Full documentation: Instant
- Audit trail: Complete

## Key Takeaways

You have learned how to:
1. **Build AI applications** that solve real business problems
2. **Use LangGraph** to orchestrate complex workflows
3. **Implement MCP servers** for specialized capabilities
4. **Integrate with your platform** (LiteLLM + Langfuse)
5. **Deploy on Kubernetes** with proper configuration
6. **Monitor AI decisions** with complete observability

You have built and deployed an AI-powered loan processing system that:
- Saves hours of manual work
- Provides consistent decisions
- Maintains complete audit trails
- Integrates with your GenAI platform

---

**[Next: Workshop Summary and Next Steps →](../../summary/)**
