# Quickstart

Run your first CAID experiment in under 15 minutes.

This takes about 10 minutes of setup and then however long the agent run takes (typically 20–60 minutes depending on the task and model).

Prerequisites: [Python 3.12+, uv, Docker, API credentials](prerequisites.md)

---

## 1. Clone and install

```bash
git clone https://github.com/<your-org>/async-swe-agents.git
cd async-swe-agents
uv sync
```

Expected output:
```
Resolved N packages in Xs
Installed N packages in Xs
```

---

## 2. Set environment variables

```bash
export LLM_BASE_URL=https://your-proxy.example.com
export LLM_API_KEY=your-api-key
export LLM_MODEL=litellm_proxy/neulab/gpt-5-mini
```

---

## 3. Download task data

Choose **one** of the tasks below.

### Commit0 data

Download the [commit0_combined](https://huggingface.co/datasets/wentingzhao/commit0_combined) dataset and place it at:

```
data/commit0/commit0_combined/
```

The directory structure should look like:

```
data/
└── commit0/
    └── commit0_combined/
        ├── minitorch/
        ├── sympy/
        └── ...
```

### PaperBench data

Clone OpenAI's frontier-evals repository and copy the data:

```bash
git clone https://github.com/openai/frontier-evals.git
```

Place the PaperBench data at:

```
data/paperbench/
├── papers/
│   └── rice/
│       ├── config.yaml
│       ├── paper.pdf
│       ├── paper.md
│       ├── rubric.json
│       └── addendum.md
└── src/
    └── paperbench/
        └── instructions/
            └── instructions.txt
```

Then install the PaperBench judge packages (not on PyPI):

```bash
cd frontier-evals
uv pip install -e "project/paperbench"
uv pip install -e "project/preparedness_turn_completer"
cd ..
```

---

## 4. Run in single-agent mode (recommended first run)

Single-agent mode runs one LLM that implements the entire task. It's faster to set up and useful for validating your configuration before trying multi-agent.

```bash
bash scripts/run_single.sh
```

Before running, open `scripts/run_single.sh` and set the parameters at the top:

```bash
task="commit0"          # or "paperbench"
model=""                # leave empty to use LLM_MODEL env var
max_iterations=100
repo="minitorch"        # (commit0 only) repository name
paper_id="rice"         # (paperbench only) paper identifier
```

---

## 5. Run in multi-agent mode

Multi-agent mode dispatches parallel engineer agents and is the main CAID contribution.

```bash
bash scripts/run_multi.sh
```

Open `scripts/run_multi.sh` and set:

```bash
task="commit0"
model=""
max_subagents=2         # number of parallel engineers (start small)
sub_iterations=80
rounds_of_chat=2
```

---

## 6. Check the output

Results land in `outputs/{task}/{model}/{identifier}/{mode}/`. The key files:

| File | What it contains |
|------|-----------------|
| `cost.json` | Token usage and USD cost per agent and phase |
| `runtime.txt` | Wall-clock seconds |
| `outputs.jsonl` | Structured event log (one JSON object per line) |
| `report.json` | (Commit0) pytest results |
| `grade.json` | (PaperBench) judge score and rubric breakdown |

See [interpret output](../guides/interpret-output.md) for a full guide to reading these files.

---

## What happened

The multi-agent run executed this sequence:

1. Started a Docker container with the target codebase pre-installed.
2. The manager LLM explored the repository to understand the task.
3. The manager produced a delegation plan splitting work across engineers.
4. Git worktrees were created — one per engineer — branching from the same base commit.
5. Engineers ran in parallel, each implementing their assigned functions in their own worktree.
6. As each engineer finished a round, the manager merged their branch and assigned the next task.
7. The manager ran a final review after all engineers were done.
8. Pytest (or the PaperBench judge) evaluated the merged result.

## Next steps

- [Workflow deep-dive](../concepts/workflow.md) — understand every step in detail
- [Configuration reference](../reference/configuration.md) — all tunable parameters
- [Add a task](../guides/add-a-task.md) — bring your own benchmark
