---
title: "Kong API Gateway 설정"
weight: 1
---

이 페이지에서는 **Kong Operator 2.x**(인그레스 컨트롤러를 번들로 포함하는 현행 Kubernetes 네이티브 오퍼레이터)를 사용하여 **Kong API Gateway** 인프라를 처음부터 끝까지 설정합니다: Kong Konnect 구독, Personal Access Token, 오퍼레이터, 그리고 Amazon Bedrock 액세스가 가능한 Amazon EKS 클러스터에 연결된 Control Plane + DataPlane을 구성합니다. 이는 LiteLLM을 대체하는 대체 스택 구성 요소입니다.

::alert[**Kong Operator 2.x (최신)** — Kong은 2.0 버전을 기점으로 *Kong Gateway Operator (KGO)*의 이름을 *Kong Operator (KO)*로 변경했으며, **2.0+에는 Kong Ingress Controller가 포함**됩니다. 즉, 게이트웨이는 `kubectl`로 적용하는 **네이티브 Kubernetes CRD**(`KongService`, `KongRoute`, `KongPlugin`)로 구성되며 `decK`가 필요하지 않습니다. 구성은 Git에서 YAML로 작성되고 지속적으로 reconcile되며(GitOps), Konnect는 여전히 UI, 분석, AI Manager를 제공합니다.]{type="info"}

::alert[완료하면 Konnect에 `kong-aws`라는 컨트롤 플레인과 클러스터의 `kong` 네임스페이스에서 실행되는 `dataplane` DataPlane이 생기며, EKS Pod Identity를 통해 Bedrock을 호출할 수 있게 됩니다. **AI Gateway** 라우트(Bedrock으로의 `ai-proxy`, key-auth, 속도 제한)는 이 게이트웨이 위에 **Module B2**에서 CRD로 구성합니다.]{type="info"}

## Step 0: CLI 도구 설치 (kubectl, helm)

Kong Operator 2.x는 네이티브 Kubernetes 리소스로 구성되므로 **kubectl**과 **helm**만 필요합니다 — 두 도구 모두 워크샵 Code Editor에 이미 설치되어 있습니다.

:::code{language=bash showCopyAction=true}
kubectl version --client
helm version
:::

::alert[**decK가 필요하지 않습니다.** Kong Operator 2.x는 인그레스 컨트롤러를 번들로 포함하므로 게이트웨이 구성은 `kubectl apply`(KongService/KongRoute/KongPlugin CRD)로 적용됩니다. `decK`는 구성을 내보내거나 백업하기 위한 선택적 도구로 남아 있지만, 이 배포에는 필요하지 않습니다.]{type="success"}

## Step 1: Kong Konnect 구독

Kong Konnect 계정이 필요합니다. 다음 단계에 따라 **Konnect Plus** 구독을 확보하십시오(크레딧이 포함된 무료 Plus 트라이얼이 제공됩니다).

