# Interpret output files

What each output file contains and how to read it.

All output files land in a directory structured as:

```
outputs/{task}/{model}/{identifier}/{mode}/{params}/
```

For example: `outputs/commit0/gpt-5-mini/minitorch/multi/s4_r2_m50/`

See [output files reference](../reference/output-files.md) for a complete list of every file.

---

## cost.json

The most useful file for understanding what a run cost and where time was spent.

```json
{
  "total_cost": 2.34,
  "total_tokens": 182400,
  "wall_clock_duration": 1847.2,
  "model": "litellm_proxy/neulab/gpt-5-mini",
  "manager": {
    "cost": 0.87,
    "total_tokens": 68000,
    "duration": 312.4,
    "breakdown": {
      "analysis_cost": 0.21,
      "delegation_cost": 0.14,
      "assign_task_cost": 0.38,
      "final_review_cost": 0.14
    }
  },
  "subagents": [
    {
      "engineer_id": "engineer_1",
      "task_id": "autodiff-forward",
      "cost": 0.52,
      "total_tokens": 40200,
      "duration_seconds": 423.1,
      "round_num": 1,
      "success": true
    }
  ]
}
```

**What to look for:**

- `wall_clock_duration` — total elapsed time in seconds. Compare this to single-agent runs to measure speedup.
- `manager.breakdown` — which manager phase consumed the most tokens. High `assign_task_cost` means many rounds of task assignment; consider increasing `--sub_iterations` so engineers finish in fewer rounds.
- Per-engineer `success: false` entries — engineers that didn't commit. Their work may still have been merged via worktree recovery (check `merge_method`).

---

## outputs.jsonl

Structured event log. One JSON object per line, emitted chronologically.

Each event has:

```json
{
  "event_type": "manager_instruction",
  "source": "manager",
  "target": "engineer_1",
  "round_num": 1,
  "timestamp": "2026-06-13T14:23:11.234567",
  "start_time": "2026-06-13T14:23:11.234567",
  "end_time": "2026-06-13T14:25:43.891234",
  "content": { ... }
}
```

**Useful event types:**

| event_type | When emitted | Key content fields |
|------------|-------------|--------------------|
| `analysis_phase_complete` | After scan_and_analyze | `actual_iterations`, `cost`, `duration` |
| `delegation_complete` | After delegate_tasks | `num_agents`, `first_round`, `remaining` |
| `onboarding_complete` | After worktrees created | `num_subagents`, `base_commit` |
| `manager_instruction` | Each task assignment | `engineer_id`, `task_id`, `assignments` |
| `agent_response` | Each engineer round | `engineer_id`, `task_id`, `success`, `cost` |
| `manager_review` | Each collect_and_merge | `merged`, `merge_method`, `conflict_files` |
| `manager_final_review_all` | After final review | `duration`, `cost`, `engineers_reviewed` |
| `background_exploration` | Commit0 exploration | `cancelled`, `remaining_tasks_explored` |

To read the file in Python:

```python
import json

with open("outputs.jsonl") as f:
    events = [json.loads(line) for line in f]

# Find all merge events
merges = [e for e in events if e["event_type"] == "manager_review"]
for m in merges:
    print(m["content"]["engineer_id"], m["content"]["merged"], m["content"]["merge_method"])
```

---

## delegations.json

The manager's delegation plan. Shows how tasks were originally distributed.

```json
{
  "delegation_plan": {
    "first_round": {
      "num_agents": 2,
      "reasoning": "autodiff.py and scalar.py are independent and have the most stubs",
      "tasks": [
        {
          "engineer_id": "engineer_1",
          "task_id": "autodiff-forward",
          "file_path": "minitorch/autodiff.py",
          "functions_to_implement": ["forward", "backward", "topological_sort"],
          "instruction": "..."
        }
      ]
    },
    "remaining_tasks": [
      {
        "task_id": "tensor-ops",
        "file_path": "minitorch/tensor_ops.py",
        "functions_to_implement": ["map", "zip", "reduce"],
        "depends_on": ["minitorch/autodiff.py"]
      }
    ]
  }
}
```

Use this to understand the manager's reasoning about task ordering and dependency handling.

---

## report.json (Commit0 only)

The pytest JSON report. Standard pytest-json-report format:

```json
{
  "exitcode": 0,
  "summary": {
    "passed": 47,
    "failed": 3,
    "total": 50
  },
  "tests": [
    {
      "nodeid": "tests/test_autodiff.py::test_forward",
      "outcome": "passed",
      "duration": 0.023
    }
  ]
}
```

The `exitcode` field is also saved separately to `{repo}_pytest_exit_code.txt`. A value of `0` means all tests passed.

---

## grade.json (PaperBench only)

The LLM judge's evaluation of the paper reproduction.

```json
{
  "paper_id": "rice",
  "agent_model": "litellm_proxy/neulab/gpt-5-mini",
  "score": 0.732,
  "judge_output": {
    "judge_type": "simple",
    "score": 0.732,
    "num_leaf_nodes": 41,
    "num_invalid_leaf_nodes": 2,
    "judge_model": "gpt-5-mini",
    "max_depth": 3,
    "graded_task_tree": { ... }
  },
  "reproduction_metadata": {
    "repro_script_exists": true,
    "repro_success": true,
    "repro_duration": 847.3
  }
}
```

**`score`** is the fraction of leaf rubric nodes the judge assessed as passing (0.0–1.0).

**`graded_task_tree`** contains the full rubric tree with per-node pass/fail verdicts. Useful for debugging which sub-experiments failed.

---

## patch.diff (Commit0 multi-agent only)

A standard git diff showing all changes between the base commit and the final merged state. Useful for comparing multi-agent vs. single-agent implementations, or for resubmitting to the Commit0 evaluation server.

```bash
# Apply the patch to a fresh checkout for verification
git apply outputs/commit0/.../patch.diff
```

---

## agent_events/ directory

Per-agent conversation event logs. Each file is `{engineer_id}_events.jsonl`.

Events include raw LLM interaction records: tool calls, tool results, agent actions, and observations. Useful for debugging why an engineer failed or made unexpected decisions.

```python
import json

with open("agent_events/engineer_1_events.jsonl") as f:
    events = [json.loads(line) for line in f]

# Find tool calls
tool_calls = [e for e in events if e.get("type") == "tool_call"]
```

---

## run_{timestamp}.log

Full stdout from the run, including all `[Manager]` and `[engineer_N]` log lines. Useful when the terminal session is lost or when running in batch mode.
