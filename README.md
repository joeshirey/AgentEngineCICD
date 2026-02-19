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

The `Deploy to Vertex AI Agent Engine` workflow in `.github/workflows/deploy.yml` deploys the agent to Vertex AI Agent Engine. It is triggered **manually** from the GitHub Actions tab.

**Before running the workflow**, you must complete the one-time setup described in [SECURITY.md](SECURITY.md) (Workload Identity Federation) and configure the following GitHub secrets and variables:

| Type | Name | Description |
|------|------|-------------|
| Secret | `GCP_WORKLOAD_IDENTITY_PROVIDER` | WIF provider resource name |
| Secret | `GCP_SERVICE_ACCOUNT` | Service account email |
| Variable | `GCP_PROJECT_ID` | GCP project ID |
| Variable | `GCP_REGION` | Deployment region (e.g. `us-central1`) |
| Variable | `AGENT_DISPLAY_NAME` | Display name in Agent Engine |

To trigger a deployment: **GitHub → Actions → Deploy to Vertex AI Agent Engine → Run workflow**.

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