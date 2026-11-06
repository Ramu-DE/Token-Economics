---
title: "인프라 설정"
weight: 18
---

Agentic AI 애플리케이션을 구축하고 배포하기 전에, 이 섹션에서는 워크샵 환경에 사전 구성된 인프라를 살펴봅니다. 이 기반은 Amazon EKS에서 생성형 AI 워크로드를 실행하는 데 필요한 환경을 제공합니다.

::alert[모든 인프라 컴포넌트가 사전 배포 및 구성되어 있습니다. 별도의 설정 작업 없이 Agentic AI 애플리케이션 학습 및 실험에 집중할 수 있습니다.]{type="success"}

## 사전 구성된 인프라 개요

워크샵 환경에는 다음 컴포넌트가 포함된 완전히 작동하는 Amazon EKS 클러스터가 있습니다:

### Amazon EKS Cluster with Auto Mode

워크샵은 **Amazon EKS Auto Mode** 를 사용합니다. 이는 컴퓨팅, 스토리지, 네트워킹 리소스 관리의 운영 오버헤드를 줄여주는 관리형 AWS Kubernetes 서비스입니다.

#### EKS Auto Mode란?

EKS Auto Mode는 AWS가 다음을 처리하는 완전 관리형 Kubernetes 경험을 제공합니다:

- **자동 컴퓨팅 프로비저닝** - EC2 인스턴스를 수동으로 생성하거나 관리할 필요 없음
- **동적 확장** - 워크로드 수요에 따라 용량을 자동으로 조정
- **패치 관리** - OS 및 Kubernetes 컴포넌트 자동 업데이트
- **비용 최적화** - 지능형 인스턴스 선택 및 Spot 인스턴스 통합

:::code{language=bash showCopyAction=true}
# View your EKS cluster details
kubectl cluster-info

# Check the EKS Auto Mode configuration
kubectl get nodes -L eks.amazonaws.com/compute-type
:::

### Node Pools 구성

클러스터에는 다양한 워크로드 유형에 최적화된 특수 노드 풀이 포함되어 있습니다:

#### 1. **General Purpose Nodes (Multi-Architecture)**
- **Instance Types**: 최신 세대 컴퓨팅 (C), 메모리 (M), 범용 (R) 최적화 인스턴스
  - **AWS Graviton (ARM64)**: 최대 40% 향상된 가격 대비 성능을 제공하는 ARM 기반 인스턴스
  - **x86 (AMD/Intel)**: 기존 x86 아키텍처 인스턴스
- **Architecture Support**: 워크로드 호환성에 따라 자동 선택되는 멀티 아키텍처 (amd64/arm64)
- **Purpose**: 일반 워크로드, 웹 서비스, 제어 플레인 컴포넌트 실행
- **Cost Optimization**: 
  - Graviton 인스턴스는 비슷한 x86 대비 최대 40% 향상된 가격 대비 성능 제공
  - 추가 절감을 위한 Spot 및 On-Demand 용량 유형
  - Karpenter는 사용 가능한 옵션에서 가장 비용 효율적인 인스턴스 유형을 자동으로 선택
- **Scaling**: Pod 요구사항에 따라 자동으로 확장되고 활용도가 낮은 노드를 통합

#### 2. **GPU-Accelerated Nodes** 
- **Instance Types**: NVIDIA GPU (g5, g6, p4 및 p5 패밀리)
- **Purpose**: GPU 집약적 AI/ML 워크로드 실행
- **Features**: CUDA 지원, GPU 오퍼레이터 사전 설치

#### 3. **AWS Neuron Nodes**
- **Instance Types**: inf2, trn1 및 trn2 인스턴스
- **Purpose**: 대규모 언어 모델을 위한 최적화된 추론
- **Benefits**: 
  - GPU 대비 추론당 최대 50% 낮은 비용
  - 최적 성능을 위한 사전 컴파일된 모델
  - 네이티브 int8 양자화 지원

:::code{language=bash showCopyAction=true}
# View available node pools and their configurations
kubectl get nodepools -o wide
:::

### Storage 구성

클러스터는 영구 스토리지로 **Amazon EFS** (Elastic File System)를 사용하며, 다음을 제공합니다:

#### EFS Storage Class Features:
- **Shared Storage**: 여러 Pod가 동시에 읽기/쓰기 가능
- **Elastic Throughput Mode**: 워크로드에 따라 성능을 자동으로 확장
- **Model Caching**: 빠른 배포를 위해 EFS에 저장된 사전 다운로드된 모델
- **Cross-AZ Availability**: 여러 가용 영역에 걸친 고가용성

