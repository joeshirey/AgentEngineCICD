# AgentEngineCICD

A minimal Google ADK agent with a CI/CD GitHub Actions workflow for deploying to [Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview).

> [!TIP]
> **New to this repository?** Start with the [Comprehensive Walkthrough](WALKTHROUGH.md) for a deep dive into the architecture, naming conventions, and scaling strategies used in this example.

---

## Project Structure

```
AgentEngineCICD/
├── my_agent/
│   ├── agent.py          # Root agent definition
│   ├── __init__.py
│   └── .env              # Local dev environment variables (not committed)
├── requirements.txt      # Python dependencies
├── .github/
│   └── workflows/
│       ├── deploy.yml    # CI/CD deployment workflow
│       └── cleanup.yml   # Automatic branch cleanup workflow
├── SECURITY.md           # Workload Identity Federation setup guide
├── WALKTHROUGH.md        # Architectural guide & CI/CD walkthrough
└── LICENSE               # Repository license
```

---

## Local Development

### Prerequisites

- Python 3.11+
- [Google ADK](https://google.github.io/adk-docs/) (`pip install google-adk`)
- [gcloud CLI](https://cloud.google.com/sdk/docs/install) authenticated to your GCP project

### Setup

```bash
# Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### Running Locally

```bash
# Start the ADK web UI for local testing
adk web
```

The ADK web UI will be available at `http://localhost:8000`.

> **Note:** Your `my_agent/.env` file must contain:
> ```env
> GOOGLE_GENAI_USE_VERTEXAI=1
> GOOGLE_CLOUD_PROJECT=your-project-id
> GOOGLE_CLOUD_LOCATION=us-central1
> ```

---

## Deploying to Vertex AI Agent Engine

### Automated (GitHub Actions)

This project uses two workflows to manage the lifecycle of your agents:

1. **Deploy Workflow** (`deploy.yml`): Handles deployments to DEV and PROD.
2. **Cleanup Workflow** (`cleanup.yml`): Automatically deletes Agent Engine resources when branches are removed.

#### Deployment Strategy

| Environment | Trigger | Target Branch | Notes |
|-------------|---------|---------------|-------|
| **DEV** | `push` | `dev/**` | Continuous deployment on every push to your feature branch. |
| **PROD** | `merge` | `main` | Triggered automatically when a Pull Request is merged into `main`. |

#### How It Works

- **Development**: Every code push to a `dev/` branch triggers an immediate deployment. Duplicate runs are automatically deduplicated via concurrency groups.
- **Production**: A Pull Request to `main` will trigger a resolution check (to show what will happen), but the actual deployment is gated until the PR is merged.
- **Deduplication**: We use GitHub Concurrency Groups to ensure that if multiple pushes happen rapidly, only the latest one completes, saving time and resources.

#### GitHub Environments Setup

Secrets and variables are scoped **per environment** using [GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment). Go to **Settings → Environments** and create one environment per deployment target (e.g. `PROD`, `DEV`).

Each environment needs the following secrets and variables:

| Type | Name | Description |
|------|------|-------------|
| Secret | `GCP_WORKLOAD_IDENTITY_PROVIDER` | WIF provider resource name for this environment's GCP project |
| Secret | `GCP_SERVICE_ACCOUNT` | Service account email for this environment |
| Variable | `GCP_PROJECT_ID` | GCP project ID for this environment |
| Variable | `GCP_REGION` | Deployment region (e.g. `us-central1`) |
| Variable | `AGENT_SOURCE_PATH` | Optional. Path to the agent directory (defaults to `my_agent`) |

> **Display name** is set automatically — `"Production"` for `main`, or the branch name (e.g. `dev/my-feature`) for all other branches. No variable needed.

See [SECURITY.md](SECURITY.md) for the full Workload Identity Federation setup walkthrough (run it once per environment/GCP project).

#### Adding More Environments

The workflow is designed to be extended. To add a new environment (e.g. `STAGING`):

1. **Create the GitHub Environment** — Settings → Environments → New environment → `STAGING`
2. **Add its secrets and variables** (same keys as above, pointing to the STAGING GCP project)
3. **Add the branch pattern** to the `on.pull_request.branches` list in `deploy.yml`:
   ```yaml
   branches:
     - main
     - 'dev/**'
     - 'staging/**'   # ← add this
   ```
4. **Add the branch → environment mapping** in the `resolve-environment` job:
   ```yaml
   elif echo "$TARGET_BRANCH" | grep -q "^staging/"; then
     echo "environment=STAGING" >> "$GITHUB_OUTPUT"
   ```

That's it — no other changes needed.

### Automatic Cleanup

When you delete a branch prefixed with `dev/` in GitHub, the `Cleanup` workflow automatically:
1.  Identifies the associated reasoning engine in Vertex AI.
2.  Issues a `DELETE` request (with `force=true`) to clean up the engine and any active sessions.
3.  Handles concurrent operations with a 10-minute retry loop.

### Manual Deployment

You can also deploy directly from your terminal:

```bash
adk deploy agent_engine \
  --project=YOUR_PROJECT_ID \
  --region=us-central1 \
  --display_name="My Agent" \
  my_agent
```

---

## Adding Dependencies

This project uses `requirements.txt` to declare Python dependencies. The `adk deploy` command installs these dependencies inside the Agent Engine container.

For a more complex agent with additional dependencies, add them to `requirements.txt`:

```
google-adk
# Add your agent's dependencies below:
# google-cloud-bigquery
# langchain
# requests
```

> **Pinning versions:** For production deployments, pin dependency versions to ensure
> reproducible builds (e.g. `google-adk==1.2.3`). Use `pip freeze > requirements.txt`
> after validating your local environment.

> **Private packages:** If your agent uses private PyPI packages, you can pass a
> `--requirements_file` flag or configure a custom Artifact Registry repository.
> See the [ADK CLI reference](https://google.github.io/adk-docs/api-reference/cli/cli.html#adk-deploy-agent-engine)
> for all available options.

---

## References

- [ADK Deploy to Agent Engine docs](https://google.github.io/adk-docs/deploy/agent-engine/deploy/)
- [ADK CLI Reference](https://google.github.io/adk-docs/api-reference/cli/cli.html#adk-deploy-agent-engine)
- [Agent Engine overview](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview)
- [Supported regions for Agent Engine](https://cloud.google.com/agent-builder/docs/locations#supported-regions-agent-engine)