# Running PaperBench locally (this machine)

Local runbook for running CAID on a PaperBench task on this Windows box. Generated during setup; references absolute local paths and the gitignored `frontier-evals/` + `.judge-venv/`.

> For the **full story** — every blocker, the clean step-by-step replication path, and the **measured results** of all three runs — see [`JOURNEY.md`](JOURNEY.md). This runbook is the quick-reference; `JOURNEY.md` is the complete record.

## What's already done

| Step | Status |
|------|--------|
| `uv sync` (main project deps) | ✅ done |
| `.env` auto-loading wired into `run_infer.py` | ✅ done (via `python-dotenv`) |
| PaperBench dataset for `rice` (PDF, markdown, rubric, assets) | ✅ pulled + scaffolded at `data/paperbench/papers/rice/` |
| Judge packages (`paperbench`, `preparedness_turn_completer`) | ✅ installed in `frontier-evals/.judge-venv` |
| Windows `resource` shim for the judge | ✅ installed in judge venv site-packages |
| Judge imports + resolves `rice` from `data/paperbench` | ✅ verified |
| CAID `PaperbenchTask` loads in main venv | ✅ verified |

## What you still need to do

### 1. Start Docker Desktop
The agent runs inside a Linux container. The daemon is currently down. Start Docker Desktop and confirm:
```bash
docker ps
```
returns without error.

### 2. Create your `.env`
Copy the template and fill in your key + base URL:
```bash
cp .env.example .env
```
Then edit `.env` — set `LLM_API_KEY` and `LLM_BASE_URL`. `LLM_MODEL`, `JUDGE_PYTHON`, and `SDK_SOURCE_DIR` are already pre-filled correctly.

Recommended (OpenAI direct — serves both the agent and the OpenAI-native judge with one key):
```
LLM_API_KEY=sk-...your-openai-key...
LLM_BASE_URL=https://api.openai.com/v1
LLM_MODEL=openai/gpt-4o-mini
JUDGE_PYTHON=C:/Users/<you>/Downloads/CAID_cmu/frontier-evals/.judge-venv/Scripts/python.exe
SDK_SOURCE_DIR=C:/Users/<you>/Downloads/CAID_cmu
```

> **Gotcha:** `run_infer.py` unconditionally `os.chdir()`s to `SDK_SOURCE_DIR` before building the Docker workspace. It defaults to a non-existent `../software-agent-sdk` and crashes if unset. PaperBench uses the pre-built image, so any existing dir works — the repo root is fine.

> The `! ` prefix runs a command in this session if you'd rather I see the output — e.g. `! docker ps`.

## Run it (single-agent smoke test, rice)

```bash
uv run python run_infer.py \
  --task paperbench \
  --paper_id rice \
  --nomulti_agent \
  --max_iterations 30 \
  --model openai/gpt-4o-mini \
  --paperbench_dir data/paperbench \
  --test_max_depth 999 \
  --test_reproduce_timeout 300 \
  --judge_type simple \
  --judge_model gpt-4o-mini \
  --code_dev
```

Why these choices:
- `--nomulti_agent` — single agent first, to validate the pipeline (workspace → paper upload → `reproduce.sh` → judge) cheaply before a full CAID run.
- `--max_iterations 30` — low cap to keep the smoke test cheap. Raise to 100+ for a serious attempt.
- Call `run_infer.py` directly, **not** `scripts/run_single.sh` — the bash script sets `JUDGE_PYTHON=""`, which breaks the judge subprocess (empty string overrides the fallback), and assumes Git-Bash.

## Then run the real thing (multi-agent CAID)

Once the smoke test confirms the plumbing:
```bash
uv run python run_infer.py \
  --task paperbench \
  --paper_id rice \
  --max_iterations 50 \
  --max_subagents 2 \
  --sub_iterations 80 \
  --rounds_of_chat 2 \
  --model openai/gpt-4o-mini \
  --paperbench_dir data/paperbench \
  --test_max_depth 999 \
  --test_reproduce_timeout 300 \
  --judge_type simple \
  --judge_model gpt-4o-mini \
  --code_dev
```

## Output

Results land in `outputs/paperbench/gpt-5-mini/rice/<mode>/<params>/`:
- `grade.json` — judge score + per-rubric-node verdicts
- `cost.json`, `runtime.txt`, `outputs.jsonl` — metrics and event log

See [docs/guides/interpret-output.md](docs/guides/interpret-output.md).

## Judge venv needs `fire` + `litellm` (easy to miss)

`judge/judge_runner.py` imports `fire` (CLI) and `litellm` (`cost_per_token`), but **neither is a dependency of the `paperbench` editable install**, so a freshly-built judge venv lacks them. The judge subprocess then dies with `ModuleNotFoundError: No module named 'fire'` and `grade.json` comes back with `score: null`. Install them into the judge venv:

```bash
# <JUDGE_PYTHON> = the interpreter your JUDGE_PYTHON env var points at
uv pip install --python <JUDGE_PYTHON> fire litellm
# verify:
<JUDGE_PYTHON> -c "import fire; from litellm import cost_per_token; import paperbench; print('judge deps OK')"
```

This is a one-time fix per judge venv (already applied to the current WSL judge venv at `~/CAID_cmu/frontier-evals/.judge-venv`).

## Caveats

- **No GPU.** Docker Desktop on Windows typically has no NVIDIA passthrough. Compute-heavy reproductions may fail or score low even with a correct setup — the agent is told it has no GPU. `rice` is relatively tractable but not guaranteed. On this CPU-only box, RL reproductions (`rice`) fail and the score is ~0 regardless of model quality — that's a hardware ceiling, not a setup issue.
- **Judge 600s subprocess timeout.** `tasks/paperbench.py` runs the judge with a **hardcoded `timeout=600`**. Grading `rice` at `--test_max_depth 999` (~40+ leaf nodes, one LLM call each) can exceed 600s, so the judge dies with `TimeoutExpired` and `grade.json` gets `score: null` even though the judge ran. To get a recorded score: lower `--test_max_depth` (e.g. `1` grades only the top level and finishes fast), or raise the `timeout=600` in `tasks/paperbench.py`'s `evaluate()`.
- **`judge_model`.** Set to `gpt-4o-mini` — the judge calls OpenAI directly (via `LLM_BASE_URL`/`LLM_API_KEY`), so use the bare model id without the `openai/` prefix.
- **Running under WSL (current setup):** the working install lives in WSL at `~/CAID_cmu` (not the Windows paths above, which reflect the initial Windows attempt). Run with `~/.local/bin/uv run --no-sync python run_infer.py ...`; `JUDGE_PYTHON` / `SDK_SOURCE_DIR` in `.env` point at Linux paths. See `WSL_TROUBLESHOOTING.md` for why.

## Adding another paper later

The dataset uses Git LFS with `project/paperbench/data/**` excluded by default. To pull and scaffold a different paper (e.g. `pinn`):
```bash
cd frontier-evals
git -c lfs.fetchexclude="" lfs fetch origin main --include "project/paperbench/data/papers/pinn/**"
git -c lfs.fetchexclude="" lfs checkout
cd ..
cp -r frontier-evals/project/paperbench/data/papers/pinn data/paperbench/papers/pinn
```
