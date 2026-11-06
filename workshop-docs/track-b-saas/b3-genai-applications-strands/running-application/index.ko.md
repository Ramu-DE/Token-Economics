---
title: "애플리케이션 배포 및 테스트"
weight: 30
---

# Loan Buddy (Strands) 배포

이 섹션에서는 Strands Loan Buddy를 배포하고 John Michael Doe의 대출 신청서가 처리되는 과정을 확인합니다: **Strands 에이전트 → Kong `/loan-strands` → Bedrock** 경로로 요청이 흐르고, 3개 MCP 도구가 사용되며, **Arize AX** 에서 전체 추적이 이루어집니다.

![처리 준비가 된 John의 신청서](/static/images/shared/example1.png)

*처리를 기다리는 John의 신청서*

## 사전 요구사항 확인

배포하기 전에 모듈 B2의 대안 스택 플랫폼이 준비되었는지 확인합니다.

::alert[**대화형 zsh 참고:** 붙여넣기 시 `command not found: #` 또는 `unknown username` 오류가 발생하면 `setopt interactive_comments`를 한 번 실행하십시오 — zsh는 기본적으로 `#`을 주석으로 처리하지 않습니다. 아래 블록은 안전을 위해 인라인 주석을 사용하지 않습니다.]{type="info"}

Kong `/loan-strands` 라우트가 작동하는지 확인합니다 (키 없이 `401`이 나오면 key-auth가 활성 상태):

:::code{language=bash showCopyAction=true}
export KONG_PROXY="http://$(kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
curl -s -o /dev/null -w "kong /loan-strands no-key: HTTP %{http_code} (expect 401)\n" \
  -X POST "$KONG_PROXY/loan-strands" -H "Content-Type: application/json" -d '{"messages":[{"role":"user","content":"hi"}]}'
:::

Arize 자격 증명이 설치되어 있는지 확인합니다:

:::code{language=bash showCopyAction=true}
kubectl get secret arize-credentials -n arize
:::

3개의 MCP 도구 서버가 실행 중인지 확인합니다 ([Track A → 모듈 A2: Agentic AI 애플리케이션 구축](/track-a-opensource/a2-genai-applications/)에서 `workshop` 네임스페이스에 8000 포트로 배포됨):

:::code{language=bash showCopyAction=true}
kubectl get svc -n workshop | grep -E "mcp-image-processor|mcp-address-validator|mcp-employment-validator"
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
kong /loan-strands no-key: HTTP 401 (expect 401)

NAME                TYPE     DATA   AGE
arize-credentials   Opaque   3      *h

mcp-address-validator      ClusterIP   172.20.x.x   <none>   8000/TCP   *
mcp-employment-validator   ClusterIP   172.20.x.x   <none>   8000/TCP   *
mcp-image-processor        ClusterIP   172.20.x.x   <none>   8000/TCP   *
:::

::alert[**플랫폼 필요**: Kong 라우트는 `HTTP 401`을 반환해야 하고, `arize-credentials`가 존재해야 하며, 3개의 MCP 서비스가 `workshop` 네임스페이스에 있어야 합니다. Strands 에이전트는 [Track A → 모듈 A2](/track-a-opensource/a2-genai-applications/)에서 배포한 (해당 모듈의 `agentic-application-deployment.yaml`에 포함된) **동일한 MCP 서버** 를 재사용합니다 — MCP는 프레임워크에 독립적이므로 Strands 에이전트는 변경 없이 연결됩니다. 해당 서비스가 없다면 먼저 모듈 A2를 통해 배포하고, Kong 라우트 + Arize 자격 증명을 위해 [모듈 B2](/track-b-saas/b2-ai-gateway-observability/)를 완료했는지 확인하십시오.]{type="warning"}

## 단계 1: 구성 이해

배포하기 전에 Strands Loan Buddy가 대안 스택 플랫폼에 연결하는 데 필요한 구성을 확인합니다:

### 필요한 구성

1. **Kong 라우트 + 키** - 모듈 B2의 `KONG_BASE_URL` (`http://<kong-proxy-lb>/loan-strands`) + consumer 키 (`apikey: loan-strands-key-123`)
2. **Arize 자격 증명** - 에이전트에 마운트되는 `arize-credentials` Secret (Service 키 + Space ID)
3. **S3 Bucket** - 업로드된 대출 문서를 저장할 위치 (langfuse 버킷 재사용)
4. **AWS Region** - `us-east-1`
5. **Service Account with S3 Access** - **S3** 권한을 부여하는 EKS Pod Identity (Bedrock 아님 — Bedrock은 Kong을 통해 접근)

