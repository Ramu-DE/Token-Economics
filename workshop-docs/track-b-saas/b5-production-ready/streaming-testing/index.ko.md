---
title: "스트리밍 응답과 프로덕션 테스트"
weight: 30
---

# 🌊 스트리밍 응답과 프로덕션 테스트

대출 담당자는 에이전트가 추론하는 동안 60초 내내 빈 화면을 쳐다보고 있어서는 안 됩니다. 스트리밍은 평가 결과가 생성되는 대로 전달합니다. 이 페이지에서는 **Strands SDK** 를 활용한 SSE 스트리밍, Pydantic 출력 검증, 그리고 SaaS 스택을 위한 프로덕션 준비 체크리스트를 추가합니다.

::alert[**이 페이지는 [Track A의 스트리밍 & 테스트 페이지](/track-a-opensource/a4-production-ready/streaming-testing/)의 SaaS 스택 대응 버전입니다.** UX 근거와 Pydantic 검증은 스택과 무관합니다. 차이점은 스트리밍이 **Strands SDK** 로 구현되며 **Kong을 통해** 흐른다는 것입니다(Kong은 스트리밍 응답을 변경 없이 그대로 통과시킵니다).]{type="info"}

## 🛠️ 실습: Strands로 스트리밍 활성화

### UX 차이 (Track A와 공유)

스트리밍이 없으면 대출 담당자는 다단계 평가 전체가 끝날 때까지 기다립니다. 스트리밍을 사용하면 토큰이 생성되는 대로 나타납니다. [Track A의 UX 비교](/track-a-opensource/a4-production-ready/streaming-testing/)를 참고하십시오 — 동일하게 적용됩니다.

### 차이점: Strands 비동기 스트리밍

Track A는 LangGraph 앱에서 스트리밍합니다. Track B에서는 Strands SDK가 에이전트에 **`stream_async()`** 를 노출하여 에이전트 루프가 실행되는 동안 이벤트를 산출(yield)합니다. FastAPI 앱에서 이를 SSE 엔드포인트로 노출합니다:

:::code{language=python showCopyAction=true}
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

@app.post("/api/process_credit_application_stream")
async def process_stream(image_file: UploadFile = File(...)):
    # ... store image, load MCP tools, build agent (as in B3) ...

    async def event_generator():
        async for event in agent.stream_async(user_prompt):
            # Strands yields text deltas + tool events; forward as SSE
            if "data" in event:
                yield f"data: {event['data']}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(event_generator(), media_type="text/event-stream")
:::

::alert[**스트리밍은 Kong을 통해 변경 없이 작동합니다.** Kong의 `ai-proxy` 는 스트리밍 Bedrock 응답을 에이전트로 다시 프록시하고, 에이전트는 SSE를 통해 클라이언트로 재스트리밍합니다. Kong은 심지어 AI 메트릭에서 스트리밍 호출을 `request_mode="stream"` 으로 라벨링하므로(v3.12+), B4 대시보드에서 스트리밍 트래픽과 원샷 트래픽을 구분할 수 있습니다. 라우트에서 스트리밍이 활성화되어 있는지 확인하십시오(`llm/v1/chat` 의 경우 기본적으로 활성화되어 있습니다).]{type="info"}

### 스트리밍 + 트레이싱

Strands는 스트리밍 실행에 대해서도 동일한 OpenInference 스팬을 방출하므로 **Arize는 여전히 전체 트레이스를 캡처합니다**(AGENT/LLM/TOOL 스팬, 토큰) — 스트리밍은 클라이언트 전달 방식을 바꿀 뿐 관측성은 바꾸지 않습니다.

## 📐 Pydantic을 활용한 구조화된 출력 검증 (공유)

Pydantic 신용 결정 스키마와 검증 로직은 **스택과 무관합니다** — **[Track A의 Pydantic 섹션](/track-a-opensource/a4-production-ready/streaming-testing/)** 을 따르십시오. 유일한 차이는 에이전트 내에서의 위치입니다: Strands 에이전트의 최종 `result` 를 반환하기 전에 스키마에 대해 검증합니다.

:::code{language=python showCopyAction=true}
# Validate the Strands agent's final output against the shared schema
from pydantic import ValidationError

result = agent(user_prompt)
try:
    decision = CreditDecision.model_validate_json(str(result))   # same schema as Track A
except ValidationError as e:
    logger.error("Output failed schema validation: %s", e)
    # re-prompt or return a safe error
:::

## ✅ 프로덕션 준비 체크리스트 (Track B)

출시 전에 모든 SaaS 스택 프로덕션 요구사항을 확인하십시오:

| 영역 | 요구사항 | 위치 |
|---|---|---|
| **게이트웨이** | `/loan-strands` 라우트 활성화; `key-auth` 강제; 모델 고정(`allow_override:false`) | B2 |
| **비용 제어** | Kong AI 메트릭 활성화; per-Consumer 어트리뷰션; `ai-rate-limiting-advanced` 예산 설정 | B4 |
| **관측성** | Arize가 AGENT/LLM/TOOL 스팬 수신; 비용 구성(수집 전); 평가자 실행 중 | B2, B4 |
| **RAG** | Milvus 배포; 정책 문서 수집; 검색 도구 트레이싱; Ragas 평가 | B5 RAG |
| **보안** | 에이전트 Pod Identity = **S3 전용**; ESO를 통한 시크릿; 라우트에 `ai-prompt-guard`; Bedrock Guardrails 적용 | B5 보안 |
| **스트리밍** | `stream_async` 를 통한 SSE 엔드포인트; Pydantic으로 검증 | 이 페이지 |
| **모니터링** | Kong AI 메트릭 → AMP → Grafana; KEDA 오토스케일링 | B4 모니터링 |
| **멀티 아키텍처** | amd64 + arm64용으로 빌드된 에이전트 이미지(혼합 노드 클러스터) | B3 |

## 🎊 지금까지 구축한 것 — Track B 완료

모듈 B1-B5 전반에 걸쳐, **Kong + Arize SaaS 스택** 위에 완전한 프로덕션급 에이전틱 AI 플랫폼을 구축했습니다:

- ✅ Amazon Bedrock 앞에 위치한 **Kong AI Gateway** (Konnect + DataPlane, CRD로 구성) — 앱에 LLM 키 없음
- ✅ 평가와 비용을 갖춘 **Arize AX** 전체 트레이스 관측성
- ✅ MCP 도구, RAG, 스트리밍을 갖춘 **Strands** Loan Buddy 에이전트
- ✅ **테넌트별 토큰/비용 어트리뷰션 AND 엣지 강제 적용** — SaaS 스택의 강력한 능력
- ✅ **프로덕션 강화** — 최소 권한 IAM, 외부 시크릿, 게이트웨이 + 모델 콘텐츠 안전성

::alert[**두 트랙 이야기:** 이제 동일한 에이전틱 AI 플랫폼이 두 가지 방식으로 구축되는 것을 보았습니다 — [Track A](/track-a-opensource/)(자체 호스팅 오픈소스)와 Track B(관리형 SaaS). 고객은 운영 모델을 비교하고 자신의 팀에 맞는 것을 선택할 수 있습니다.]{type="success"}

---

**[다음: 워크샵 요약 →](/summary/)**
