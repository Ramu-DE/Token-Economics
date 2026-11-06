---
title: "모듈 B1: 플랫폼 설정 - Kong Operator + Arize AX"
weight: 10
---

**Alternative Stack**으로 GenAI 애플리케이션을 구축하고 배포하기 전에, 이 섹션에서는 LiteLLM과 Langfuse를 대체하는 두 가지 SaaS 플랫폼인 **Kong AI Gateway**(Kong Konnect 기반)와 **Arize AX**(관측성)를 설정하는 과정을 안내합니다. LiteLLM과 Langfuse가 클러스터 내에 사전 배포되어 있는 기본 스택과 달리, 대체 스택은 관리형 SaaS 컨트롤 플레인을 등록하고 Amazon EKS 클러스터에 연결하는 방식을 사용합니다.

::alert[이 섹션은 **Alternative Stack** 설정입니다. 기본 LiteLLM + Langfuse 스택과 병렬로 실행되며 기본 스택을 수정하지 않습니다. 공유 컴포넌트(vLLM, Open WebUI, Qdrant, TEI)는 사전 구성된 상태로 유지되며, 게이트웨이와 관측성 레이어만 변경됩니다.]{type="info"}

## 설정할 항목

| 레이어 | 기본 스택 (사전 구성됨) | 대체 스택 (여기서 설정) |
|---|---|---|
| AI Gateway | LiteLLM (클러스터 내 파드) | **Kong AI Gateway** -- Konnect (SaaS) 컨트롤 플레인 + 클러스터 내 DataPlane |
| 관측성 | Langfuse (클러스터 내) | **Arize AX** (SaaS) -- Space + API 키 |
| LLM 제공자 | Amazon Bedrock | Amazon Bedrock *(변경 없음)* |
| 자체 호스팅 모델 | vLLM Qwen | vLLM Qwen *(변경 없음)* |

## SaaS 컨트롤 플레인을 사용하는 이유

프로덕션 팀은 데이터베이스와 컨트롤 플레인 인프라를 직접 운영하는 부담을 피하기 위해 **관리형** 게이트웨이와 관측성 레이어를 표준으로 채택하는 경우가 많습니다. 이 경로는 게이트웨이(Kong Konnect)와 관측성(Arize AX)가 SaaS일 때 *동일한* 플랫폼이 어떻게 구축되는지 보여줍니다:

- **Kong Konnect**는 게이트웨이 구성, 분석, AI Manager UI를 호스팅합니다. 클러스터에서는 상태 비저장 **DataPlane**만 실행합니다(Kong Gateway Operator가 관리).
- **Arize AX**는 트레이스, 스팬, 세션, 평가를 호스팅합니다. 에이전트가 **OpenInference** 스팬을 Arize AX로 내보내며, 클러스터 내에서 실행할 것이 없습니다.

## 설정 섹션

Module B2(Kong 및 Arize UI를 구성하고 사용하는 모듈)를 시작하기 **전에** 두 설정 페이지를 모두 완료하십시오:

#### 1. [Kong AI Gateway 설정 →](kong-setup/)
Kong Konnect 구독을 등록하고, Personal Access Token을 생성하고, Kong Gateway Operator를 설치한 다음, Bedrock 액세스가 가능한 EKS 클러스터에 연결된 Control Plane + DataPlane을 배포합니다.

#### 2. [Arize AX 설정 →](arize-setup/)
Arize AX 계정을 생성하고, Space를 설정하고, Service API 키를 생성한 다음, 에이전트가 트레이스를 내보낼 수 있도록 자격 증명을 클러스터에 설치합니다.

## 사전 요구 사항

기본 워크샵의 [인프라 설정](../infra-setup/)에 포함된 모든 항목(EKS Auto Mode 클러스터, vLLM, Open WebUI, kubectl 액세스)에 **더하여** 다음이 필요합니다:

- **Kong Konnect** 계정 (프리 티어 / Plus 트라이얼) -- 다음 페이지에서 설정
- **Arize AX** 계정 (프리 티어) -- Arize 페이지에서 설정
- `kubectl`, `helm`, `deck`, `aws` CLI를 사용할 수 있는 **AWS CloudShell** 또는 워크샵 Code Editor 터미널

::alert[**CLI 도구:** 대체 스택은 Kong용 **decK** CLI(`deck`)를 추가로 필요로 합니다. 워크샵 Code Editor에는 사전 설치되어 있습니다. 로컬에 설치하려면 `brew install kong/deck/deck`(macOS)를 사용하거나 [decK 설치 가이드](https://docs.konghq.com/deck/latest/installation/)를 참조하십시오.]{type="info"}

## 성공 체크리스트

Module B2로 진행하기 전에 다음을 확인하십시오:

- **Kong Konnect** 계정 생성 완료 및 컨트롤 플레인 `kong-aws`가 Gateway Manager에 표시됨
- **Kong DataPlane**(`dataplane1`)이 `kong` 네임스페이스에서 실행 중이고 연결됨(healthy)
- DataPlane의 **Bedrock 액세스** 확인 완료 (Pod Identity → IAM 역할)
- **Arize AX** 계정 생성, Space 설정, Service API 키 생성 완료
- **`arize-credentials`** Secret이 클러스터에 존재

## Module B2 준비 완료

두 설정을 모두 완료하면 다음을 갖추게 됩니다:

- Amazon Bedrock에 연결된 Kong Konnect 컨트롤 플레인 + 클러스터 내 DataPlane
- 에이전트 트레이싱을 위한 Arize AX Space + 자격 증명
- 기본 스택과 동일한 공유 모델 레이어 (vLLM, Open WebUI)

이제 Kong Bedrock 라우트와 Arize 관측성를 구성할 준비가 되었습니다.

---

**[시작: Kong AI Gateway 설정 →](kong-setup/)**