1. [Kong Konnect 등록 페이지](https://konghq.com/products/kong-konnect/register?utm_medium=partner&utm_source=aws)를 열고 가입합니다.
2. Konnect가 보내는 이메일을 통해 구독을 확인합니다.
3. [cloud.konghq.com](https://cloud.konghq.com)에 로그인합니다.
4. **organization** 이름을 생성하고, **region**을 선택한 후(아래의 `serverURL: us.api.konghq.com`과 일치하도록 **US** 사용), 설정 질문에 답합니다.

::alert[Konnect Plus는 트라이얼 크레딧(일반적으로 $500 / 30일)을 제공합니다. 이 워크샵은 그중 극히 일부만 사용합니다. 워크샵 후에 Konnect 환경을 삭제할 수 있습니다.]{type="info"}

로그인하면 Konnect 홈에 도착합니다. 왼쪽 내비게이션의 **Connectivity** 아래 **API Gateway** 섹션이 컨트롤 플레인이 위치하는 곳입니다:

![Kong Konnect home - Connectivity section](/static/images/track-b-saas/b1-platform-setup/kong-login-page.png)

## Step 2: Personal Access Token (PAT) 생성

Kong Operator는 Control Plane을 생성하기 위해 **Konnect Personal Access Token**이 필요합니다.

1. Konnect에서 **이니셜**(오른쪽 상단)을 클릭 → **Personal access tokens**를 선택합니다.
2. **+ Generate Token**을 클릭하고, 이름을 지정하고, 만료 기간을 설정한 후, **지금 바로 복사하십시오** -- Konnect는 토큰을 다시 보여주지 않습니다.

![Kong Konnect Personal Access Tokens page](/static/images/track-b-saas/b1-platform-setup/kong-pat.png)

**PAT를 셸에 저장하고, 네임스페이스 + 서비스 어카운트를 생성한 후, PAT를 레이블이 지정된 시크릿으로 저장합니다:**

:::code{language=bash showCopyAction=true}
# Paste your copied PAT here
export KONNECT_TOKEN="kpat_PASTE_YOUR_TOKEN_HERE"

# Create the kong namespace and the service account the DataPlane runs as.
# DO NOT rename the service account -- Pod Identity + Bedrock IAM are bound to this exact name.
kubectl create namespace kong
kubectl create sa kaigateway-podid-sa -n kong
:::

Konnect 인증 시크릿을 생성합니다(Kong Operator 2.x는 `konghq.com/credential=konnect` 및 `konghq.com/secret=true` 레이블과 `token` 키 아래의 토큰을 기대합니다):

:::code{language=bash showCopyAction=true}
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: konnect-api-auth-secret
  namespace: kong
  labels:
    konghq.com/credential: konnect
    konghq.com/secret: "true"
stringData:
  token: "$KONNECT_TOKEN"
EOF
:::

::alert[**Pod Identity / Bedrock 액세스:** 서비스 어카운트 `kaigateway-podid-sa`는 `bedrock:InvokeModel` 권한을 부여하는 IAM 역할과 (EKS Pod Identity를 통해) 연결되어 있어야 합니다. 워크샵에서는 이 연결이 환경 설정에 의해 자동으로 생성됩니다. 새로 설정하는 경우, DataPlane을 배포하기 전에 `kong/kaigateway-podid-sa` → Bedrock 역할에 대한 Pod Identity 연결을 생성하십시오.]{type="warning"}

## Step 3: Kubernetes Gateway API CRD 설치 (반드시 먼저 필요)

::alert[**오퍼레이터를 설치하기 전에 이 작업을 수행하십시오.** Kong Operator 2.x는 Kubernetes **Gateway API CRD v1.5.1+**가 필요합니다(`ReferenceGrant`를 `v1` 버전으로 감시합니다). CRD가 없거나 **이전 버전이 설치된 경우**(예: `ReferenceGrant`가 `v1beta1`만 제공하는 v1.3.0), 오퍼레이터가 `no matches for kind "ReferenceGrant" in version "gateway.networking.k8s.io/v1"` 오류와 함께 **크래시 루프**에 빠지고, 어드미션 웹훅이 불안정해지며, 이후의 `kubectl apply`가 `no endpoints available for service "...-webhook"` 오류로 실패합니다.]{type="warning"}

:::code{language=bash showCopyAction=true}
# Install/upgrade Gateway API CRDs to v1.5.1 (server-side).
# Use --force-conflicts if an older version was previously installed by another tool (e.g. helm).
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/standard-install.yaml \
  --server-side --force-conflicts
:::

`ReferenceGrant`가 이제 `v1`을 제공하는지 확인합니다:

:::code{language=bash showCopyAction=true}
kubectl get crd referencegrants.gateway.networking.k8s.io \
  -o jsonpath='{range .spec.versions[*]}{.name}{" served="}{.served}{"\n"}{end}'
# expect a line: v1 served=true
:::

## Step 4: Kong Operator 2.x 설치

Konnect 컨트롤러 지원을 활성화한 상태로 **`kong/kong-operator`** 차트(인그레스 컨트롤러를 번들로 포함하는 2.x 오퍼레이터)를 설치합니다. 이 설치는 **`kong-system`** 네임스페이스(오퍼레이터 자체 네임스페이스)에 이루어지며, 게이트웨이 리소스가 위치하는 **`kong`** 네임스페이스(Step 2에서 생성)와는 별개입니다.

:::code{language=bash showCopyAction=true}
helm repo add kong https://charts.konghq.com
helm repo update kong

helm upgrade --install kong-operator kong/kong-operator \
  -n kong-system \
  --create-namespace \
  --set image.tag=2.2 \
  --set env.ENABLE_CONTROLLER_KONNECT=true
:::

::alert[**Gateway API CRD보다 먼저 오퍼레이터를 설치한 경우**(또는 v1.5.1로 업그레이드하기 전에 설치한 경우), 오퍼레이터 파드가 크래시 루프 상태일 것입니다. 지금 재시작하여 CRD를 인식하도록 하십시오: `kubectl rollout restart deployment/kong-operator-kong-operator-controller-manager -n kong-system`]{type="info"}

오퍼레이터가 준비될 때까지 기다립니다:

:::code{language=bash showCopyAction=true}
kubectl -n kong-system wait --for=condition=Available=true --timeout=120s \
  deployment/kong-operator-kong-operator-controller-manager
:::

::alert[**버전 참고:** `image.tag=2.2`는 현행 Kong Operator 2.x 라인입니다(2.0+에는 Kong Ingress Controller가 포함됨). 이는 약 1년 전 Kong 워크샵에서 사용된 구버전 `kong/gateway-operator` 1.6 차트를 대체합니다. 더 새로운 2.x 패치가 있다면 그것을 사용하십시오 — 아래의 CRD API 버전은 2.x 전반에 걸쳐 안정적입니다.]{type="info"}

## Step 5: Konnect 인증 구성 및 Control Plane 생성

::alert[**사전 요구 사항 확인:** 이 단계에는 **Step 2**의 `kong` 네임스페이스와 `konnect-api-auth-secret`이 필요합니다. Step 2를 건너뛰었다면 `namespaces "kong" not found` 오류가 표시됩니다. 먼저 확인하십시오: `kubectl get ns kong && kubectl get secret konnect-api-auth-secret -n kong` — 둘 중 하나라도 없으면 돌아가서 Step 2를 실행하십시오.]{type="warning"}

이 단계에서는 (PAT 시크릿을 참조하는) Konnect **API 인증 구성**과 `kong-aws`라는 이름의 Konnect **Control Plane**을 생성합니다. 현재 API 버전에 유의하십시오: `KonnectAPIAuthConfiguration`은 `v1alpha1`이고, `KonnectGatewayControlPlane`은 `v1alpha2`(`createControlPlaneRequest` 사용)입니다:

:::code{language=bash showCopyAction=true}
cat <<EOF | kubectl apply -f -
kind: KonnectAPIAuthConfiguration
apiVersion: konnect.konghq.com/v1alpha1
metadata:
  name: konnect-api-auth
  namespace: kong
spec:
  type: secretRef
  secretRef:
    name: konnect-api-auth-secret
  serverURL: us.api.konghq.com
---
kind: KonnectGatewayControlPlane
apiVersion: konnect.konghq.com/v1alpha2
metadata:
  name: kong-aws
  namespace: kong
spec:
  createControlPlaneRequest:
    name: kong-aws
  konnect:
    authRef:
      name: konnect-api-auth
EOF
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
konnectapiauthconfiguration.konnect.konghq.com/konnect-api-auth created
konnectgatewaycontrolplane.konnect.konghq.com/kong-aws created
:::

컨트롤 플레인이 성공적으로 프로그래밍되었는지(Konnect ID를 받았는지) 확인합니다:

:::code{language=bash showCopyAction=true}
kubectl get konnectgatewaycontrolplane kong-aws -n kong
:::

Konnect UI → **Gateway Manager** → **Control planes**에서 `kong-aws`라는 새 컨트롤 플레인이 표시됩니다:

![Kong Konnect control plane kong-aws in Gateway Manager](/static/images/track-b-saas/b1-platform-setup/kong-control-plane.png)

## Step 6: Data Plane 배포

이 단계에서는 `KonnectExtension`(Kong Operator가 인증서를 자동으로 관리)과 `DataPlane`을 생성합니다 — Kong Gateway 이미지에 고정되어 있고, (Bedrock Pod Identity를 위한) `kaigateway-podid-sa` 서비스 어카운트를 사용하며, 인터넷 연결 LoadBalancer를 통해 노출됩니다:

:::code{language=bash showCopyAction=true}
cat <<EOF | kubectl apply -f -
kind: KonnectExtension
apiVersion: konnect.konghq.com/v1alpha2
metadata:
  name: konnect-config1
  namespace: kong
spec:
  clientAuth:
    certificateSecret:
      provisioning: Automatic
  konnect:
    controlPlane:
      ref:
        type: konnectNamespacedRef
        konnectNamespacedRef:
          name: kong-aws
---
apiVersion: gateway-operator.konghq.com/v1beta1
kind: DataPlane
metadata:
  name: dataplane
  namespace: kong
spec:
  extensions:
  - kind: KonnectExtension
    name: konnect-config1
    group: konnect.konghq.com
  deployment:
    podTemplateSpec:
      spec:
        containers:
        - name: proxy
          image: kong/kong-gateway:3.13
          readinessProbe:
            initialDelaySeconds: 1
            periodSeconds: 1
        serviceAccountName: kaigateway-podid-sa
  network:
    services:
      ingress:
        name: proxy1
        type: LoadBalancer
        annotations:
          "service.beta.kubernetes.io/aws-load-balancer-scheme": "internet-facing"
          "service.beta.kubernetes.io/aws-load-balancer-nlb-target-type": "ip"
EOF
:::

::alert[`serviceAccountName`을 **절대 수정하지 마십시오** -- Pod Identity와 Bedrock IAM 권한이 `kaigateway-podid-sa`에 바인딩되어 있습니다. Kong Gateway 이미지 `kong/kong-gateway:3.13`은 Kong Operator 2.x와 짝을 이루는 현행 버전입니다.]{type="warning"}

## Step 7: Data Plane 엔드포인트 확인

LoadBalancer 프로비저닝에는 몇 분이 걸립니다. 호스트네임을 캡처합니다:

:::code{language=bash showCopyAction=true}
export DATA_PLANE_LB="http://$(kubectl get svc -n kong proxy1 --output=jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
echo "Data Plane: $DATA_PLANE_LB"
:::

게이트웨이가 응답하는지 확인합니다(아직 라우트가 구성되지 않았으므로 "no Route matched" 메시지가 예상되는 정상 응답입니다):

:::code{language=bash showCopyAction=true}
curl -w '\n' $DATA_PLANE_LB
:::

**예상 출력:**

:::code{language=json showCopyAction=false}
{
  "message":"no Route matched with those values",
  "request_id":"d364362a60b32142fed73712a9ea1948"
}
:::

## Step 8: Kubernetes에서 Data Plane 확인

:::code{language=bash showCopyAction=true}
# DataPlane CR should be Ready=True
kubectl get dataplane -n kong

# The proxy pod runs as kaigateway-podid-sa
kubectl get pods -n kong -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.serviceAccountName}{"\n"}{end}' | grep dataplane
:::

**예상 출력:**

:::code{language=text showCopyAction=false}
# kubectl get dataplane -n kong
NAME        READY
dataplane   True

# proxy pod -> service account
dataplane-dataplane-xxxxx-xxxxxxxxx-xxxxx   kaigateway-podid-sa
:::

::alert[DataPlane의 `READY   True`와 `kaigateway-podid-sa`에 바인딩된 프록시 파드는 게이트웨이가 Bedrock이 활성화된 서비스 어카운트 아래에서 정상 실행 중임을 확인해 줍니다. 정확한 파드 이름 접미사는 환경마다 다릅니다.]{type="success"}

Konnect 측에서도 연결을 확인할 수 있습니다: **Gateway Manager → `kong-aws` → Data plane nodes**. DataPlane 노드가 **Connected** 및 **In sync**로 표시됩니다:

![Kong Konnect data plane node connected and in sync](/static/images/track-b-saas/b1-platform-setup/kong-data-plane.png)

::alert[**Connected** + **In sync**(Sync status)는 Konnect가 클러스터 내 DataPlane을 성공적으로 관리하고 있음을 의미합니다. Kong Operator 2.x는 인그레스 컨트롤러를 포함하므로, `kubectl`로 적용하는 KongService/KongRoute/KongPlugin CRD가 이 컨트롤 플레인으로 reconcile되어 Konnect UI에 (읽기 전용으로) 표시됩니다.]{type="info"}

## Kong 설정 완료

이제 다음을 갖추게 되었습니다:

- Kong Konnect 계정 + 컨트롤 플레인 `kong-aws`
- **Kong Operator 2.x** 설치 완료 (인그레스 컨트롤러 번들 포함 — K8s 네이티브 CRD 구성)
- Konnect에 연결되고 `proxy1` LoadBalancer를 통해 노출된 DataPlane(`dataplane`)
- `kaigateway-podid-sa`의 Pod Identity를 통한 Bedrock 액세스

게이트웨이는 실행 중이지만 아직 **AI 라우트가 없습니다** -- Bedrock `ai-proxy` 라우트는 **Module B2**에서 (`kubectl`로 적용하는) **KongService / KongRoute / KongPlugin CRD**로 구성합니다.

---

**[다음: Arize AX 설정 →](../arize-setup/)**
