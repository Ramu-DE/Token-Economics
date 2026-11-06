---
title: "게이트웨이에서의 프롬프트 엔지니어링"
weight: 25
---

# 🛡️ 게이트웨이에서의 프롬프트 엔지니어링 (Kong AI 플러그인)

Kong은 애플리케이션 코드를 수정하지 않고도 **게이트웨이 계층에서** 프롬프트를 엔지니어링, 템플릿화, 변환할 수 있는 세 가지 플러그인을 제공합니다. 이는 프로덕션에서 강력합니다. 관리자는 프롬프트 동작을 중앙에서 제어하고 개발자는 비즈니스 로직에 집중할 수 있습니다.

| 플러그인 | 목적 |
|---|---|
| **AI Prompt Decorator** | 모든 요청의 시작 또는 끝에 system/assistant 메시지 주입 |
| **AI Prompt Template** | `{{variables}}`가 포함된 재사용 가능한 프롬프트 템플릿 정의 — 클라이언트가 빈칸을 채움 |
| **AI Request/Response Transformer** | LLM을 사용하여 요청/응답 페이로드를 실시간으로 변환 |

---

## Part 1 — AI Prompt Decorator (System 프롬프트 주입)

`ai-prompt-decorator` 플러그인은 컨슈머의 채팅 기록의 **시작 또는 끝** 에 메시지를 투명하게 추가합니다. 다음 용도로 사용합니다:
- system 프롬프트 주입 (예: "항상 스페인어로 응답하시오")
- 앱을 변경하지 않고 안전 가드레일 추가
- 모든 요청에 있어야 하는 컨텍스트를 앞에 추가

### 🎯 플러그인 배포

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-prompt-decorator-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-prompt-decorator
config:
  prompts:
    prepend:
    - role: system
      content: "You are a strict credit underwriting AI. Always respond with a structured assessment including APPROVED, CONDITIONAL, or REJECTED decision. Never approve loans with DTI above 43%."
EOF
:::

라우트에 연결합니다 (기존 플러그인과 함께):

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-loan-strands,key-auth-loan-strands,prometheus-metrics,ai-prompt-decorator-loan-strands --overwrite
:::

### 작동 방식

:::code{language=text showCopyAction=false}
Client sends:
  messages: [{"role":"user","content":"Should I approve this loan?"}]

Kong PREPENDS the system message:
  messages: [
    {"role":"system","content":"You are a strict credit underwriting AI..."},  ← injected by decorator
    {"role":"user","content":"Should I approve this loan?"}                    ← original
  ]

Bedrock receives both messages — client never sees the injection.
:::

### 🚀 테스트: System 프롬프트 주입 확인

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "=== Without mentioning format, the decorator forces structured output ==="
curl -s --max-time 15 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{"model":"global.anthropic.claude-sonnet-4-5-20250929-v1:0","messages":[{"role":"user","content":"Applicant: income $50k, DTI 48%, credit 650. Decision?"}],"max_tokens":100}' \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('choices',[{}])[0].get('message',{}).get('content','ERR: '+str(r)[:200])[:300])"
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
# CREDIT UNDERWRITING ASSESSMENT

## DECISION: **REJECTED**

---

## Key Metrics Analysis:

| Factor | Value | Status |
|--------|-------|--------|
| Annual Income | $50,000 | ✓ Acceptable |
| Debt-to-Income Ratio | 48% | ✗ EXCEEDS 43% MAXIMUM |
| Credit Score | 650 | ⚠ Below preferred threshold |
:::

**왜 REJECTED인가?** decorator가 "Never approve loans with DTI above 43%"라고 명시한 system 프롬프트를 주입했습니다. 모델은 그 규칙을 따라 48% DTI 신청을 거부했습니다 — 클라이언트가 이 제약을 전혀 언급하지 않았는데도 말입니다. decorator는 게이트웨이에서 비즈니스 규칙을 투명하게 적용합니다.

### 정리

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
kubectl delete kongplugin ai-prompt-decorator-loan-strands -n kong
:::

---

## Part 2 — AI Prompt Template (재사용 가능한 프롬프트 API)

