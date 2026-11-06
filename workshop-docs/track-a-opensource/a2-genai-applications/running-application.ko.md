---
title: "애플리케이션 배포 및 테스트"
weight: 30
---

# Loan Buddy 배포

이 섹션에서는 Loan Buddy를 배포하고 John Michael Doe의 대출 신청서가 실시간으로 처리되는 과정을 확인합니다.

![처리 준비가 된 John의 신청서](/static/images/shared/example1.png) 

*AI 기반 처리를 위해 준비된 John의 신청서입니다.*

## 사전 요구사항 확인

배포하기 전에 모델과 상호작용하기 & 모듈 A1의 GenAI 플랫폼이 준비되었는지 확인합니다:

:::code{language=bash showCopyAction=true}
# 플랫폼 구성요소가 실행 중인지 확인
kubectl get pods -n litellm
kubectl get pods -n langfuse

# LiteLLM 게이트웨이가 접근 가능한지 확인
echo "LiteLLM URL: http://$(kubectl get ingress -n litellm litellm -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"

# Langfuse 관측성 확인
echo "Langfuse URL: http://$(kubectl get ingress -n langfuse langfuse -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
:::

::alert[**플랫폼 필요**: 모든 Pod가 "Running" 상태여야 합니다. 구성요소가 누락된 경우 먼저 모델과 상호작용하기 & 모듈 A1을 완료하십시오.]{type="warning"}

## 단계 1: 구성 이해

배포하기 전에 Loan Buddy가 GenAI 플랫폼에 연결하는 데 필요한 구성을 검토합니다:

### 필요한 구성

1. **S3 Bucket** - 업로드된 대출 문서를 저장할 위치
   - Langfuse 배포 시 Terraform이 자동으로 생성
   - 다음에서 확인: `terraform output langfuse_bucket_name`

2. **AWS Region** - 리소스가 위치한 리전
   - `.env` 파일에서 `REGION`으로 구성
   - S3 및 기타 AWS 서비스 호출에 사용

3. **LiteLLM API Key** - 모듈 A1의 Virtual Key
   - 에이전트가 게이트웨이를 통해 LLM 모델을 호출할 수 있게 합니다
   - 모듈 A1, 단계 9에서 생성

4. **Langfuse Credentials** - 관측성 및 추적용
   - `.env` 파일의 공개 및 비밀 키
   - AI 결정에 대한 완전한 감사 추적을 지원합니다

5. **Service Account with S3 Access** - Kubernetes RBAC + AWS IAM
   - Langfuse 서비스 계정을 워크샵 네임스페이스로 복사
   - S3 권한을 위한 Pod Identity Association 생성
   - Pod가 하드코딩된 자격 증명 없이 S3에 안전하게 접근할 수 있습니다

### 자격 증명 확인

먼저 모듈 A1의 Virtual Key가 있는지 확인합니다:

:::code{language=bash showCopyAction=true}
# 모듈 A1의 AGENT_KEY가 있는지 확인
if [ -z "$AGENT_KEY" ]; then
  echo "AGENT_KEY not found. Please set it from Module A1"
  echo "Run: export AGENT_KEY='your-sk-key-from-module-a1'"
else
  echo "AGENT_KEY found: ${AGENT_KEY:0:10}..."
fi
:::

::alert[**AGENT_KEY가 없는 경우** 모듈 A1, Virtual Key 섹션의 단계 3으로 돌아가서 키를 복사하십시오. 그런 다음 실행합니다: `export AGENT_KEY="sk-your-actual-key"`]{type="warning"}

## 단계 2: 자동 배포

### 배포 스크립트 실행

자동 배포 스크립트가 모든 구성을 처리합니다:

:::code{language=bash showCopyAction=true}
# credit validation 디렉토리로 이동
cd /workshop/workshops/eks-genai-workshop/static/code/module3/credit-validation

# 스크립트를 실행 가능하게 만들기
chmod +x deploy-workshop-app.sh

# 배포 스크립트 실행
./deploy-workshop-app.sh
:::

### 스크립트가 수행하는 작업

스크립트는 다음 단계를 자동화합니다:

