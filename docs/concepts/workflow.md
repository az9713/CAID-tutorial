# The CAID workflow

The full 10-step pipeline that runs when you invoke `run_infer.py` in multi-agent mode. Single-agent mode follows a simpler path (steps 3–5 replaced by a single agent run).

---

## Step 1: Configure

`run_infer.py main()` parses CLI arguments and builds a `WorkflowConfig`. The `build_task_module()` utility instantiates the correct `TaskModule` subclass (`Commit0Task` or `PaperbenchTask`).

The output directory is constructed as:

```
outputs/{task}/{model}/{identifier}/{mode}/{params}/
```

For example: `outputs/commit0/gpt-5-mini/minitorch/multi/s4_r2_m50/`

---

## Step 2: Start Docker workspace

A Docker container is started using the OpenHands workspace SDK. CAID supports two workspace modes:

- **`DockerWorkspace`**: pulls a pre-built server image specified by `get_workspace_config()`.
- **`DockerDevWorkspace`**: builds from source using a `base_image` + `target` (used for tasks needing custom dependencies).

All subsequent steps run commands inside this container via `workspace.execute_command()`.

---

## Step 3: Initialize manager agent

`Manager.setup(mode="multi_agent")` creates an OpenHands `Agent` with:

- The default tool set (shell, file read/write, no browser).
- A `user_instruction` system suffix from the task's prompt YAML.
- An `LLMSummarizingCondenser` (max 200 events, keep first 4) to prevent context overflow on long runs.

The agent's `Conversation` is created with `max_iteration_per_run = manager_max_iterations`.

---

## Step 4: Scan and analyze

`Manager.scan_and_analyze()` sends the `scan_analysis` prompt to the manager LLM. The manager explores the codebase or paper inside the Docker container and produces an `AnalysisResult`.

**For Commit0:** The manager discovers all `pass`-stub functions, their file locations, import dependencies, and implementation order. It may also add missing function stubs that aren't yet defined anywhere.

**For PaperBench:** The manager reads the paper PDF/markdown, the rubric JSON, and the `instructions.txt`, building a `TaskNode` tree of scoreable sub-experiments.

The raw LLM conversation events are saved to `agent_events/manager_events.jsonl`.

---

## Step 5: Delegate tasks

`Manager.delegate_tasks()` sends the `task_delegation` prompt. The manager LLM outputs a `delegation_plan` JSON block:

```json
{
  "delegation_plan": {
    "first_round": {
      "num_agents": 2,
      "tasks": [
        {
          "engineer_id": "engineer_1",
          "task_id": "autodiff-forward",
          "file_path": "minitorch/autodiff.py",
          "functions_to_implement": ["forward", "backward"],
          "instruction": "..."
        }
      ]
    },
    "remaining_tasks": [...]
  }
}
```

`first_round` tasks start immediately. `remaining_tasks` are queued and assigned as engineers complete rounds.

The parsed plan is saved to `delegations.json`. If the LLM fails to produce valid JSON, a fallback distributes tasks evenly.

---

## Step 6: Onboard subagents

`Manager.onboard_subagents()` sets up one git worktree per engineer:

```bash
# Record the base commit
git rev-parse HEAD   # → e.g., a1b2c3d4

# Create branch from that commit
git branch feature/engineer_1 a1b2c3d4

# Create worktree
git worktree add /workspace/minitorch_engineer_1_worktree feature/engineer_1
```

Each engineer's `SubAgent` dataclass is populated with `worktree_path`, `branch_name`, and `base_commit`. Engineers with a failed worktree creation get `status = "failed"` and are skipped.

---

## Step 7: Setup subagent runners

For each ready subagent, a `SubAgentRunner` is created. Each runner holds:

- Its own OpenHands `Agent` and `Conversation` (independent LLM context).
- The task-specific `task_module` for prompt building and result extraction.
- `max_iterations` and `max_rounds_chat` limits.

Runners persist their `Conversation` across rounds so engineers maintain context from prior work.

---

## Step 8: Run engineers in parallel

`run_subagents_parallel()` drives the async event loop:

```
asyncio tasks: [engineer_1, engineer_2, ...]
     │
     ▼
asyncio.wait(FIRST_COMPLETED)
     │
     ├─ Engineer finishes round
     │       │
     │       ├─ manager.collect_and_merge()  ← merges their git branch
     │       │
     │       ├─ [if conflict] → conflict resolution round
     │       ├─ [if no commit] → auto-reassign same task
     │       └─ [if merged] → manager.assign_task() → new asyncio task
     │
     └─ Background exploration (Commit0 only)
             └─ Manager concurrently reads upcoming files
                Cancelled the instant any engineer completes
```

The loop continues until all engineers reach `max_rounds_chat` or no tasks remain.

---

## Step 8.5 / Step 9: Manager final review

`Manager.final_review_all()` sends the `manager_final_review_all` prompt. The manager:

1. Checks for unmerged worktrees and copies useful code.
2. Scans for infinite loops, `input()` calls, or syntax errors.
3. Verifies cross-file naming consistency (e.g., one engineer named a class differently than another expected).
4. Commits any manual fixes.

This step runs with a lower `max_iterations` cap (default 30) to keep runtime cost bounded.

---

## Step 9 / Step 10: Evaluate

**Commit0:** Runs `pytest` inside the container. Results are saved as `report.json`, `{repo}_pytest_exit_code.txt`, and `{repo}_test_output.txt`. The final repo state is archived as `final_repo/{repo}.tar.gz`.

**PaperBench:** Runs `reproduce.sh` inside the container, then an LLM judge scores the output against the rubric. Results are saved as `grade.json`.

---

## Step 10: Save costs and patch

`save_all_costs()` writes `cost.json` with a full breakdown by agent and phase. For Commit0, `generate_patch()` produces `patch.diff` — the git diff between the base commit and the final merged state.

All stdout is also tee'd to `run_{timestamp}.log`.

---

## Single-agent mode

Single-agent mode skips steps 5–8. Instead:

- Step 3: Manager is set up in `single_agent` mode (no condenser, no system suffix).
- Step 4: The entire task is sent as one user message. The agent works until `max_iterations`.
- Step 5: Evaluate directly.

This is the baseline for comparing against the multi-agent CAID approach.
