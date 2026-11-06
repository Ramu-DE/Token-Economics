---
title: "Building a RAG Pipeline (Milvus + Strands)"
weight: 10
---

# 🗄️ Building a RAG Pipeline with Milvus (Strands + Kong)

Your Strands Loan Buddy makes decisions from the applicant's data — but a production underwriting agent must also ground its decisions in the bank's **loan policy documents**. This page builds a production-grade RAG pipeline so the agent retrieves relevant policy before deciding.

## 🎯 What You Will Learn

- 🗄️ Deploy **Milvus** vector database on EKS
- 📄 Ingest loan policy documents (chunk → embed with Bedrock Titan → store)
- 🤖 Add a **retrieval tool** to the Strands agent (auto-traced in Arize)
- 📊 Evaluate RAG quality with **Ragas** (faithfulness, relevancy, precision)

---

## 🛠️ Hands-On: Deploy Your Vector Database

### Step 1: Check Prerequisites

:::code{language=bash showCopyAction=true}
kubectl get pods -n milvus 2>/dev/null || echo "Milvus not deployed yet — we'll create it"

# Verify Bedrock Titan Embeddings access
aws bedrock list-foundation-models --region us-east-1 \
  --query "modelSummaries[?contains(modelId, 'titan-embed')].modelId" --output table
:::

### Step 2: Deploy Milvus on EKS

:::code{language=bash showCopyAction=true}
helm repo add milvus https://zilliztech.github.io/milvus-helm/
helm repo update

# upgrade --install is idempotent: installs on first run, no-ops on re-runs (no "already exists" error)
helm upgrade --install milvus milvus/milvus \
  --namespace milvus \
  --create-namespace \
  --set cluster.enabled=false \
  --set standalone.enabled=true \
  --set standalone.resources.requests.cpu=500m \
  --set standalone.resources.requests.memory=1Gi \
  --set standalone.resources.limits.memory=2Gi \
  --set etcd.replicaCount=1 \
  --set minio.mode=standalone \
  --set pulsarv3.enabled=false \
  --set standalone.messageQueue=woodpecker \
  --set woodpecker.enabled=true \
  --wait --timeout 420s
:::

### Step 3: Verify the Deployment

:::code{language=bash showCopyAction=true}
kubectl get pods -n milvus
:::

**Expected:**

:::code{language=text showCopyAction=false}
milvus-standalone-xxx    1/1   Running
milvus-etcd-0            1/1   Running
milvus-minio-xxx         1/1   Running
:::

- **milvus-standalone**: Vector database engine
- **milvus-etcd**: Metadata storage
- **milvus-minio**: Object storage for vector persistence

---

## 📄 Ingest Loan Policy Documents

### Step 4: Prepare Policy Documents

:::code{language=bash showCopyAction=true}
cd /workshop/workshops/eks-genai-workshop/static/code/module5/rag-pipeline
ls -la policies/
:::

**Policy documents:**
- `home-improvement-policy.md` — Rules for home improvement loans ($5K-$50K)
- `personal-loan-policy.md` — Rules for personal loans ($1K-$25K)
- `income-requirements.md` — Minimum income and DTI ratio requirements
- `risk-assessment-guidelines.md` — Risk scoring criteria and thresholds

### Step 5: Run the Document Ingestion Pipeline

:::code{language=bash showCopyAction=true}
# Install ingestion dependencies
python3 -m venv .ingest-venv
source .ingest-venv/bin/activate
pip install --quiet langchain-milvus langchain-aws langchain-text-splitters "pymilvus>=2.5,<2.6"

# Port-forward Milvus
# Idempotent — start the Milvus port-forward only if 19530 isn't already forwarding
pgrep -f "port-forward.*19530:19530" >/dev/null || kubectl port-forward svc/milvus -n milvus 19530:19530 >/tmp/milvus-pf.log 2>&1 &
sleep 3

# Run ingestion
python3 ingest_policies.py
:::

