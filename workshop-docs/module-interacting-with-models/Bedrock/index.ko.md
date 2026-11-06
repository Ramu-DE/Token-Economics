---
title: "AWS Bedrock - 관리형 AI 서비스"
weight: 23
---

AWS Bedrock은 단일 API를 통해 주요 AI 기업의 고성능 파운데이션 모델에 대한 액세스를 제공하는 완전 관리형 서비스입니다. 이 섹션에서는 Claude Sonnet 4.5에 대한 액세스를 구성하고 EKS에서 구성하는 Agentic AI 플랫폼을 위해 Bedrock 모델을 사용하는 방법을 배웁니다.

## AWS Bedrock이란 무엇입니까?

Amazon Bedrock은 다음을 제공합니다:

- **즉시 액세스**: 관리할 인프라나 배포할 모델이 없습니다
- **파운데이션 모델**: Claude(Anthropic), Llama(Meta), Mistral, Amazon Titan 등을 제공합니다
- **사용량 기반 과금**: 처리된 토큰에 대해서만 비용을 지불하며 유휴 비용이 없습니다
- **보안**: 데이터 프라이버시, 규정 준수 및 거버넌스가 내장되어 있습니다
- **성능**: 일관되게 낮은 지연 시간을 제공하는 최적화된 인프라를 사용합니다
- **추가 기능**: 파인 튜닝, RAG, 에이전트 및 가드레일을 지원합니다

## 이 워크샵에서 Bedrock을 사용하는 이유는?

이전 섹션에서 워크샵 인프라의 자체 호스팅 모델을 사용해 보았습니다. 하드웨어 제약(inf2.xlarge 인스턴스)으로 인해 자체 호스팅 모델의 성능은 제한적입니다. 나머지 모듈에서는 다음과 같은 이유로 Bedrock 사용을 권장합니다:

1. **하드웨어 제약 없음**: 워크샵 인스턴스 크기에 제한받지 않습니다
2. **일관된 성능**: 실습에 필요한 안정적인 응답 시간을 제공합니다
3. **최신 모델**: 고급 기능을 갖춘 Claude Sonnet 4.5에 액세스할 수 있습니다
4. **학습에 집중**: 응답 대기 시간 대신 개념 학습에 시간을 사용할 수 있습니다

::alert[**워크샵 제안**: 나머지 모듈에서는 Bedrock(Claude Sonnet 4.5)을 사용하는 것을 권장합니다. 자체 호스팅과 관리형 접근 방식은 각각 적합한 시나리오가 다릅니다.]{type="info"}

## Open WebUI에서 Bedrock 모델 사용

모델 액세스가 활성화되면 LiteLLM을 통해 Open WebUI를 통해 Bedrock 모델과 상호작용할 수 있습니다:

### 1단계: Open WebUI 액세스

1. 이전 명령에서 OpenWebUI URL 사용
2. 자격 증명으로 로그인

### 2단계: Bedrock 모델 선택

1. 채팅 상단의 모델 드롭다운 클릭
2. **"claude-4.5-sonnet"** 찾기(자동으로 나타나야 함)
3. 활성 모델로 선택

### 3단계: Claude Sonnet 4.5 테스트

몇 가지 실용적인 예제로 Claude의 기능을 테스트합니다.

#### 테스트 1: 고급 기술 추론

다음 질문을 입력하여 Claude의 기술적 응답 수준을 확인합니다:

:::code{language=markdown showCopyAction=true}
Kubernetes operators의 개념을 설명하고 커스텀 operator를 생성하는 경우의 간단한 예를 제공하십시오.
:::

**예상 응답**: Claude는 명확한 구조로 포괄적인 설명을 제공합니다:

![Claude's Kubernetes Operators Response](/static/images/module-interacting-with-models/claude-kubernetes-operators-response.png)

**주목할 점**:
- **구조화된 응답**: 명확한 제목과 논리적 흐름
- **기술적 깊이**: 제어 루프, CRD 및 실용적인 예제 설명
- **실제 컨텍스트**: 구체적인 사용 사례 제공

#### 테스트 2: 코드 생성 (선택 사항)

코드 생성 기능을 시도해 보십시오:

:::code{language=markdown showCopyAction=true}
get 및 put 메서드를 사용하여 간단한 LRU 캐시를 구현하는 Python 클래스를 작성하십시오. 각 작업의 시간 복잡도를 설명하십시오.
:::

#### 테스트 3: 장문 분석 (선택 사항)

복잡한 분석 작업의 경우:

:::code{language=markdown showCopyAction=true}
Kubernetes에서 AI 워크로드를 실행하는 이점과 과제를 분석하십시오. 확장성, 리소스 관리 및 운영 복잡성과 같은 측면을 고려하십시오.
:::

**주요 관찰**: Claude는 이후 모듈의 학습 실습에 적합한 상세하고 잘 구조화된 응답을 제공합니다.

## 모델 비교: 다양한 접근 방식 탐색

Open WebUI에는 서로 다른 모델의 응답을 나란히 비교할 수 있는 기능이 있습니다. 이 기능을 사용하면 서로 다른 모델이 같은 문제에 어떻게 접근하는지 이해하고 특정 작업에 적합한 모델을 선택할 수 있습니다.

### 모델 비교 방법

1. **첫 번째 모델 선택** (예: Claude Sonnet 4.5)
2. 모델 이름 옆의 **"+" 버튼 클릭** 하여 두 번째 모델 추가
3. **두 번째 모델 선택** (예: vLLM Deepseek)
4. 모델 이름 옆의 **"+" 버튼 클릭** 하여 세 번째 모델 추가
5. **세 번째 모델 선택** (예: vLLM Qwen 3)
6. 선택한 모델들에게 동시에 **같은 질문 입력**

### 비교 예시

다음 수학 질문으로 모델들의 응답을 비교해 봅니다:

**테스트 질문**: "What is squareroot of 144 divided by 29 multiplied by pi"

![Model Comparison: Side-by-Side Results](/static/images/module-interacting-with-models/model-comparison-claude-vs-llama.png)

::alert[**/no_think**: 일부 추론 모델(예: Qwen)은 응답하기 전에 "사고" 단계에 추가 시간을 소비합니다. 이것이 너무 오래 걸리면(이 예제에서는 6분), 프롬프트에 `/no_think`를 추가하여 추론 단계를 건너뛰고 직접 답변을 얻으십시오.]{type="info"}

### 관찰할 내용

**다양한 응답 스타일**: 각 모델은 문제 해결에 대한 고유한 접근 방식을 가지고 있습니다:
- 일부 모델은 단계별 설명을 제공합니다
- 다른 모델은 직접적이고 간결한 답변을 제공합니다
- 응답 형식과 세부 수준이 모델마다 다릅니다

**학습 기회**: 이 비교 기능을 통해 다음을 수행할 수 있습니다:
- 모델 특성 이해
- 다양한 사용 사례에 적합한 모델 선택
- 응답 품질 및 스타일 선호도 평가
- 다양한 문제 해결 접근 방식에서 학습

## 고급 기능: Open WebUI에서의 RAG

Open WebUI에는 기본 RAG(Retrieval Augmented Generation) 기능이 포함되어 있습니다. 다음 데모에서는 RAG가 문서의 특정 컨텍스트를 제공하여 모델 응답을 어떻게 바꾸는지 보여줍니다.

### 1단계: 테스트 문서 생성

먼저 VSC IDE 터미널에서 샘플 문서를 생성합니다:

:::code{language=bash showCopyAction=true}
# VSC IDE 터미널에서 테스트 파일 생성
cat > super-secret.txt << 'EOF'
Super Secret Document

This document contains classified information about the secret project.
Only authorized personnel are allowed to access this data.

Project Codename: Nightfall
Objective: Develop an untraceable communication device.
Status: In progress

Remember, this information is top secret and must not be shared with anyone.
EOF

code super-secret.txt
:::

### 2단계: 로컬 장치로 다운로드

VSC IDE에서 파일을 생성한 후:

1. VSC IDE 파일 탐색기에서 **파일 찾기**
2. `super-secret.txt`에서 **마우스 오른쪽 버튼 클릭**
3. **"Download" 선택** 하여 로컬 머신에 저장
4. **다운로드 위치 기억** - OpenWebUI에 업로드해야 합니다

::alert[**왜 다운로드합니까?** OpenWebUI는 브라우저에서 실행되며 원격 VSC IDE 환경이 아닌 로컬 머신의 파일이 필요합니다.]{type="info"}

### 3단계: OpenWebUI에서 Knowledge Base 생성

OpenWebUI에서 knowledge base를 설정합니다:

1. **Workspace 액세스**: **"Workspace"** 클릭 후 **"Knowledge"** 클릭

![Knowledge Tab](/static/images/module-interacting-with-models/openwebui-knowledge-workspace.png)

2. **새 Knowledge Base 생성**: 오른쪽의 **"+"** 버튼 클릭

3. **양식 작성**:
   - **Name**: `super-secret-info`
   - **Description**: `Store secret info in knowledge base`
   - **Visibility**: **"Private"** 선택 

![Create Knowledge Base Form](/static/images/module-interacting-with-models/openwebui-create-knowledge-base.png)

4. **생성 및 업로드**:
   - **"Create Knowledge"** 클릭
   - **+** 클릭
   - **"Upload files"** 클릭
   - 다운로드한 `super-secret.txt` 파일 선택
   - 업로드가 완료될 때까지 대기

### 4단계: RAG 없이 테스트 (기준선)

먼저 문서에 액세스할 수 없는 상태에서 Claude가 어떻게 응답하는지 확인합니다:

**다음 질문을 입력하십시오**: "What is the objective of project nightfall?"

![Claude Without RAG Context](/static/images/module-interacting-with-models/claude-without-rag-context.png)

**주목**: Claude는 문서 내용에 대한 지식이 없기 때문에 소셜 미디어 플랫폼에 대한 일반적인 응답을 제공합니다.

### 5단계: RAG와 함께 테스트

이제 RAG를 적용했을 때의 효과를 확인합니다:

1. **새 채팅 시작**
2. 메시지 상자에 **`#` 입력** - knowledge base 선택이 활성화됩니다

![Knowledge Base Selection](/static/images/module-interacting-with-models/openwebui-select-knowledge-base.png)

3. **knowledge base 선택**: **"super-secret.txt"** 클릭
4. **정확히 같은 질문**: "What is the objective of project nightfall?"

![Claude With RAG Context](/static/images/module-interacting-with-models/claude-with-rag-context.png)

이제 Claude가 문서를 올바르게 참조하고 정확한 정보를 제공합니다: "Develop an untraceable communication device."

### RAG의 효과

이 과정은 RAG의 기본 기능을 보여줍니다:

- **RAG 없이**: Claude는 훈련 데이터만 사용하여 일반적이고 잘못된 응답을 생성합니다
- **RAG와 함께**: Claude는 지정한 문서를 컨텍스트로 사용하여 정확하고 관련성 있는 응답을 생성합니다

같은 모델에 같은 질문을 하더라도 제공되는 컨텍스트에 따라 다른 답변이 생성됩니다.

### 방금 경험한 것

이것은 OpenWebUI의 내장 RAG 기능입니다. 다음을 확인했습니다:

- 문서를 업로드하고 인덱싱할 수 있습니다
- AI 모델이 문서에서 관련 정보를 검색할 수 있습니다
- 응답이 데이터에 기반하여 정확하고 구체적으로 생성됩니다

::alert[**간단한 RAG**: 이것은 입문용 예제입니다. OpenWebUI는 기본 RAG 기능을 제공하며, 더 고급 RAG 기능을 구현하는 다양한 방법이 있습니다.]{type="success"}

## 주요 요점

- **즉시 배포**: 관리할 인프라가 없으며 모델을 즉시 사용할 수 있습니다
- **관리형 서비스 이점**: 운영 오버헤드 없이 최적화된 인프라를 사용합니다
- **고급 모델**: 200K 컨텍스트 윈도우를 갖춘 Claude 4.5에 액세스할 수 있습니다
- **사용량 기반 과금**: 가변 워크로드에 적합한 비용 모델입니다
- **추가 기능**: 보안, 규정 준수 및 거버넌스가 내장되어 있습니다

## 모듈 요약

이제 세 가지 모델 배포 접근 방식을 사용해 보았습니다:

1. **Open WebUI**: 모든 모델을 위한 통합 채팅 인터페이스
2. **vLLM**: 완전한 제어가 가능하지만 하드웨어 제약이 있는 자체 호스팅 모델
3. **Bedrock**: 일관된 성능을 제공하는 관리형 서비스

### 나머지 모듈을 위한 제안 접근 방식

나머지 모듈에서는 **Bedrock(Claude Sonnet 4.5)** 사용을 권장합니다:
- 실습에 필요한 일관된 성능
- 복잡한 학습 시나리오를 위한 고급 기능
- 인프라 관리 오버헤드 없음
- 대화형 학습을 위한 신뢰할 수 있는 응답

### 학습한 내용

- Kubernetes에서 LLM 인프라를 배포하고 구성하는 방법
- 자체 호스팅 모델과 관리형 모델 간의 트레이드오프
- 통합 API를 통해 여러 모델 제공자를 통합하는 방법
- AI 워크로드를 위한 하드웨어 선택의 중요성
- 비용 및 성능 최적화 전략

---

## 트랙 선택

공유 파운데이션을 완료했습니다. 이제 Agentic AI 플랫폼을 어떤 스택 위에 구축할지 선택합니다. 두 트랙은 병렬이며 하나를 선택하면 됩니다:

- **[Track A — 오픈소스 스택](/track-a-opensource/)** — LiteLLM + Langfuse + LangGraph를 클러스터에 자체 호스팅합니다. 플랫폼 구축부터 애플리케이션, 확장, 프로덕션까지 전 과정을 다룹니다.
- **[Track B — SaaS 스택](/track-b-saas/)** — Kong Konnect + Arize AX + Strands 매니지드 SaaS 컨트롤 플레인을 사용합니다. 동일한 플랫폼과 에이전트를 구축합니다.

---

**[모듈 개요로 돌아가기](../) | [Track A 시작하기 →](/track-a-opensource/) | [Track B 시작하기 →](/track-b-saas/)**