::alert[**기본 Loan Buddy와의 차이점:** 기본 에이전트의 Pod는 Bedrock IAM과 LiteLLM Virtual Key를 보유합니다. Strands 에이전트는 **S3 IAM만** 보유합니다 — 모든 Bedrock 접근은 Kong의 DataPlane (Bedrock IAM 보유)을 통해 이루어지며, Kong 인증은 `apikey` 헤더입니다. 앱에 보관하는 시크릿이 더 적습니다.]{type="info"}

## 단계 2: 자동 배포

### 배포 명령 실행

starter-kit CLI가 이미지를 빌드하고, Kong + Arize + MCP + S3를 연결하며, `strands-agents` 네임스페이스에 배포합니다:

:::code{language=bash showCopyAction=true}
cd /workshop
./cli strands-agents loan-buddy-agent install
:::

### 명령이 수행하는 작업

::::expand{header="내부 동작: 자동화되는 내용"}

1. **Terraform apply** — 에이전트의 ECR 저장소와 **S3 Pod Identity** 연결을 생성합니다 (에이전트는 업로드된 이미지를 저장하기 위해 S3가 필요하며, Bedrock 접근은 에이전트가 아닌 Kong을 통해 이루어집니다).
2. **`docker buildx build --push`** — 에이전트 이미지를 빌드하여 ECR에 푸시합니다.
3. **Kong 프록시 LB 호스트명 확인** → `KONG_BASE_URL=http://<lb>/loan-strands` 설정.
4. **`arize-credentials` Secret 복사** — `arize` 네임스페이스에서 `strands-agents`로 복사합니다.
5. **Deployment / Service / ServiceAccount 렌더링 + 적용**.

**이 명령은 위의 모든 작업을 한 단계로 수행합니다** — 프로덕션 CI/CD 파이프라인에서 사용되는 것과 동일한 자동화 패턴입니다.

::::

배포가 준비될 때까지 대기합니다:

:::code{language=bash showCopyAction=true}
kubectl rollout status deployment/loan-buddy-agent -n strands-agents
kubectl get pods -n strands-agents -l app=loan-buddy-agent
:::

### 예상 출력

:::code{language=text showCopyAction=false}
deployment "loan-buddy-agent" successfully rolled out
NAME                                READY   STATUS    RESTARTS   AGE
loan-buddy-agent-xxxxxxxxx-xxxxx    1/1     Running   0          1m
:::

## 단계 3: 에이전트 모니터링

### 실시간 모니터링 설정

두 번째 터미널을 열어 에이전트의 의사결정 과정을 확인합니다:

:::code{language=bash showCopyAction=true}
# 새 터미널에서 에이전트 로그 관찰
kubectl logs -f deployment/loan-buddy-agent -n strands-agents
:::

**이 터미널을 계속 표시해 두십시오** - 에이전트가 요청을 처리하는 동안 의사결정 과정이 로그에 나타납니다.

### 애플리케이션 노출

메인 터미널로 돌아갑니다:

:::code{language=bash showCopyAction=true}
kubectl port-forward service/loan-buddy-agent 8890:8080 -n strands-agents &

# use the same sample application image as Track A's Module A2
cd /workshop/workshops/eks-genai-workshop/static/code/module3/credit-validation/
:::

::alert[**API 전용 서비스**: Loan Buddy는 FastAPI REST 서버입니다 — 웹 UI가 없습니다. `http://localhost:8890/`는 `{"detail":"Not Found"}`를 반환하며 (정상), Swagger는 `http://localhost:8890/docs`에서 확인할 수 있습니다.]{type="info"}

::alert[**`port-forward`가 끊어지는 경우** (`lost connection to pod`): 클러스터 오토스케일러가 노드를 통합하면서 Pod를 재스케줄링할 수 있습니다. Pod가 다시 `Running` 상태가 될 때까지 기다린 후 (`kubectl get pods -n strands-agents -l app=loan-buddy-agent`) `port-forward`를 다시 실행하십시오. 단일 처리 요청은 **60-90초** 가 소요되므로 (여러 LLM 턴 + 7회 도구 호출) 응답이 반환될 때까지 연결을 유지하십시오.]{type="info"}

## 단계 4: John의 대출 신청서 처리

### John의 신청서 제출

:::code{language=bash showCopyAction=true}
curl -s --max-time 180 -X POST -F "image_file=@./example1.png" \
  http://localhost:8890/api/process_credit_application_with_upload \
  -o /tmp/loan_strands.json

python3 -c "import json;d=json.load(open('/tmp/loan_strands.json'));print('status:',d['status']);print();print(d.get('credit_assessment') or d.get('message',''))"
:::