`ai-prompt-template` 플러그인을 사용하면 관리자가 `{{variable}}` 플레이스홀더가 있는 프롬프트 템플릿을 정의할 수 있습니다. 클라이언트는 변수만 전송하고 — Kong이 전체 프롬프트를 조립합니다. 이는:
- 모든 컨슈머에 걸쳐 프롬프트를 표준화
- 프롬프트 주입 방지 (입력이 정제됨)
- LLM을 목적에 특화된 API로 제시 (예: "신용 결정 API")

### 🎯 플러그인 배포

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-prompt-template-loan-strands
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-prompt-template
config:
  allow_untemplated_requests: false
  templates:
  - name: credit-decision
    template: |-
      {
        "messages": [
          {
            "role": "system",
            "content": "You are a credit underwriting AI. Provide a one-paragraph decision: APPROVED, CONDITIONAL, or REJECTED with reasoning."
          },
          {
            "role": "user",
            "content": "Evaluate this loan application: Applicant {{name}}, annual income {{income}}, credit score {{credit_score}}, loan amount {{loan_amount}}, purpose: {{purpose}}. Give your credit decision."
          }
        ]
      }
EOF
:::

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-loan-strands,key-auth-loan-strands,prometheus-metrics,ai-prompt-template-loan-strands --overwrite
:::

### 작동 방식

:::code{language=text showCopyAction=false}
Client sends (only variables — no prompt engineering needed):
{
  "messages": "{template://credit-decision}",
  "properties": {
    "name": "Sarah Johnson",
    "income": "$92,000",
    "credit_score": "780",
    "loan_amount": "$15,000",
    "purpose": "Debt Consolidation"
  }
}

Kong assembles the full prompt from the template + variables → sends to Bedrock.
Client never writes the prompt — just fills in the blanks.
:::

### 🚀 테스트: 템플릿 API 호출

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "=== Call credit-decision template with variables ==="
curl -s --max-time 15 -X POST "$KONG_PROXY/loan-strands/v1/chat/completions" \
  -H "Content-Type: application/json" -H "apikey: loan-strands-key-123" \
  -d '{
    "model":"global.anthropic.claude-sonnet-4-5-20250929-v1:0",
    "messages": "{template://credit-decision}",
    "properties": {
      "name": "Sarah Johnson",
      "income": "$92,000",
      "credit_score": "780",
      "loan_amount": "$15,000",
      "purpose": "Debt Consolidation"
    }
  }' | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('choices',[{}])[0].get('message',{}).get('content','ERR: '+str(r)[:200])[:300])"
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
**DECISION: APPROVED**

Sarah Johnson's loan application for $15,000 demonstrates strong creditworthiness
and represents a low-risk lending opportunity. With an excellent credit score of 780,
she has proven a solid track record of responsible credit management and timely
payments. Her annual income of $92,000 provides more than adequate capacity to
service this debt, as the loan amount represents only approximately 16% of her gross
annual income, well within conservative debt-to-income parameters. The stated purpose
of debt consolidation is financially prudent.
:::

**왜 APPROVED인가?** 클라이언트는 5개의 변수(name, income, credit_score, loan_amount, purpose)만 전송했습니다 — Kong이 `credit-decision` 템플릿에서 전체 프롬프트를 조립하여 Bedrock으로 전송했습니다. 모델은 템플릿의 system 프롬프트 형식을 따라 구조화된 결정을 생성했습니다.

::alert[**프로덕션 유스케이스:** 서로 다른 유스케이스에 대해 여러 템플릿(`credit-decision`, `risk-assessment`, `fraud-check`)을 생성하고 이를 목적에 특화된 API로 노출하십시오. 개발자는 변수와 함께 `{template://credit-decision}`을 호출합니다 — 기저의 프롬프트 엔지니어링을 알 필요가 전혀 없습니다.]{type="info"}

### 정리

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-loan-strands,key-auth-loan-strands,prometheus-metrics --overwrite
kubectl delete kongplugin ai-prompt-template-loan-strands -n kong
:::

---

