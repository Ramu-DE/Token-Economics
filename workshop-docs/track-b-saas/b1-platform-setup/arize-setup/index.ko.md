---
title: "Arize AX 설정"
weight: 2
---

이 페이지에서는 대체 스택에서 Langfuse를 대체하는 SaaS 관측성 및 평가 플랫폼인 **Arize AX**를 설정합니다. 계정을 생성하고, Space를 설정하고, **Service** API 키를 생성한 다음, 에이전트가 OpenInference 트레이스를 내보낼 수 있도록 자격 증명을 클러스터에 설치합니다.

::alert[Arize AX는 [app.arize.com](https://app.arize.com)에서 완전 호스팅됩니다. 클러스터 내에 배포할 것이 없으며, 이 설정은 계정을 등록하고 에이전트가 트레이스를 전송하는 데 사용하는 자격 증명을 저장하는 과정입니다.]{type="info"}

## Step 1: Arize AX 계정 생성

1. [app.arize.com](https://app.arize.com)으로 이동하여 가입합니다(프리 티어가 제공됩니다).
2. 이메일을 확인하고 로그인합니다.
3. 최초 로그인 시 Arize가 **Organization**과 기본 **Space**를 자동으로 생성합니다.

로그인하면 Arize AX 홈에 도착합니다. 왼쪽 내비게이션에는 **Tracing Projects**, **Datasets**, **Evaluators** 등이 표시됩니다 -- 이것이 Langfuse를 대체하는 관측성 및 평가 기능입니다:

![Arize AX home dashboard](/static/images/track-b-saas/b1-platform-setup/arize-login-page.png)

## Step 2: Space 및 Space ID 찾기

**Space**는 트레이스, 데이터셋, 평가가 위치하는 곳입니다(Langfuse 프로젝트와 유사).

1. **Space ID** 버튼(오른쪽 상단)을 클릭하거나 **Settings → Space Settings → Config**를 엽니다.
2. **Space ID**를 찾습니다 -- base64 스타일 문자열입니다(예: `U3BhY2U6MzUwMjk6c3p2Qw==`). URL에도 표시됩니다: `app.arize.com/organizations/.../spaces/<SPACE_ID>`.
3. **Copy Space ID**를 클릭합니다 -- 자격 증명에 필요합니다.

![Arize AX Space Settings - Copy Space ID](/static/images/track-b-saas/b1-platform-setup/arize-space-id.png)

:::code{language=bash showCopyAction=true}
# Save your Space ID
export ARIZE_SPACE_ID="U3BhY2U6..."   # paste your Space ID
:::

## Step 3: Service API 키 생성

Arize에는 두 종류의 키가 있습니다 -- 사용자 키가 아닌 **Service 키를 사용하십시오**.

1. **Settings → API Keys**로 이동한 후 **Service Keys** 탭을 선택합니다.
2. **+ New Service Key**를 클릭합니다(또는 기존 키를 복사합니다). 키는 `ak-`로 시작하며 약 72자입니다. 이 페이지에는 **Current SpaceID**도 표시됩니다.
3. Service 키를 복사합니다.

![Arize AX API Keys - Service Keys tab](/static/images/track-b-saas/b1-platform-setup/arize-key.png)

::alert[**USER 키가 아닌 SERVICE 키를 사용하십시오.** 둘 다 `ak-`로 시작하지만, (로그인에 연결된) **사용자 키**는 자동화된 시스템에서 트레이스 수집에 사용할 경우 `PERMISSION_DENIED`를 반환합니다. **Service 키**는 자동화된 시스템을 위한 영구 자격 증명이며, 에이전트 트레이싱에 작동하는 키입니다.]{type="warning"}

:::code{language=bash showCopyAction=true}
# Save your SERVICE API key (ak-..., ~72 chars)
export ARIZE_API_KEY="ak-xxxxxxxx-...-xxxxxxxx"
:::

## Step 4: 자격 증명을 클러스터에 설치

대체 스택의 `o11y/arize` 컴포넌트는 Arize 자격 증명을 Kubernetes Secret(`arize` 네임스페이스의 `arize-credentials`)으로 저장합니다. 에이전트는 이 Secret을 참조하여 트레이스를 내보냅니다.

::alert[**이 Secret이 필요한 경우:** Module B3의 **클러스터 내** Strands 에이전트에만 필요합니다(`secretKeyRef`를 통해 Secret을 마운트합니다). 터미널에서 Arize 스크립트만 실행하는 경우(예: Step 6의 스모크 테스트)에는 `ARIZE_API_KEY`와 `ARIZE_SPACE_ID`를 환경 변수로 내보내기만 하면 되며 클러스터 Secret은 필요하지 않습니다. 에이전트를 배포하기 전에 준비되도록 여기서 Secret을 생성해 두십시오.]{type="info"}

**Option A -- starter-kit CLI (권장):**

:::code{language=bash showCopyAction=true}
# Put your creds in .env.local (do not edit the committed .env)
cat >> /workshop/.env.local <<EOF
ARIZE_API_KEY=$ARIZE_API_KEY
ARIZE_SPACE_ID=$ARIZE_SPACE_ID
EOF

cd /workshop && ./cli o11y arize install
:::

**Option B -- Secret 직접 생성:**

:::code{language=bash showCopyAction=true}
kubectl create namespace arize --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic arize-credentials -n arize \
  --from-literal=ARIZE_API_KEY="$ARIZE_API_KEY" \
  --from-literal=ARIZE_SPACE_ID="$ARIZE_SPACE_ID" \
  --from-literal=ARIZE_OTLP_ENDPOINT="https://otlp.arize.com" \
  --dry-run=client -o yaml | kubectl apply -f -
:::

## Step 5: 자격 증명 확인

:::code{language=bash showCopyAction=true}
kubectl get secret arize-credentials -n arize \
  -o jsonpath='{.data.ARIZE_OTLP_ENDPOINT}' | base64 -d; echo
# expect: https://otlp.arize.com
:::

## Step 6: (선택 사항) 테스트 트레이스 전송

스팬 하나를 전송하여 키가 작동하는지 확인합니다. Workshop Studio Code Editor, AWS CloudShell, 로컬 머신에서 동일하게 작동하는 안정적이고 이식성 있는 방법은 **전용 가상 환경**을 생성하고, 그 안에 설치한 다음, **동일한 venv의 python**으로 실행하는 것입니다. 이렇게 하면 `pip`과 `python`이 동일한 인터프리터임이 보장됩니다(셸의 `python3`가 pip이 설치한 인터프리터와 다르거나 별칭 처리된 경우 발생하는 `ModuleNotFoundError`를 방지).

:::code{language=bash showCopyAction=true}
# 1. Create an isolated venv and install the SDK INTO it
python3 -m venv /tmp/arize-venv
/tmp/arize-venv/bin/pip install -q --upgrade pip
/tmp/arize-venv/bin/pip install -q arize-otel openinference-instrumentation opentelemetry-sdk

# 2. Run the smoke test with the SAME venv's python (not the shell's python3)
/tmp/arize-venv/bin/python - <<'PY'
import os
from arize.otel import register
tp = register(
    space_id=os.environ["ARIZE_SPACE_ID"],
    api_key=os.environ["ARIZE_API_KEY"],
    project_name="alt-stack-smoketest",
    set_global_tracer_provider=True,
)
from opentelemetry import trace
with trace.get_tracer("smoke").start_as_current_span("hello-arize"):
    print("sent a span to Arize AX project 'alt-stack-smoketest'")
if hasattr(tp, "force_flush"): tp.force_flush()
PY
:::

::alert[**전용 venv를 사용하는 이유:** `pip install ...` 후 `python3 ...`를 실행하면, 셸의 `python3`가 `pip`이 설치한 인터프리터와 *다른* 인터프리터로 해석될 때마다 `ModuleNotFoundError: No module named 'arize'`로 실패합니다(macOS에서 `python3`가 Homebrew로 별칭 처리된 경우와 일부 관리형 환경에서 흔히 발생). `/tmp/arize-venv`를 생성하고 **`/tmp/arize-venv/bin/pip`** + **`/tmp/arize-venv/bin/python`**을 사용하면 구조적으로 둘이 동일한 바이너리가 됩니다 — Code Editor, CloudShell, 로컬 머신 전반에 이식 가능합니다. 검증 방법: `/tmp/arize-venv/bin/python -c "import arize; print(arize.__file__)"`.]{type="info"}

[app.arize.com](https://app.arize.com) → **Tracing Projects** → 프로젝트 **`alt-stack-smoketest`** → **Traces** 탭을 열고 `hello-ar`(ize) 스팬이 표시되는지 확인합니다(1 trace, 1 span):

![Arize AX smoke-test trace in alt-stack-smoketest project](/static/images/track-b-saas/b1-platform-setup/arize-smoke-test.png)

`PERMISSION_DENIED`가 표시되면 사용자 키를 사용하고 있는 것입니다 -- **Service** 키로 전환하십시오(Step 3). 트레이스가 표시되지 않으면 올바른 **Space**(왼쪽 상단 셀렉터)에 있는지, 그리고 시간 필터(오른쪽 상단)가 최근 몇 분을 포함하는지 확인하십시오.

## Arize 설정 완료

이제 다음을 갖추게 되었습니다:

- Space와 Space ID가 있는 Arize AX 계정
- 트레이스 수집에 사용할 수 있는 **Service** API 키
- 클러스터에 설치된 `arize-credentials` Secret

Module B3의 에이전트가 이 자격 증명을 사용하여 전체 OpenInference 트레이스(LLM + 도구 스팬)를 Arize AX로 내보냅니다.

---

**[대체 스택 개요로 돌아가기 →](../) · [Module B2 시작: Kong AI Gateway →](/track-b-saas/b2-ai-gateway-observability/ai-gateway-kong/)**
