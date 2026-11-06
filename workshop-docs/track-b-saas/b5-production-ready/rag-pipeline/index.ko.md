---
title: "RAG 파이프라인 구축 (Milvus + Strands)"
weight: 10
---

# 🗄️ Milvus로 RAG 파이프라인 구축하기 (Strands + Kong)

Strands Loan Buddy는 신청자의 데이터를 기반으로 의사결정을 내립니다 — 하지만 프로덕션 심사 에이전트는 그 결정을 은행의 **대출 정책 문서** 에도 근거를 두어야 합니다. 이 페이지에서는 에이전트가 의사결정 전에 관련 정책을 검색하도록 프로덕션 수준의 RAG 파이프라인을 구축합니다.

## 🎯 학습 내용

- 🗄️ EKS에 **Milvus** 벡터 데이터베이스 배포
- 📄 대출 정책 문서 수집 (청킹 → Bedrock Titan으로 임베딩 → 저장)
- 🤖 Strands 에이전트에 **검색 도구(retrieval tool)** 추가 (Arize에서 자동 트레이싱)
- 📊 **Ragas** 로 RAG 품질 평가 (충실성, 관련성, 정밀도)

---

## 🛠️ 실습: 벡터 데이터베이스 배포

### 1단계: 사전 요구 사항 확인

:::code{language=bash showCopyAction=true}
kubectl get pods -n milvus 2>/dev/null || echo "Milvus not deployed yet — we'll create it"

# Verify Bedrock Titan Embeddings access
aws bedrock list-foundation-models --region us-east-1 \
  --query "modelSummaries[?contains(modelId, 'titan-embed')].modelId" --output table
:::

### 2단계: EKS에 Milvus 배포

:::code{language=bash showCopyAction=true}
helm repo add milvus https://zilliztech.github.io/milvus-helm/
helm repo update

helm install milvus milvus/milvus \
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

### 3단계: 배포 확인

:::code{language=bash showCopyAction=true}
kubectl get pods -n milvus
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
milvus-standalone-xxx    1/1   Running
milvus-etcd-0            1/1   Running
milvus-minio-xxx         1/1   Running
:::

- **milvus-standalone**: 벡터 데이터베이스 엔진
- **milvus-etcd**: 메타데이터 저장소
- **milvus-minio**: 벡터 영속성을 위한 객체 스토리지

---

## 📄 대출 정책 문서 수집

### 4단계: 정책 문서 준비

:::code{language=bash showCopyAction=true}
cd /workshop/workshops/eks-genai-workshop/static/code/module5/rag-pipeline
ls -la policies/
:::

**정책 문서:**
- `home-improvement-policy.md` — 주택 개보수 대출 규정 ($5K-$50K)
- `personal-loan-policy.md` — 개인 대출 규정 ($1K-$25K)
- `income-requirements.md` — 최소 소득 및 DTI 비율 요건
- `risk-assessment-guidelines.md` — 리스크 점수 기준 및 임계값

### 5단계: 문서 수집 파이프라인 실행

:::code{language=bash showCopyAction=true}
# Install ingestion dependencies
python3 -m venv .ingest-venv
source .ingest-venv/bin/activate
pip install --quiet langchain-milvus langchain-aws langchain-text-splitters "pymilvus>=2.5,<2.6"

# Port-forward Milvus
kubectl port-forward svc/milvus -n milvus 19530:19530 &
sleep 3

# Run ingestion
python3 ingest_policies.py
:::

**수집 과정에서 수행하는 작업:**

:::code{language=text showCopyAction=false}
1. Load 4 policy markdown files
2. Split into ~500-char chunks (22 chunks total)
3. Embed each chunk using Bedrock Titan Embed Text v2 (1024-dim vectors)
4. Store vectors + metadata in Milvus collection "loan_policies"
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
Loading 4 policy documents...
Splitting into chunks: 22 chunks created
Generating embeddings via Bedrock Titan...
Storing in Milvus collection 'loan_policies'...
Ingested 22 policy chunks into Milvus
:::

### 6단계: 벡터 스토어 확인

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

주택 개보수 대출과 가장 관련성이 높은 상위 3개 정책 섹션이 표시됩니다.

---

## 🤖 RAG로 Loan Buddy 강화하기 (Strands `@tool`)

### 7단계: Strands 에이전트에 검색 도구 추가

Track B에서 검색은 에이전트가 루프 중에 호출하는 **Strands `@tool`** 로 구현됩니다. Strands 도구이기 때문에 검색은 자동으로 **Arize의 TOOL 스팬** 으로 표시됩니다:

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

