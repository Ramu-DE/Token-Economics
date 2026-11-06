---
title: "LiteLLM으로 A/B 모델 라우팅"
weight: 2
---

평가 데이터에 따르면 Qwen 3 8B는 결정당 $0.002에 유용성 8.1/10 점수를 받는 반면, Claude 4.5 Sonnet은 $0.045에 9.5/10을 받습니다. A/B 모델 라우팅을 사용하면 간단한 케이스는 Qwen으로 자동 라우팅하여 비용을 95% 절감하고, 복잡한 시나리오에는 Claude를 사용할 수 있습니다.

## 실습: 현재 라우팅 이해

먼저 LiteLLM이 현재 요청을 어떻게 라우팅하는지 확인합니다:

### 단계 1: 현재 모델 구성 확인

:::code{language=bash showCopyAction=true}
# 현재 LiteLLM 모델 구성 보기
kubectl get configmap litellm-config -n litellm -o yaml | grep -A 20 "model_list:"
:::

**보게 될 것:**

:::code{language=yaml showCopyAction=false}
# 현재 구성 (모든 모델 동일한 가중치):
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

**현재 동작:**
- 애플리케이션이 사용할 모델을 명시적으로 지정
- 자동 라우팅 또는 폴백 없음
- 유사한 모델 간 로드 밸런싱 없음

### 단계 2: 현재 요청 분포 확인

현재 요청이 어떻게 분포되어 있는지 확인하겠습니다:

:::code{language=bash showCopyAction=true}
# 모델 사용 분포를 위해 Langfuse 쿼리
echo "Langfuse를 열고 다음으로 이동: Dashboards → Usage → Model Distribution"
echo "Langfuse URL: http://$(kubectl get ingress -n langfuse langfuse -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
:::

**예상 현재 분포:**
- **Claude 4.5 Sonnet**: ~90% (Loan Buddy 기본값)
- **Qwen 3 8B**: ~10% (수동 테스트만)
- **총 비용**: 높음 (대부분 프리미엄 모델)

## A/B 모델 라우팅이란?

**A/B 라우팅** 을 사용하면 LiteLLM이 구성 가능한 가중치를 기반으로 여러 모델에 트래픽을 자동으로 분배할 수 있습니다. 80/20, 50/50 등 원하는 비율로 트래픽을 분할하는 로드 밸런서 역할을 수행합니다.

**주요 기능:**

::::tabs

:::tab{label="Weighted Distribution"}
**트래픽 분할**

백분율로 모델 간 요청 분할:

```yaml
# 예시: 80% Claude, 20% Qwen
model_list:
  - model_name: loan-processing
    litellm_params:
      model: bedrock/global.anthropic.claude-sonnet-4-5-20250929-v1:0
      weight: 8  # 80% 트래픽
  - model_name: loan-processing
    litellm_params:
      model: openai/qwen3-8b-neuron
      weight: 2  # 20% 트래픽
```

**사용 사례:**
- **점진적 롤아웃**: 새 모델을 10%로 시작하여 50%, 100%로 확대
- **비용 최적화**: 80%는 저비용 모델, 20%는 검증용 프리미엄 모델
- **성능 테스트**: 트래픽을 분할하여 실제 성능 측정
:::

:::tab{label="Automatic Fallback"}
**복원력 및 신뢰성**

기본 모델을 사용할 수 없을 때 자동 폴오버:

```yaml
# 폴백 캐스케이드:
router_settings:
  fallbacks: [{"vllm/qwen3-8b-neuron": ["bedrock/claude-4.5-sonnet"]}]
  # 빠르고 저렴한 모델 먼저 시도 → 신뢰할 수 있는 프리미엄으로 폴백
```

**폴백 트리거:**
- 모델 타임아웃 (>30초 응답 시간)
- 서비스 사용 불가 (503 오류)
- 속도 제한 초과 (429 오류)
- 모델 충돌 또는 재시작
:::

:::tab{label="Load Balancing"}
**수평 스케일링**

여러 복제본에 로드 분산:

```yaml
# 3개의 Qwen 복제본에 균형 (동일한 model_name = 자동 로드 밸런싱):
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

**이점:**
- 더 높은 처리량 (3배 용량)
- 향상된 복원력 (복제본 1개 다운 시에도 66% 용량 유지)
- 롤링 업데이트 (복제본을 한 번에 하나씩 업데이트)
:::

:::tab{label="Smart Routing Rules"}
**컨텍스트 인식 라우팅**

요청 특성에 따라 라우팅:

```python
# 예시: 요청 특성에 따른 애플리케이션 레벨 라우팅
def route_by_complexity(prompt: str, metadata: dict) -> str:
    """요청 복잡성에 따라 라우팅."""
    prompt_length = len(prompt.split())
    
    if prompt_length < 100:  # 간단하고 짧은 요청
        return "loan-processing-simple"  # Qwen
    elif metadata.get("user_tier") == "premium":
        return "bedrock/claude-4.5-sonnet"  # 프리미엄 사용자는 Claude
    else:
        return "loan-processing"  # 표준 요청을 위한 A/B 분할
```

**라우팅 요소:**
- 프롬프트 길이 (토큰 수)
- 요청 태그/메타데이터
- 사용자 계층 (무료 vs 프리미엄)
- 시간대 (비피크 시간대 = 저비용 모델)
- 비용 예산 제약
:::

::::

## 실습: 80/20 A/B 라우팅 구성

비용 비교를 위해 트래픽의 80%를 Claude로, 20%를 Qwen으로 보내는 가중치 기반 라우팅을 설정합니다:

### 단계 1: 현재 구성 백업

:::code{language=bash showCopyAction=true}
# 기존 LiteLLM 값 백업
cp /workshop/components/ai-gateway/litellm/values.rendered.yaml \
   /workshop/components/ai-gateway/litellm/values.rendered.yaml.backup

# 백업 확인
ls -lh /workshop/components/ai-gateway/litellm/values.rendered.yaml*
:::

### 단계 2: LiteLLM 구성 편집

구성 파일 열기:

:::code{language=bash showCopyAction=true}
# 에디터에서 열기
code /workshop/components/ai-gateway/litellm/values.rendered.yaml
:::

### 단계 3: 라우터 구성 추가

80/20 가중치 라우팅을 활성화하려면 다음 구성을 추가합니다. LiteLLM에서는 **동일한** `model_name`과 다른 `weight` 값을 가진 여러 배포를 생성합니다:

:::code{language=yaml showCopyAction=true}
# 모든 LiteLLM 구성은 최상위 `proxy_config:` 키 아래에 위치합니다
# (모듈 A1에서 도입한 차트 스키마). 새 `loan-processing` 항목을
# model_list에, router_settings 블록을 기존 `proxy_config:` 아래에
# 추가합니다.
proxy_config:
  model_list:
    # 개별 모델 (직접 액세스를 위해 유지)
    - model_name: bedrock/claude-4.5-sonnet
      litellm_params:
        model: bedrock/global.anthropic.claude-sonnet-4-5-20250929-v1:0
        aws_region_name: us-west-2

    - model_name: vllm/qwen3-8b-neuron
      litellm_params:
        model: openai/qwen3-8b-neuron
        api_base: http://qwen3-8b-neuron.vllm:8000/v1
        api_key: fake-key

    # A/B 라우팅 그룹: 동일한 model_name, 다른 가중치
    - model_name: loan-processing
      litellm_params:
        model: bedrock/global.anthropic.claude-sonnet-4-5-20250929-v1:0
        aws_region_name: us-west-2
        weight: 8    # 80% 트래픽

    - model_name: loan-processing
      litellm_params:
        model: openai/qwen3-8b-neuron
        api_base: http://qwen3-8b-neuron.vllm:8000/v1
        api_key: fake-key
        weight: 2    # 20% 트래픽

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

**구성 분석:**

| 설정 | 값 | 목적 |
|---------|-------|---------|
| `routing_strategy` | simple-shuffle | 확률 기반 분배 사용 |
| `num_retries` | 2 | 실패한 요청을 최대 2번까지 재시도 |
| `timeout` | 30 | 요청당 최대 30초 |
| `weight: 8` | 80% | Claude가 트래픽의 80% 받음 |
| `weight: 2` | 20% | Qwen이 트래픽의 20% 받음 |
| `fallbacks` | loan-processing→Claude | 그룹의 모델이 실패하면 Claude 시도 |

**수행하는 작업:**
- "loan-processing"이라는 가상 모델 그룹 생성
- 애플리케이션이 "loan-processing"을 요청하면 LiteLLM이 가중치에 따라 80/20으로 자동 라우팅
- 개별 모델 (bedrock/claude-4.5-sonnet, vllm/qwen3-8b-neuron)은 테스트를 위해 여전히 접근 가능
- 배포가 실패하면 요청이 자동으로 Claude로 폴백

### 단계 4: 구성 변경 사항 적용

업데이트된 구성 배포:

:::code{language=bash showCopyAction=true}
# 새 구성 적용
helm upgrade --install litellm oci://ghcr.io/berriai/litellm-helm \
  --namespace litellm \
  -f /workshop/components/ai-gateway/litellm/values.rendered.yaml \
  --wait

# 롤아웃 완료 확인
kubectl rollout status deployment/litellm -n litellm

# 라우터 초기화를 위한 로그 확인
kubectl logs -n litellm deployment/litellm --tail=50 | grep -i "router"
:::

**예상 출력:** 롤아웃이 완료되면 라우터 초기화 로그가 다음과 유사하게 표시됩니다 (정확한 문구와 순서는 LiteLLM 버전에 따라 다릅니다):

:::code{language=yaml showCopyAction=false}
# 라우터 초기화 (예시):
INFO: Router initialized with model group: loan-processing
INFO: Model group 'loan-processing' has 2 deployments with weights ~[0.8, 0.2]
INFO: Fallbacks configured for loan-processing
INFO: LiteLLM proxy started successfully on port 4000
:::

::alert[**배포 시간**: 구성 변경 사항을 적용하는 데 약 30-60초가 걸립니다. 테스트하기 전에 "rollout status"가 "successfully rolled out"을 표시할 때까지 기다리십시오.]{type="info"}

## 실습: A/B 라우팅 테스트

80/20 분포가 작동하는지 확인합니다:

### 단계 1: 에이전트가 모델 그룹을 가리키도록 설정

80/20 분할은 방금 구성한 LiteLLM 게이트웨이 안에서 전적으로 이루어지므로 에이전트 코드의 라우팅 로직을 변경할 필요가 없습니다. 에이전트는 이미 모든 요청을 게이트웨이로 보내고 있으며(`GATEWAY_MODEL_ACCESS_KEY` 사용), 단일 모델 대신 새 `loan-processing` 모델 **이름** 을 요청하기만 하면 됩니다.

에이전트가 사용하는 모델 이름은 YAML 구성 파일이 아니라 코드(`credit-underwriting-agent.py`의 `llm_model`)에 설정되어 있습니다. 에이전트를 전환하려면 해당 한 줄을 수정하고 재배포합니다:

:::code{language=bash showCopyAction=true}
# 모델 이름은 구성 파일이 아니라 에이전트 소스에 하드코딩되어 있습니다
grep -n 'llm_model' \
  /workshop/apps/loan-buddy/credit-validation/credit-underwriting-agent.py
:::

:::code{language=python showCopyAction=true}
# 변경 전:
llm_model = "bedrock/claude-4.5-sonnet"

# 변경 후 (게이트웨이 80/20 라우팅을 자동으로 사용):
llm_model = "loan-processing"
:::

변경 사항을 적용하려면 Loan Buddy 재배포:

:::code{language=bash showCopyAction=true}
# 에이전트 Pod 재시작
kubectl rollout restart deployment/loan-buddy-agent -n workshop

# 재시작 대기
kubectl rollout status deployment/loan-buddy-agent -n workshop
:::

::alert[**에이전트를 건드리지 않으시겠습니까?** 라우팅이 게이트웨이에서 수행되므로 에이전트를 임의의 모델 이름에 그대로 둔 채 다음 단계의 `curl`로 게이트웨이에 직접 80/20 분할을 테스트할 수도 있습니다 — 재배포가 필요 없습니다.]{type="info"}

### 단계 2: 테스트 요청 보내기

80/20 분포를 보기 위해 LiteLLM을 통해 직접 라우팅 테스트:

:::code{language=bash showCopyAction=true}
# LiteLLM 서비스 포트 포워드
kubectl port-forward svc/litellm 4000:4000 -n litellm &

# 게이트웨이 API 키 가져오기 (에이전트가 사용하는 것과 동일한 키)
source /workshop/.env
export LITELLM_KEY=$AGENT_KEY

# 20개의 테스트 요청 보내기
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
  
  # 요청 간 작은 지연
  sleep 0.5
done

# 모든 백그라운드 작업 대기
wait

echo "loan-processing 모델 그룹에 20개의 테스트 요청 전송"
:::

### 단계 3: Langfuse에서 분포 확인

Langfuse를 열고 모델 분포 확인:

:::code{language=bash showCopyAction=true}
# Langfuse 열기
echo "Langfuse URL: http://$(kubectl get ingress -n langfuse langfuse -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
echo "다음으로 이동: Dashboards → Create custom dashboard"
:::

빠른 분포 위젯 생성:

**위젯 구성:**
- **View**: Traces
- **Metric**: Count
- **Breakdown**: Model Name
- **Date Range**: Past 1 hour
- **Chart Type**: Donut Chart

**예상 결과:**

:::code{language=yaml showCopyAction=false}
# 20개 요청의 모델 분포:
bedrock/global.anthropic.claude-sonnet-4-5: 16 requests (80%)
openai/qwen3-8b-neuron: 4 requests (20%)

# 정확한 숫자는 무작위성으로 인해 약간 다를 수 있지만
# 80/20 분할에 가까워야 합니다
:::

::alert[**통계적 참고**: 20개의 요청만으로는 무작위 변동으로 인해 15/5 또는 17/3 분할을 볼 수 있습니다. 정확히 80/20으로 수렴하는 분포를 보려면 100개 이상의 요청을 실행하십시오.]{type="info"}

### 단계 4: 폴백 동작 테스트

Qwen을 사용할 수 없을 때 요청이 자동으로 폴오버되는지 확인하겠습니다:

:::code{language=bash showCopyAction=true}
# Qwen을 0개 복제본으로 일시적으로 스케일 (실패 시뮬레이션)
kubectl scale deployment qwen3-8b-neuron -n vllm --replicas=0

# Pod 종료 대기
sleep 10

# 테스트 요청 보내기 (폴백을 통해 자동으로 Claude 사용해야 함)
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

# Langfuse에서 어떤 모델이 처리했는지 확인
echo "Langfuse 확인 - 추적이 Claude를 표시해야 함 (폴백 활성화됨)"

# Qwen 복원
kubectl scale deployment qwen3-8b-neuron -n vllm --replicas=1
:::

**Langfuse에서 볼 수 있는 것:**

이 요청에 대한 Langfuse 추적에서 폴백 메타데이터는 다음과 같이 표시됩니다:

**추적 메타데이터:**
- **Requested Model**: loan-processing
- **Attempted Model**: vllm/qwen3-8b-neuron
- **Fallback Reason**: "Connection refused - target service unavailable"
- **Actual Model**: bedrock/global.anthropic.claude-sonnet-4-5
- **Fallback Time**: 2.1초 (즉각적인 감지 + Claude 응답)

## 실습: 비용 대 품질 트레이드오프 분석

이제 A/B 결과를 비교하여 데이터 기반 라우팅 결정을 내립니다:

### 단계 1: 비용 비교 대시보드 생성

Langfuse에서 포괄적인 A/B 비교 대시보드 생성:

**대시보드 이름**: `A/B Test: Claude vs Qwen (80/20)`

다음 위젯 추가:

**위젯 1: 모델별 비용**

:::code{language=yaml showCopyAction=false}
View: Traces
Metric: Total Cost (Sum)
Breakdown: Model Name
Chart: Vertical Bar
Date Range: Past 1 hour
:::

**위젯 2: 모델별 평균 지연시간**

:::code{language=yaml showCopyAction=false}
View: Traces
Metric: Latency (Average)
Breakdown: Model Name
Chart: Horizontal Bar
Date Range: Past 1 hour
:::

**위젯 3: 모델별 품질 점수**

:::code{language=yaml showCopyAction=false}
View: Scores
Metric: Score Value (Average)
Breakdown: Model Name + Score Name
Chart: Grouped Bar
Date Range: Past 1 hour
Filter: Score Name in [loan-accuracy, loan-helpfulness, loan-safety]
:::

### 단계 2: 비교 결과 보기

**분석 결과 (100개 요청에서):**

| 메트릭 | Claude 4.5 Sonnet (80 req) | Qwen 3 8B (20 req) | 차이 |
|--------|----------------------------|---------------------|------------|
| **총 비용** | $3.60 | $0.04 | Claude가 90배 더 비쌈 |
| **평균 요청당 비용** | $0.045 | $0.002 | Qwen 요청당 $0.043 절약 |
| **평균 지연시간** | 3.2초 | 5.8초 | Qwen이 2.6초 느림 |
| **Accuracy 점수** | 9.2/10 | 8.3/10 | -0.9점 |
| **Helpfulness 점수** | 9.5/10 | 8.1/10 | -1.4점 |
| **Safety 점수** | 10.0/10 | 9.7/10 | -0.3점 |

**비즈니스 영향 계산:**

:::code{language=yaml showCopyAction=false}
# 현재 80/20 분할 결과:
Daily volume: 1,000 applications
Claude (80%): 800 req × $0.045 = $36.00
Qwen (20%): 200 req × $0.002 = $0.40
Daily cost: $36.40

# 100% Claude와 비교:
1,000 req × $0.045 = $45.00/day
Savings: $8.60/day = $258/month (19% 절감)

# 100% Qwen과 비교:
1,000 req × $0.002 = $2.00/day
Savings: $34.40/day = $1,032/month (95% 절감)
하지만: 주요 메트릭에서 품질이 1-1.4점 하락
:::

### 단계 3: 복잡성 기반 라우팅 구현

LiteLLM에는 "품질 인식" 라우팅이 내장되어 있지 않지만, 여러 모델 그룹을 정의하고 요청 특성에 따라 라우팅하여 애플리케이션 레벨에서 지능형 라우팅을 구현할 수 있습니다.

간단한 케이스와 복잡한 케이스를 위한 별도의 모델 그룹으로 LiteLLM 구성 업데이트:

:::code{language=yaml showCopyAction=true}
model_list:
  # 간단한 케이스: Qwen만 (저렴, 빠름)
  - model_name: loan-processing-simple
    litellm_params:
      model: openai/qwen3-8b-neuron
      api_base: http://qwen3-8b-neuron.vllm:8000/v1
      api_key: fake-key
  
  # 복잡한 케이스: Claude만 (프리미엄, 정확)
  - model_name: loan-processing-complex
    litellm_params:
      model: bedrock/global.anthropic.claude-sonnet-4-5-20250929-v1:0
      aws_region_name: us-west-2
  
  # 일반 A/B 테스트 그룹 (80/20 분할)
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

그런 다음 Loan Buddy 에이전트에서 애플리케이션 레벨 라우팅 로직을 구현:

:::code{language=python showCopyAction=true}
def get_model_for_application(credit_score: int, dti_ratio: float) -> str:
    """신청 복잡성에 따라 적절한 모델로 라우팅."""
    if credit_score >= 700 and dti_ratio <= 36:
        # 높은 신용 점수 + 낮은 DTI = 간단한 케이스
        return "loan-processing-simple"   # Qwen: 빠르고 저렴
    else:
        # 낮은 신용 또는 높은 DTI = 신중한 분석이 필요한 엣지 케이스
        return "loan-processing-complex"  # Claude: 엣지 케이스에 정확


# 에이전트에서 사용:
model_name = get_model_for_application(
    credit_score=applicant.credit_score,
    dti_ratio=applicant.debt_to_income_ratio
)

response = litellm_client.chat.completions.create(
    model=model_name,
    messages=[{"role": "user", "content": prompt}]
)
:::

**예상 최적화:**

:::code{language=yaml showCopyAction=false}
# 지능형 라우팅 결과 (일반적인 분포 가정):
신청의 70%가 간단함 (신용 ≥700, DTI ≤36) → 100% Qwen
신청의 30%가 복잡함 → 100% Claude

새로운 비용 분석:
- Simple (700/day): 700 Qwen × $0.002 = $1.40
- Complex (300/day): 300 Claude × $0.045 = $13.50
- Total: $14.90/day vs $45/day 기준선 = 67% 비용 절감
- 품질 유지: 복잡한 케이스는 여전히 최고의 모델을 받음
:::

## 실습: 점진적 롤아웃 구현

안전한 모델 배포를 위해 A/B 라우팅을 사용합니다:

### 단계 1: 점진적 트래픽 전환 전략

**시나리오**: 새로운 미세 조정된 Qwen 모델을 롤아웃하려고 합니다. 위험을 최소화하기 위해 점진적 롤아웃 사용:

:::code{language=yaml showCopyAction=true}
# 1주차: 카나리 (5% 새 모델)
model_list:
  - model_name: loan-processing
    litellm_params:
      model: openai/qwen3-8b-neuron-v2  # 새 미세 조정된 모델
      api_base: http://qwen3-8b-neuron-v2.vllm:8000/v1
      weight: 1  # 5% 트래픽
  - model_name: loan-processing
    litellm_params:
      model: bedrock/global.anthropic.claude-sonnet-4-5-20250929-v1:0
      weight: 19  # 95% 트래픽

# 2주차: 품질이 유지되면 25%로 증가 (weight: 5 vs 15)
# 3주차: 품질이 유지되면 75%로 증가 (weight: 15 vs 5)
# 4주차: 성공하면 100%로 전체 롤아웃 (weight: 20 vs 0)
:::

### 단계 2: 롤아웃 건전성 모니터링

롤아웃 건전성 대시보드 생성:

**모니터링할 주요 메트릭:**
- **오류율**: 1% 미만으로 유지해야 함
- **P95 지연시간**: 20%를 초과하여 증가하지 않아야 함
- **품질 점수**: 0.5점을 초과하여 하락하지 않아야 함
- **비용**: 절감 실현 추적
- **롤백 수**: 폴백 활성화 모니터링

**롤아웃 결정 규칙:**

| 메트릭 상태 | 조치 |
|---------------|--------|
| 24시간 동안 모든 메트릭 정상 | 트래픽을 25% 증가 |
| 일부 메트릭 저하 | 현재 비율 유지, 원인 조사 |
| 메트릭 기준 미달 | 이전 비율로 롤백 |

## A/B 라우팅의 프로덕션 패턴

**패턴 1: 시간 기반 라우팅**

:::code{language=python showCopyAction=true}
# 낮은 위험 시간대에 더 저렴한 모델 사용 (애플리케이션 레벨 로직):
from datetime import datetime
import pytz

def get_model_by_time() -> str:
    """하루 시간에 따라 적절한 모델로 라우팅."""
    et = pytz.timezone('US/Eastern')
    current_hour = datetime.now(et).hour
    
    if 9 <= current_hour < 17:  # 영업 시간 (오전 9시-오후 5시 ET)
        return "bedrock/claude-4.5-sonnet"  # 프리미엄 모델
    else:  # 영업 외 시간
        return "loan-processing"  # 80/20 분할 (주로 Qwen)
:::

**패턴 2: 사용자 계층 기반 라우팅**

:::code{language=python showCopyAction=true}
# 프리미엄 고객이 프리미엄 모델을 받음 (애플리케이션 레벨 로직):
def get_model_by_tier(user_tier: str) -> str:
    """사용자 구독 계층에 따라 적절한 모델로 라우팅."""
    if user_tier == "premium":
        return "bedrock/claude-4.5-sonnet"  # 유료 고객을 위한 100% Claude
    elif user_tier == "free":
        return "loan-processing-simple"  # 무료 계층을 위한 Qwen (비용 제어)
    else:  # 표준 계층
        return "loan-processing"  # 80/20 A/B 분할
:::

**패턴 3: 로드 기반 라우팅**

:::code{language=yaml showCopyAction=true}
# LiteLLM은 동일한 model_name을 가진 여러 복제본에 자동으로 로드 밸런싱:
model_list:
  # 수평 스케일링을 위해 3개의 Qwen 복제본 배포
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
  routing_strategy: simple-shuffle  # 복제본 간 라운드 로빈
:::

## A/B 모델 라우팅의 주요 이점

이 섹션의 실습을 통해 다음을 확인했습니다:

- **비용 최적화**: 복잡성 기반 라우팅을 통한 67% 비용 절감
- **품질 유지**: 복잡한 케이스는 계속 프리미엄 모델을 사용
- **복원력**: 자동 폴오버로 다운타임 방지
- **안전한 롤아웃**: 점진적 트래픽 전환으로 배포 위험 최소화
- **데이터 기반 결정**: 실제 메트릭이 모델 선택을 안내
- **유연성**: 비즈니스 요구에 따라 라우팅 전략 조정

## 다음 단계

이제 A/B 모델 라우팅을 구현했습니다. 플랫폼은 다음을 수행할 수 있습니다:
- 모델 간 트래픽을 80/20으로 자동 분배
- 모델을 사용할 수 없을 때 즉시 폴오버
- 실제 데이터로 비용과 품질 트레이드오프 비교
- 안전 장치를 갖춘 상태로 새 모델을 점진적으로 롤아웃

다음 단계는 이러한 모델을 지탱하는 인프라 상태를 모니터링하는 것입니다. 다음 섹션에서는 Amazon Managed Prometheus 및 Grafana를 사용한 프로덕션급 모니터링을 추가하여 vLLM 성능과 Neuron 활용률을 추적하고 실시간 메트릭 기반 오토스케일링을 구성합니다.

---

**[다음: 인프라 모니터링 및 오토스케일링 →](../monitoring-scaling/)**
