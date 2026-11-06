---
title: "프로덕션을 위한 보안 강화"
weight: 20
---

# 🔐 프로덕션을 위한 보안 강화

Loan Buddy가 실제 신청자 데이터를 처리하기 전에, 프로덕션 보안 요구사항을 충족해야 합니다: 하드코딩된 시크릿 없음, 최소 권한 IAM, 그리고 콘텐츠 안전성. SaaS 스택에서는 IAM과 시크릿 작업이 Track A와 공유되며, **콘텐츠 안전성은 Track A의 앱 측 접근 방식에는 없는 게이트웨이 수준 계층을 추가로 확보** 합니다.

::alert[**Pod Identity와 External Secrets는 스택에 독립적입니다** — 게이트웨이와 무관하게 파드와 그 자격 증명을 보호합니다. 이 부분은 Track A를 따른 다음, 아래의 **Track-B 콘텐츠 안전성 델타** 를 적용하세요.]{type="info"}

## 🔐 파트 1: Pod Identity — 최소 권한 IAM (Track A 따르기)

**[Track A의 Pod Identity 섹션](/track-a-opensource/a4-production-ready/security-hardening/)** 을 완료하세요 — 변경 없이 그대로 적용됩니다. 에이전트에게 *어떤* IAM이 필요한지에 대한 중요한 Track-B 참고 사항이 하나 있습니다:

::alert[**Strands 에이전트는 Track A보다 더 적은 IAM만 필요합니다.** Bedrock에 **Kong을 통해** 접근하기 때문에(Kong DataPlane이 B1에서 자체 Pod Identity를 통해 Bedrock IAM을 보유), Strands 에이전트의 Pod Identity는 **S3만**(이미지 저장) 부여하며 **`bedrock:InvokeModel`은 부여하지 않습니다**. 이것은 진정한 Track-B 보안 이점입니다: 애플리케이션이 절대 Bedrock 자격 증명을 보유하지 않습니다. 이는 이미 [B3](/track-b-saas/b3-genai-applications-strands/)에서 설정되었으며, 여기서 프로덕션 수준으로 강화합니다(S3 정책을 특정 버킷 ARN으로 범위 지정).]{type="success"}

## 🔑 파트 2: External Secrets Operator (Track A 따르기)

**[Track A의 External Secrets 섹션](/track-a-opensource/a4-production-ready/security-hardening/)** 을 완료하세요 — ESO를 배포하고 AWS Secrets Manager에서 자격 증명을 동기화합니다. Track-B 델타는 *어떤* 시크릿인가입니다:

| 시크릿 | Track A | Track B |
|---|---|---|
| LLM 게이트웨이 인증 | LiteLLM Virtual Key (`sk-...`) | Kong consumer 키(`apikey`) → Secrets Manager에 저장, ESO로 동기화 |
| 관측성 | Langfuse 키 | **Arize** Service 키 + Space ID(B1의 `arize-credentials` Secret) |

::alert[**Arize + Kong 자격 증명 강화:** B1-B3에서는 `arize-credentials` Secret과 Kong `apikey`가 직접 생성되었습니다. 프로덕션에서는 이를 **AWS Secrets Manager** 에 저장하고 ESO로 동기화하여(Track A가 자체 키에 대해 하는 것과 정확히 동일하게), 민감한 정보가 평문 Kubernetes Secret이나 매니페스트에 존재하지 않도록 하세요.]{type="info"}

## 🛡️ 파트 3: 콘텐츠 안전성 — Track B의 두 계층

여기가 Track B가 가장 크게 다른 부분입니다. Track A는 LiteLLM을 통해 Bedrock Guardrails를 적용합니다. Track B에는 **두 가지 상호 보완적인 옵션** 이 있습니다 — 게이트웨이 수준 가드와 앱 수준 Bedrock Guardrails입니다.

### 계층 1: Kong `ai-prompt-guard` (게이트웨이 적용) — Track-B 델타

Kong의 **`ai-prompt-guard`** plugin은 채팅 요청이 Bedrock에 도달하기 전에 **게이트웨이에서** 필터링합니다 — 사용자 메시지에 대한 허용/거부 목록(PCRE 정규식)과 숨겨진 유니코드 프롬프트 인젝션 시도 탐지를 제공합니다. 규칙을 위반하는 요청은 **엣지에서 400** 을 받게 되므로, 잘못된 입력이 절대 Bedrock 호출을 소비하지 않습니다.

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-prompt-guard-loan-strands
plugin: ai-prompt-guard
config:
  # deny obvious injection / out-of-scope requests; deny wins over allow
  deny_patterns:
    - "(?i)ignore (all|previous) instructions"
    - "(?i)system prompt"
  allow_patterns:
    - ".*"
EOF
:::