이제 에이전트는 추론 루프 중에 `retrieve_loan_policy`를 호출하여 실제 정책에 의사결정의 근거를 둡니다. 검색은 Arize 트레이스에서 **TOOL 스팬** 으로 나타납니다.

::alert[**임베딩은 검색 도구를 위해 Bedrock을 직접** 통과합니다(Kong을 거치지 않음). Kong `/loan-strands` 라우트는 LLM 채팅 완성을 처리하며, 임베딩은 에이전트 Pod(자체 Pod Identity를 통해 또는 전용 Kong 임베딩 라우트를 통해 Bedrock 접근 권한을 가짐)에서의 별도 Bedrock API 호출입니다.]{type="info"}

### 8단계: RAG로 강화된 처리 테스트

:::code{language=bash showCopyAction=true}
kubectl port-forward -n strands-agents svc/loan-buddy-agent 8890:8080 &
sleep 3

export LOAN_IMAGE="/workshop/static/images/shared/example1.png"

curl -s --max-time 180 -X POST \
  http://localhost:8890/api/process_credit_application_with_upload \
  -H "X-Consumer-ID: loan-strands-agent" \
  -H "X-Consumer-ApiKey: loan-strands-key-123" \
  -F "image_file=@${LOAN_IMAGE}" \
  -o /tmp/rag_result.json

python3 -c "import json;d=json.load(open('/tmp/rag_result.json'));print('Status:',d['status']);print(d.get('credit_assessment','')[:300])"
:::

이제 응답의 평가 내용에 **정책 참조** 가 포함되어야 합니다(예: "주택 개보수 정책 3.2 섹션에 따라...").

### 9단계: Arize에서 RAG 트레이스 보기

Arize AX → `loan-strands` → Traces에서 RAG로 강화된 트레이스는 다음을 보여줍니다:
- **AGENT** 스팬: `invoke_agent Strands Agents` (루트)
  - **TOOL** 스팬: `retrieve_loan_policy` (입력: 쿼리, 출력: 정책 구절)
  - **LLM** 스팬: `chat` (프롬프트에서 정책 컨텍스트를 확인)

검색 단계가 완전히 트레이싱됩니다 — 각 의사결정에 대해 어떤 정책 청크가 검색되었는지 검사할 수 있습니다.

---

## 📊 Ragas로 RAG 품질 평가

### 10단계: Ragas 평가 실행

[Ragas](https://docs.ragas.io/)는 다음 메트릭으로 RAG 품질을 평가합니다:
- **Faithfulness(충실성)**: 답변이 검색된 컨텍스트에 근거하고 있는가?
- **Answer Relevancy(답변 관련성)**: 답변이 질문에 응답하고 있는가?
- **Context Precision(컨텍스트 정밀도)**: 검색된 청크가 질문과 관련이 있는가?

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

**예상 점수 (양호한 RAG 파이프라인):**

:::code{language=text showCopyAction=false}
{'faithfulness': 0.92, 'answer_relevancy': 0.88, 'context_precision': 0.95}
:::

| 메트릭 | 양호 | 의미 |
|---|---|---|
| Faithfulness > 0.8 | ✅ | 답변이 검색된 컨텍스트에 근거함 |
| Answer Relevancy > 0.8 | ✅ | 답변이 질문에 응답함 |
| Context Precision > 0.8 | ✅ | 검색된 청크가 관련 있음 |

---

## 📊 Track A와의 매핑

| | Track A | Track B |
|---|---|---|
| 벡터 DB | Milvus | **동일** |
| 임베딩 | LiteLLM을 통한 Bedrock Titan | Bedrock Titan (직접 또는 Kong `/loan-embeddings` 라우트를 통해) |
| 검색 연결 | LangGraph `@tool` | **Strands `@tool`** (Arize에서 TOOL 스팬으로 자동 트레이싱) |
| RAG 트레이스 | Langfuse | **Arize AX** (OpenInference TOOL 스팬) |
| RAG 평가 | Ragas | **동일** |

## ✅ 완료한 작업

- ✅ EKS에 Milvus 벡터 데이터베이스 배포 (standalone 모드)
- ✅ 대출 정책 문서 수집 (청킹 → 임베딩 → 저장)
- ✅ Strands 에이전트에 검색 도구 추가 (Arize에서 자동 트레이싱)
- ✅ 정책 문서에 대출 의사결정의 근거 부여
- ✅ Ragas로 RAG 품질 평가 (충실성, 관련성, 정밀도)

---

**[다음: 프로덕션을 위한 보안 강화 →](../security-hardening/)**
