# Security — Workload Identity Federation Setup

This document walks you through setting up **Workload Identity Federation (WIF)** so GitHub Actions can authenticate to Google Cloud without storing long-lived service account keys. This is the recommended, keyless approach.

---

## Overview

WIF creates a trust relationship between GitHub and Google Cloud. When the workflow runs, GitHub mints a short-lived OIDC token, which GCP exchanges for a temporary access token scoped to a specific service account. No JSON key is ever stored or transmitted.

**Official docs:**
- [Workload Identity Federation overview](https://cloud.google.com/iam/docs/workload-identity-federation)
- [Configuring WIF for GitHub Actions](https://cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines#github-actions)
- [`google-github-actions/auth` action README](https://github.com/google-github-actions/auth)

---

## Prerequisites

- [gcloud CLI](https://cloud.google.com/sdk/docs/install) installed and authenticated
- Owner or `roles/iam.workloadIdentityPoolAdmin` on the GCP project
- The following APIs enabled in your project:
  - [IAM Credentials API](https://console.cloud.google.com/apis/library/iamcredentials.googleapis.com)
  - [Vertex AI API](https://console.cloud.google.com/apis/library/aiplatform.googleapis.com)
  - [Cloud Resource Manager API](https://console.developers.google.com/apis/api/cloudresourcemanager.googleapis.com/overview)

---

## Step 1 — Create a Service Account

This service account is what GitHub Actions will impersonate.

```bash
export PROJECT_ID="your-gcp-project-id"
export SA_NAME="github-agent-engine-deploy"

gcloud iam service-accounts create "$SA_NAME" \
  --project="$PROJECT_ID" \
  --display-name="GitHub Actions — Agent Engine Deploy"
```

Grant it the minimum roles needed to deploy to Agent Engine:

```bash
export SA_EMAIL="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

# Deploy and manage Agent Engine resources
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/aiplatform.user"

# Upload agent artifacts to GCS during deployment
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/storage.objectAdmin"
```

> [!TIP]
> Prefer a dedicated GCS bucket for the deployment artifacts instead of
> `storage.objectAdmin` on the whole project. See the
> [ADK deploy docs](https://google.github.io/adk-docs/deploy/agent-engine/deploy/)
> for the `--gcs_source_bucket` flag.

---

## Step 2 — Create a Workload Identity Pool

```bash
export POOL_NAME="github-actions-pool"

gcloud iam workload-identity-pools create "$POOL_NAME" \
  --project="$PROJECT_ID" \
  --location="global" \
  --display-name="GitHub Actions Pool"
```

---

## Step 3 — Create a Workload Identity Provider

Replace `your-github-org-or-user` and `your-repo-name` with your actual GitHub owner and repository.

```bash
export REPO_OWNER="your-github-org-or-user"
export REPO_NAME="your-repo-name"
export PROVIDER_NAME="github-provider"

gcloud iam workload-identity-pools providers create-oidc "$PROVIDER_NAME" \
  --project="$PROJECT_ID" \
  --location="global" \
  --workload-identity-pool="$POOL_NAME" \
  --display-name="GitHub Provider" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository == '${REPO_OWNER}/${REPO_NAME}'"
```

> [!IMPORTANT]
> The `--attribute-condition` restricts authentication to tokens from **your specific
> repository only**. Do not skip this — without it, any GitHub Actions workflow
> could impersonate your service account.

---

## Step 4 — Allow the Provider to Impersonate the Service Account

```bash
export PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format="value(projectNumber)")

gcloud iam service-accounts add-iam-policy-binding "$SA_EMAIL" \
  --project="$PROJECT_ID" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/${POOL_NAME}/attribute.repository/${REPO_OWNER}/${REPO_NAME}"
```

---

## Step 5 — Get the Provider Resource Name

You need this value for the `GCP_WORKLOAD_IDENTITY_PROVIDER` GitHub secret.

```bash
gcloud iam workload-identity-pools providers describe "$PROVIDER_NAME" \
  --project="$PROJECT_ID" \
  --location="global" \
  --workload-identity-pool="$POOL_NAME" \
  --format="value(name)"
```

The output looks like:
```
projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_NAME/providers/PROVIDER_NAME
```

---

## Step 6 — Configure GitHub Secrets & Variables

Go to your GitHub repository → **Settings → Secrets and variables → Actions**.

### Secrets (Settings → Secrets and variables → Actions → Secrets)

| Name | Value |
|------|-------|
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | Output from Step 5 |
| `GCP_SERVICE_ACCOUNT` | `github-agent-engine-deploy@<PROJECT_ID>.iam.gserviceaccount.com` |

### Variables (Settings → Secrets and variables → Actions → Variables)

| Name | Example value | Description |
|------|--------------|-------------|
| `GCP_PROJECT_ID` | `my-project-123` | GCP project ID |
| `GCP_REGION` | `us-central1` | Agent Engine deployment region ([supported regions](https://cloud.google.com/agent-builder/docs/locations#supported-regions-agent-engine)) |
| `AGENT_SOURCE_PATH` | `my_agent` | Optional. Path to the agent directory (defaults to `my_agent`) |

> **Display name** is set automatically by the workflow — `"Production"` for `main`, or the branch name for all other environments. No variable needed.

---

## Verification

After completing setup, trigger the workflow by merging a pull request into `main` (deploys to PROD) or a `dev/**` branch (deploys to DEV).

If authentication fails, the most common causes are:
- The `--attribute-condition` in Step 3 doesn't match your actual `REPO_OWNER/REPO_NAME`
- The `principalSet` binding in Step 4 uses the wrong project number or pool name
- The IAM Credentials API is not enabled

---

## References

| Resource | Link |
|----------|------|
| WIF with GitHub Actions | https://cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines#github-actions |
| `google-github-actions/auth` | https://github.com/google-github-actions/auth |
| ADK Deploy docs | https://google.github.io/adk-docs/deploy/agent-engine/deploy/ |
| Agent Engine regions | https://cloud.google.com/agent-builder/docs/locations#supported-regions-agent-engine |