## Part 3 — AI Request/Response Transformer (LLM 기반 페이로드 변환)

`ai-request-transformer`와 `ai-response-transformer` 플러그인은 LLM을 사용하여 **요청/응답 페이로드를 실시간으로 변환** 합니다 — 업스트림 서비스에 도달하기 전 또는 클라이언트로 반환되기 전에 말입니다.

| 플러그인 | 실행 시점 | 유스케이스 |
|---|---|---|
| `ai-request-transformer` | 업스트림으로 프록시하기 전 | 요청 데이터 보강, 누락된 필드 추가, 형식 정규화 |
| `ai-response-transformer` | 업스트림 응답 후, 클라이언트에 도달하기 전 | 요약, 번역, 구조화된 데이터 추출 |

::alert[**이 플러그인들은 실제 업스트림 서비스(예: httpbin)가 있는 별도의 라우트를 사용합니다.** LLM(Bedrock)을 사용하여 페이로드를 변환한 다음, 변환된 본문을 업스트림으로 전달합니다. LLM 호출은 `config.llm`에서 구성되며 — ai-proxy와 동일한 형식입니다.]{type="info"}

### 🎯 httpbin + Transformer 플러그인 배포

먼저, 테스트 업스트림으로 httpbin을 배포합니다 (수신한 것을 그대로 되돌려 보냄):

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
  namespace: kong
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
  template:
    metadata:
      labels:
        app: httpbin
    spec:
      containers:
      - name: httpbin
        image: kong/httpbin:0.1.0
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  namespace: kong
spec:
  selector:
    app: httpbin
  ports:
  - port: 80
    targetPort: 80
EOF
:::

request transformer가 연결된 httpbin을 가리키는 Kong 라우트를 생성합니다:

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1alpha1
kind: KongService
metadata:
  name: httpbin-service
  namespace: kong
spec:
  name: httpbin-service
  host: httpbin.kong.svc.cluster.local
  port: 80
  controlPlaneRef:
    type: konnectNamespacedRef
    konnectNamespacedRef:
      name: kong-aws
---
apiVersion: configuration.konghq.com/v1alpha1
kind: KongRoute
metadata:
  name: httpbin-route
  namespace: kong
  annotations:
    konghq.com/plugins: ai-request-transformer-demo,ai-response-transformer-demo
spec:
  name: httpbin-route
  paths:
  - /httpbin-route
  protocols:
  - http
  - https
  serviceRef:
    type: namespacedRef
    namespacedRef:
      name: httpbin-service
---
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-request-transformer-demo
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-request-transformer
config:
  prompt: "In my JSON message, anywhere there is a JSON tag for a 'city', also add a 'country' tag with the name of the country in which the city resides. Return me only the JSON message, no extra text."
  llm:
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    route_type: llm/v1/chat
    model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
---
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-response-transformer-demo
  namespace: kong
  annotations:
    kubernetes.io/ingress.class: kong
plugin: ai-response-transformer
config:
  prompt: "For any city name in the JSON, add its approximate current temperature in Celsius in brackets next to it. Reply with the JSON result only, no markdown."
  llm:
    auth:
      header_name: x-dummy
      header_value: unused
      allow_override: false
    route_type: llm/v1/chat
    model:
      name: global.anthropic.claude-sonnet-4-5-20250929-v1:0
      provider: bedrock
      options:
        bedrock:
          aws_region: us-east-1
EOF
:::

### 작동 방식

:::code{language=text showCopyAction=false}
Request flow (ai-request-transformer):
  1. Client sends: {"user": {"name": "Kong User", "city": "Tokyo"}}
  2. Kong sends body to Claude with prompt: "add a 'country' for any 'city'"
  3. Claude returns: {"user": {"name": "Kong User", "city": "Tokyo", "country": "Japan"}}
  4. Kong forwards TRANSFORMED body to httpbin

Response flow (ai-response-transformer):
  5. httpbin responds (echoes the enriched body)
  6. Kong sends the response body to Claude with prompt: "add temperature for any city"
  7. Claude returns: {"user": {"name": "Kong User", "city": "Tokyo [25°C]", "country": "Japan"}}
  8. Kong returns the TRANSFORMED response to client
