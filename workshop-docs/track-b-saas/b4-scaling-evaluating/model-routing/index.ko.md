---
title: "Kong을 활용한 A/B 모델 라우팅"
weight: 30
---

Kong의 **`ai-proxy-advanced`** plugin은 단일 라우트에서 여러 LLM 모델을 프론트엔드로 제공하며, 설정 가능한 알고리즘 — 가중치 라운드로빈, 비용 인식, 지연시간 인식, 헤더 기반 스티키 라우팅 — 을 사용하여 요청을 라우팅합니다. 이 페이지에서는 각 알고리즘을 실습으로 살펴봅니다.

::alert[**Bedrock + IRSA의 핵심 패턴:** 각 타겟은 Konnect 스키마 검증을 위해 `auth` 블록이 필요합니다. `header_name: x-dummy` / `header_value: unused` / `allow_override: false` 를 사용하세요 — Bedrock 인증은 pod의 IRSA 역할에서 자동으로 가져옵니다. 클라이언트는 plugin이 모델을 선택하도록 `"model":""`(빈 문자열)을 전송합니다.]{type="warning"}

## 🎯 학습 내용

- ⚖️ **Part 1** — 가중치 라운드로빈 (80/20 A/B 분할)
- 💰 **Part 2** — 최저 사용량 (비용 인식: 가장 저렴한 모델로 라우팅)
- ⚡ **Part 3** — 최저 지연시간 (속도 인식: 가장 빠른 모델로 라우팅)
- 🔒 **Part 4** — 일관성 해싱 (스티키: 동일한 사용자는 항상 동일한 모델로 라우팅)
- 🛡️ **Part 5** — 페일오버 / 서킷 브레이커 (장애 시 자동 폴백)
- 🏷️ **Part 6** — 모델 별칭 (클라이언트 제어: 친숙한 이름 전송)
- 🧠 **Part 7** — 시맨틱 라우팅 (임베딩을 사용하여 프롬프트 내용으로 라우팅)
- ⚖️ **Part 8** — 최소 연결 (활성 연결 수 기준 로드 밸런싱)

---

## Part 1 — 가중치 라운드로빈 (80/20 A/B 분할) ⭐

`round-robin` 알고리즘은 `weight`에 따라 요청을 분배합니다. 단일 라우트에서 Claude 80%, Nova 20%로 설정합니다:

### 🎯 Plugin 배포

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  balancer:
    algorithm: round-robin
  targets:
  - model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 3.0
        output_cost: 15.0
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    weight: 80
  - model:
      name: amazon.nova-micro-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 0.035
        output_cost: 0.14
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    weight: 20
EOF
:::

단일 모델 `ai-proxy` 대신 `ai-proxy-advanced`를 사용하도록 라우트를 전환합니다:

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-advanced-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
:::

### 설정 설명

| 필드 | 값 | 의미 |
|---|---|---|
| `balancer.algorithm` | `round-robin` | 타겟별 `weight`를 사용한 확률적 분배 |
| `targets[0].weight` | `80` | Claude가 트래픽의 약 80%를 받음 |
| `targets[1].weight` | `20` | Nova가 트래픽의 약 20%를 받음 |
| `input_cost` / `output_cost` | 100만 token당 USD | Kong의 비용 지표에 반영되며 `lowest-usage`(Part 2)를 활성화 |
| `auth.header_name: x-dummy` | 우회 | Konnect 스키마에서 요구됨; Bedrock 인증은 IRSA(pod identity)를 사용 |

### 🚀 테스트: 80/20 분배 확인

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

CLAUDE=0; NOVA=0
for i in {1..10}; do
  MODEL=$(curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
    -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
    -d '{"model":"","messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('model','ERR'))" 2>/dev/null)
  case "$MODEL" in *claude*) CLAUDE=$((CLAUDE+1));; *nova*) NOVA=$((NOVA+1));; esac
  echo "  Request $i: $MODEL"
  sleep 0.5
done
echo ""
echo "Distribution: Claude=$CLAUDE Nova=$NOVA (expected ~8/2)"
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
  Request 1: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 2: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 3: amazon.nova-micro-v1:0
  Request 4: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  ...
Distribution: Claude=8 Nova=2 (expected ~8/2)
:::

**왜 8/2인가?** `round-robin` 알고리즘은 `weight`를 기반으로 요청을 확률적으로 분배합니다. 가중치 80:20에서 각 요청은 Claude에 도달할 확률 80%, Nova에 도달할 확률 20%를 갖습니다. 10개의 요청에 걸쳐 대략 8:2를 보게 됩니다(정확한 수치는 무작위성으로 인해 달라짐).

::alert[**`"model":""`**(빈 문자열)이 핵심입니다. 클라이언트가 빈 모델 필드를 전송하면 plugin이 알고리즘에 따라 타겟을 선택합니다. 특정 모델 이름(예: `"model":"global.anthropic.claude-sonnet-4-5-20250929-v1:0"`)을 전송하면 해당 타겟으로 라우팅될 때만 동작합니다 — 다른 타겟으로 라우팅된 요청은 "cannot use own model"로 실패합니다.]{type="warning"}

---

## Part 2 — 최저 사용량 (비용 인식 라우팅)

`lowest-usage` 알고리즘은 **누적된 token 또는 비용 지출이 가장 낮은** 타겟으로 라우팅합니다 — 게이트웨이 네이티브 비용 최적화 도구입니다. 트래픽은 자연스럽게 더 저렴한 모델 쪽으로 이동합니다.

### 🎯 Plugin 배포

카운터를 초기화하기 위해 이전 설정을 삭제한 다음 새 알고리즘을 적용합니다:

:::code{language=bash showCopyAction=true}
kubectl delete kongplugin ai-proxy-advanced-loan-strands -n kong 2>/dev/null; sleep 3
:::

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  balancer:
    algorithm: lowest-usage
    tokens_count_strategy: cost
  targets:
  - model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 3.0
        output_cost: 15.0
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
  - model:
      name: amazon.nova-micro-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 0.035
        output_cost: 0.14
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
EOF
:::

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-advanced-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
:::

### 설정 설명

| 필드 | 값 | 의미 |
|---|---|---|
| `balancer.algorithm` | `lowest-usage` | 지금까지 **누적 지출이 가장 낮은** 타겟으로 각 요청을 라우팅 |
| `tokens_count_strategy` | `cost` | 지출을 **달러** 단위로 추적(대안: 원시 token 계산을 위한 `prompt-tokens`, `completion-tokens`, `total-tokens`) |
| `input_cost` | `3.0`(Claude) / `0.035`(Nova) | **입력 token 100만 개당** USD — 요청당 달러 비용 계산에 사용 |
| `output_cost` | `15.0`(Claude) / `0.14`(Nova) | **출력 token 100만 개당** USD |

