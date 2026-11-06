---
title: "Track A: LiteLLM, Langfuse, LangGraph 기반 오픈소스 스택"
weight: 30
---

# Track A — 오픈소스 스택

이 트랙에서는 Amazon EKS 클러스터에서 직접 운영하는 **자체 호스팅 오픈소스** 컴포넌트로 완전한 Agentic AI 플랫폼을 구축합니다:

- **AI Gateway** — [LiteLLM](https://docs.litellm.ai/) (클러스터 내 배포): Amazon Bedrock과 vLLM 모델에 대한 통합 OpenAI 호환 API 제공
- **Observability** — [Langfuse](https://langfuse.com/docs) (클러스터 내 배포): 트레이싱, 세션, 평가(Evaluation) 지원
- **Agent framework** — [LangChain / LangGraph](https://langchain-ai.github.io/langgraph/): 에이전틱 애플리케이션 구현

플랫폼을 구축하고(Module A1), 신용 심사(credit-underwriting) 에이전트를 만들고(A2), 확장 및 평가하고(A3), 프로덕션 수준으로 강화(A4)합니다.

::alert[**트랙 선택 안내:** Track A는 직접 실행하고 운영하는 자체 호스팅 오픈소스 도구를 사용합니다. **관리형 SaaS** 컨트롤 플레인(Kong Konnect + Arize AX)과 Strands SDK를 사용하고 싶다면 [Track B — SaaS 스택](../track-b-saas/)을 참고하세요. 게이트웨이와 관측성 계층은 두 트랙 간에 완전히 독립적이므로, 플랫폼을 운영하고 싶은 방식에 맞는 트랙을 선택하면 됩니다. (참고: MCP는 프레임워크에 종속되지 않으므로, Track B의 에이전트는 여기 Module A2에서 배포하는 MCP 도구 서버를 재사용합니다.)]{type="info"}

## 이 트랙의 구성

| 모듈 | 주요 내용 |
|---|---|
| **A1: GenAI 플랫폼 컴포넌트** | Helm으로 배포하는 LiteLLM 게이트웨이 + Langfuse 관측성 |
| **A2: Agentic AI 애플리케이션 구축** | LangGraph 에이전트 + MCP 도구 (신용 심사) |
| **A3: 플랫폼 확장과 평가** | LLM-as-a-Judge 평가, A/B 모델 라우팅, AMP/AMG 모니터링, KEDA 오토스케일링 |
| **A4: 프로덕션 수준의 Agentic AI 애플리케이션** | RAG 파이프라인, Pod Identity, External Secrets, Bedrock Guardrails, SSE 스트리밍 |

## 트랙 비교

| | Track A (현재 트랙) | [Track B](../track-b-saas/) |
|---|---|---|
| AI Gateway | **LiteLLM** (자체 호스팅) | Kong AI Gateway — Konnect (SaaS) |
| Observability | **Langfuse** (자체 호스팅) | Arize AX (SaaS) |
| Agent framework | **LangChain / LangGraph** | Strands Agents SDK |
| 모델 액세스 IAM | 앱이 Bedrock + 게이트웨이 키 보유 | 앱은 S3만 보유; Bedrock은 게이트웨이 경유 |
| 운영 모델 | 컨트롤 플레인을 직접 운영 | 관리형 SaaS 컨트롤 플레인 |

---

**[Module A1: GenAI 플랫폼 컴포넌트부터 시작하기 →](a1-genai-components/)**
