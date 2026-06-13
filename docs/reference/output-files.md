# Output files reference

Every file produced by a CAID run, including format, location, and when it is written.

All files are written to the output directory (`--output_dir` or the auto-generated path).

---

## Always produced

### `outputs.jsonl`

**Format:** JSONL (one JSON object per line)
**When written:** Continuously throughout the run

Structured event log. Every significant event in the workflow is recorded here: scan start/end, delegation, manager instructions, engineer responses, merge results, final review, evaluation. See [interpret output](../guides/interpret-output.md#outputsjsonl) for field details.

---

### `cost.json`

**Format:** JSON
**When written:** End of run (after evaluation)

Full cost breakdown by agent and phase. Includes `total_cost`, `wall_clock_duration`, per-engineer breakdowns, and manager phase breakdowns (analysis, delegation, assign_task, final_review). See [interpret output](../guides/interpret-output.md#costjson).

---

### `runtime.txt`

**Format:** Plain text (single float)
**When written:** After all agents finish, before evaluation

Wall-clock runtime in seconds from workflow start to the end of the agent execution phase. Does not include evaluation time.

Example:
```
1847.2
```

---

### `delegations.json`

**Format:** JSON
**When written:** After `delegate_tasks()` completes (multi-agent only)

The manager's delegation plan: `first_round` task assignments and `remaining_tasks` queue. See [interpret output](../guides/interpret-output.md#delegationsjson).

---

### `agent_events/manager_events.jsonl`

**Format:** JSONL
**When written:** After each manager phase (scan, delegate, assign, final review)

Raw OpenHands conversation events from the manager agent, including all tool calls and LLM responses. Each event has `engineer_id = "manager"` and a `phase` label.

---

### `agent_events/{engineer_id}_events.jsonl`

**Format:** JSONL
**When written:** After each engineer round (multi-agent only)

Raw OpenHands conversation events from each engineer agent. One file per engineer. Events include `task_id` and `round_num` for filtering.

---

### `run_{timestamp}.log`

**Format:** Plain text
**When written:** Throughout the run (tee'd from stdout)

Full terminal output from the run. Includes all `[Manager]` and `[engineer_N]` log lines, Docker setup output, and evaluation results.

---

## Commit0 only

### `report.json`

**Format:** JSON (pytest-json-report format)
**When written:** After pytest evaluation

Pytest results including `exitcode`, per-test outcomes, and summary counts (passed/failed/total). See [interpret output](../guides/interpret-output.md#reportjson-commit0-only).

---

### `{repo_name}_pytest_exit_code.txt`

**Format:** Plain text
**When written:** After pytest evaluation

The pytest process exit code as a string. `"0"` = all tests passed. `"1"` = some tests failed. `"2"` = pytest error.

Example filename: `minitorch_pytest_exit_code.txt`

---

### `{repo_name}_test_output.txt`

**Format:** Plain text
**When written:** After pytest evaluation

Full raw output from the pytest run, including individual test output and failure tracebacks.

Example filename: `minitorch_test_output.txt`

---

### `patch.diff`

**Format:** Unified diff
**When written:** After all agents finish (multi-agent only)

Git diff between the base commit and the final merged state. Shows all changes made by all engineers combined. Can be applied to a fresh clone for verification or resubmission.

```bash
# Verify the patch applies cleanly
git apply --check outputs/commit0/.../patch.diff
```

---

### `final_repo/{repo_name}.tar.gz`

**Format:** Gzipped tar archive
**When written:** After evaluation

Full snapshot of the repository at its final state. Used for running `retest.py` (the Commit0 offline evaluation script) without re-running the agent.

---

## PaperBench only

### `grade.json`

**Format:** JSON
**When written:** After LLM judge evaluation

Judge scores for the paper reproduction. Includes overall `score` (0.0–1.0), per-rubric-node verdicts in `graded_task_tree`, and reproduction metadata. See [interpret output](../guides/interpret-output.md#gradejson-paperbench-only).

---

## Output directory naming

When `--output_dir` is not specified, the path is constructed as:

```
outputs/{task}/{model_slug}/{identifier}/{mode}/{params}/
```

Where:
- `model_slug` — the model identifier with `/` replaced by `_`
- `identifier` — repo name (Commit0) or paper ID (PaperBench)
- `mode` — `"multi"` or `"single"`
- `params` — a short string encoding key hyperparameters, e.g., `s4_r2_m50` (max_subagents=4, rounds=2, max_iter=50)