1. **사전 요구사항 검증** - kubectl 및 클러스터 연결 확인
2. **S3 Bucket 검색** - Terraform 출력에서 버킷 이름 가져오기
3. **AWS Region 가져오기** - .env 파일에서 로드
4. **자격 증명 구성** - LiteLLM 및 Langfuse 키 설정
5. **워크샵 네임스페이스 생성** - 애플리케이션을 위한 격리된 환경
6. **서비스 계정 설정** - S3 권한이 있는 Langfuse SA 복사
7. **Pod Identity Association 생성** - 워크샵 Pod에 S3 접근 권한 부여
8. **배포 구성** - YAML의 모든 플레이스홀더 교체
9. **애플리케이션 배포** - 구성된 Kubernetes 매니페스트 적용
10. **준비 대기** - 모든 Pod가 실행 중인지 확인

::alert[**학습 기회**: 스크립트는 모든 값이 채워진 `agentic-application-deployment.configured.yaml` 파일을 생성합니다. 구성이 어떻게 동작하는지 확인하려면 이 파일을 검토하십시오.]{type="info"}

::::expand{header="내부 동작: 자동화되는 내용"}

**스크립트 없이는 다음을 수행해야 합니다:**

1. **S3 Bucket Name 가져오기:**
   ```bash
   cd /workshop/components/o11y/langfuse
   terraform output langfuse_bucket_name
   ```

2. **AWS Region 가져오기:**
   ```bash
   # .env 파일은 인프라 설정 중에 자동 생성되며
   # REGION, CLUSTER_NAME 및 기타 워크샵 변수를 포함합니다
   source /workshop/.env
   echo $REGION
   ```

3. **YAML 파일 수동 편집:**
   - `<>`를 S3 버킷 이름으로 교체 (2곳)
   - `<>`를 AWS 리전으로 교체 (2곳)
   - Langfuse 키 교체 (2곳)
   - LiteLLM 키 교체 (2곳)

4. **서비스 계정 생성:**
   ```bash
   kubectl get serviceaccount langfuse -n langfuse -o yaml | \
     sed 's/namespace: langfuse/namespace: workshop/' | \
     kubectl apply -f -
   ```

5. **Pod Identity Association 생성:**
   ```bash
   aws eks create-pod-identity-association \
     --cluster-name genai-on-eks \
     --namespace workshop \
     --service-account langfuse \
     --role-arn <IAM_ROLE_ARN>
   ```

6. **애플리케이션 배포:**
   ```bash
   kubectl apply -f agentic-application-deployment.yaml
   ```

**스크립트는 이 모든 작업을 하나의 명령으로 수행합니다.** 이는 프로덕션 CI/CD 파이프라인에서 사용되는 것과 동일한 자동화 패턴입니다.

::::

### 예상 출력

다음과 유사한 출력이 표시됩니다:

```
  Loan Buddy Workshop Deployment

Step 1: Checking prerequisites...
kubectl found
Connected to Kubernetes cluster

Step 2: Retrieving S3 bucket name from Terraform...
S3 Bucket: genai-on-eks-bucket-langfuse-xxxxx

Step 3: Getting AWS region...
AWS Region: us-west-2

Step 4: Getting LiteLLM API key...
LiteLLM API Key: sk-xxxxx...

Step 5: Getting Langfuse credentials...
Langfuse Public Key: lf_pk_xxxxx...
Langfuse Secret Key: lf_sk_xxxxx...

Step 6: Creating workshop namespace...
Created namespace 'workshop'

Step 7: Setting up service account for S3 access...
Service account 'langfuse' created in workshop namespace

Step 7b: Creating Pod Identity Association for S3 access...
Pod Identity Association created for workshop namespace

Step 8: Configuring deployment file...
Deployment file configured

Step 9: Deploying Loan Buddy application...
Application deployed

Step 10: Waiting for deployments to be ready...
All deployments ready

Step 11: Verifying deployment...
NAME                                     READY   STATUS    RESTARTS   AGE
loan-buddy-agent-xxx                     1/1     Running   0          1m
mcp-address-validator-xxx                1/1     Running   0          1m
mcp-employment-validator-xxx             1/1     Running   0          1m
mcp-image-processor-xxx                  1/1     Running   0          1m

  Deployment Complete!
```