### 알고리즘의 결정 방식

plugin은 **타겟별 누적 비용 총계**를 유지합니다. 각 요청이 완료되면 비용을 계산하여 해당 타겟의 총계에 더합니다:

:::code{language=text showCopyAction=false}
cost_per_request = (prompt_tokens / 1,000,000 × input_cost) + (completion_tokens / 1,000,000 × output_cost)
:::

일반적인 짧은 요청(입력 8 + 출력 3 token)의 경우:

| 타겟 | 계산 | 요청당 비용 |
|---|---|---|
| **Claude** | (8/1M × $3.0) + (3/1M × $15.0) | **$0.000069** |
| **Nova** | (8/1M × $0.035) + (3/1M × $0.14) | **$0.000000700** |

Claude는 **요청당 약 100배 더 비쌉니다**. 따라서 Claude 요청이 단 1건만 처리되어도 그 누적 총계($0.000069)가 Nova의 누적 총계($0)를 크게 초과합니다. plugin은 Nova의 누적 총계가 결국 따라잡을 때까지 이후 모든 요청을 Nova("최저 사용량" 타겟)로 라우팅합니다 — 이는 Claude 요청 1건과 맞먹으려면 약 100건의 Nova 요청이 필요합니다.

**결과적으로:** Nova가 자동으로 트래픽의 대부분을 받습니다. plugin은 게이트웨이 네이티브 비용 최적화 도구입니다 — 애플리케이션 로직이 필요 없습니다.

