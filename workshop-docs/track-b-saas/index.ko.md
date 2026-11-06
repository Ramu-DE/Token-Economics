---
title: "Track B: Kong Konnect, Arize AX, Strands 기반 SaaS 스택"
weight: 40
---

# Track B — SaaS 스택

이 트랙에서는 **관리형 SaaS** 컨트롤 플레인과 AWS의 자체 에이전트 SDK를 사용해 동일한 Agentic AI 플랫폼을 구축합니다:

- **AI Gateway** — [Kong AI Gateway](https://developer.konghq.com/): **Kong Konnect**(SaaS 컨트롤 플레인) + **Kong Operator 2.x**가 관리하는 클러스터 내 DataPlane, 네이티브 Kubernetes CRD로 구성
- **Observability** — [Arize AX](https://arize.com/docs/ax) (SaaS): OpenInference/OpenTelemetry 트레이스 수집
- **Agent framework** — [Strands Agents SDK](https://github.com/strands-agents/sdk-python): AWS의 모델 우선(model-first) 에이전트 프레임워크

SaaS 플랫폼을 설정하고(Module B1), AI 게이트웨이와 관측성를 구성하고(B2), Strands로 신용 심사(credit-underwriting) 에이전트를 구축(B3)합니다.

::alert[**트랙 선택 기준:** Track B는 관리형 SaaS 컨트롤 플레인을 사용합니다 — 클러스터에는 상태 비저장(stateless) 게이트웨이 DataPlane만 실행하고 트레이스는 Arize AX로 내보내므로, 그 외에 직접 운영할 구성 요소가 없습니다. 게이트웨이와 관측성를 오픈소스 도구로 자체 호스팅하려면 [Track A — 오픈소스 스택](../track-a-opensource/)을 참조하십시오. 게이트웨이와 관측성 계층은 두 트랙 간에 독립적이므로, 플랫폼 운영 방식에 맞는 트랙을 선택합니다.]{type="info"}

::alert[**사전 요구사항:** Track B는 **Introduction**과 **Interacting with Models**에서 만드는 공통 기반(EKS 클러스터, vLLM, Open WebUI, Amazon Bedrock 액세스) 위에서 진행되므로 이를 먼저 완료해야 합니다. 또한 **B3**의 Strands 에이전트는 [Track A → Module A2](../track-a-opensource/a2-genai-applications/)에서 배포하는 **MCP 도구 서버 3종**을 재사용합니다(MCP는 프레임워크에 종속되지 않으므로 두 트랙이 동일한 도구 서버를 공유합니다). Track B만 단독으로 진행하는 경우, B3 전에 Module A2의 MCP 배포 단계를 먼저 실행하세요.]{type="warning"}

## 이 트랙의 구성

| 모듈 | 주요 내용 |
|---|---|
| **B1: 플랫폼 설정 — Kong Operator + Arize AX** | Kong Konnect 컨트롤 플레인 + DataPlane 등록; Arize AX Space 및 자격 증명 생성 |
| **B2: AI Gateway & Observability (Kong + Arize)** | KongService/KongRoute/KongPlugin CRD로 구성하는 `ai-proxy` Bedrock 라우트; Arize AX 트레이싱 연동 |
| **B3: Strands로 Agentic 애플리케이션 구축** | Kong → Bedrock 경로로 동작하고 Arize AX에서 트레이싱되는 Strands 신용 심사 에이전트 |

## 트랙 비교

| | [Track A](../track-a-opensource/) | Track B (현재 트랙) |
|---|---|---|
| AI Gateway | LiteLLM (자체 호스팅) | **Kong AI Gateway** — Konnect (SaaS) |
| Observability | Langfuse (자체 호스팅) | **Arize AX** (SaaS) |
| Agent framework | LangChain / LangGraph | **Strands Agents SDK** |
| 모델 액세스 IAM | 앱이 Bedrock + 게이트웨이 키 보유 | **앱은 S3만 보유; Bedrock은 게이트웨이 경유** |
| 운영 모델 | 컨트롤 플레인을 직접 운영 | **관리형 SaaS 컨트롤 플레인** |

---

**[Module B1: 플랫폼 설정부터 시작하기 →](b1-platform-setup/)**