**What the ingestion does:**

:::code{language=text showCopyAction=false}
1. Load 4 policy markdown files
2. Split into ~500-char chunks (22 chunks total)
3. Embed each chunk using Bedrock Titan Embed Text v2 (1024-dim vectors)
4. Store vectors + metadata in Milvus collection "loan_policies"
:::

::::expand{header="Check Output"}

```text
Loading 4 policy documents...
Splitting into chunks: 22 chunks created
Generating embeddings via Bedrock Titan...
Storing in Milvus collection 'loan_policies'...
Ingested 22 policy chunks into Milvus
```

::::

### Step 6: Verify the Vector Store

:::code{language=bash showCopyAction=true}
python3 -c "
from langchain_milvus import Milvus
from langchain_aws import BedrockEmbeddings
import os

embeddings = BedrockEmbeddings(
    model_id='amazon.titan-embed-text-v2:0',
    region_name='us-east-1'
)

vectorstore = Milvus(
    embedding_function=embeddings,
    connection_args={'uri': 'http://localhost:19530'},
    collection_name='loan_policies'
)

results = vectorstore.similarity_search('home improvement loan requirements', k=3)
for i, doc in enumerate(results):
    print(f'--- Result {i+1} ---')
    print(doc.page_content[:200])
    print()
"
:::

You should see the top-3 most relevant policy sections for home improvement loans.

---

## 🤖 Enhance Loan Buddy with RAG (Strands `@tool`)

### Step 7: Add Retrieval Tool to the Strands Agent

In Track B, retrieval is implemented as a **Strands `@tool`** that the agent calls during its loop. Because it's a Strands tool, the retrieval shows up as a **TOOL span in Arize** automatically:

:::code{language=python showCopyAction=false}
from strands import tool
from pymilvus import MilvusClient
from langchain_aws import BedrockEmbeddings

embeddings = BedrockEmbeddings(model_id="amazon.titan-embed-text-v2:0", region_name="us-east-1")
milvus_client = MilvusClient(uri="http://milvus.milvus.svc.cluster.local:19530")

@tool
def retrieve_loan_policy(query: str) -> str:
    """Retrieve relevant loan-policy passages for the given query.

    Args:
        query: the applicant scenario or policy question to look up
    """
    query_vector = embeddings.embed_query(query)
    results = milvus_client.search(
        collection_name="loan_policies",
        data=[query_vector],
        limit=4,
        output_fields=["text", "source"]
    )
    passages = []
    for hits in results:
        for hit in hits:
            passages.append(hit["entity"]["text"])
    return "\n\n---\n\n".join(passages)

# Add to the agent alongside MCP tools:
agent = Agent(
    model=request_model,
    system_prompt=SYSTEM_PROMPT,
    tools=[*mcp_tools, retrieve_loan_policy],
    trace_attributes={...}
)
:::

The agent now calls `retrieve_loan_policy` during its reasoning loop, grounding decisions in actual policy. The retrieval appears as a **TOOL span** in Arize traces.

::alert[**Embeddings go through Bedrock directly** (not Kong) for the retrieval tool. The Kong `/loan-strands` route handles the LLM chat completion; embeddings are a separate Bedrock API call from the agent pod (which has Bedrock access via its own Pod Identity or through a dedicated Kong embeddings route).]{type="info"}

### Step 8: Test RAG-Enhanced Processing

:::code{language=bash showCopyAction=true}
# Idempotent — start the port-forward only if 8890 isn't already forwarding (safe to re-run in any module)
pgrep -f "port-forward.*8890:8080" >/dev/null || kubectl port-forward -n strands-agents svc/loan-buddy-agent 8890:8080 >/tmp/loanbuddy-pf.log 2>&1 &
sleep 3

export LOAN_IMAGE="/workshop/workshops/eks-genai-workshop/static/code/module3/credit-validation/example1.png"

