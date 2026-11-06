---
title: "워크샵 요약 및 다음 단계"
weight: 70
---

# 워크샵 요약 및 다음 단계

**Amazon EKS에서 고급 Agentic AI 플랫폼 설계 및 배포** 워크샵을 완료하셨습니다. 이 워크샵을 통해 Amazon EKS에 완전한 Agentic AI 플랫폼을 구축하고 이를 사용하여 비즈니스 애플리케이션을 구현했습니다. 이 실습 과정을 통해 프로덕션 환경에서 Agentic AI 솔루션을 배포하고 확장하는 실용적인 기술을 습득했습니다.

## 완료한 내용

이 워크샵 과정에서 다음을 완료했습니다:

### 모델과 상호작용하기 _(공통 기반)_
**EKS에서 실제 LLM 배포 및 상호작용**
- ChatGPT와 유사한 인터페이스인 Open WebUI 설정
- AWS Neuron 칩에서 Qwen 3 8B 및 DeepSeek R1 Qwen 3 8B를 서빙하는 vLLM 탐색
- AWS Bedrock을 통해 Claude Sonnet 4.5 통합
- 자체 호스팅 vs. 관리형 모델 배포 패턴 비교
- LLM 호스팅에 필요한 Kubernetes 리소스 이해

### Track A · 모듈 A1: GenAI 플랫폼 컴포넌트
**포괄적인 관측성을 갖춘 통합 Agentic AI 플랫폼 구축**
- 모델 전반에 걸쳐 요청을 라우팅하는 통합 API 게이트웨이로 LiteLLM 구성
- 완전한 관측성 및 추적을 위한 Langfuse 구현
- Helm 구성을 통해 플랫폼에 새 모델 추가
- 비용 추적, 성능 모니터링, 사용량 분석 탐색
- 플랫폼 컴포넌트가 통합되는 방식 이해

### Track A · 모듈 A2: Agentic AI 애플리케이션 구축
**대출 처리 애플리케이션 생성**
- "Loan Buddy" 배포 - 대출 검토를 자동화하는 AI 에이전트
- 복잡한 의사결정을 위한 LangGraph 워크플로우 구현
- 외부 도구를 위한 Model Context Protocol (MCP) 서버 통합:
  - 주소 검증
  - 고용 확인
  - 이미지 처리 및 데이터 추출
- Langfuse에서 전체 워크플로우 실행 관찰
- 수동 검토 프로세스를 2-3시간에서 약 2분으로 단축

### Track A · 모듈 A3: 플랫폼 확장 및 평가
**품질 검증 및 프로덕션 트래픽을 위한 플랫폼 확장**
- Langfuse에서 LLM-as-a-Judge 평가를 구성하여 에이전트 출력을 자동으로 채점
- LiteLLM에서 A/B 모델 라우팅 설정 (Claude vs Qwen 가중치 비교)
- 인프라 모니터링을 위한 Amazon Managed Prometheus (AMP) 및 Amazon Managed Grafana (AMG) 배포
- Agentic AI 특화 대시보드 구축 (토큰 처리량, 추론 지연시간, Neuron 활용률)
- HPA 오토스케일링 구성 및 부하 상황에서 Karpenter 노드 프로비저닝 관찰

### Track A · 모듈 A4: 프로덕션 준비 애플리케이션
**Loan Buddy를 엔터프라이즈 요구사항에 맞게 준비**
- Milvus 벡터 데이터베이스 배포 및 Bedrock Titan Embeddings로 RAG 파이프라인 구축
- 실제 문서에 기반한 정책 인식 결정을 내리도록 Loan Buddy 개선
- Ragas 메트릭을 사용하여 RAG 품질 평가 (충실도, 관련성, 정확도, 재현율)
- 하드코딩된 자격 증명을 External Secrets Operator + AWS Secrets Manager로 대체
- 최소 권한 IAM 액세스를 위한 EKS Pod Identity 구성
- 콘텐츠 안전 및 PII 익명화를 위한 Bedrock Guardrails 추가
- 실시간 에이전트 응답을 위한 SSE 스트리밍 구현
- 규정 준수를 위한 Pydantic 스키마로 출력 검증
- 전체 프로덕션 준비 체크리스트 완료

::alert[위 성과는 **Track A(오픈소스 스택)** 경로에 해당합니다. **Track B(SaaS 스택)** 를 진행하셨다면 아래를 참고하십시오.]{type="info"}

### Track B: SaaS 스택 (Kong Konnect + Arize AX + Strands)
**관리형 SaaS 컨트롤 플레인에서 동일한 플랫폼과 에이전트 구축** _(Track B를 진행한 경우)_
- **Kong Konnect** 컨트롤 플레인을 등록하고 상태 비저장 **DataPlane** 을 클러스터 내에서 실행 (Kong Operator 2.x)
- Amazon Bedrock `ai-proxy` 라우트를 **네이티브 Kubernetes CRD**(`KongService`/`KongRoute`/`KongPlugin`/`KongConsumer`)로 구성 — decK 없이 GitOps 친화적
- **Arize AX**(SaaS)를 설정하고 **OpenInference** 트레이스(AGENT/CHAIN/LLM/TOOL 스팬)를 내보내기
- **Strands Agents SDK** 로 Loan Buddy 에이전트를 구축하여 **Kong 경유** 로 Bedrock 호출 (애플리케이션은 S3 IAM 만 보유 — Bedrock 키 없음)
- Track A와 **동일한 3개 MCP 도구 서버** 를 재사용하여 MCP가 프레임워크에 종속되지 않음을 입증

