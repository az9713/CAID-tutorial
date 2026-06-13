# Prerequisites

Everything you need before running CAID.

---

### Python 3.12+

Verify:
```bash
python --version
# Should print Python 3.12.x or higher
```

Install: [python.org/downloads](https://www.python.org/downloads/) or via your OS package manager.

---

### uv

CAID uses [uv](https://docs.astral.sh/uv/) for dependency management.

Verify:
```bash
uv --version
# Should print uv 0.x.x
```

Install:
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

---

### Docker

OpenHands workspaces run inside Docker containers. Docker Desktop or Docker Engine both work.

Verify:
```bash
docker --version
# Should print Docker version 24.x or higher
docker ps
# Should return without error (daemon is running)
```

Install: [docs.docker.com/get-docker](https://docs.docker.com/get-docker/)

> **Note:** On Linux, add your user to the `docker` group so you can run Docker without `sudo`: `sudo usermod -aG docker $USER`

---

### LLM API credentials

CAID routes all LLM calls through a LiteLLM proxy. Set these environment variables:

```bash
export LLM_BASE_URL=<your-proxy-url>
export LLM_API_KEY=<your-api-key>
```

To use a specific model by default (instead of passing `--model` on every run):

```bash
export LLM_MODEL=litellm_proxy/neulab/gpt-5-mini
```

You can also set a separate model for subagents:

```bash
export LLM_SUBAGENT_MODEL=litellm_proxy/neulab/gpt-5-mini
```

> **Tip:** Add these exports to your `.bashrc` or `.zshrc` to avoid re-setting them every session.

---

### Disk space

Each run creates a Docker container with a full copy of the target repository and all dependencies. Expect:

- Commit0: 2–5 GB per run (depends on repository and Docker layers)
- PaperBench: 5–20 GB per run (research codebases with large datasets)

---

### Task-specific data

Before running, download the benchmark dataset for your task:

- **Commit0**: see [commit0 data setup](../getting-started/quickstart.md#commit0-data)
- **PaperBench**: see [paperbench data setup](../getting-started/quickstart.md#paperbench-data)
