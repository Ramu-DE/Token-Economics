---
title: "Security Hardening — Gateway Guardrails"
weight: 20
---

# 🔐 Security Hardening for Production

Before Loan Buddy handles real applicant data, it must meet production security requirements: no hardcoded secrets, least-privilege IAM, and content safety. On the SaaS stack, the IAM and secrets work is shared with Track A, while **content safety gains a gateway-level layer** that Track A's app-side approach doesn't have.

::alert[**Pod Identity and External Secrets are stack-independent** — they secure the pod and its credentials regardless of gateway. Follow Track A for those, then apply the **Track-B content-safety deltas** below.]{type="info"}

## 🔐 Part 1: Pod Identity — Least-Privilege IAM (follow Track A)

Complete **[Track A's Pod Identity section](/track-a-opensource/a4-production-ready/security-hardening/)** — it applies unchanged. One important Track-B note on *what* IAM the agent needs:

::alert[**The Strands agent needs LESS IAM than Track A's.** Because Bedrock is reached **through Kong** (the Kong DataPlane holds the Bedrock IAM via its own Pod Identity from B1), the Strands agent's Pod Identity grants **S3 only** (image storage) — **not** `bedrock:InvokeModel`. This is a genuine Track-B security advantage: the application never holds Bedrock credentials. This was already set up in [B3](/track-b-saas/b3-genai-applications-strands/); production-harden it here (scope the S3 policy to the specific bucket ARN).]{type="success"}

## 🔑 Part 2: External Secrets Operator (follow Track A)

Complete **[Track A's External Secrets section](/track-a-opensource/a4-production-ready/security-hardening/)** — deploy ESO and sync credentials from AWS Secrets Manager. The Track-B delta is *which* secrets:

| Secret | Track A | Track B |
|---|---|---|
| LLM gateway auth | LiteLLM Virtual Key (`sk-...`) | Kong consumer key (`apikey`) → store in Secrets Manager, sync via ESO |
| Observability | Langfuse keys | **Arize** Service key + Space ID (the `arize-credentials` Secret from B1) |

::alert[**Harden the Arize + Kong credentials:** in B1-B3 the `arize-credentials` Secret and the Kong `apikey` were created directly. For production, store them in **AWS Secrets Manager** and sync via ESO (exactly as Track A does for its keys) so nothing sensitive lives in plain Kubernetes Secrets or manifests.]{type="info"}

## 🛡️ Part 3: Content Safety — Two Layers on Track B

This is where Track B differs most. Track A applies Bedrock Guardrails through LiteLLM. Track B has **two complementary options** — a gateway-level guard and app-level Bedrock Guardrails.

### Layer 1: Kong `ai-prompt-guard` (gateway-enforced) — Track-B delta

Kong's **`ai-prompt-guard`** plugin filters chat requests **at the gateway**, before they reach Bedrock — allow/deny lists (PCRE regex) on user messages, plus detection of hidden-unicode prompt-injection attempts. Requests that violate the rules get a **400 at the edge**, so bad input never consumes a Bedrock call.

:::code{language=yaml showCopyAction=true}
cat <<'EOF' | kubectl apply -f -
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: ai-prompt-guard-loan-strands
  namespace: kong
plugin: ai-prompt-guard
config:
  # deny obvious injection / out-of-scope requests; deny wins over allow
  deny_patterns:
    - "(?i)ignore (all|previous) instructions"
    - "(?i)system prompt"
  allow_patterns:
    - ".*"
EOF
:::

Attach it to the `/loan-strands` route (add to the route's `konghq.com/plugins` annotation alongside `ai-proxy` + `key-auth` from B2):

:::code{language=bash showCopyAction=true}
kubectl annotate kongroute loan-strands-route -n kong \
  konghq.com/plugins=ai-proxy-loan-strands,key-auth-loan-strands,prometheus-metrics,ai-prompt-guard-loan-strands --overwrite
:::

This is **gateway-enforced safety** — it protects every consumer of the route regardless of the application code, something an app-side-only approach can't guarantee.

::alert[**Verified:** `ai-prompt-guard` is a Kong plugin that scans `user`-role messages against allow/deny regex and blocks violations with a 400, and detects hidden-unicode injection. Supported via the Kong Ingress Controller (KongPlugin CRD).]{type="success"}

### Layer 2: Amazon Bedrock Guardrails (app-side) — parity with Track A

For managed content policies (PII redaction, denied topics, toxicity), use **Amazon Bedrock Guardrails**. In Track B, attach the guardrail on the **application's model call** — the Strands `BedrockModel` (or via the OpenAI-compatible request `extra_body`), passing `guardrailIdentifier` + `guardrailVersion`.

::alert[**Honest scope note:** whether Kong's `ai-proxy` can *inject* Bedrock Guardrails into the Bedrock call at the gateway (so the app doesn't specify them) is **not confirmed** in the current Kong docs I verified — `ai-proxy` documents `extra_body` pass-through but not a first-class guardrail option. Until verified against the Kong Bedrock provider reference for your version, treat **Bedrock Guardrails as app-side** (set on the agent's model call) and use **`ai-prompt-guard` for the gateway-enforced layer**. Do not claim gateway-injected Bedrock Guardrails without verifying it live.]{type="warning"}

Create the guardrail once (same as Track A's Step 8), then reference it from the agent:

:::code{language=python showCopyAction=true}
# Strands agent — attach a Bedrock Guardrail to the model call (app-side)
# via the OpenAI-compatible request the agent sends through Kong:
params={
    "max_tokens": 5000,
    "temperature": 0,
    "extra_headers": {"apikey": KONG_API_KEY},
    "extra_body": {
        "guardrailConfig": {
            "guardrailIdentifier": GUARDRAIL_ID,
            "guardrailVersion": GUARDRAIL_VERSION,
        }
    },
}
:::

::alert[**Verify the pass-through path live:** confirm Kong's `ai-proxy` forwards `extra_body.guardrailConfig` to Bedrock's Converse/InvokeModel for your Kong version before relying on it. If it does not pass through, apply Bedrock Guardrails by calling Bedrock's guardrail-enabled API directly, or rely on Layer 1 (`ai-prompt-guard`) plus Bedrock Guardrails configured at the model/inference-profile level.]{type="warning"}

## 📊 How It Maps to Track A

| | Track A | Track B |
|---|---|---|
| Pod Identity | app holds Bedrock + S3 IAM | **app holds S3 only** (Bedrock via Kong) — stronger |
| External Secrets | LiteLLM + Langfuse keys | Kong key + Arize creds via ESO |
| Gateway content guard | — (app-side only) | **`ai-prompt-guard`** at Kong edge (delta) |
| Managed content policy | Bedrock Guardrails via LiteLLM | Bedrock Guardrails **app-side** (gateway-inject unverified) |

## ✅ What You Accomplished

- ✅ Reused Track A's Pod Identity + External Secrets (stack-independent)
- ✅ Confirmed the Track-B IAM advantage: agent holds **S3 only**, never Bedrock credentials
- ✅ Added **gateway-enforced** content safety with Kong `ai-prompt-guard`
- ✅ Applied Bedrock Guardrails app-side, with an honest note on the unverified gateway-inject path

---

**[Next: Streaming Responses and Production Testing →](../streaming-testing/)**
