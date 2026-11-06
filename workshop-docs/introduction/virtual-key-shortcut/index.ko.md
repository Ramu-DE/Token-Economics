---
title: "Virtual Key 단축 경로 (Advanced)"
weight: 20
---

이 페이지는 **Advanced 경로 단축 페이지** 입니다. 모듈 A2에서 필요한 LiteLLM Virtual Key 단계만 추출하여, 숙련된 참가자가 공통 기반(모델과 상호작용하기)과 모듈 A1을 건너뛰고 바로 에이전트 애플리케이션으로 진입할 수 있도록 구성되어 있습니다.

::alert[**이 페이지 이용 대상**: Kubernetes에서 vLLM, LiteLLM, Langfuse를 이미 이해하고 있으며, 약 2시간의 이벤트 시간 안에 모듈 A2 / A3 / A4에 도달하고자 하는 참가자. 스택이 처음이라면 [Beginner 경로 (모델과 상호작용하기 → A1 → A2)](../)를 따르십시오.]{type="info"}

## 사전 요구사항

단축 경로를 시작하기 전에 워크샵 환경이 준비되었는지 확인하십시오. CloudFormation 스택이 vLLM, LiteLLM, Langfuse, Open WebUI를 자동으로 프로비저닝합니다.

:::code{language=bash showCopyAction=true}
kubectl get pods -A | grep -E "vllm|litellm|langfuse|open-webui"
:::

모든 파드가 `Running` 상태여야 합니다. 아직 대기 중인 파드가 있다면 프로비저닝이 완료될 때까지 기다리십시오. CloudFormation의 `VerifyPodReadiness` 단계는 스택 생성 후 일반적으로 10-15분이 소요됩니다.

## 단계 1: LiteLLM UI 접속

LiteLLM URL을 확인하고 브라우저에서 엽니다.

:::code{language=bash showCopyAction=true}
echo "LiteLLM URL: http://$(kubectl get ingress -n litellm litellm -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
:::

**로그인 정보:**
- **Username**: `admin`
- **Password**: `/workshop/.env` 의 `LITELLM_MASTER_KEY` 값 (`source /workshop/.env && echo $LITELLM_MASTER_KEY` 명령으로 확인)

## 단계 2: Agent Virtual Key 생성

1. LiteLLM 사이드바에서 **"Virtual Keys"** 를 클릭합니다.
2. **"+ Create New Key"** 를 클릭합니다.
3. 다음과 같이 구성합니다:
   - **Owned By**: You
   - **Key Name**: `agent-key`
   - **Models**: 네 개 모두 선택:
     - `bedrock/claude-4.5-sonnet`
     - `bedrock/gpt-oss-20b`
     - `vllm/qwen3-8b-neuron`
     - `vllm/deepseek-r1-qwen3-8b-neuron`
   - **Key Type**: Default
4. **"Create Key"** 를 클릭합니다.

::alert[**지금 키를 복사하십시오**: LiteLLM은 `sk-...` 시크릿을 **단 한 번만** 표시합니다. 복사하지 않고 대화상자를 닫으면 새 키를 다시 생성해야 합니다.]{type="warning"}

## 단계 3: 모듈 A2를 위한 AGENT_KEY 내보내기

모듈 A2의 Loan Buddy 배포가 자동으로 인식할 수 있도록 키를 환경 변수로 저장합니다.

:::code{language=bash showCopyAction=true}
# sk-... 부분을 방금 복사한 키로 교체하십시오
export AGENT_KEY="sk-your-copied-key"

# 이후 터미널에서도 상속받을 수 있도록 워크샵 env 파일에 영구 저장
grep -v '^AGENT_KEY=' /workshop/.env > /tmp/.env.new && mv /tmp/.env.new /workshop/.env
echo "AGENT_KEY=${AGENT_KEY}" >> /workshop/.env
:::

## 단계 4: 접근 권한 검증

키가 LiteLLM을 통해 정상적으로 인증 및 라우팅되는지 간단히 확인합니다:

:::code{language=bash showCopyAction=true}
LITELLM_URL="http://$(kubectl get ingress -n litellm litellm -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
curl -sS -X POST "${LITELLM_URL}/v1/chat/completions" \
  -H "Authorization: Bearer ${AGENT_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"model":"bedrock/claude-4.5-sonnet","messages":[{"role":"user","content":"Reply with just OK"}]}' \
  | head -c 500
:::

예상 응답: Claude Sonnet 4.5으로부터 `"content":"OK"` (또는 유사한) 문자열이 포함된 JSON 응답.

## 다음 단계

이제 모듈 A2에서 요구하는 결과물인 `AGENT_KEY` 환경 변수를 확보했습니다. 모듈 A2로 바로 이동하여 "Running the Application" 섹션부터 시작하십시오.

---

**[모듈 A2: Agentic AI 애플리케이션 구축으로 계속 →](/track-a-opensource/a2-genai-applications/)**