이를 `/loan-strands` 라우트에 연결하세요(B2의 `ai-proxy` + `key-auth`와 함께 라우트의 `konghq.com/plugins` 어노테이션에 추가). 이것은 **게이트웨이에서 적용되는 안전성** 입니다 — 애플리케이션 코드와 무관하게 라우트의 모든 consumer를 보호하며, 앱 측에서만 적용하는 접근 방식으로는 보장할 수 없는 것입니다.

::alert[**검증됨:** `ai-prompt-guard`는 `user` 역할 메시지를 허용/거부 정규식과 대조하여 스캔하고 위반을 400으로 차단하며, 숨겨진 유니코드 인젝션을 탐지하는 Kong plugin입니다. Kong Ingress Controller(KongPlugin CRD)를 통해 지원됩니다.]{type="success"}

### 계층 2: Amazon Bedrock Guardrails (앱 측) — Track A와 동등

관리형 콘텐츠 정책(PII 마스킹, 거부 주제, 유해성)의 경우 **Amazon Bedrock Guardrails** 를 사용하세요. Track B에서는 **애플리케이션의 모델 호출** 에 guardrail을 연결합니다 — Strands `BedrockModel`(또는 OpenAI 호환 요청 `extra_body`를 통해)에 `guardrailIdentifier` + `guardrailVersion`을 전달합니다.

::alert[**솔직한 범위 참고:** Kong의 `ai-proxy`가 게이트웨이에서 Bedrock Guardrails를 Bedrock 호출에 *주입* 할 수 있는지(그래서 앱이 이를 지정하지 않아도 되는지)는 제가 검증한 현재 Kong 문서에서 **확인되지 않았습니다** — `ai-proxy`는 `extra_body` 통과(pass-through)를 문서화하지만 일급(first-class) guardrail 옵션은 문서화하지 않습니다. 사용 중인 버전에 대해 Kong Bedrock 프로바이더 레퍼런스로 검증하기 전까지는 **Bedrock Guardrails를 앱 측**(에이전트의 모델 호출에 설정)으로 취급하고, **게이트웨이 적용 계층에는 `ai-prompt-guard`** 를 사용하세요. 직접 검증하지 않고 게이트웨이 주입 방식의 Bedrock Guardrails를 주장하지 마세요.]{type="warning"}

guardrail을 한 번 생성한 다음(Track A의 8단계와 동일), 에이전트에서 이를 참조하세요:

:::code{language=python showCopyAction=true}
# Strands agent — attach a Bedrock Guardrail to the model call (app-side)
# via the OpenAI-compatible request the agent sends through Kong:
params={
    "max_tokens": 5000,
    "temperature": 0,
    "extra_headers": {"apikey": KONG_API_KEY},
    "extra_body": {
        "guardrailConfig": {
            "guardrailIdentifier": GUARDRAIL_ID,
            "guardrailVersion": GUARDRAIL_VERSION,
        }
    },
}
:::

::alert[**통과 경로를 직접 검증하세요:** 이에 의존하기 전에 사용 중인 Kong 버전에 대해 Kong의 `ai-proxy`가 `extra_body.guardrailConfig`를 Bedrock의 Converse/InvokeModel로 전달하는지 확인하세요. 통과하지 않는다면, Bedrock의 guardrail 활성화 API를 직접 호출하여 Bedrock Guardrails를 적용하거나, 계층 1(`ai-prompt-guard`)과 모델/추론 프로파일 수준에서 구성된 Bedrock Guardrails에 의존하세요.]{type="warning"}

## 📊 Track A와의 매핑

| | Track A | Track B |
|---|---|---|
| Pod Identity | 앱이 Bedrock + S3 IAM 보유 | **앱이 S3만 보유**(Bedrock은 Kong 경유) — 더 강력함 |
| External Secrets | LiteLLM + Langfuse 키 | ESO를 통한 Kong 키 + Arize 자격 증명 |
| 게이트웨이 콘텐츠 가드 | — (앱 측만) | Kong 엣지의 **`ai-prompt-guard`**(델타) |
| 관리형 콘텐츠 정책 | LiteLLM을 통한 Bedrock Guardrails | **앱 측** Bedrock Guardrails(게이트웨이 주입은 미검증) |

## ✅ 완료한 작업

- ✅ Track A의 Pod Identity + External Secrets 재사용(스택 독립적)
- ✅ Track-B IAM 이점 확인: 에이전트는 **S3만** 보유하며 절대 Bedrock 자격 증명을 보유하지 않음
- ✅ Kong `ai-prompt-guard`로 **게이트웨이에서 적용되는** 콘텐츠 안전성 추가
- ✅ 검증되지 않은 게이트웨이 주입 경로에 대한 솔직한 참고와 함께 Bedrock Guardrails를 앱 측에 적용

---

**[다음: 스트리밍 응답과 프로덕션 테스트 →](../streaming-testing/)**