curl -s --max-time 180 -X POST \
  http://localhost:8890/api/process_credit_application_with_upload \
  -H "X-Consumer-ID: loan-strands-agent" \
  -H "X-Consumer-ApiKey: loan-strands-key-123" \
  -F "image_file=@${LOAN_IMAGE}" \
  -o /tmp/rag_result.json

python3 -c "import json;d=json.load(open('/tmp/rag_result.json'));print('Status:',d['status']);print(d.get('credit_assessment','')[:300])"
:::

The response should now include **policy references** in the assessment (e.g., "Per home improvement policy section 3.2...").

### Step 9: View RAG Traces in Arize

In Arize AX → `loan-strands` → Traces, the RAG-enhanced trace shows:
- **AGENT** span: `invoke_agent Strands Agents` (root)
  - **TOOL** span: `retrieve_loan_policy` (input: query, output: policy passages)
  - **LLM** span: `chat` (sees the policy context in the prompt)

The retrieval step is fully traced — you can inspect which policy chunks were retrieved for each decision.

---

## 📊 Evaluate RAG Quality with Ragas

### Step 10: Run Ragas Evaluation

[Ragas](https://docs.ragas.io/) evaluates RAG quality with these metrics:
- **Faithfulness**: Is the answer grounded in the retrieved context?
- **Answer Relevancy**: Does the answer address the question?
- **Context Precision**: Are the retrieved chunks relevant to the question?

:::code{language=bash showCopyAction=true}
pip install --quiet ragas langchain-aws

python3 -c "
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision
from langchain_aws import ChatBedrock, BedrockEmbeddings
from datasets import Dataset

# Prepare evaluation dataset from your test traces
eval_data = {
    'question': ['Should we approve a home improvement loan for $8,500?'],
    'answer': ['APPROVED - Per home improvement policy, applicant meets all criteria...'],
    'contexts': [['Home improvement loans between $5K-$50K require credit score >= 680...']],
    'ground_truth': ['Approve if credit >= 680, DTI <= 43%, stable employment >= 2 years']
}

dataset = Dataset.from_dict(eval_data)
results = evaluate(
    dataset,
    metrics=[faithfulness, answer_relevancy, context_precision],
    llm=ChatBedrock(model_id='anthropic.claude-3-5-sonnet-20241022-v2:0', region_name='us-east-1'),
    embeddings=BedrockEmbeddings(model_id='amazon.titan-embed-text-v2:0', region_name='us-east-1')
)
print(results)
"
:::

**Expected scores (good RAG pipeline):**

:::code{language=text showCopyAction=false}
{'faithfulness': 0.92, 'answer_relevancy': 0.88, 'context_precision': 0.95}
:::

| Metric | Good | Meaning |
|---|---|---|
| Faithfulness > 0.8 | ✅ | Answer is grounded in retrieved context |
| Answer Relevancy > 0.8 | ✅ | Answer addresses the question |
| Context Precision > 0.8 | ✅ | Retrieved chunks are relevant |

---

## 📊 How It Maps to Track A

| | Track A | Track B |
|---|---|---|
| Vector DB | Milvus | **same** |
| Embeddings | Bedrock Titan via LiteLLM | Bedrock Titan (direct or via Kong `/loan-embeddings` route) |
| Retrieval wiring | LangGraph `@tool` | **Strands `@tool`** (auto-traced as TOOL span in Arize) |
| RAG traces | Langfuse | **Arize AX** (OpenInference TOOL spans) |
| RAG eval | Ragas | **same** |

## ✅ What You Accomplished

- ✅ Deployed Milvus vector database on EKS (standalone mode)
- ✅ Ingested loan policy documents (chunk → embed → store)
- ✅ Added retrieval tool to Strands agent (auto-traced in Arize)
- ✅ Grounded loan decisions in policy documents
- ✅ Evaluated RAG quality with Ragas (faithfulness, relevancy, precision)

---

**[Next: Security Hardening for Production →](../security-hardening/)**