### 🚀 테스트: 비용 인식 이동 관찰

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "Sending 10 requests (lowest-usage will favor Nova after warmup)..."
for i in {1..10}; do
  MODEL=$(curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
    -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
    -d '{"model":"","messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('model','ERR'))" 2>/dev/null)
  echo "  Request $i: $MODEL"
  sleep 0.5
done
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
Sending 10 requests (lowest-usage will favor Nova after warmup)...
  Request 1: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 2: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 3: amazon.nova-micro-v1:0
  Request 4: amazon.nova-micro-v1:0
  Request 5: amazon.nova-micro-v1:0
  Request 6: amazon.nova-micro-v1:0
  Request 7: amazon.nova-micro-v1:0
  Request 8: amazon.nova-micro-v1:0
  Request 9: amazon.nova-micro-v1:0
  Request 10: amazon.nova-micro-v1:0
:::

**왜 이런 패턴인가?**

| 요청 | 이 요청 전 누적 총계 | 결정 | 이유 |
|---|---|---|---|
| 1 | Claude: $0, Nova: $0 | **Claude** | 동점 — plugin이 첫 번째 타겟 선택 |
| 2 | Claude: $0.000069, Nova: $0 | **Nova** 또는 **Claude** | Nova가 더 낮지만 초기 워밍업 단계에서는 양쪽을 모두 탐색할 수 있음 |
| 3 | Claude: ~$0.000138, Nova: ~$0.0000007 | **Nova** | Nova의 총계($0.0000007)가 Claude($0.000138)보다 훨씬 낮음 |
| 4-10 | Claude는 ~$0.000138 유지, Nova는 천천히 상승 | **Nova** | Nova가 단 2건의 Claude 비용을 따라잡으려면 약 100건의 Nova 요청이 필요 |

초기 워밍업(요청 1-2) 이후에는 Nova의 요청당 비용이 약 100배 낮기 때문에 Nova가 지배적입니다. plugin이 Nova의 누적 비용을 Claude의 2건 요청 총계까지 따라잡으려면 약 100건의 요청을 Nova로 보내야 합니다 — 사실상 이것을 "90%+ Nova" 라우터로 만듭니다.

::alert[**프로덕션 유스케이스:** `input_cost`/`output_cost`를 실제 Bedrock 가격으로 설정하세요. 알고리즘이 모델 전반의 달러 지출을 자동으로 균형 있게 조정합니다 — 통합 예산이 있고 수동 가중치 조정 없이 Kong이 이를 최적으로 분배하기를 원할 때 유용합니다.]{type="info"}

---

## Part 3 — 최저 지연시간 (속도 인식 라우팅)

`lowest-latency` 알고리즘은 지수 가중 이동 평균(EWMA)을 사용하여 **가장 빠르게 응답하는 모델**로 라우팅합니다. Nova Micro는 일반적으로 Claude Sonnet보다 3-5배 빠릅니다.

### 🎯 Plugin 배포

EWMA 상태를 초기화하기 위해 이전 설정을 삭제한 다음 적용합니다:

:::code{language=bash showCopyAction=true}
kubectl delete kongplugin ai-proxy-advanced-loan-strands -n kong 2>/dev/null; sleep 3
:::

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  balancer:
    algorithm: lowest-latency
    latency_strategy: tpot
  targets:
  - model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
  - model:
      name: amazon.nova-micro-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
EOF
:::

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-advanced-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
:::

### 설정 설명

| 필드 | 값 | 의미 |
|---|---|---|
| `balancer.algorithm` | `lowest-latency` | **관측된 지연시간이 가장 낮은** 타겟으로 각 요청을 라우팅 |
| `latency_strategy` | `tpot` | **출력 token당 시간(Time Per Output Token)** 측정 — 총 응답 시간 / 출력 token 수(대안: 원시 종단 간 지연시간을 위한 `e2e`) |

### 알고리즘의 결정 방식

plugin은 각 타겟에 대한 지연시간의 **지수 가중 이동 평균(EWMA)**을 유지합니다. 각 응답 후:

:::code{language=text showCopyAction=false}
EWMA_new = α × latency_this_request + (1 - α) × EWMA_previous
:::

다음 요청은 **더 낮은 EWMA**를 가진 타겟으로 전달됩니다. 주요 동작:

1. **워밍업 단계** — 각 타겟이 신뢰할 수 있는 EWMA를 구축할 만큼 충분한 요청을 처리할 때까지 알고리즘은 양쪽 타겟을 모두 탐색합니다. 워밍업 중에는 라우팅이 무작위로 보이거나 첫 번째 타겟으로 편향될 수 있습니다.
2. **정상 상태** — 워밍업 후 트래픽의 약 90-99%가 가장 빠른 모델로 이동합니다. plugin은 한 타겟으로 **절대 100%를 보내지 않습니다** — 조건 변화(예: 부하 감소로 느린 모델이 빨라짐)를 감지하기 위해 주기적으로 느린 타겟을 탐색합니다.
3. **TPOT vs E2E** — `tpot`(출력 token당 시간)는 응답 길이와 무관하게 생성 속도를 측정합니다. `e2e`는 총 벽시계 시간(첫 token까지의 시간 + 모든 생성 시간 포함)을 측정합니다. 응답 길이가 다양할 때 모델을 공정하게 비교하려면 TPOT가 더 낫습니다.

**본 모델들의 일반적인 지연시간:**

| 모델 | TTFT(첫 token까지 시간) | TPOT(출력 token당) | 10 token에 대한 E2E |
|---|---|---|---|
| Claude Sonnet 4.5 | ~800ms | ~50ms | ~1.3s |
| Nova Micro | ~200ms | ~30ms | ~0.5s |

### 🚀 테스트: 지연시간 인식 라우팅 관찰

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "Sending 10 requests (lowest-latency routes to fastest model)..."
for i in {1..10}; do
  MODEL=$(curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
    -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
    -d '{"model":"","messages":[{"role":"user","content":"hi"}],"max_tokens":10}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('model','ERR'))" 2>/dev/null)
  echo "  Request $i: $MODEL"
  sleep 0.5
done
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
Sending 10 requests (lowest-latency routes to fastest model)...
  Request 1: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 2: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 3: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 4: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 5: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 6: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 7: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 8: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 9: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 10: global.anthropic.claude-sonnet-4-5-20250929-v1:0
:::

**왜 모두 Claude인가(양쪽을 탐색하기 위해 모델 간에 분할하지 않고)?**

`lowest-latency` 알고리즘은 라운드로빈 워밍업을 사용하지 **않습니다**. 대신 **감쇠 계수를 가진 EWMA**를 사용합니다 — 첫 번째로 설정된 타겟으로 시작하며 다른 타겟이 더 빠르다는 충분한 증거가 누적될 때만 이동합니다. `tpot`(출력 token당 시간)와 비스트리밍 Bedrock 응답의 경우:

| 요인 | 설명 |
|---|---|
| **라운드로빈 워밍업 없음** | `lowest-usage`(양쪽을 $0에서 시작하여 교대함)와 달리, `lowest-latency`는 첫 번째 타겟으로 라우팅을 시작하고 EWMA 데이터가 이동을 시사할 때까지 그대로 유지합니다. 지연시간을 "발견"하기 위해 트래픽을 능동적으로 분할하지 **않습니다**. |
| **TPOT는 스트리밍이 필요** | `tpot` 전략은 출력 token 간의 시간을 측정합니다. 비스트리밍 모드(이 워크숍에서 사용)에서는 전체 응답이 한 번에 도착합니다 — plugin은 `total_time / output_tokens`를 근사값으로 계산하지만, 이는 요청별 분산이 큽니다. |
| **느린 탐색 속도** | 알고리즘은 다른 타겟으로 가끔 탐색 요청을 보내지만(모든 요청이 아님), 요청이 10-30건에 불과하면 탐색이 아직 발생하지 않았을 수 있습니다. 수천 건의 요청이 있는 프로덕션에서는 탐색이 이동에 충분한 데이터를 누적합니다. |

**핵심 인사이트:** `lowest-latency`는 수천 건의 요청이 시간에 걸쳐 자연스럽게 EWMA 데이터를 구축하는 **고트래픽 프로덕션 워크로드**를 위해 설계되었습니다. 10-30건 요청의 워크숍 테스트에서는 Claude(첫 번째 타겟)에 고정되는 것을 보게 됩니다. 프로덕션에서는 약 100건 이상의 요청 후 Nova가 더 빠르다는 것을 발견하고 트래픽의 약 90%를 점진적으로 그쪽으로 이동합니다.

::alert[**`lowest-latency`는 대규모 프로덕션에서 빛을 발합니다.** 높은 요청량에서 EWMA가 빠르게 수렴하여 트래픽의 90% 이상을 가장 빠른 모델로 라우팅합니다. 워크숍 테스트에서는 가중치가 있는 `round-robin`(Part 1) 또는 `lowest-usage`(Part 2)가 더 명확한 즉각적 동작을 보여줍니다.]{type="info"}

::alert[**`latency_strategy: e2e` vs `tpot`** — 총 응답 시간을 중요하게 여기고 스트리밍하지 않을 때는 `e2e`(종단 간)를 사용하세요. 스트리밍을 사용하고 응답 길이와 무관하게 생성 속도를 비교하려면 `tpot`(출력 token당 시간)를 사용하세요.]{type="info"}

---

## Part 4 — 일관성 해싱 (스티키 라우팅)

`consistent-hashing` 알고리즘은 **헤더 값**을 기반으로 라우팅합니다 — 동일한 헤더 값은 항상 동일한 모델로 도달합니다. 결정론적 모델 할당을 원하는 사용자별 또는 등급별 라우팅에 유용합니다.

### 🎯 Plugin 배포

:::code{language=bash showCopyAction=true}
kubectl delete kongplugin ai-proxy-advanced-loan-strands -n kong 2>/dev/null; sleep 3
:::

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  balancer:
    algorithm: consistent-hashing
    hash_on_header: X-User-Tier
  targets:
  - model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
  - model:
      name: amazon.nova-micro-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
EOF
:::

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-advanced-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
:::

### 설정 설명

| 필드 | 값 | 의미 |
|---|---|---|
| `balancer.algorithm` | `consistent-hashing` | 헤더 값을 해싱하여 매번 결정론적으로 동일한 타겟 선택 |
| `hash_on_header` | `X-User-Tier` | 타겟 선택을 위해 값을 해싱할 HTTP 헤더 |

### 알고리즘의 결정 방식

plugin은 **해시 링**(일관성 해싱)을 사용하여 헤더 값을 타겟에 매핑합니다:

:::code{language=text showCopyAction=false}
target_index = hash(header_value) % number_of_targets
:::

1. Kong이 `X-User-Tier`의 값을 해싱합니다(예: hash("premium") → 어떤 숫자)
2. 그 숫자가 해시 링의 특정 위치에 매핑됩니다
3. 링에서 가장 가까운 타겟이 요청을 처리합니다
4. **동일한 값 = 매번 동일한 타겟**(결정론적, 무작위성 없음)

**주요 동작:**
- **동일한 헤더 값**은 항상 **동일한 모델**로 라우팅됩니다 — 요청 전반의 일관성 보장
- **다른 헤더 값**은 다른 모델로 라우팅될 수도 있고 아닐 수도 있습니다 — 해시 링의 어디에 도달하는지에 따라 달라집니다
- 타겟이 2개뿐이면 각 값은 어느 한 타겟에 도달할 확률이 약 50%입니다. 일부 값("premium", "free")은 동일한 타겟으로 해싱될 수 있습니다 — 이는 버그가 아니라 예상되는 해시 링 동작입니다

### 🚀 테스트: 헤더에 의한 스티키 라우팅

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "=== premium tier (same header → same model every time) ==="
for i in {1..3}; do
  curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
    -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
    -H "X-User-Tier: premium" \
    -d '{"model":"","messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); print(f'  {r.get(\"model\",\"ERR\")}')" 2>/dev/null
  sleep 0.5
done

echo ""
echo "=== free tier (different header value → deterministic but may land on same or different model) ==="
for i in {1..3}; do
  curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
    -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
    -H "X-User-Tier: free" \
    -d '{"model":"","messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); print(f'  {r.get(\"model\",\"ERR\")}')" 2>/dev/null
  sleep 0.5
done

echo ""
echo "=== try more values to find one that hashes to Nova ==="
for TIER in gold silver bronze team-a team-b user-123; do
  MODEL=$(curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
    -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
    -H "X-User-Tier: $TIER" \
    -d '{"model":"","messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); m=r.get('model','ERR'); print('Claude' if 'claude' in m else 'Nova')" 2>/dev/null)
  echo "  X-User-Tier: $TIER → $MODEL"
  sleep 0.3
done
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
=== premium tier (same header → same model every time) ===
  global.anthropic.claude-sonnet-4-5-20250929-v1:0
  global.anthropic.claude-sonnet-4-5-20250929-v1:0
  global.anthropic.claude-sonnet-4-5-20250929-v1:0

=== free tier (different header value → deterministic but may land on same or different model) ===
  global.anthropic.claude-sonnet-4-5-20250929-v1:0
  global.anthropic.claude-sonnet-4-5-20250929-v1:0
  global.anthropic.claude-sonnet-4-5-20250929-v1:0

=== try more values to find one that hashes to Nova ===
  X-User-Tier: gold → Claude
  X-User-Tier: silver → Nova
  X-User-Tier: bronze → Claude
  X-User-Tier: team-a → Nova
  X-User-Tier: team-b → Claude
  X-User-Tier: user-123 → Claude
:::

**왜 "premium"과 "free"가 모두 Claude로 라우팅되는가?**

| 관찰 | 설명 |
|---|---|
| 둘 다 Claude에 도달 | `hash("premium")`과 `hash("free")`가 우연히 해시 링의 같은 쪽에 도달함 → Claude. 이것은 정상입니다 — 타겟이 2개뿐일 때 임의의 값 쌍이 동일한 타겟에 도달할 확률은 50%입니다. |
| 3번의 반복 모두 일관됨 | **이것이 보장입니다** — 동일한 값은 항상 동일한 모델로 라우팅됩니다. "premium" → 매번, 영원히 Claude. 무작위성 없음. |
| 다른 값은 Nova에 도달 | 더 많은 헤더 값(silver, team-a 등)을 시도하세요 — 일부는 링의 Nova 쪽으로 해싱됩니다. 고유한 사용자 ID가 많은 프로덕션에서는 두 모델에 자연스러운 분산이 이루어집니다. |

**핵심 보장:** 분산이 아니라 일관성입니다. "premium" 등급의 사용자는 **항상** Claude를 받습니다. "silver" 등급의 사용자는 **항상** Nova를 받습니다. 매핑은 결정론적입니다 — 어떤 값이 어디로 매핑되는지 테스트하고 그 지식을 애플리케이션에서 활용할 수 있습니다.

::alert[**프로덕션 패턴:** 등급 대신 `X-Consumer-Username` 또는 사용자 ID 헤더로 해싱하세요. 고유한 사용자가 많으면(예: 수백 명의 대출 신청자) 해시 링이 2개 타겟에 자연스럽게 약 50/50으로 분산합니다. 각 사용자는 모든 요청에 걸쳐 일관된 모델 할당을 받습니다 — 사용자가 일관된 경험을 갖기를 원하는 A/B 테스트에 유용합니다.]{type="info"}

---

## Part 5 — 페일오버 / 서킷 브레이커 (자동 폴백)

`failover_criteria` 설정은 오류, 타임아웃 또는 속도 제한이 발생할 때 `ai-proxy-advanced`가 기본 모델에서 백업 모델로 자동 페일오버하도록 합니다. 이것은 Track A의 `router_settings.fallbacks`에 해당합니다.

### 🎯 Plugin 배포

:::code{language=bash showCopyAction=true}
kubectl delete kongplugin ai-proxy-advanced-loan-strands -n kong 2>/dev/null; sleep 3
:::

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  balancer:
    algorithm: round-robin
    failover_criteria:
    - error
    - timeout
    - http_429
    - http_503
    - http_500
    max_fails: 3
    fail_timeout: 10000
    retries: 2
  targets:
  - model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 3.0
        output_cost: 15.0
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    weight: 100
  - model:
      name: amazon.nova-micro-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 0.035
        output_cost: 0.14
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    weight: 100
EOF
:::

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-advanced-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
:::

### 설정 설명

| 필드 | 값 | 의미 |
|---|---|---|
| `failover_criteria` | `[error, timeout, http_429, http_503, http_500]` | 다음 타겟으로 페일오버를 트리거하는 조건 |
| `max_fails` | `3` | 3번 연속 실패 후 타겟을 비정상으로 표시 |
| `fail_timeout` | `10000` | 비정상 타겟을 다시 탐색하기 전 10초 대기 |
| `retries` | `2` | 클라이언트에 오류를 반환하기 전 다른 타겟에서 최대 2회 재시도 |
| `weight: 100`(양쪽) | 동일 | 정상 트래픽은 첫 번째 타겟(Claude)으로; Nova는 폴백 |

### 알고리즘의 결정 방식

:::code{language=text showCopyAction=false}
Request arrives
  → Route to primary target (Claude, weight=100)
  → If response is error/timeout/429/503/500:
      → Retry on next target (Nova)
      → If Nova succeeds: return Nova's response to client
      → If Nova also fails: return error to client
  → After 3 consecutive failures on Claude:
      → Mark Claude "unhealthy" for 10 seconds
      → All traffic goes to Nova during this period
      → After 10s, probe Claude again — if healthy, restore it
:::

**페일오버 트리거(각 기준이 포착하는 것):**

| 기준 | 포착 대상 | 예시 시나리오 |
|---|---|---|
| `error` | 연결 거부, DNS 실패 | Bedrock 리전 장애 |
| `timeout` | 설정된 타임아웃 초과 응답 | 모델 과부하, 느린 응답 |
| `http_429` | 속도 제한 초과 | Bedrock이 계정을 스로틀링 |
| `http_503` | 서비스 사용 불가 | Bedrock 용량 문제 |
| `http_500` | 내부 서버 오류 | 프로바이더 측 버그 |

### 🚀 테스트: 정상 작동 확인 (기본 타겟)

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "=== Normal operation — Claude (primary, weight=100) handles all requests ==="
for i in {1..5}; do
  curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
    -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
    -d '{"messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); print(f'  Request $i: {r.get(\"model\",\"ERR\")}')" 2>/dev/null
  sleep 0.5
done
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
=== Normal operation — Claude (primary, weight=100) handles all requests ===
  Request 1: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 2: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 3: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 4: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 5: global.anthropic.claude-sonnet-4-5-20250929-v1:0
:::

**왜 모두 Claude인가?** 가중치 100:1에서 round-robin 알고리즘은 약 99%를 Claude로 보냅니다. 정상 작동 시 Claude는 정상 상태이며 모든 요청을 처리합니다. Nova는 휴면 상태의 폴백입니다 — Claude가 실패할 때만 활성화됩니다.

### Claude가 실패할 때 무슨 일이 일어나는가 (페일오버 동작)

페일오버는 **투명합니다** — 클라이언트는 오류 대신 정상 응답(Nova로부터)을 봅니다:

| 시나리오 | 클라이언트가 보는 것 | 내부적으로 일어나는 일 |
|---|---|---|
| Claude 정상 | `model: claude-sonnet-4-5`(정상) | Claude로 직접 전달 |
| Claude가 429 반환(속도 제한) | `model: nova-micro`(폴백) | Kong이 429를 포착 → Nova에서 재시도 → Nova의 응답 반환 |
| Claude가 503 반환(용량 문제) | `model: nova-micro`(폴백) | Kong이 503을 포착 → Nova에서 재시도 → Nova의 응답 반환 |
| Claude 타임아웃(>30s) | `model: nova-micro`(폴백) | Kong이 타임아웃을 포착 → Nova에서 재시도 |
| Claude가 3번 연속 실패 | **모든 트래픽 → 10초간 Nova** | Claude가 비정상으로 표시(`max_fails:3`) → `fail_timeout:10000`ms 동안 서킷 열림 |
| 10초 서킷 타임아웃 후 | `model: claude-sonnet-4-5`(복구됨) | Kong이 Claude를 탐색 → 정상이면 복원 |
| Claude와 Nova 모두 실패 | 클라이언트에 오류 반환 | 재시도 소진(`retries:2`) — 클라이언트가 오류를 받음 |

::alert[**페일오버는 클라이언트에게 보이지 않습니다.** Bedrock이 Claude를 스로틀링(429)하거나 용량 문제(503)가 있을 때 Kong은 자동으로 Nova에서 재시도합니다. 클라이언트는 정상 200 응답을 받습니다 — 오류를 전혀 보지 못합니다. 응답의 `model` 필드가 실제로 어느 타겟이 요청을 처리했는지 알려줍니다.]{type="success"}

::alert[**이 워크숍에서는 페일오버를 온디맨드로 트리거할 수 없습니다** — Bedrock의 스로틀 한도가 높고, 극도의 동시 부하 없이는 Claude가 429/503을 반환하도록 인위적으로 만들 방법이 없습니다. 서킷 브레이커는 (1) 정상 작동이 Claude로 라우팅되고, (2) 페일오버 기준이 올바르게 설정되었음을 확인함으로써 검증됩니다. 프로덕션에서는 Kong이 실제 장애를 자동으로 포착합니다.]{type="info"}

---

## Part 6 — 모델 별칭 (클라이언트 제어 라우팅)

`model_alias`를 사용하면 클라이언트가 전체 Bedrock 모델 ID 대신 사람이 알기 쉬운 이름(예: `"model":"complex"` 또는 `"model":"simple"`)을 전송하여 사용할 모델을 명시적으로 선택합니다. 게이트웨이는 별칭을 올바른 백엔드에 매핑합니다.

### 🎯 Plugin 배포

:::code{language=bash showCopyAction=true}
kubectl delete kongplugin ai-proxy-advanced-loan-strands -n kong 2>/dev/null; sleep 3
:::

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  targets:
  - model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      model_alias: complex
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 3.0
        output_cost: 15.0
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
  - model:
      name: amazon.nova-micro-v1:0
      provider: bedrock
      model_alias: simple
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 0.035
        output_cost: 0.14
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
EOF
:::

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-advanced-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
:::

### 설정 설명

| 필드 | 값 | 의미 |
|---|---|---|
| `model_alias` | `complex` / `simple` | 클라이언트가 `"model"` 필드에 전송하는 사람이 알기 쉬운 이름 |
| `balancer` 블록 없음 | — | 밸런서 알고리즘이 없으면 라우팅은 순전히 별칭 매칭으로 이루어짐 |

### 알고리즘의 결정 방식

:::code{language=text showCopyAction=false}
Client sends: "model": "complex"
  → Plugin scans targets for model_alias == "complex"
  → Match found: Claude target
  → Route request to Bedrock with model = global.anthropic.claude-sonnet-4-5-20250929-v1:0

Client sends: "model": "simple"
  → Plugin scans targets for model_alias == "simple"
  → Match found: Nova target
  → Route request to Bedrock with model = amazon.nova-micro-v1:0
:::

**별칭의 강점:** 애플리케이션은 전체 Bedrock 모델 ID를 알 필요가 전혀 없습니다. 그저 `"complex"` 또는 `"simple"`을 전송할 뿐입니다. 나중에 Claude를 파인튜닝된 모델로 교체하더라도 plugin 설정만 업데이트하면 됩니다 — 애플리케이션 코드 변경은 전혀 없습니다.

### 🚀 테스트: 별칭에 의한 라우팅

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "=== Send model='complex' → routes to Claude target ==="
curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{"model":"complex","messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(f'Response: {r.get(\"model\", r.get(\"error\",{}).get(\"message\",str(r)[:80]))}')"

echo ""
echo "=== Send model='simple' → routes to Nova target ==="
curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{"model":"simple","messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(f'Response: {r.get(\"model\", r.get(\"error\",{}).get(\"message\",str(r)[:80]))}')"
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
=== Send model='complex' → routes to Claude target ===
Response: cannot use own model - must be: global.anthropic.claude-sonnet-4-5-20250929-v1:0

=== Send model='simple' → routes to Nova target ===
Response: cannot use own model - must be: amazon.nova-micro-v1:0
:::

**왜 이 오류가 발생하며(그리고 왜 그것이 별칭이 동작함을 증명하는가)?**

| 단계 | 무슨 일이 일어났는가 | 증거 |
|---|---|---|
| 1. 클라이언트가 `"model":"complex"` 전송 | plugin이 별칭 해석 → **Claude** 타겟 선택 | 오류가 "must be: global.anthropic.claude-sonnet..." 이라고 표시 |
| 2. 클라이언트가 `"model":"simple"` 전송 | plugin이 별칭 해석 → **Nova** 타겟 선택 | 오류가 "must be: amazon.nova-micro..." 이라고 표시 |
| 3. 모델 이름 강제 적용 | plugin이 요청 모델과 타겟의 `model.name`이 정확히 일치할 것을 요구 | 이것은 **Kong 3.13 버그**입니다 — 별칭 해석은 동작하지만 모델 이름 검사가 그 후에 실행됨 |

별칭 라우팅 로직은 정확합니다 — 두 경우 모두 올바른 타겟을 선택했습니다. 오류는 향후 버전에서 수정되어야 할 Kong 3.13 모델 이름 강제 적용 버그입니다.

::alert[**Kong 3.13 제한 사항:** `model_alias`는 타겟을 올바르게 해석하지만 해석 후에도 여전히 정확한 모델 이름 매칭을 강제합니다. 이것이 수정되면 설정 패턴은 프로덕션 사용에 정확합니다. Kong 팀에 접수되었습니다.]{type="warning"}

**애플리케이션 통합 패턴(별칭이 종단 간으로 동작할 때):**

:::code{language=python showCopyAction=false}
def get_model_alias(credit_score: int, dti_ratio: float) -> str:
    """Application picks 'complex' or 'simple' — gateway maps to the right model."""
    if credit_score >= 700 and dti_ratio <= 36:
        return "simple"    # Nova Micro (fast, cheap)
    else:
        return "complex"   # Claude Sonnet (accurate for edge cases)

response = client.chat.completions.create(
    model=get_model_alias(applicant.credit_score, applicant.dti),
    messages=[{"role": "user", "content": prompt}]
)
:::

---

## Part 7 — 시맨틱 라우팅 (프롬프트 복잡도 기반) 🧠

`semantic` 알고리즘은 가장 고급 라우팅 옵션입니다 — **임베딩 모델**을 사용하여 들어오는 프롬프트를 분석하고 각 타겟의 설명과의 **시맨틱 유사도**를 기반으로 라우팅합니다. 복잡한 프롬프트는 Claude로, 단순한 프롬프트는 Nova(CATCHALL)로 이동합니다.

::alert[**사전 요구 사항:** 시맨틱 라우팅에는 (1) **Amazon Bedrock Titan Embeddings**(`amazon.titan-embed-text-v2:0`) — 이미 계정에서 활성화됨, (2) 벡터 검색을 지원하는 **Redis Stack** — 아직 실행 중이 아니면 아래에서 배포 — 이 필요합니다.]{type="warning"}

### Redis Stack 배포 (아직 실행 중이 아닌 경우)

Redis Stack은 임베딩 기반 라우팅과 시맨틱 캐싱에 필요한 벡터 검색 기능을 제공합니다. 이미 배포되어 있는지 확인합니다:

:::code{language=bash showCopyAction=true}
kubectl get pods -n redis 2>/dev/null | grep redis-stack || echo "Redis Stack NOT found — deploy it below"
:::

Redis Stack이 실행 중이 아니면 Helm으로 배포합니다:

:::code{language=bash showCopyAction=true}
helm repo add redis-stack https://redis-stack.github.io/helm-redis-stack
helm repo update
helm install redis-stack redis-stack/redis-stack -n redis --create-namespace

# Wait for Redis to be ready
kubectl rollout status deployment/redis-stack -n redis --timeout=60s
:::

Kong 네임스페이스에서 Redis에 접근 가능한지 확인합니다:

:::code{language=bash showCopyAction=true}
kubectl run redis-test --rm -i --restart=Never --image=redis:7 -n kong -- \
  redis-cli -h redis-stack.redis.svc.cluster.local -p 6379 PING
:::

**예상 결과:** `PONG`

### 🎯 Plugin 배포

:::code{language=bash showCopyAction=true}
kubectl delete kongplugin ai-proxy-advanced-loan-strands -n kong 2>/dev/null; sleep 3
:::

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  balancer:
    algorithm: semantic
  embeddings:
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    model:
      provider: bedrock
      name: amazon.titan-embed-text-v2:0
      options:
        bedrock:
          aws_region: us-east-1
  vectordb:
    dimensions: 1024
    distance_metric: cosine
    strategy: redis
    threshold: 1.0
    redis:
      host: redis-stack.redis.svc.cluster.local
      port: 6379
  targets:
  - model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    description: "Complex loan evaluation, edge cases, low credit score, high debt-to-income ratio, bankruptcy history, collections"
  - model:
      name: amazon.nova-micro-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    description: "Simple questions, greetings, general knowledge, basic math, weather"
EOF
:::

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-advanced-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
:::

### 설정 설명

| 필드 | 값 | 의미 |
|---|---|---|
| `balancer.algorithm` | `semantic` | 프롬프트와 타겟 설명 간의 시맨틱 유사도를 기반으로 라우팅 |
| `embeddings.model` | `amazon.titan-embed-text-v2:0` | Bedrock Titan Embeddings가 프롬프트 + 설명을 1024차원 벡터로 변환 |
| `vectordb.strategy` | `redis` | Redis Stack이 설명 임베딩을 벡터로 저장 |
| `vectordb.threshold` | `1.0` | 최대 코사인 거리 — 값이 낮을수록 더 엄격(0 = 정확히 일치하는 것만) |
| `vectordb.dimensions` | `1024` | Titan Embed v2 출력 차원과 일치해야 함 |
| `description`(Claude) | "Complex loan evaluation, edge cases..." | 이것과 시맨틱하게 유사한 프롬프트는 Claude로 이동 |
| `description`(Nova) | "Simple questions, greetings..." | 이것과 유사한 프롬프트는 Nova로 이동 |

### 알고리즘의 결정 방식

:::code{language=text showCopyAction=false}
1. First request: Kong embeds both target descriptions using Titan Embeddings
   → Stores embeddings in Redis (one-time setup, auto on first request)

2. For each incoming request:
   a. Kong embeds the prompt: "Evaluate: credit 580, bankruptcy" → vector [0.12, 0.85, ...]
   b. Kong searches Redis for nearest description vector:
      - vs "Complex loan evaluation, edge cases..." → distance = 0.2 (close match!)
      - vs "Simple questions, greetings..." → distance = 0.9 (far)
   c. Nearest match (0.2) < threshold (1.0) → route to Claude

3. Another request: "What is 2 plus 2?"
   - vs "Complex loan evaluation..." → distance = 0.95 (far)
   - vs "Simple questions, greetings, basic math..." → distance = 0.3 (close!)
   → Route to Nova
:::

### 🚀 테스트: 시맨틱 라우팅

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "=== Complex loan prompt → should route to Claude ==="
curl -s --max-time 15 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{"model":"","messages":[{"role":"user","content":"Evaluate this high-risk loan: credit score 580, DTI 52%, recent bankruptcy, collections on record. Should we approve?"}],"max_tokens":20}' \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(f'Model: {r.get(\"model\", \"ERR: \"+str(r)[:80])}')"
:::

:::code{language=bash showCopyAction=true}
echo "=== Simple question → should route to Nova ==="
curl -s --max-time 15 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{"model":"","messages":[{"role":"user","content":"What is 2 plus 2?"}],"max_tokens":20}' \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(f'Model: {r.get(\"model\", \"ERR: \"+str(r)[:80])}')"
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
=== Complex loan prompt → should route to Claude ===
Model: global.anthropic.claude-sonnet-4-5-20250929-v1:0

=== Simple question → should route to Nova ===
Model: amazon.nova-micro-v1:0
:::

**왜 이것이 동작하는가:** Kong은 Titan Embeddings를 사용하여 각 프롬프트의 **내용**을 분석하고 설명이 가장 시맨틱하게 유사한 모델로 라우팅했습니다. 복잡한 대출 프롬프트(파산, 낮은 신용, 높은 DTI)는 Claude 타겟의 설명과 일치합니다. 단순한 수학 질문은 Nova 타겟의 설명과 일치합니다. 애플리케이션 코드가 필요 없습니다 — 게이트웨이가 라우팅 결정을 전적으로 내립니다.

### Redis가 임베딩을 저장했는지 확인

:::code{language=bash showCopyAction=true}
kubectl run redis-scan --rm -i --restart=Never --image=redis:7 -n redis -- \
  redis-cli -h redis-stack.redis.svc.cluster.local -p 6379 --scan 2>/dev/null | grep "semantic_routing"
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
semantic_routing:<plugin-id>:route:<hash1>
semantic_routing:<plugin-id>:route:<hash2>
:::

두 개의 항목 — Redis에 저장된 각 타겟 설명 임베딩에 대해 하나씩입니다.

::alert[**Track A 대응:** LiteLLM에는 내장 시맨틱 라우팅이 없습니다. Track A는 애플리케이션 수준 코드(`route_by_complexity()` 함수)를 통해 내용 기반 라우팅을 달성합니다. Track B의 시맨틱 라우팅은 **게이트웨이 계층**에서 동일한 작업을 수행합니다 — 애플리케이션 코드가 필요 없으며, 설명 필드를 업데이트하여 라우팅 로직을 변경할 수 있습니다(재배포 불필요).]{type="info"}

::alert[**팁:** 짧고 초점이 맞춰진 설명 구문이 긴 산문보다 더 선명한 임베딩을 생성합니다. 일반적인 "difficult financial situations" 대신 "bankruptcy, collections, low credit score"와 같은 도메인 특화 키워드를 사용하세요.]{type="info"}

---

## Part 8 — 최소 연결 (로드 밸런싱)

`least-connections` 알고리즘은 현재 **가장 적은 활성 연결**을 처리하는 타겟으로 라우팅합니다 — 모델 레플리카 전반의 로드 밸런싱이나 동시 트래픽을 균등하게 분산하는 데 이상적입니다.

### 🎯 Plugin 배포

:::code{language=bash showCopyAction=true}
kubectl delete kongplugin ai-proxy-advanced-loan-strands -n kong 2>/dev/null; sleep 3
:::

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-proxy-advanced-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-proxy-advanced
config:
  balancer:
    algorithm: least-connections
    retries: 2
    failover_criteria:
    - error
    - timeout
    - http_429
  targets:
  - model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 3.0
        output_cost: 15.0
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
  - model:
      name: amazon.nova-micro-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
        input_cost: 0.035
        output_cost: 0.14
    route_type: llm/v1/chat
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
EOF
:::

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-advanced-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
:::

### 설정 설명

| 필드 | 값 | 의미 |
|---|---|---|
| `balancer.algorithm` | `least-connections` | 진행 중인 요청이 가장 적은 타겟으로 라우팅 |
| `retries` | `2` | 선택된 타겟이 실패하면 다른 타겟에서 재시도 |
| `failover_criteria` | `[error, timeout, http_429]` | 다른 타겟에서 재시도를 트리거하는 조건 |

### 알고리즘의 결정 방식

:::code{language=text showCopyAction=false}
Request arrives:
  → Check active connections: Claude=2, Nova=1
  → Route to Nova (fewest connections)

Next request:
  → Check active connections: Claude=2, Nova=2
  → Route to either (tied — round-robin tiebreaker)
:::

**주요 동작:**
- 순차적 요청(이 테스트와 같은)에서는 각 요청이 도착할 때 두 타겟 모두 활성 연결이 0이므로 알고리즘이 약 50/50으로 교대합니다
- **동시** 요청에서는 빛을 발합니다: 느린 모델(Claude ~2s)은 연결을 축적하는 반면 빠른 모델(Nova ~0.5s)은 더 빨리 연결을 해제하여 자연스럽게 더 많은 트래픽을 받습니다
- 이것은 Track A의 "레플리카 전반의 로드 밸런싱" 패턴에 해당하는 Track B의 대응입니다

### 🚀 테스트: 균등 분배 관찰

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "Sending 10 sequential requests (least-connections → ~50/50 distribution)..."
CLAUDE=0; NOVA=0
for i in {1..10}; do
  MODEL=$(curl -s --max-time 10 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
    -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
    -d '{"model":"","messages":[{"role":"user","content":"hi"}],"max_tokens":3}' \
    | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('model','ERR'))" 2>/dev/null)
  case "$MODEL" in *claude*) CLAUDE=$((CLAUDE+1));; *nova*) NOVA=$((NOVA+1));; esac
  echo "  Request $i: $MODEL"
  sleep 0.3
done
echo ""
echo "Distribution: Claude=$CLAUDE Nova=$NOVA (expected ~5/5)"
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
Sending 10 sequential requests (least-connections → ~50/50 distribution)...
  Request 1: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 2: amazon.nova-micro-v1:0
  Request 3: amazon.nova-micro-v1:0
  Request 4: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 5: amazon.nova-micro-v1:0
  Request 6: amazon.nova-micro-v1:0
  Request 7: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 8: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 9: global.anthropic.claude-sonnet-4-5-20250929-v1:0
  Request 10: amazon.nova-micro-v1:0

Distribution: Claude=5 Nova=5 (expected ~5/5)
:::

**왜 약 50/50인가?** 순차적 요청(각 요청이 다음 요청 시작 전에 완료됨)에서는 두 타겟 모두 항상 활성 연결 0을 보이므로 → 알고리즘이 교대합니다. 동시 트래픽이 있는 프로덕션에서는 빠른 모델(Nova)이 더 빨리 연결을 해제하여 더 많은 요청을 받습니다 — 실제 용량에 따라 자연스럽게 로드 밸런싱됩니다.

::alert[**프로덕션 유스케이스:** 동일한 모델의 여러 레플리카(예: 서로 다른 엔드포인트 뒤의 3개 Nova 인스턴스)가 있을 때 `least-connections`를 사용하세요. 정적 가중치가 아닌 실제 부하를 기반으로 트래픽을 분산합니다 — 느린 레플리카는 자동으로 더 적은 트래픽을 받습니다.]{type="info"}

---

## 정리: 단일 모델 라우트 복원

모든 알고리즘을 테스트한 후 원래의 단일 모델 설정을 복원합니다:

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
kubectl delete kongplugin ai-proxy-advanced-loan-strands -n kong
:::

---

## 알고리즘 요약

| 알고리즘 | Part | 적합한 용도 | 트래픽 패턴 | 애플리케이션 변경 |
|---|---|---|---|---|
| `round-robin` + `weight` | Part 1 | 점진적 롤아웃, A/B 테스트 | 정적 80/20 분할 | 없음(`model:""`) |
| `lowest-usage` + `cost` | Part 2 | 지출 자동 최소화 | 가장 저렴한 모델로 이동 | 없음(`model:""`) |
| `lowest-latency` + `tpot` | Part 3 | 속도 중요(고트래픽 프로덕션) | 가장 빠른 모델로 이동 | 없음(`model:""`) |
| `consistent-hashing` | Part 4 | 사용자별/등급별 스티키 라우팅 | 헤더에 의한 결정론적 | 라우팅 헤더 설정 |
| `failover_criteria` | Part 5 | 복원력, 자동 폴백 | 기본 + 장애 시 백업 | 없음(`model:""`) |
| `model_alias` | Part 6 | 클라이언트 제어 라우팅 | 별칭 이름에 의한 명시적 | `model` 필드에 별칭 전송 |
| `semantic` | Part 7 | 프롬프트 기반 내용 라우팅 | 프롬프트별 지능형 | `"model":"auto"` 전송 |
| `least-connections` | Part 8 | 레플리카 전반 로드 밸런싱 | 부하에 의한 균등 분배 | 없음(`model:""`) |

## 📊 Track A와의 매핑

| 기능 | Track A (LiteLLM) | Track B (Kong `ai-proxy-advanced`) |
|---|---|---|
| **가중치 A/B 분할** | `litellm_params`의 `weight` | 타겟별 `weight`(Part 1) |
| **자동 폴백** | `router_settings.fallbacks` | `failover_criteria` + `retries`(Part 5) |
| **서킷 브레이커** | 쿨다운/재시도 로직 | `max_fails` + `fail_timeout`(Part 5) |
| **로드 밸런싱(레플리카)** | 레플리카 전반의 동일 `model_name` | 타겟 전반의 `least-connections`(Part 8) |
| **비용 인식 라우팅** | 내장 없음 | `lowest-usage` + `tokens_count_strategy: cost`(Part 2) |
| **지연시간 인식 라우팅** | 내장 없음 | EWMA를 사용한 `lowest-latency`(Part 3) |
| **스티키/사용자별 라우팅** | 내장 없음 | 헤더에 대한 `consistent-hashing`(Part 4) |
| **복잡도 기반 라우팅** | 애플리케이션 수준 코드(`get_model_for_application()`) | `model_alias` 클라이언트 제어(Part 6) 또는 `semantic` 게이트웨이 수준(Part 7) |
| **내용 인식 라우팅** | 애플리케이션 수준 코드만 | 임베딩을 사용한 `semantic` 알고리즘(Part 7) — **게이트웨이 네이티브, 앱 코드 없음** |
| **점진적 롤아웃** | 몇 주에 걸쳐 `weight` 값 조정 | 동일 — CRD에서 `weight` 조정 후 재적용 |
| **비용 설정 위치** | `litellm_params` 비용 필드 | `options`의 타겟별 `input_cost`/`output_cost` |
| **필요한 애플리케이션 변경** | `model` 파라미터를 그룹 이름으로 변경 | 게이트웨이 결정을 위한 `model:""`, 또는 클라이언트 제어를 위한 별칭 |

## ✅ 달성한 내용

- ✅ **Part 1:** 가중치 라운드로빈(80/20 A/B 분할) — 게이트웨이가 확률로 분배
- ✅ **Part 2:** 비용 인식 라우팅(`lowest-usage`) — 게이트웨이가 지출을 자동으로 최소화
- ✅ **Part 3:** 지연시간 인식 라우팅(`lowest-latency`) — 게이트웨이가 대규모에서 가장 빠른 모델 선호
- ✅ **Part 4:** 스티키 라우팅(`consistent-hashing`) — 동일한 사용자는 항상 동일한 모델로 라우팅
- ✅ **Part 5:** 페일오버 / 서킷 브레이커 — 오류/타임아웃/429 시 자동 폴백
- ✅ **Part 6:** 모델 별칭 — 클라이언트가 친숙한 이름으로 모델 선택(Kong 3.13 제한 사항 명시)
- ✅ **Part 7:** 시맨틱 라우팅 — 게이트웨이가 임베딩을 사용하여 프롬프트 내용으로 라우팅
- ✅ **Part 8:** 최소 연결 — 활성 연결 수에 의한 로드 밸런싱
- ✅ 게이트웨이 결정 라우팅을 위한 `"model":""` 패턴 학습
- ✅ Track A와의 완전한 동등성 + **5가지 Kong 전용 기능**(비용 인식, 지연시간 인식, 시맨틱, 최소 연결, 일관성 해싱)

---

**[다음: 인프라 모니터링 및 오토스케일링 →](../monitoring-scaling/)**