::alert[**60-90초 정도 기다려 주십시오.** 에이전트는 Kong을 통한 여러 번의 LLM 왕복과 함께 다단계 루프 (추출 → 주소/고용/소유권 검증 → 결정)를 실행하므로 요청에 약 1분이 소요되는 것이 정상입니다. `--max-time 180`은 `curl`이 조기에 타임아웃되는 것을 방지합니다.]{type="info"}

::alert[**모델 참고:** Kong이 Claude Sonnet 4.5을 고정하고 주입하므로 (`allow_override: false`), 에이전트는 모델 이름을 전송하지 않습니다 — 게이트웨이가 모델과 비용을 제어합니다. 도구 호출은 Kong의 `ai-proxy`를 통해 작동합니다 (OpenAI `tools` ↔ Bedrock `toolConfig` 변환은 게이트웨이에서 처리됨).]{type="info"}

### 처리 과정 관찰

**로그 터미널에서 다음을 확인할 수 있습니다** — 에이전트가 이미지를 S3에 저장하고, MCP 도구를 로드한 다음, 루프를 주도합니다 — 데이터 추출 → 주소 검증 → 고용 검증 → 결정:

```
Starting credit application processing (Strands)...
Image stored in S3 with ID: 858ce1976c4d05cc...
Loading MCP tools...
Available tools: ['extract_credit_application_data', 'validate_document_authenticity', 'validate_address', 'perform_address_fraud_check', 'verify_address_ownership', 'validate_income_employment', 'check_employment_stability']
Processing with Strands agent...

Tool #1: extract_credit_application_data
Tool #2: validate_document_authenticity
Tool #3: validate_income_employment
Tool #4: check_employment_stability
Tool #5: validate_address
Tool #6: perform_address_fraud_check
Tool #7: verify_address_ownership

stop_reason=end_turn
metrics summary: {'total_cycles': 4, 'tool_usage': {... 'success_rate': 1.0 ...}}
```

::alert[에이전트가 루프를 스스로 주도합니다 (model-first): **모델이** 결정한 순서로 도구를 호출합니다 — 일반적으로 추출 + 진위 확인 먼저, 그다음 고용/소득, 그다음 주소 확인 — 약 4회의 이벤트 루프 사이클에 걸쳐 진행됩니다. `metrics summary` 줄은 도구별 호출 수와 성공률을 보고합니다 (정상 실행 시 모두 `1.0`).]{type="info"}

### 응답 검토

응답은 완전한 평가가 포함된 JSON 객체입니다:

:::code{language=json showCopyAction=false}
{
  "status": "COMPLETED",
  "image_id": "858ce1976c4d05cc...",
  "credit_assessment": "# COMPREHENSIVE CREDIT ASSESSMENT REPORT ... ## FINAL CREDIT DECISION: CONDITIONAL APPROVAL — verified employment at Tech Solutions Inc ($75,000/year), valid residential address, loan-to-income ratio 11.3%; conditions: obtain wet signature, verify full SSN, confirm housing status (document authenticity score 75/100).",
  "processing_note": "Strands agent via Kong->Bedrock; traced in Arize AX"
}
:::

::alert[**Claude Sonnet 4.5에서는 `CONDITIONAL APPROVAL`이 예상되는 결과입니다.** 모델은 견고한 기본 요건 (검증된 소득/고용, 낮은 11.3% 대출 대 소득 비율)을 근거로 승인하지만, 문서의 타이핑된 서명과 비표준 SSN 형식을 진위 우려 사항으로 표시합니다 — 그래서 무조건 승인 대신 조건을 첨부합니다. 이는 모델이 샘플 문서에 대해 올바르게 추론한 것이지 결함이 아닙니다. 전체 `credit_assessment`는 여러 섹션으로 구성된 markdown 리포트입니다 (신청자 정보, 검증 결과, 위험 분석, 결정, 다음 단계).]{type="info"}

## 단계 5: Arize AX에서 전체 워크플로우 탐색

### Arize AX 열기

