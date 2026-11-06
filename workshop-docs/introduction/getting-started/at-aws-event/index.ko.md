---
title: "AWS 이벤트에서"
weight: 14
---

Agentic AI 인프라가 이벤트 주최자에 의해 사전 배포되어 있습니다. 이 섹션에서는 워크샵 환경에 연결하고 모든 것이 준비되었는지 확인합니다.

## 이미 완료된 것

::alert[인프라 배포가 이미 완료되어 있습니다. 워크샵 내용에 집중할 수 있습니다.]{type="success"}

워크샵 환경에 포함된 것:
- **EKS 클러스터**: Auto Mode 활성화
- **Agentic AI 플랫폼** 컴포넌트 (vLLM, LiteLLM, Langfuse, Open WebUI)
- **사전 로드된 모델** (Qwen 3 8B, DeepSeek R1 Qwen 3 8B)
- **특수 하드웨어** (추론을 위한 Neuron 노드)
- **관측성 스택**: 구성되고 실행 중

## 단계 1: Workshop Studio 액세스

Workshop Studio는 이 이벤트를 위한 격리된 AWS 환경을 제공합니다.

### 1.1 Workshop Studio에 연결

강사가 제공한 이벤트 URL로 이동합니다. 액세스 코드가 제공된 경우 다음 링크에서 액세스 코드를 입력합니다:

```
https://catalog.workshops.aws/join
```

Workshop Studio 랜딩 페이지가 표시됩니다:

![Workshop Studio Event Page](/static/images/introduction/workshopstudio-event1.jpg)

### 1.2 Email OTP로 로그인

비밀번호 없는 인증을 진행하려면 **Email One-Time Password (OTP)** 를 선택합니다:

1. 이메일 주소를 입력합니다
2. **Send passcode** 를 선택합니다
3. **"Your one-time passcode"** 제목의 이메일을 확인합니다
4. 코드를 입력합니다
5. **Sign in** 을 선택합니다

::alert[**팁**: 패스코드는 10분 후에 만료됩니다. 만료되면 새로 요청하십시오.]{type="info"}

### 1.3 이벤트 참가

로그인 후 이벤트 액세스 페이지가 표시됩니다:

1. **Event access code** 가 미리 입력되어 있어야 합니다. **Next** 를 선택합니다.
2. **"I agree with the Terms and Conditions"** 를 선택합니다
   ![Workshop Studio Event Page](/static/images/introduction/workshopstudio-event3.jpg)
3. **Join event** 를 선택합니다

이제 워크샵 환경에 접속되었습니다.

## 단계 2: AWS 계정 액세스

### 2.1 AWS Console 열기

Workshop Studio 왼쪽 사이드바에서 다음을 찾아 선택합니다:

**Open AWS Console**
![alt text](/static/images/introduction/open-aws-console.png)

임시 자격 증명이 이미 구성된 상태로 새 탭에서 AWS Management Console이 열립니다.

::alert[**중요**: 반드시 이 콘솔 링크를 사용하십시오. 이 워크샵에 개인 AWS 계정을 사용하지 마십시오.]{type="warning"}

## 단계 3: 개발 환경 액세스

### 3.1 Visual Studio Code (VSC) IDE 열기

Workshop Studio를 통해 클라우드 기반 개발 환경에 액세스합니다:

1. **이벤트 페이지로 이동**:
   - Workshop Studio 인터페이스에서 **왼쪽 상단 모서리** 를 확인합니다
   - "Architect and deploy advanced Agentic AI platforms on Amazon EKS event"를 표시하는 **이벤트 제목 상자** 를 선택합니다 (제목이 다를 수 있음)
   - 모든 워크샵 세부 정보가 있는 메인 이벤트 페이지로 이동합니다

2. **Event Outputs 섹션 찾기**:
   - 이벤트 페이지에서 아래로 스크롤하여 **"Event Outputs (1)"** 섹션을 찾습니다
   - 워크샵 환경의 출력 값이 있는 테이블이 표시됩니다

![Workshop Studio Event Outputs](/static/images/introduction/url.png)

3. **VSC IDE URL 찾기**:
   - 출력 테이블에서 **Key** 가 "URL"인 행을 찾습니다
   - 이 URL(주황색으로 강조 표시)은 클라우드 기반 Visual Studio Code 환경을 가리킵니다
   - URL은 다음과 유사합니다: `https://d3fbc6117q00r.cloudfront.net/...`

4. **개발 환경 열기**:
   - Value 열의 **URL 링크** 를 선택합니다
   - 전체 Visual Studio Code IDE가 있는 새 브라우저 탭이 열립니다
   - 환경이 완전히 로드될 때까지 잠시 기다립니다

5. **제공되는 환경**:
   클라우드 기반 VSC IDE에는 다음이 포함됩니다:
   - **전체 Visual Studio Code 인터페이스**: 구문 강조 및 확장 기능 포함
   - **통합 터미널**: bash 액세스 포함
   - **kubectl 사전 구성됨**: EKS 클러스터에 연결됨
   - **AWS CLI 사용 준비**: 임시 자격 증명 포함
   - **코드 에디터**: 모든 워크샵 실습 및 수정 작업용

::alert[**중요**: 이 창을 닫지 마십시오. 워크샵 중에 개발 환경으로 여러 번 돌아와야 합니다.]{type="info"}

### 3.2 kubectl 액세스 확인

VSC 터미널에서 EKS 클러스터에 대한 연결을 확인합니다:

:::code{language=bash showCopyAction=true}
# Check cluster access
kubectl cluster-info

# Expected output:
# Kubernetes control plane is running at https://...
:::

## 설정 완료

이제 다음에 액세스할 수 있습니다:

- **AWS Console**: 임시 자격 증명 포함
- **VSC IDE**: 터미널 및 kubectl 구성됨
- **EKS 클러스터**: 연결 확인됨

## 다음 단계

개발 환경에 액세스할 수 있으므로, 다음 단계에서는 배포된 Agentic AI 인프라를 살펴보고 모든 것이 올바르게 작동하는지 확인합니다.

---

**[인프라 개요로 계속 →](/introduction/infra-setup/)**