## 단계 3: AI 동작 확인

### 실시간 모니터링 설정

두 번째 터미널을 열어 에이전트의 처리 과정을 관찰합니다:

:::code{language=bash showCopyAction=true}
# 새 터미널에서 에이전트 로그 관찰
kubectl logs -f deployment/loan-buddy-agent -n workshop
:::

**이 터미널을 계속 표시해 두십시오** - AI가 실시간으로 결정을 내리는 과정을 확인할 수 있습니다.

### 애플리케이션 노출

메인 터미널로 돌아갑니다:

:::code{language=bash showCopyAction=true}
# Loan Buddy를 접근 가능하게 만들기
kubectl port-forward service/loan-buddy-agent 8888:8080 -n workshop &

# 테스트 데이터 디렉토리로 이동
cd /workshop/workshops/eks-genai-workshop/static/code/module3/credit-validation/
:::

::alert[**API 전용 서비스**: Loan Buddy는 FastAPI로 구축된 REST API 서버입니다. 웹 UI가 없으며 모든 상호작용은 API 호출(예\: `curl`)을 통해 이루어집니다. 브라우저에서 `http://localhost:8888`에 접속하면 `{"detail":"Not Found"}`가 표시되는데, 이는 정상입니다. API 문서는 `http://localhost:8888/docs` (Swagger UI)에서 확인할 수 있습니다.]{type="info"}

::alert[**8888 포트를 사용하는 이유**: 워크샵의 Code Editor 서버가 이미 8080 포트를 사용하고 있습니다. 충돌을 방지하기 위해 Loan Buddy는 로컬 포트 8888로 포워딩하며, 컨테이너 내부에서는 여전히 8080 포트로 동작합니다.]{type="info"}

## 단계 4: John의 대출 신청서 처리

### John의 신청서 제출

::alert[이 명령은 JSON 형식화를 위해 `jq`를 사용하며, 워크샵 환경에 사전 설치되어 있습니다.]{type="info"}

:::code{language=bash showCopyAction=true}
# John Michael Doe의 대출 신청서 처리
curl -X POST -F "image_file=@./example1.png" \
  http://localhost:8888/api/process_credit_application_with_upload \
  | (data=$(cat); echo "$data" | jq -C 'del(.credit_assessment) + {"credit_assessment": "(formatted below)"}'; echo -e "\n\033[1;36m═══ CREDIT ASSESSMENT ═══\033[0m\n"; echo "$data" | jq -r '.credit_assessment' | sed 's/\\n/\n/g')
:::

### 처리 과정 관찰

**로그 터미널에서 다음을 확인할 수 있습니다:**

```
Starting credit application processing...
Processing uploaded loan application image...
Image stored in S3 with ID: a7b9c2d4e5f6...
Loading MCP tools...
Processing with agent...

[Agent]: I'll help you process this credit application. Let me start by extracting the information from the document.

[Image Processor MCP]: Extracting data from loan application...
Found: John Michael Doe, DOB: 03/15/1985, SSN: ***-**-7890
Income: $75,000, Employer: Tech Solutions Inc
Address: 123 Main Street, Anytown, CA 90210

[Agent]: Now validating the address information...

[Address Validator MCP]: Checking 123 Main Street, Anytown, CA 90210
Valid residential address - Single family home
Owner occupied - 4 years at address
Risk score: 15 (Low risk)

[Agent]: Verifying employment and income...

[Employment Validator MCP]: Verifying Tech Solutions Inc
Employment confirmed - Software Engineer
Income verified - $75,000 annual
Employment duration - 3.5 years
Stability score: 85 (High stability)

[Agent]: Calculating loan metrics...
- Loan amount: $8,500
- Monthly payment: $275
- Monthly income: $6,250
- Debt-to-income ratio: 4.4% (Excellent)

FINAL DECISION: APPROVED
- Low risk profile
- Stable employment
- Excellent debt-to-income ratio
- Home ownership indicates stability
```

### 응답 검토

curl 명령은 완전한 평가가 포함된 JSON 응답을 반환합니다:

