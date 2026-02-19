# AgentEngineCICD

A minimal Google ADK agent with a CI/CD GitHub Actions workflow for deploying to [Vertex AI Agent Engine](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview).

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
│       └── deploy.yml    # CI/CD workflow
└── SECURITY.md           # Workload Identity Federation setup guide
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

The `Deploy to Vertex AI Agent Engine` workflow (`.github/workflows/deploy.yml`) automatically deploys the agent when a **pull request is merged**. The target environment is resolved from the PR's base branch:

| Base branch | GitHub Environment | GCP target |
|-------------|-------------------|------------|
| `main` | `PROD` | Production project/region |
| `dev/**` | `DEV` | Development project/region |

The workflow is ignored for all other branches.

#### How It Works

1. A PR is opened targeting `main` or a `dev/*` branch
2. When the PR is **merged** (not just closed), the workflow triggers
3. The `resolve-environment` job maps the base branch to a GitHub Environment
4. The `deploy` job runs within that environment, pulling its own secrets and variables
5. `adk deploy agent_engine` packages and deploys the agent to the correct GCP project

#### GitHub Environments Setup

Secrets and variables are scoped **per environment** using [GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment). Go to **Settings → Environments** and create one environment per deployment target (e.g. `PROD`, `DEV`).

Each environment needs the following secrets and variables:

| Type | Name | Description |
|------|------|-------------|
| Secret | `GCP_WORKLOAD_IDENTITY_PROVIDER` | WIF provider resource name for this environment's GCP project |
| Secret | `GCP_SERVICE_ACCOUNT` | Service account email for this environment |
| Variable | `GCP_PROJECT_ID` | GCP project ID for this environment |
| Variable | `GCP_REGION` | Deployment region (e.g. `us-central1`) |
| Variable | `AGENT_DISPLAY_NAME` | Display name in Agent Engine |

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
   elif echo "$BASE_BRANCH" | grep -q "^staging/"; then
     echo "environment=STAGING" >> "$GITHUB_OUTPUT"
   ```

That's it — no other changes needed.

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