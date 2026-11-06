---
title: "vLLM - 자체 호스팅 모델 서빙"
weight: 22
---

Open WebUI에서 상호작용한 모델은 Kubernetes에서 실행되는 vLLM이 서빙합니다. 이 섹션에서는 해당 대화를 지원하는 vLLM 인프라를 살펴봅니다.

## 실습: 실행 중인 모델 탐색

클러스터에서 무엇이 실행되고 있는지 확인하는 것부터 시작합니다:

### 1단계: 실행 중인 모델 확인

:::code{language=bash showCopyAction=true}
# 현재 실행 중인 vLLM 모델 확인
kubectl get pods -n vllm

# 채팅 경험 뒤의 실제 deployments 확인
kubectl get deployments -n vllm -o wide

# 모델을 호스팅하는 노드 확인
kubectl get pods -n vllm -o wide
:::

`qwen3-8b-neuron-xxx` 및 `deepseek-r1-qwen3-8b-neuron-xxx`와 같은 Pod가 표시됩니다. 이것이 Open WebUI에서 사용한 모델입니다.

### 2단계: 실제 구성 파일 살펴보기

VSC IDE에서 실제 배포 파일을 확인합니다:

:::code{language=bash showCopyAction=true}
# vLLM 구성으로 이동
ls /workshop/components/llm-model/vllm/

# 방금 사용한 Qwen 3 배포 살펴보기
cat /workshop/components/llm-model/vllm/model-qwen3-8b-neuron.rendered.yaml

# DeepSeek 배포와 비교
cat /workshop/components/llm-model/vllm/model-deepseek-r1-qwen3-8b-neuron.rendered.yaml
:::

### 3단계: 채팅 경험과 인프라 연결

Qwen 3 8B에서 받은 응답은 다음 경로를 따랐습니다:

1. **사용자의 메시지** → Open WebUI → LiteLLM → **이 vLLM Pod**
2. 지금 확인하고 있는 **Pod** 가 AWS Neuron 하드웨어에서 요청을 처리했습니다
3. **응답** 이 같은 경로를 통해 브라우저로 돌아왔습니다

## vLLM이란 무엇입니까?