:::code{language=bash showCopyAction=true}
# View available storage classes
kubectl get storageclass

# Check EFS persistent volumes
kubectl get pv | grep efs
:::

::alert[EFS 스토리지는 Qwen 3 8B 및 DeepSeek R1 Qwen 3 8B 모델을 위한 모델 캐시로 사전 구성되어 새 모델 인스턴스의 배포 시간을 크게 단축합니다.]{type="info"}

### 사전 설치된 Agentic AI 스택

다음 Agentic AI 컴포넌트가 배포되어 사용할 준비가 되어 있습니다:

#### **1. Open WebUI**
대규모 언어 모델과 상호작용하기 위한 웹 인터페이스입니다.

- **URL**: 워크샵 도메인에서 사용 가능
- **Features**: 채팅 인터페이스, 대화 기록, 모델 전환
- **Integration**: LiteLLM 게이트웨이와 작동하도록 사전 구성됨

#### **2. LiteLLM**
여러 LLM 제공자에 대한 단일 인터페이스를 제공하는 통합 API 게이트웨이입니다.

- **Purpose**: 다양한 모델 제공자 간 API 호출 표준화
- **Supported Models**: vLLM, AWS Bedrock, OpenAI 호환 엔드포인트
- **Load Balancing**: 사용 가능한 모델 전반에 걸쳐 요청을 자동으로 분산

#### **3. Langfuse**
LLM 애플리케이션을 위한 관측성 플랫폼입니다.

- **Monitoring**: 토큰 사용량, 지연시간, 비용 추적
- **Debugging**: 전체 스택을 통해 개별 요청 추적
- **Analytics**: 모델 성능 및 사용자 상호작용 파악

#### **4. vLLM**
대규모 언어 모델을 위한 고성능 추론 엔진입니다.

- **Pre-loaded Models**:
  - Qwen 3 8B (Neuron용으로 컴파일됨)
  - DeepSeek R1 Qwen 3 8B (Neuron용으로 컴파일됨)
- **Optimization**: 연속 배치 처리, PagedAttention, 텐서 병렬화
- **Hardware Acceleration**: AWS Neuron 칩에 최적화됨

:::code{language=bash showCopyAction=true}
# Check deployed Agentic AI components
kubectl get pods -A | grep -E "litellm|langfuse|openwebui|vllm"
:::

:::code{language=bash showCopyAction=true}
# View service endpoints
kubectl get svc -A | grep -E "litellm|langfuse|openwebui|vllm"
:::

## 성공 체크리스트

모델과 상호작용하기로 진행하기 전에 다음을 확인하십시오:

- **AWS Console** 액세스 작동
- **VSC IDE** 열림 및 터미널 준비
- **kubectl**: EKS 클러스터에 연결됨
- **All pods**: Running 상태

## 주요 요점

::::tabs

:::tab{label="Infrastructure"}
- **EKS Auto Mode**: 기반 인프라 관리
- **특수 노드 풀**: 다양한 워크로드 유형에 최적화
- **EFS 스토리지**: 모델을 위한 공유, 확장 가능한 스토리지 제공
:::

:::tab{label="Agentic AI Stack"}
- **Open WebUI**: 채팅 인터페이스 제공
- **LiteLLM**: 여러 모델 제공자에 대한 통합 액세스
- **Langfuse**: 관측성 제공
- **vLLM**: 고성능 모델 서빙 제공
:::

:::tab{label="Benefits"}
- **설정 시간 불필요** - 구성이 아닌 학습에 집중
- **프로덕션 패턴** - 실제 배포에 사용되는 동일한 패턴
- **비용 최적화** - Spot 인스턴스 및 Neuron 칩 사용
- **완전 통합** - 모든 컴포넌트가 함께 작동
:::

::::

## 모델과 상호작용하기 준비 완료

Agentic AI 환경이 검증되고 준비되었습니다. 다음을 갖추었습니다:

- 완전히 구성된 EKS 클러스터에 대한 액세스
- Agentic AI 플랫폼 컴포넌트 실행 및 확인됨
- 개발 환경 설정
- 모든 도구 및 액세스 확인됨

이제 실습 Agentic AI 과정을 시작할 준비가 되었습니다.

---

**[모델과 상호작용하기 시작 →](/module-interacting-with-models/)**