```json
{
  "status": "COMPLETED",
  "image_id": "a7b9c2d4e5f6...",
  "credit_assessment": "APPROVED - John Michael Doe's application for $8,500 home improvement loan is approved. Verified employment at Tech Solutions Inc ($75,000/year), confirmed residential address ownership, excellent debt-to-income ratio of 4.4%.",
  "processing_note": "Complete audit trail available in Langfuse"
}
```

## 단계 5: Langfuse에서 전체 워크플로우 탐색

### Langfuse 대시보드 열기

1. **Langfuse URL 확인** (사전 요구사항 확인 단계에서 조회)
2. **브라우저에서 열기**
3. **왼쪽 사이드바에서 "Traces"로 이동**

### John의 신청서 추적 찾기

다음 조건에 해당하는 추적을 찾습니다:
- **Name**: `credit_underwriting_agent_with_image_id`
- **Recent timestamp** (방금 처리됨)

### 추적에서 확인할 수 있는 내용

![Langfuse 추적 예시](/static/images/track-a-opensource/a2-genai-applications/LoanBuddy-Observability.png)
*AI의 의사결정 프로세스에 대한 완전한 가시성*

**추적에 표시되는 내용:**
- **완전한 워크플로우** - 업로드에서 결정까지 모든 단계
- **모든 도구 호출** - 이미지 추출, 주소 검증, 고용 확인
- **AI 추론** - 대출이 승인된 이유
- **성능 메트릭** - 각 단계의 처리 시간, 지연시간
- **비용 추적** - 사용된 토큰, 예상 비용
- **전체 감사 추적** - 규정 준수를 위한 완전한 기록

### 메트릭 분석

Langfuse에서 추적을 클릭하여 다음을 확인합니다:
- **Total tokens**: 약 2,500 토큰 사용
- **Processing time**: 총 45-60초
- **Tool calls**: 3개의 MCP 서버 호출
- **Decision path**: 완전한 추론 체인

## 단계 6: 다양한 시나리오 시도 (선택 사항)

### 두 번째 신청서로 테스트

다른 테스트 신청서가 있는 경우:

:::code{language=bash showCopyAction=true}
# 다른 신청서 처리
curl -X POST -F "image_file=@./example2.png" \
  http://localhost:8888/api/process_credit_application_with_upload
:::

### 엣지 케이스 실험

MCP 서버 모의 데이터를 수정하여 다른 결과를 확인할 수 있습니다:
- 고용 확인이 실패하는 경우
- 주소가 유효하지 않은 경우
- 소득이 일치하지 않는 경우

## 요약

### 수행한 작업

다음을 완료했습니다:
- Kubernetes에 완전한 AI 대출 처리 시스템 **배포**
- 1분 이내에 John의 대출 신청서 **처리**
- 2-3시간의 수동 작업 **자동화**
- Langfuse에 완전한 감사 추적 **생성**
- 모델과 상호작용하기 & 모듈 A1의 GenAI 플랫폼과 **통합**

### 비즈니스 영향

이 모듈의 시작 부분에서 소개한 Sarah의 사례를 되짚어 보겠습니다.

**Loan Buddy 이전:**
- John의 데이터를 수동으로 입력하는 데 30분
- Tech Solutions Inc에 전화하는 데 45분
- 주소 확인에 20분
- 비율 계산 및 문서화에 30분
- **총**: 2시간 이상

**Loan Buddy 이후:**
- 완전한 처리: 47초
- 모든 검증: 자동화됨
- 전체 문서화: 즉시
- 감사 추적: 완료

## 주요 요점

다음을 학습했습니다:
1. 실제 비즈니스 문제를 해결하는 **AI 애플리케이션 구축**
2. 복잡한 워크플로우 오케스트레이션을 위한 **LangGraph 사용**
3. 전문화된 기능을 위한 **MCP 서버 구현**
4. **플랫폼과 통합** (LiteLLM + Langfuse)
5. 적절한 구성으로 **Kubernetes에 배포**
6. 완전한 관측성으로 **AI 결정 모니터링**

다음을 수행하는 AI 기반 대출 처리 시스템을 구축하고 배포했습니다:
- 수 시간의 수동 작업 절감
- 일관된 결정 제공
- 완전한 감사 추적 유지
- GenAI 플랫폼과 통합

---

**[다음: 워크샵 요약 및 다음 단계 →](../../summary/)**
