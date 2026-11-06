---
title: "Install the Kong Gateway (Operator + DataPlane)"
weight: 1
---

This page sets up the **Kong API Gateway** infrastructure end to end using **Kong Operator 2.x** (the current, Kubernetes-native operator that bundles the ingress controller): a Kong Konnect subscription, a Personal Access Token, the operator, and a Control Plane + DataPlane connected to your Amazon EKS cluster with Amazon Bedrock access. This is the Track B stack replacement for LiteLLM.

::alert[**Kong Operator 2.x (latest)** — Kong renamed *Kong Gateway Operator (KGO)* to *Kong Operator (KO)* at the 2.0 boundary, and **2.0+ includes the Kong Ingress Controller**. This means the gateway is configured with **native Kubernetes CRDs** (`KongService`, `KongRoute`, `KongPlugin`) applied via `kubectl` — no `decK` required. Config is authored as YAML in Git and reconciled continuously (GitOps), while Konnect still provides the UI, analytics, and AI Manager.]{type="info"}

::alert[When done, you will have a control plane named `kong-aws` in Konnect and a `dataplane` DataPlane running in your cluster's `kong` namespace, able to call Bedrock via EKS Pod Identity. The **AI Gateway** routes (`ai-proxy` to Bedrock, key-auth, rate limiting) are configured as CRDs in **Module B2**, on top of this gateway.]{type="info"}

## Step 0: Install CLI Tools (kubectl, helm)

Kong Operator 2.x is configured with native Kubernetes resources, so you only need **kubectl** and **helm** — both already present in the workshop Code Editor.

:::code{language=bash showCopyAction=true}
kubectl version --client
helm version
:::

::::expand{header="Check Output"}

```bash
Client Version: v1.36.3
Kustomize Version: v5.8.1
version.BuildInfo{Version:"v3.21.4", GitCommit:"813176c51bb5c181dbbd7901298ddcc104cd3417", GitTreeState:"clean", GoVersion:"go1.26.5"}
```

::::

::alert[**No decK needed.** Because Kong Operator 2.x bundles the ingress controller, gateway configuration is applied with `kubectl apply` (KongService/KongRoute/KongPlugin CRDs). `decK` remains an optional tool for exporting/backing up config, but it is not required for this deployment.]{type="success"}

## Step 1: Kong Konnect Subscription

You need a Kong Konnect account. Follow these steps to obtain a **Konnect** subscription (a free trial with credits is available -- no credit card required).

1. Open the [Kong Konnect registration page](https://konghq.com/products/kong-konnect/register?utm_medium=partner&utm_source=aws). Under **Sign Up**, enter your **Name**, a personal **Email address** (e.g. a Gmail address), and a **Password** that meets the listed requirements, then click **Sign Up**. Confirm your subscription via the email Konnect sends you.

![Kong Konnect Sign Up — name, email, and password](/static/images/track-b-saas/b1-platform-setup/kong-setup-1.png)

2. When prompted to **Create an Organization**, enter an **Organization Name** (for example `AWS`) and click **Next**.

![Kong Konnect Create an Organization — name AWS](/static/images/track-b-saas/b1-platform-setup/kong-setup-3.png)

3. You land on the Konnect **Welcome** home for your organization. In the **top-right corner**, set the **region selector to US** (the flag icon). This is important: it must match `serverURL: us.api.konghq.com` used in Step 5 — otherwise your control plane is created in one region while the UI shows another, and it will appear to be "missing." The **API Gateway** section under **Connectivity** (left nav) is where your control planes will live.

![Kong Konnect Welcome home — US region selected (top-right flag)](/static/images/track-b-saas/b1-platform-setup/kong-setup-6.png)

::alert[The Konnect trial provides credits (typically $500 / 30 days). This workshop uses a small fraction of that. You can delete the Konnect organization after the workshop.]{type="info"}

::alert[**Data region:** this workshop assumes your Konnect org is in the **US** region, which matches `serverURL: us.api.konghq.com` used in Step 5. If your org is in a different region, set `serverURL` accordingly (e.g. `eu.api.konghq.com`).]{type="warning"}

## Step 2: Create a Personal Access Token (PAT)

Kong Operator requires a **Konnect Personal Access Token** to create the Control Plane.

1. In Konnect, click your **initials** (upper-right corner) → **My Account**. On the account page, open the **Personal Access Tokens** tab. (The token menu is a **tab on your account/profile page**, not a direct item in the initials dropdown.)
2. Click **+ Generate Token**, name it, set an expiration, and **copy it now** -- Konnect will not show it again.

![Kong Konnect — open the initials menu (upper-right) and click My Account](/static/images/track-b-saas/b1-platform-setup/kong-setup-pat-1.png)

![Kong Konnect — on the account page, open the Personal Access Tokens tab](/static/images/track-b-saas/b1-platform-setup/kong-setup-pat-2.png)

**Save the PAT in your shell** (edit the value first -- paste your copied token in place of the placeholder):

:::code{language=bash showCopyAction=true}
# Paste your copied PAT here
export KONNECT_TOKEN="PASTE_YOUR_TOKEN_HERE"   # paste your full PAT -- it already starts with kpat_
:::

**Create the `kong` namespace and the service account the DataPlane runs as** (copy/paste as-is):

:::code{language=bash showCopyAction=true}
# DO NOT rename the service account -- Pod Identity + Bedrock IAM are bound to this exact name.
# Idempotent: dry-run|apply creates on first run and reports "unchanged" on re-runs (no error).
kubectl create namespace kong --dry-run=client -o yaml | kubectl apply -f -
kubectl create sa kaigateway-podid-sa -n kong --dry-run=client -o yaml | kubectl apply -f -
:::

Create the Konnect auth secret (Kong Operator 2.x expects the `konghq.com/credential=konnect` and `konghq.com/secret=true` labels, with the token under the `token` key):

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

::::expand{header="Check Output"}

```bash
secret/konnect-api-auth-secret configured
```

::::

::alert[**Bedrock access is configured in Module B2.** The DataPlane reaches Amazon Bedrock through **EKS Pod Identity** on `kaigateway-podid-sa`. You create that association (service account → Bedrock IAM role) and restart the DataPlane in [Module B2 → Step 2: Confirm Bedrock Access (Pod Identity)](/track-b-saas/b2-ai-gateway-observability/ai-gateway-kong/). The DataPlane you deploy below therefore starts **without** Bedrock credentials — that is expected until B2 wires it up.]{type="info"}

## Step 3: Install the Kubernetes Gateway API CRDs (REQUIRED first)

::alert[**Do this BEFORE installing the operator.** Kong Operator 2.x requires the Kubernetes **Gateway API CRDs at v1.5.1+** (it watches `ReferenceGrant` in version `v1`). If they are missing — or an **older version is installed** (e.g. v1.3.0, where `ReferenceGrant` only serves `v1beta1`) — the operator **crash-loops** with `no matches for kind "ReferenceGrant" in version "gateway.networking.k8s.io/v1"`, its admission webhook flaps, and later `kubectl apply`s fail with `no endpoints available for service "...-webhook"`.]{type="warning"}

:::code{language=bash showCopyAction=true}
# Install/upgrade Gateway API CRDs to v1.5.1 (server-side).
# Use --force-conflicts if an older version was previously installed by another tool (e.g. helm).
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/standard-install.yaml \
  --server-side --force-conflicts
:::

::::expand{header="Check Output"}

```bash
customresourcedefinition.apiextensions.k8s.io/backendtlspolicies.gateway.networking.k8s.io serverside-applied
customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io serverside-applied
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io serverside-applied
customresourcedefinition.apiextensions.k8s.io/grpcroutes.gateway.networking.k8s.io serverside-applied
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io serverside-applied
customresourcedefinition.apiextensions.k8s.io/listenersets.gateway.networking.k8s.io serverside-applied
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io serverside-applied
customresourcedefinition.apiextensions.k8s.io/tlsroutes.gateway.networking.k8s.io serverside-applied
validatingadmissionpolicy.admissionregistration.k8s.io/safe-upgrades.gateway.networking.k8s.io serverside-applied
validatingadmissionpolicybinding.admissionregistration.k8s.io/safe-upgrades.gateway.networking.k8s.io serverside-applied
```

::::

Verify `ReferenceGrant` now serves `v1`:

:::code{language=bash showCopyAction=true}
kubectl get crd referencegrants.gateway.networking.k8s.io \
  -o jsonpath='{range .spec.versions[*]}{.name}{" served="}{.served}{"\n"}{end}'
:::

::::expand{header="Check Output"}

```bash
v1 served=true
v1beta1 served=true
```

::::

## Step 4: Install Kong Operator 2.x

Install the **`kong/kong-operator`** chart (the 2.x operator that bundles the ingress controller), with Konnect controller support enabled. Note this installs into the **`kong-system`** namespace (the operator's own namespace) — distinct from the **`kong`** namespace (from Step 2) where your gateway resources live.

*Why this matters: the operator is the in-cluster controller that turns your Kong CRDs (ControlPlane, DataPlane, KongService/Route/Plugin) into running config and continuously reconciles them — nothing else renders your gateway config.*

:::code{language=bash showCopyAction=true}
helm repo add kong https://charts.konghq.com
helm repo update kong

helm upgrade --install kong-operator kong/kong-operator \
  -n kong-system \
  --create-namespace \
  --set image.tag=2.2 \
  --set env.ENABLE_CONTROLLER_KONNECT=true
:::

::::expand{header="Check Output"}

```bash
"kong" has been added to your repositories
...Successfully got an update from the "kong" chart repository
Update Complete. ⎈Happy Helming!⎈
Release "kong-operator" has been installed. Happy Helming!
NAME: kong-operator
LAST DEPLOYED: Fri Aug 21 11:29:08 2026
NAMESPACE: kong-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
kong-operator-kong-operator has been installed. Check its status by running:

  kubectl --namespace kong-system get pods

For more details, please refer to the following documents:

* https://developer.konghq.com/operator/dataplanes/get-started/kic/create-gateway/
* https://developer.konghq.com/operator/dataplanes/konnectextension/
```

::::

::alert[**If you installed the operator BEFORE the Gateway API CRDs** (or before upgrading them to v1.5.1), the operator pod will be crash-looping. Restart it now so it picks up the CRDs: `kubectl rollout restart deployment/kong-operator-kong-operator-controller-manager -n kong-system`]{type="info"}

Wait for the operator to become ready:

:::code{language=bash showCopyAction=true}
kubectl -n kong-system wait --for=condition=Available=true --timeout=120s \
  deployment/kong-operator-kong-operator-controller-manager
:::

::::expand{header="Check Output"}

```bash
deployment.apps/kong-operator-kong-operator-controller-manager condition met
```

::::

Confirm the operator pod is running:

:::code{language=bash showCopyAction=true}
kubectl --namespace kong-system get pods
:::

::::expand{header="Check Output"}

```bash
NAME                                                              READY   STATUS    RESTARTS   AGE
kong-operator-kong-operator-controller-manager-xxxxxxxxxx-xxxxx   1/1     Running   0          60s
```

::::

::alert[**Version note:** `image.tag=2.2` is the current Kong Operator 2.x line (2.0+ includes the Kong Ingress Controller, replacing the older `kong/gateway-operator` 1.6 chart). If a newer 2.x patch exists, use it — the CRD API versions below are stable across 2.x.]{type="info"}

## Step 5: Create the Konnect Auth Config and Control Plane

*Why this matters: the `KonnectAPIAuthConfiguration` binds your cluster to your Konnect org (via the PAT), and the `KonnectGatewayControlPlane` creates the managed control plane `kong-aws` that stores and serves your gateway config — the DataPlane attaches to it.*

::alert[**Prerequisite check:** this step requires the `kong` namespace and the `konnect-api-auth-secret` from **Step 2**. If you skipped Step 2 you will see `namespaces "kong" not found`. Verify first: `kubectl get ns kong && kubectl get secret konnect-api-auth-secret -n kong` — if either is missing, go back and run Step 2.]{type="warning"}

This creates a Konnect **API auth configuration** (referencing the PAT secret) and a Konnect **Control Plane** named `kong-aws`. Note the current API versions: `KonnectAPIAuthConfiguration` is `v1alpha1`, `KonnectGatewayControlPlane` is `v1alpha2` (which uses `createControlPlaneRequest`):

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

::::expand{header="Check Output"}

```text
konnectapiauthconfiguration.konnect.konghq.com/konnect-api-auth created
konnectgatewaycontrolplane.konnect.konghq.com/kong-aws created
```

::::

Confirm the control plane programmed successfully (gets a Konnect ID):

:::code{language=bash showCopyAction=true}
kubectl get konnectgatewaycontrolplane kong-aws -n kong
:::

::::expand{header="Check Output"}

```bash
NAME       PROGRAMMED   ID                                     ORGID                                  AGE
kong-aws   True         xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   30s
```

::::

`PROGRAMMED   True` means Kong Operator successfully created the control plane in Konnect (US region) and it received a Konnect `ID`.

In the Konnect UI → **Gateway Manager** → **Control planes** (with the **US** region selected, top-right), a new control plane named `kong-aws` will appear:

![Kong Konnect control plane kong-aws in Gateway Manager](/static/images/track-b-saas/b1-platform-setup/kong-control-plane.png)

::alert[**Don't see `kong-aws` in the UI?** It was created in the **US** region (`serverURL: us.api.konghq.com`). If the list looks empty, your UI is viewing a different region — switch the **region selector (top-right) to US**. The CLI check above confirms it exists regardless of the UI.]{type="info"}

## Step 6: Deploy the Data Plane

*Why this matters: the DataPlane is the only Kong component that runs in your cluster and handles live traffic (the proxy clients hit via the LoadBalancer); it pulls its config from the Konnect control plane created in Step 5.*

This creates a `KonnectExtension` (Kong Operator manages certificates automatically) and the `DataPlane` — pinned to the Kong Gateway image, using the `kaigateway-podid-sa` service account (for Bedrock Pod Identity), and exposed via an internet-facing LoadBalancer:

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
          env:
          # Agentic requests carry a growing multi-turn tool history (tool results
          # can be long, LLM-generated). Those bodies exceed nginx's default 8k
          # client body buffer, so nginx spills the body to a temp file; Kong's
          # ai-proxy reads the body via the PDK get_raw_body(), which FAILS on a
          # buffered-to-disk body and returns the misleading "request body doesn't
          # contain valid inputs". Raise the buffer, AND set client_max_body_size to
          # the SAME value so every request up to the limit stays in memory (no disk
          # spill possible) and anything larger is rejected with an explicit 413.
          - name: KONG_NGINX_HTTP_CLIENT_BODY_BUFFER_SIZE
            value: "16m"
          - name: KONG_NGINX_HTTP_CLIENT_MAX_BODY_SIZE
            value: "16m"
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

::::expand{header="Check Output"}

```bash
konnectextension.konnect.konghq.com/konnect-config1 created
dataplane.gateway-operator.konghq.com/dataplane created
```

::::

::alert[**DO NOT** modify the `serviceAccountName` -- Pod Identity and the Bedrock IAM permissions are bound to `kaigateway-podid-sa`. The Kong Gateway image `kong/kong-gateway:3.13` is the current version paired with Kong Operator 2.x.]{type="warning"}

## Step 7: Get the Data Plane Endpoint

The LoadBalancer (an AWS NLB) takes **~2-4 minutes** to provision. First wait until its hostname is assigned — this portable loop needs no extra tools and exits on its own once the hostname appears:

:::code{language=bash showCopyAction=true}
until kubectl get svc -n kong proxy1 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' | grep -q .; do
  echo "waiting for the LoadBalancer to provision..."; sleep 15
done
echo "LoadBalancer provisioned."
:::

Then capture its hostname:

:::code{language=bash showCopyAction=true}
export DATA_PLANE_LB="http://$(kubectl get svc -n kong proxy1 --output=jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
echo "Data Plane: $DATA_PLANE_LB"
:::

::::expand{header="Check Output"}

```bash
Data Plane: http://k8s-kong-proxy1-xxxxxxxxxx-xxxxxxxxxxxxxxxx.elb.us-east-1.amazonaws.com
```

::::

Verify the gateway responds (no route is configured yet, so a "no Route matched" message with **HTTP 404** is the expected, healthy response). The `-w '...%{http_code}'` prints the status code so you can tell a real gateway response from "still provisioning" (an empty body or a connection/DNS error):

:::code{language=bash showCopyAction=true}
curl -sS --max-time 10 -w '\nHTTP_CODE:%{http_code}\n' "$DATA_PLANE_LB"
:::

::::expand{header="Check Output"}

```json
{
  "message":"no Route matched with those values",
  "request_id":"d364362a60b32142fed73712a9ea1948"
}
HTTP_CODE:404
```

::::

::alert[**If you see `curl: (6) Could not resolve host: k8s-kong-proxy1-…elb.us-east-1.amazonaws.com`**, the LoadBalancer's DNS record has not propagated yet. Wait **2-3 minutes** and run the `curl` again — the hostname resolves once the NLB finishes provisioning.]{type="info"}

## Step 8: Verify the Data Plane in Kubernetes

:::code{language=bash showCopyAction=true}
# DataPlane CR should be Ready=True
kubectl get dataplane -n kong

# The proxy pod runs as kaigateway-podid-sa
kubectl get pods -n kong -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.serviceAccountName}{"\n"}{end}' | grep dataplane
:::

::::expand{header="Check Output"}

```text
# kubectl get dataplane -n kong
NAME        READY
dataplane   True

# proxy pod -> service account
dataplane-dataplane-xxxxx-xxxxxxxxx-xxxxx   kaigateway-podid-sa
```

::::

::alert[`READY   True` on the DataPlane and the proxy pod bound to `kaigateway-podid-sa` confirm the gateway is up. **Bedrock access itself is granted via EKS Pod Identity (an IAM role associated with this service account) in Module B2** -- the service account alone does not grant Bedrock. The exact pod name suffix will differ in your environment.]{type="success"}

You can also confirm the connection from the Konnect side: **Gateway Manager → `kong-aws` → Data plane nodes**. Your DataPlane node shows as **Connected** and **In sync**:

![Kong Konnect data plane node connected and in sync](/static/images/track-b-saas/b1-platform-setup/kong-data-plane.png)

::alert[**Connected** + **In sync** (Sync status) means Konnect is successfully managing your in-cluster DataPlane. Because Kong Operator 2.x includes the ingress controller, KongService/KongRoute/KongPlugin CRDs you apply with `kubectl` are reconciled to this control plane and appear (read-only) in the Konnect UI.]{type="info"}

## Kong Setup Complete

You now have:

- A Kong Konnect account + control plane `kong-aws`
- **Kong Operator 2.x** installed (bundles the ingress controller — K8s-native CRD config)
- A DataPlane (`dataplane`) connected to Konnect and exposed via `proxy1` LoadBalancer
- The `kaigateway-podid-sa` service account created and bound to the DataPlane, ready for **EKS Pod Identity** (the Bedrock IAM role is associated with it in **Module B2** — Bedrock access is not yet active at this point)

The gateway is running but has **no AI routes yet** -- you configure the Bedrock `ai-proxy` route as **KongService / KongRoute / KongPlugin CRDs** (applied with `kubectl`) in **Module B2**.

---

**[Next: Arize AX Setup →](../arize-setup/)**