[vLLM](https://docs.vllm.ai/en/latest/)은 다음을 제공하는 오픈소스 LLM 추론 및 서빙 라이브러리입니다:

- **높은 처리량**: 여러 요청을 효율적으로 서빙하도록 최적화
- **Continuous Batching**: 최적의 하드웨어 활용을 위한 동적 요청 배칭
- **PagedAttention**: 긴 컨텍스트를 위한 효율적인 메모리 관리
- **Tensor Parallelism**: 여러 가속기에 모델 분산
- **OpenAI 호환 API**: OpenAI API의 드롭인 대체

## 실시간 모델 모니터링

모델이 요청을 처리하는 과정을 실시간으로 관찰합니다.

### 4단계: 모델의 요청 처리 관찰

VSC IDE에서 두 번째 터미널을 열고 실행합니다:

:::code{language=bash showCopyAction=true}
# Deepseek Qwen 3 모델 로그를 실시간으로 관찰
kubectl logs -f --tail=0 -n vllm deployment/deepseek-r1-qwen3-8b-neuron
:::

이제 Open WebUI 탭으로 돌아가서 모델을 Deepseek 모델로 변경하고 메시지를 보냅니다. 로그에서 요청이 실시간으로 처리되는 것을 확인할 수 있습니다.

**다음 예시 질문을 시도해 보십시오:**

![Open WebUI Question](/static/images/module-interacting-with-models/flies.png)

"why would a fly fly into a fly pant" 메시지를 보내자마자 터미널을 확인하십시오. 다음을 보여주는 상세한 로그가 표시됩니다:

![vLLM Processing Logs](/static/images/module-interacting-with-models/logs.png)

**로그에서 확인할 수 있는 내용:**
- **요청 수신**: vLLM이 처리하는 프롬프트
- **모델 처리**: 토큰 생성 및 처리 지표
- **성능 통계**: 처리량, 지연시간 및 캐시 사용량
- **실시간 업데이트**: 실시간으로 생성되는 각 토큰

**주목할 주요 지표:**
- **Prompt throughput**: ~1.3 tokens/s (질문을 읽는 속도 - 입력이 짧아 낮게 표시됨)
- **Generation throughput**: ~24 tokens/s (응답을 생성하는 속도)
- **GPU KV cache usage**: 메모리 활용률 표시
- **요청 처리**: 시작부터 완료까지의 전체 요청 수명 주기

이 로그를 통해 모델이 인프라 수준에서 어떻게 동작하는지 확인할 수 있습니다.

**탐색을 마쳤으면 Ctrl+C를 눌러 로그를 중지하십시오.**


## AWS Neuron: AI 워크로드를 위한 전용 하드웨어

[AWS Neuron](https://aws.amazon.com/ai/machine-learning/neuron/)은 AWS Inferentia(inf2) 및 Trainium(trn1) 칩을 위한 SDK입니다:

- **비용 효율성**: GPU 대비 추론당 최대 50% 낮은 비용
- **LLM을 위한 최적화**: 트랜스포머 아키텍처에 대한 기본 지원
- **Quantization**: 메모리 사용을 줄이기 위한 INT8 및 FP8 지원
- **컴파일**: 최적의 성능을 위해 사전 컴파일된 모델

::alert[**워크샵 제약**: 워크샵 제한으로 인해 inf2.xlarge 인스턴스(각 2개의 Neuron 코어)를 사용하고 있습니다. 프로덕션 배포는 일반적으로 더 나은 성능을 위해 GPU 또는 더 큰 Neuron 인스턴스를 사용합니다.]{type="warning"}

## 기술 심층 분석 (선택 사항)

Kubernetes 구현 세부 사항은 다음과 같습니다:

::alert[**전체 파일 사용 가능**: 이 탭에 표시된 모든 YAML 매니페스트는 VSC IDE의 `/workshop/components/llm-model/vllm/`에서 확인할 수 있으며, 전체 구성을 자세히 살펴볼 수 있습니다.]{type="info"}

:::::tabs

::::tab{label="Namespace"}
**Namespace 격리**

먼저 vLLM 워크로드를 위한 전용 네임스페이스를 생성합니다:

:::code{language=yaml showCopyAction=true}
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: vllm
:::
::::

::::tab{label="Storage"}

**스토리지 구성**

Amazon EFS를 사용하여 다운로드한 모델과 컴파일된 Neuron 그래프를 캐시합니다:

Hugging Face 모델 캐시:
:::code{language=yaml showCopyAction=true}
# pvc-huggingface-cache.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: huggingface-cache
  namespace: vllm
spec:
  storageClassName: efs
  accessModes:
    - ReadWriteMany  # 여러 Pod가 읽기/쓰기 가능
  resources:
    requests:
      storage: 100Gi
:::

컴파일된 Neuron 모델:
:::code{language=yaml showCopyAction=true}
# pvc-neuron-cache.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: neuron-cache
  namespace: vllm
spec:
  storageClassName: efs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
:::


**두 개의 캐시를 사용하는 이유?**
- **HuggingFace 캐시**: 다운로드한 모델 가중치 저장
- **Neuron 캐시**: 컴파일된 모델 그래프 저장
- **재사용성**: 캐시된 데이터는 Pod 재시작 후에도 유지됨
::::

::::tab{label="Secret"}
**시크릿 관리**

HuggingFace 토큰은 안전하게 저장됩니다:

:::code{language=yaml showCopyAction=true}
# secret.template.yaml
apiVersion: v1
kind: Secret
metadata:
  name: hf-token
  namespace: vllm
type: Opaque
stringData:
  token: ${HF_TOKEN}  # 배포 중 주입됨
:::
::::

::::tab{label="Deployment"}
**주요 배포 매니페스트**

Neuron 최적화를 사용한 Qwen 3 8B 배포:

::alert[주요 세부 사항만 표시하기 위해 배포 매니페스트 파일을 축약했습니다.]{type="warning"}

:::code{language=yaml showCopyAction=true}
# model-qwen3-8b-neuron.rendered.yaml (주요 섹션)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qwen3-8b-neuron
  namespace: vllm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: qwen3-8b-neuron
  template:
    spec:
      # Neuron 하드웨어를 위한 노드 선택
      nodeSelector:
        eks.amazonaws.com/instance-family: inf2
        node.kubernetes.io/instance-type: inf2.xlarge
      
      containers:
        - name: vllm
          image: public.ecr.aws/agentic-ai-platforms-on-k8s/vllm-neuron:qwen3-8b
          command: ["vllm", "serve"]
          args:
            - --served-model-name=qwen3-8b-neuron
            - --trust-remote-code
            - --gpu-memory-utilization=0.90
            - --enable-auto-tool-choice
            - --tool-call-parser=hermes
            - --reasoning-parser=qwen3
            - --tensor-parallel-size=2
            - --max-num-seqs=1
            - --max-model-len=16384
          resources:
            requests:
              cpu: 3
              memory: 12Gi
              aws.amazon.com/neuroncore: 2  # 2개의 Neuron 코어 요청
            limits:
              aws.amazon.com/neuroncore: 2
:::

**주요 구성 세부 사항:**
- **노드 선택**: Pod가 Neuron이 장착된 인스턴스에서만 실행되도록 보장
- **Neuron 코어**: Neuron 코어의 요청 및 제한이 일치해야 함
- **Tensor Parallelism**: 더 빠른 추론을 위해 2개의 코어에 모델 분할
- **동시성**: 8K 컨텍스트 윈도우로 4개의 동시 요청 처리
::::

::::tab{label="Service"}
**서비스 구성**

Service는 vLLM 배포를 노출하고 다른 컴포넌트가 액세스할 수 있도록 합니다:

:::code{language=yaml showCopyAction=true}
apiVersion: v1
kind: Service
metadata:
  name: qwen3-8b-neuron
  namespace: vllm
spec:
  selector:
    app: qwen3-8b-neuron
  ports:
    - name: http
      port: 8000
:::

**서비스 세부 사항:**
- **Selector**: `app: qwen3-8b-neuron` 레이블이 있는 Pod와 일치
- **포트 8000**: OpenAI 호환 엔드포인트를 위한 표준 vLLM API 포트
- **ClusterIP**: 클러스터 내에서만 액세스 가능한 내부 서비스
- **타겟**: vLLM 컨테이너 포트 8000으로 트래픽 라우팅
::::
:::::

---

## 주요 요점

- **Kubernetes 네이티브**: 표준 K8s 리소스(Deployment, Service, PVC)를 사용하여 vLLM 배포
- **Neuron 최적화**: AWS Inferentia를 위해 사전 컴파일되고 양자화된 모델
- **리소스 관리**: 적절한 노드 선택, toleration 및 리소스 제한
- **프로덕션 패턴**: 캐싱, 시크릿 관리 및 보안 컨텍스트
- **성능 트레이드오프**: 비용(하드웨어)과 속도(성능) 간의 균형

## 다음은?

vLLM을 사용하여 EKS에서 모델을 자체 호스팅하는 방법을 살펴봤습니다. 

다음 섹션에서는 인프라 오버헤드 없이 고성능 모델에 즉시 액세스할 수 있는 완전 관리형 대안인 AWS Bedrock을 살펴봅니다.

---

**[다음: AWS Bedrock - 관리형 AI 서비스 →](../Bedrock)**
