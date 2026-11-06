---
title: "Arize AX Account & Credentials"
weight: 2
---

This page sets up **Arize AX** -- the SaaS observability and evaluation platform that replaces Langfuse in the Track B stack. You will create an account, set up a Space, generate a **Service** API key, and install the credentials into your cluster so agents can export OpenInference traces.

::alert[Arize AX is fully hosted at [app.arize.com](https://app.arize.com). There is nothing to deploy in-cluster -- this setup registers your account and stores the credentials your agents use to send traces.]{type="info"}

## Step 1: Create an Arize AX Account

1. Go to [app.arize.com](https://app.arize.com) and click **Sign Up** (a free tier is available). Use a personal email such as a Gmail address, so every participant follows the same self-service sign-up flow.

![Arize AX log-in page — click Sign Up to create an account](/static/images/track-b-saas/b1-platform-setup/arize-initial-page-1.png)

2. On the **Get started for free** page, leave **Data Region** as **US**, enter your email, and click **Sign up with email**. Confirm your email if prompted, then log in.

![Arize AX Get started for free — Sign up with email](/static/images/track-b-saas/b1-platform-setup/arize-initial-page-2.png)

3. On **Set up your workspace**, enter your name under **Your name** and choose a **Space name** (for example `my-aws-arize-space`), then click **Continue**. Arize creates your **Organization** and this **Space**.

![Arize AX Set up your workspace — enter Your name and a Space name](/static/images/track-b-saas/b1-platform-setup/arize-initial-page-3.png)

## Step 2: Copy Space ID

On first login, Arize shows the **Get started with Arize** page (manual setup is under **Instrument Manually**, top-right). You do **not** need to create a User API key here — the workshop's agents authenticate with a dedicated **Service key** you create in Step 3. From here you only need to copy your **Space ID**.

![Arize AX — Get started with Arize (Instrument Manually, top-right)](/static/images/track-b-saas/b1-platform-setup/arize-initial-get-started-page.png)

On the Arize AX home for your Space (**my-aws-arize-space**, selected top-left), the left nav shows **Tracing Projects**, **Dashboards**, **Evaluators**, **Monitors**, and more -- the observability and evaluation features that replace Langfuse. Click the **Space ID** button (top-right) to copy your Space ID.

![Arize AX home — click the Space ID button (top-right) to copy your Space ID](/static/images/track-b-saas/b1-platform-setup/arize-initial-page-8.png)

Export your Space ID and the (fixed) OTLP endpoint — the two values you keep from here:

:::code{language=bash showCopyAction=true}
# Space ID (from the Space ID button) + the fixed OTLP endpoint
export ARIZE_SPACE_ID="U3BhY2U6..."                  # your Space ID
export ARIZE_OTLP_ENDPOINT="https://otlp.arize.com/v1"
:::

## Step 3: Generate a Service API Key

The **Service key** is the persistent credential your agents use to send traces (unlike the User key from Step 2). Create one from **Settings → API Keys**.

1. Open **Settings → API Keys**, select the **Service Keys** tab, and click **+ New Service Key** (top-right).

![Arize AX API Keys — Service Keys tab, New Service Key](/static/images/track-b-saas/b1-platform-setup/arize-initial-page-9.png)

2. In **Create Service Key**, give it any name (for example `my-aws-arize-service-key`), set the **Account Role** and **Organization Role** to **Admin**, leave **Expiration** as **Never**, keep **Default Org**, and add your Space **`my-aws-arize-space`** with the **Space Admin** role. Then click **Create Service Key**.

![Arize AX Create Service Key — name, Admin Account/Organization roles, Never expiration, Default Org, and my-aws-arize-space with the Space Admin role](/static/images/track-b-saas/b1-platform-setup/arize-service-key-creation.png)

::alert[**Admin everywhere is for this workshop/demo only.** In production, grant a **least-privileged** role scoped to just the Space and the permissions the agent actually needs (trace ingestion) -- not Admin.]{type="warning"}

3. On **Service Key Created**, click the copy icon to copy the key -- you will **not** be able to see it again -- then export it as `ARIZE_API_KEY`:

![Arize AX Service Key Created — copy the key](/static/images/track-b-saas/b1-platform-setup/arize-initial-page-11.png)

:::code{language=bash showCopyAction=true}
# Save your SERVICE API key (ak-..., ~72 chars) — this is the value the agents use
export ARIZE_API_KEY="ak-xxxxxxxx-...-xxxxxxxx"
:::

## Step 4: Install the Credentials into Your Cluster

The Track B stack's `o11y/arize` component stores your Arize credentials as a Kubernetes Secret (`arize-credentials` in the `arize` namespace). Agents reference this Secret to export traces.

::alert[**When this Secret is needed:** only the **in-cluster** Strands agent in Module B3 requires it (the agent mounts the Secret via `secretKeyRef`). If you are only running Arize scripts from your terminal (such as the smoke test in Step 6), you only need `ARIZE_API_KEY` and `ARIZE_SPACE_ID` exported as environment variables -- no cluster Secret required. Install it here so it is ready before you deploy the agent.]{type="info"}

Put your credentials in `.env.local` (do not edit the committed `.env`), then run the starter-kit CLI. The `o11y/arize` component creates the `arize` namespace and the `arize-credentials` Secret for you:

:::code{language=bash showCopyAction=true}
# Put your creds in .env.local (do not edit the committed .env)
cat >> /workshop/.env.local <<EOF
ARIZE_API_KEY=$ARIZE_API_KEY
ARIZE_SPACE_ID=$ARIZE_SPACE_ID
ARIZE_OTLP_ENDPOINT=$ARIZE_OTLP_ENDPOINT
EOF

cd /workshop && ./cli o11y arize install
:::

::::expand{header="Check Output"}

```bash
$ kubectl config use-context genai-on-eks-us-east-1
Switched to context "genai-on-eks-us-east-1".
$ kubectl create namespace arize --dry-run=client -o yaml | kubectl apply -f -
namespace/arize created
$ kubectl create secret generic arize-credentials     --namespace arize     --from-literal=ARIZE_API_KEY=ak-xxxxxxxx-...-xxxxxxxx     --from-literal=ARIZE_SPACE_ID=U3BhY2U6...     --from-literal=ARIZE_OTLP_ENDPOINT=https://otlp.arize.com/v1     --dry-run=client -o yaml | kubectl apply -f -
secret/arize-credentials created

Arize AX (SaaS) configured. Traces will be sent to https://otlp.arize.com/v1.
Secret 'arize-credentials' created in namespace 'arize'.
Agents export OpenInference spans directly to Arize AX (app.arize.com).
```

::::

## Step 5: Verify the Credentials

:::code{language=bash showCopyAction=true}
kubectl get secret arize-credentials -n arize \
  -o jsonpath='{.data.ARIZE_OTLP_ENDPOINT}' | base64 -d; echo
:::

::::expand{header="Check Output"}

```bash
https://otlp.arize.com/v1
```

::::

## Step 6: (Optional) Send a Test Trace

Confirm your key works by sending one span. The reliable, portable way is to create a **dedicated virtual environment**, install into it, and run with **that same venv's python**. This guarantees `pip` and `python` are the same interpreter (avoiding `ModuleNotFoundError` when the shell's `python3` is a different/aliased interpreter than the one pip installed into).

::alert[**Workshop Studio Code Editor:** its Python 3.12 ships **without** the `venv`/`pip` modules, so `python3 -m venv` fails with `ensurepip is not available`. Run Step 0 below first (`sudo` is passwordless on the Code Editor). On AWS CloudShell or a local machine with a full Python install, you can skip Step 0.]{type="info"}

:::code{language=bash showCopyAction=true}
# 0. Code Editor only: install the venv module (Python 3.12 ships without it)
sudo apt-get update -qq && sudo apt-get install -y python3.12-venv

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
    project_name="arize-trace-smoketest",
    set_global_tracer_provider=True,
)
from opentelemetry import trace
with trace.get_tracer("smoke").start_as_current_span("hello-arize"):
    print("sent a span to Arize AX project 'arize-trace-smoketest'")
if hasattr(tp, "force_flush"): tp.force_flush()
PY
:::

::alert[**Why a dedicated venv is used:** `pip install ...` followed by `python3 ...` fails with `ModuleNotFoundError: No module named 'arize'` whenever the shell's `python3` resolves to a *different* interpreter than `pip` wrote to (common on macOS where `python3` is aliased to Homebrew, and in some managed environments). Creating `/tmp/arize-venv` and using **`/tmp/arize-venv/bin/pip`** + **`/tmp/arize-venv/bin/python`** makes them the same binary by construction — portable across the Code Editor, CloudShell, and local machines. Sanity check: `/tmp/arize-venv/bin/python -c "from arize.otel import register; print('arize-otel OK')"` (note: `import arize; arize.__file__` prints `None` because `arize` is a namespace package — that is normal, not a failure).]{type="info"}

Open [app.arize.com](https://app.arize.com) → **Tracing Projects** → project **`arize-trace-smoketest`** → **Traces** tab, and confirm the `hello-ar`(ize) span appears (1 trace, 1 span):

![Arize AX smoke-test trace in arize-trace-smoketest project](/static/images/track-b-saas/b1-platform-setup/arize-smoke-test.png)

If you see `PERMISSION_DENIED`, you are using the user key -- switch to the **Service** key (Step 3). If the trace does not appear, confirm you are on the correct **Space** (top-left selector) and that the time filter (top-right) covers the last few minutes.

## Arize Setup Complete

You now have:

- An Arize AX account with a Space and Space ID
- A **Service** API key that works for trace ingestion
- The `arize-credentials` Secret installed in the cluster

Agents in Module B3 will use these credentials to export full OpenInference traces (LLM + tool spans) to Arize AX.

::alert[**You now have both control-plane tokens** — the **Kong PAT** (from [Kong Setup](../kong-setup/)) and this **Arize Service key + Space ID**. If you set up the [Claude Code co-pilot](/introduction/claude-code-copilot/), go back to its **"Configure it for control-plane debugging"** section and export `KONNECT_TOKEN`, `ARIZE_API_KEY`, and `ARIZE_SPACE_ID`, then restart the Claude session — the co-pilot can then query Kong Konnect and Arize live to help you debug the rest of the workshop.]{type="success"}

---

**[Back to Track B Stack overview →](../) · [Begin Module B2: Kong AI Gateway →](/track-b-saas/b2-ai-gateway-observability/ai-gateway-kong/)**