1. [app.arize.com](https://app.arize.com)을 열고 Space 선택
2. **Tracing Projects** → 프로젝트 **`loan-strands`** 로 이동
3. **Traces** 탭 열기

### John의 신청서 추적 찾기

가장 최근 추적을 찾습니다 — **`invoke_agent Strands Agents`** 라는 이름의 최상위 **AGENT** 스팬 (Status `OK`, 지연시간 ~60초; 애플리케이션에서 설정한 `session.id = loan-buddy` 및 `tag.tags` 속성 포함). 클릭하여 스팬 트리를 펼칩니다.

### 추적에서 확인할 수 있는 것

**추적은** 올바른 OpenInference 스팬 종류가 포함된 전체 중첩 트리를 표시합니다:
- **AGENT** - `invoke_agent Strands Agents` (루트 실행), 사용자 프롬프트가 **Input**, 최종 평가가 **Output**
- **CHAIN** - `execute_event_loop_cycle` 스팬 (추론 사이클당 하나, 약 4개)
- **LLM** - `chat` 스팬, 각각 Kong을 통한 Bedrock (Claude Sonnet 4.5) 호출, 프롬프트/완성 **토큰 수** 포함
- **TOOL** - MCP 호출당 하나의 `execute_tool ...` 스팬 (7개 모두: 추출, 진위 확인, 주소/고용 검증, 사기 확인, 소유권), 도구 **인자** 가 Input, **JSON 결과** 가 Output
- **성능** - 스팬별 지연시간; LLM 스팬별 **토큰 사용량**; 전체 프롬프트/인자/결과는 **Input / Output** 탭에서 확인

이것은 Track A 모듈 A2의 Langfuse 추적에 해당하는 Arize AX 버전입니다 — **동일한 워크플로우, 다른 (SaaS) 관측성 백엔드** 입니다.

::alert[**스팬 내용이 채워지는 이유:** 에이전트는 `StrandsAgentsToOpenInferenceProcessor`와 함께 Arize를 등록하며, 이 프로세서가 Strands의 네이티브 OpenTelemetry `gen_ai.*` 스팬을 Arize가 렌더링하는 OpenInference 레이아웃으로 변환합니다 — 그 결과 AGENT/CHAIN/LLM/TOOL 스팬 종류와 채워진 Input/Output 탭을 볼 수 있습니다. 이 프로세서가 없으면 스팬은 도착하지만 스팬 종류가 없고 Input/Output이 비어 있습니다.]{type="info"}

## 단계 6: 다양한 시나리오 시도 (선택 사항)

:::code{language=bash showCopyAction=true}
# 다른 신청서 처리
curl -s --max-time 180 -X POST -F "image_file=@./example2.png" \
  http://localhost:8890/api/process_credit_application_with_upload -o /tmp/loan2.json
python3 -c "import json;d=json.load(open('/tmp/loan2.json'));print('status:',d['status']);print(d.get('credit_assessment') or d.get('message',''))"
:::

## 요약

### 수행한 작업

다음 작업을 완료했습니다:
- **배포** — Kubernetes에 Strands 기반 대출 처리 애플리케이션을 배포했습니다
- **처리** — 1분 이내에 John의 대출 신청서를 처리했습니다
- **모든 AI를 Kong으로 라우팅** — `ai-proxy` → Bedrock 경로를 사용하며, 앱에 LLM 키가 없습니다
- **Arize AX에서 전체 워크플로우 추적** — AGENT/LLM/TOOL 스팬을 확인했습니다
- **기본 스택과 동일한 3개 MCP 서버 재사용**

### 기본 Loan Buddy와의 비교

| | Track A · 모듈 A2 (LangChain) | Track B · 모듈 B3 (Strands) |
|---|---|---|
| 배포 | `./deploy-workshop-app.sh` | `./cli strands-agents loan-buddy-agent install` |
| 에이전트 프레임워크 | LangChain + LangGraph (`create_react_agent`) | **Strands Agents SDK** |
| LLM 접근 | LiteLLM `sk-` Virtual Key → Bedrock | Kong `apikey` → `ai-proxy` → Bedrock |
| 앱 IAM | Bedrock + S3 | **S3만** (Bedrock은 Kong 경유) |
| 추적 | Langfuse `credit_underwriting_agent_with_image_id` | Arize AX 프로젝트 `loan-strands` |
| 도구 | 3개 MCP 서버 (SSE) | **동일한 3개 MCP 서버** |

## 주요 요점

다음을 학습했습니다:
1. **Agentic 앱 구축** — Strands SDK 사용
2. **모든 AI 트래픽을 Kong AI Gateway로 라우팅** — SaaS 관리형, K8s 네이티브 구성
3. **에이전트 프레임워크 간 MCP 서버 재사용** — MCP가 상호 운용 계층
4. **Arize AX로 추적** — Langfuse 대신 OpenInference 사용
5. **Kubernetes에 배포** — 최소 권한 IAM (S3만) 적용

## 정리

:::code{language=bash showCopyAction=true}
cd /workshop && ./cli strands-agents loan-buddy-agent uninstall
:::

---

**[다음: 워크샵 요약 및 다음 단계 →](../../summary/)**