:::

### 🚀 테스트: AI가 요청과 응답을 모두 변환

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

echo "=== Request Transformer: AI adds 'country' field ==="
echo "=== Response Transformer: AI adds temperature to city ==="
curl -s --max-time 45 -X POST "$KONG_PROXY/httpbin-route/post" \
  -H "Content-Type: application/json" \
  -d '{"user":{"name":"Kong User","city":"Tokyo"}}' \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print('Final response to client:'); print(r.get('data', str(r)[:200]))"
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
=== Request Transformer: AI adds 'country' field ===
=== Response Transformer: AI adds temperature to city ===
Final response to client:
{"user":{"name":"Kong User","city":"Tokyo [25°C]","country":"Japan"}}
:::

**순서대로 일어난 일:**

| 단계 | 컴포넌트 | 동작 |
|---|---|---|
| 1 | 클라이언트 | `{"city":"Tokyo"}` 전송 — country 없음, temperature 없음 |
| 2 | **ai-request-transformer** | Claude 호출 → `"country":"Japan"` 추가 → 보강된 본문을 업스트림으로 전달 |
| 3 | httpbin (업스트림) | 보강된 요청 본문을 그대로 되돌려 보냄 |
| 4 | **ai-response-transformer** | Claude 호출 → temperature `"Tokyo [25°C]"` 추가 → 클라이언트로 반환 |
| 5 | 클라이언트 | country와 temperature가 모두 포함된 완전히 보강된 응답 수신 |

클라이언트는 단순한 `{city: Tokyo}`를 전송하고 `{city: "Tokyo [25°C]", country: "Japan"}`을 되돌려 받았습니다 — 두 변환 모두 두 개의 별도 LLM 호출을 사용하여 게이트웨이에서 투명하게 일어났습니다.

::alert[**대출 신청을 위한 프로덕션 유스케이스:** (1) **Request transformer:** 신청자 데이터를 고용주 확인으로 자동 보강하고, 주소를 정규화하고, DTI 계산을 추가합니다. (2) **Response transformer:** 신용 결정을 신청자의 언어로 번역하고, 긴 평가를 모바일용으로 요약하고, 응답에서 PII를 삭제합니다.]{type="info"}

### 정리

:::code{language=bash showCopyAction=true}
kubectl delete kongplugin ai-request-transformer-demo ai-response-transformer-demo -n kong
kubectl delete kongroute httpbin-route -n kong
kubectl delete kongservice httpbin-service -n kong
kubectl delete deploy httpbin -n kong
kubectl delete svc httpbin -n kong
:::

---

## 📊 Track A와의 매핑

| 기능 | Track A (LiteLLM) | Track B (Kong) |
|---|---|---|
| System 프롬프트 주입 | 애플리케이션 코드만 | `ai-prompt-decorator` — 게이트웨이 레벨, 코드 변경 없음 |
| 프롬프트 템플릿 | 애플리케이션 코드만 | `ai-prompt-template` — LLM을 목적에 특화된 API로 노출 |
| 요청/응답 변환 | 애플리케이션 코드만 | `ai-request-transformer` / `ai-response-transformer` — LLM 기반, 게이트웨이 레벨 |
| 프롬프트 주입 방지 | 애플리케이션 코드만 | `ai-prompt-template`이 입력을 정제 + `ai-prompt-guard` (B5) |

## ✅ 완료한 작업

- ✅ `ai-prompt-decorator`로 게이트웨이에서 system 프롬프트를 주입 — 앱 코드는 그대로
- ✅ `ai-prompt-template`로 재사용 가능한 신용 결정 템플릿 API 생성
- ✅ 게이트웨이에서의 LLM 기반 요청/응답 변환 학습
- ✅ 각 플러그인을 언제 사용할지 이해 (decorator = 항상 켜져 있는 규칙, template = 표준화된 API, transformer = 페이로드 보강)

---

**[다음: Arize AX를 활용한 관측성 →](../observability-arize/)**
