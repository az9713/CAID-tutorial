# Configuration reference

All parameters for controlling a CAID run, as CLI arguments to `run_infer.py` and as fields in `WorkflowConfig`.

---

## CLI arguments

Pass these to `uv run python run_infer.py` or set them at the top of `scripts/run_multi.sh` / `scripts/run_single.sh`.

### Core arguments

| Argument | Type | Default | Description |
|----------|------|---------|-------------|
| `--task` | string | `"commit0"` | Task to run. One of: `"commit0"`, `"paperbench"`, or a custom task name registered in `build_task_module()`. |
| `--model` | string | `$LLM_MODEL` or `"litellm_proxy/neulab/gpt-5-mini"` | LiteLLM model identifier for the manager agent. Falls back to the `LLM_MODEL` environment variable. |
| `--multi_agent` / `--nomulti_agent` | bool | `True` | Run in multi-agent mode (default) or single-agent baseline mode. |
| `--max_iterations` | int | `50` | Maximum LLM iterations for the manager per conversation run (scan, delegate, assign, final review each count separately). |
| `--max_subagents` | int | `4` | Maximum number of parallel engineer subagents. |
| `--sub_iterations` | int | `50` | Maximum LLM iterations per engineer per round. |
| `--rounds_of_chat` | int | `2` | Maximum number of rounds (task assignments) per engineer. |
| `--subagent_model` | string | `None` | LiteLLM model identifier for engineer agents. Falls back to `$LLM_SUBAGENT_MODEL`, then to `--model`. |
| `--output_dir` | string | auto-generated | Override the output directory. If not set, auto-generated as `outputs/{task}/{model}/{id}/{mode}/{params}/`. |

### Commit0-specific arguments

| Argument | Type | Default | Description |
|----------|------|---------|-------------|
| `--repo` | string | required | Repository name from the commit0_combined dataset (e.g., `"minitorch"`, `"sympy"`). |
| `--dataset_path` | string | `None` | Override the path to the commit0_combined dataset. Default is resolved by `Commit0Task`. |

### PaperBench-specific arguments

| Argument | Type | Default | Description |
|----------|------|---------|-------------|
| `--paper_id` | string | required | Paper identifier matching a directory in `data/paperbench/papers/` (e.g., `"rice"`). |
| `--paperbench_dir` | string | `None` | Override the path to the PaperBench data directory. |
| `--judge_model` | string | `"gpt-5-mini"` | Model used by the LLM judge to score the paper reproduction. |
| `--judge_type` | string | `"simple"` | Judge type. `"simple"` uses the paperbench simple judge. |
| `--test_max_depth` | int | `999` | Maximum rubric tree depth to evaluate. `999` means evaluate all levels. |
| `--test_reproduce_timeout` | int | `300` | Timeout in seconds for running `reproduce.sh`. |
| `--code_dev` / `--nocode_dev` | bool | `True` | Whether the paper requires code development (vs. just running existing code). |

---

## WorkflowConfig fields

`WorkflowConfig` (`config.py`) is the dataclass built from CLI arguments. All fields map directly to CLI arguments.

| Field | Type | Default | CLI argument |
|-------|------|---------|-------------|
| `model` | `Optional[str]` | `None` | `--model` |
| `subagent_model` | `Optional[str]` | `None` | `--subagent_model` |
| `manager_max_iterations` | `int` | `50` | `--max_iterations` |
| `max_subagents` | `int` | `4` | `--max_subagents` |
| `subagent_max_iterations` | `int` | `50` | `--sub_iterations` |
| `max_rounds_chat` | `int` | `2` | `--rounds_of_chat` |
| `output_dir` | `str` | `"outputs"` | `--output_dir` |

---

## Environment variables

| Variable | Description |
|----------|-------------|
| `LLM_BASE_URL` | Base URL for the LiteLLM proxy (required) |
| `LLM_API_KEY` | API key for the LiteLLM proxy (required) |
| `LLM_MODEL` | Default model identifier. Used when `--model` is not passed. |
| `LLM_SUBAGENT_MODEL` | Default subagent model. Used when `--subagent_model` is not passed. |
| `SDK_SOURCE_DIR` | Path to the OpenHands SDK source directory. Default: `../software-agent-sdk` relative to `run_infer.py`. Only needed when using `DockerDevWorkspace`. |
| `JUDGE_PYTHON` | Path to the Python executable for the PaperBench judge. Required for PaperBench evaluation. |

---

## Tuning guidance

**Run is too slow:**
- Reduce `--max_subagents` (fewer concurrent Docker operations)
- Reduce `--max_iterations` and `--sub_iterations` (engineers stop sooner)
- Use a faster/cheaper model for subagents: `--subagent_model litellm_proxy/.../faster-model`

**Engineers frequently fail to commit:**
- Increase `--sub_iterations` (more time to finish implementation)
- Increase `--rounds_of_chat` (more retries)
- Check if the task's prompt gives clear enough instructions

**Manager delegation quality is poor:**
- Increase `--max_iterations` (more time to scan before delegating)
- Review `delegations.json` to see the manager's reasoning
- Improve the `scan_analysis` and `task_delegation` prompts in your task's YAML

**Cost is too high:**
- Use a cheaper model for subagents: `--subagent_model litellm_proxy/.../cheap-model`
- Reduce `--rounds_of_chat` to 1
- Reduce `--max_subagents` to 2