## 다룬 핵심 기술

### Infrastructure & Platforms
- **Amazon EKS Auto Mode**: 자동 확장 기능이 있는 완전 관리형 Kubernetes
- **AWS Neuron**: 비용 효율적인 AI 추론을 위한 특수 목적 칩
- **Amazon EFS**: 모델 캐싱 및 지속성을 위한 공유 스토리지
- **Helm Charts**: 선언적 배포 및 구성 관리

### Agentic AI Stack
- **vLLM**: 연속 배치 처리를 갖춘 고성능 추론 서버
- **LiteLLM**: 여러 LLM 제공자를 지원하는 통합 게이트웨이
- **Langfuse**: LLM 애플리케이션을 위한 포괄적인 관측성
- **Open WebUI**: 모델 상호작용을 위한 채팅 인터페이스
- **Milvus**: RAG 파이프라인을 위한 클라우드 네이티브 벡터 데이터베이스
- **Amazon Managed Prometheus/Grafana**: 인프라 수준 모니터링

### Application Frameworks
- **LangChain**: LLM 애플리케이션을 위한 구성 요소
- **LangGraph**: AI 에이전트를 위한 워크플로우 오케스트레이션
- **Model Context Protocol (MCP)**: 도구 통합을 위한 표준
- **Ragas**: RAG 평가 프레임워크 (충실도, 관련성, 정확도)
- **Pydantic**: 데이터 검증 및 구조화된 출력 스키마

### Security & Production
- **EKS Pod Identity**: Kubernetes Pod를 위한 세밀한 IAM
- **External Secrets Operator**: 관리형 자격 증명 동기화
- **Bedrock Guardrails**: 콘텐츠 안전 및 PII 보호

## 비즈니스 성과

"Loan Buddy" 애플리케이션은 측정 가능한 비즈니스 가치를 보여주었습니다:

- **속도**: 처리 시간을 2-3시간에서 2분으로 단축 (99% 개선)
- **일관성**: 모든 신청서에 걸쳐 표준화된 의사결정
- **비용 절감**: 수동 검토 프로세스 자동화
- **규정 준수**: 모든 결정에 대한 완전한 감사 추적
- **경험**: 직원과 고객 모두에게 개선된 서비스

동일한 패턴을 다음과 같은 다양한 비즈니스 프로세스에 적용할 수 있습니다:
- 문서 처리 및 추출
- 고객 서비스 자동화
- 품질 보증 워크플로우
- 규정 준수 모니터링

## 다음 단계

### 즉시 수행할 수 있는 작업

#### 1. 자신의 환경에 배포
**[AWS GenAI on EKS Starter Kit](https://github.com/aws-samples/sample-genai-on-eks-starter-kit)** 을 사용하여:
- AWS 계정에 동일한 플랫폼 배포
- 사용 사례에 맞게 구성 커스터마이징
- 요구사항에 따라 확장
#### 2. llm-d를 활용한 고급 모델 서빙 탐색
**[AWS GenAI on EKS Starter Kit](https://github.com/aws-samples/sample-genai-on-eks-starter-kit)** 에는 Kubernetes 기반 분리형 LLM 서빙을 위한 오픈소스 프레임워크인 **llm-d** 가 포함되어 있습니다. 분리형 서빙(Disaggregated Serving)은 프리필(프롬프트 처리)과 디코드(토큰 생성) 단계를 독립적으로 확장 가능한 컴포넌트로 분리하여, 대규모 환경에서 처리량과 지연시간을 크게 개선합니다. 이는 본 워크샵에서 다룬 패턴을 넘어서는 LLM 추론 최적화의 다음 단계입니다.

### 학습 리소스

#### 공식 문서
- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Amazon Bedrock Documentation](https://docs.aws.amazon.com/bedrock/)
- [LangChain Documentation](https://python.langchain.com/)
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/)
- [vLLM Documentation](https://docs.vllm.ai/)
- [LiteLLM Documentation](https://docs.litellm.ai/)
- [Langfuse Documentation](https://langfuse.com/docs)
- [Milvus Documentation](https://milvus.io/docs)
- [Ragas Documentation](https://docs.ragas.io/)

## 마무리

Amazon EKS 기반 Agentic AI 워크샵을 완료했습니다. 이 워크샵을 통해 다음을 수행했습니다:

- 처음부터 완전한 Agentic AI 플랫폼 구축
- 특수 하드웨어에 실제 LLM 배포
- 관측성을 갖춘 통합 API 게이트웨이 생성
- AI 에이전트 기반 비즈니스 애플리케이션 개발
- 품질 평가 및 프로덕션 트래픽을 위한 확장
- Milvus 및 Ragas 평가를 사용한 RAG 파이프라인 구축
- Pod Identity, External Secrets, Guardrails로 보안 강화
- 최신 AI 기술에 대한 실습 경험 획득

배운 기술과 패턴은 고객 서비스 봇, 문서 프로세서, 복잡한 AI 에이전트 등 실제 시나리오에 직접 적용할 수 있습니다.

---

*이 워크샵에 참여해 주셔서 감사합니다. 질문, 피드백 또는 지원이 필요하면 AWS 커뮤니티에 참여하거나 [AWS GenAI on EKS Starter Kit](https://github.com/aws-samples/sample-genai-on-eks-starter-kit) 저장소를 참조하십시오.*
