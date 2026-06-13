# System design

Architecture overview of CAID for developers working on or extending the framework.

---

## High-level architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Host Machine                              │
│                                                              │
│  run_infer.py                                                │
│  ┌──────────┐     ┌────────────────────────────────────┐   │
│  │WorkflowCo│     │        Docker Container              │   │
│  │nfig      │────▶│                                      │   │
│  │TaskModule│     │  /workspace/                         │   │
│  └──────────┘     │  ├── {repo}_repo/       (main)       │   │
│                   │  ├── {repo}_engineer_1_worktree/     │   │
│                   │  ├── {repo}_engineer_2_worktree/     │   │
│                   │  └── ...                             │   │
│                   │                                      │   │
│  Manager          │  OpenHands Agent (manager)           │   │
│  ┌──────────┐    │  ┌──────────────────────────────┐    │   │
│  │ Agent    │◀───┤  │  LLM ↔ Tools (shell, files)  │    │   │
│  │ Convers- │    │  └──────────────────────────────┘    │   │
│  │ ation    │    │                                      │   │
│  └──────────┘    │  OpenHands Agents (engineers)        │   │
│                   │  ┌──────────┐  ┌──────────┐         │   │
│  asyncio loop     │  │engineer_1│  │engineer_2│  ...    │   │
│  ┌──────────┐    │  │Agent+Conv│  │Agent+Conv│         │   │
│  │Subagent  │◀───┤  └──────────┘  └──────────┘         │   │
│  │Runner x N│    └────────────────────────────────────┘   │
│  └──────────┘                                              │
│                                                              │
│  OutputLogger → outputs/{task}/{model}/{id}/{mode}/{params}/ │
└─────────────────────────────────────────────────────────────┘
```

## Component breakdown

### `run_infer.py`

The entry point. Responsibilities:
- Parse CLI arguments into `WorkflowConfig` and task-specific configs.
- Instantiate the correct `TaskModule`.
- Construct the output directory.
- Start the Docker workspace.
- Drive the 10-step workflow as a top-level async function.
- Save final costs and evaluation results.

### `core/manager.py` — `Manager`

A stateful controller that owns one `Agent`+`Conversation`. The manager's conversation accumulates the entire context of the workflow: scan results, delegation reasoning, and per-engineer assignment decisions.

Key state:
- `analysis_result: AnalysisResult` — populated after scan
- `delegation_plan: DelegationPlan` — populated after delegation
- Per-phase cost and token counters

See [manager agent](../concepts/manager.md) for the full lifecycle.

### `core/subagent.py` — `SubAgentRunner`, `run_subagents_parallel`

`SubAgentRunner` wraps one engineer's `Agent`+`Conversation`. It persists across rounds but gets a new task via `update_task()`.

`run_subagents_parallel` is the async coordination loop. It uses `asyncio.wait(FIRST_COMPLETED)` to process engineers as they finish. When multiple engineers finish simultaneously, they are sorted by `end_time` and processed serially (because the manager conversation is single-threaded).

### `config.py`

Pure dataclasses with no logic. Defines the data shapes passed between components:
- `WorkflowConfig` — top-level run configuration
- `SubAgent` — one engineer's current task assignment
- `SubAgentResult` — outcome of one engineer round
- `DelegationPlan` — manager's full task queue
- `AnalysisResult` — manager's scan output (task-specific)
- `TaskNode` — PaperBench rubric node

### `tasks/base.py` — `TaskModule`

Abstract interface with 22 abstract methods. Separates all benchmark-specific logic from the generic workflow machinery. See [add a task](../guides/add-a-task.md).

### `core/utils.py`

Utility functions used across components:
- `build_task_module()` — task factory
- `load_prompts()` — loads `prompts/{task}.yaml`
- `extract_json_from_events()` — scans conversation events for JSON blocks
- `build_delegation_plan()` — parses delegation JSON into `DelegationPlan`
- `save_all_costs()` — writes `cost.json`
- `generate_patch()` — produces `patch.diff`
- `OutputLogger` — writes `outputs.jsonl` and `agent_events/`
- `PanelVisualizer` — rich terminal progress display

---

## Data flows

### Manager scan flow

```
conversation.send_message(scan_analysis_prompt)
  → LLM explores container (shell/file tools)
  → LLM stops
task_module.build_analysis_from_state()
  → reads LLM output from conversation events
  → returns AnalysisResult
```

### Delegation flow

```
conversation.send_message(task_delegation_prompt)
  → LLM outputs delegation_plan JSON
extract_json_from_events(events, "delegation_plan")
  → finds the JSON block in any LLM message event
build_delegation_plan(delegation_json)
  → returns DelegationPlan with first_round_tasks + remaining_tasks
```

### Engineer round flow

```
runner.run()
  → conversation.send_message(subagent_prompt or followup_prompt)
  → conversation.run()  [engineer acts in worktree via tools]
  → get_commit_info()   [git log -1 in worktree]
  → populate SubAgentResult
  → log events to agent_events/{id}_events.jsonl
  → return SubAgentResult
```

### Collect-and-merge flow

```
manager.collect_and_merge(result)
  → if result.commit_hash:
      git merge {branch_name}  from main repo dir
      → success: return merged=True, merge_method="branch_merge"
      → conflict: abort, return merged=False, conflict_files=[...]
  → elif worktree has uncommitted changes:
      git add . && git commit  inside worktree
      git merge {branch_name}  from main repo dir
      → return merged=True, merge_method="worktree_commit_merge"
  → else:
      return merged=False, merge_method="none"
```

---

## Key design decisions

### Decision 1: Git worktrees instead of containers-per-agent

**Rationale:** Creating a new Docker container per engineer would require 1–5 minutes of image pull and container startup per engineer, and would require copying the full codebase into each container. Git worktrees take <1 second and share the same filesystem (only the working tree is isolated). For tasks with large codebases (Commit0 repos can be several hundred MB of installed packages), this is critical.

**Trade-off:** All engineers share the same Docker container, so one engineer's runaway process (infinite loop, memory exhaustion) can affect others. The final review phase is designed to catch and fix such issues before evaluation.

### Decision 2: Single-threaded manager with async engineers

**Rationale:** The manager needs a coherent conversation history to make good decisions. Interleaving concurrent manager LLM calls would cause the context to be inconsistent. Engineers, by contrast, work independently and don't need to coordinate — they only interact with the manager through discrete handoffs (round completion, merge, new task).

**Trade-off:** When N engineers finish simultaneously, they are processed serially, adding latency. In practice this is rarely a bottleneck since engineers rarely complete at exactly the same time.

### Decision 3: LLM condensation for the manager

**Rationale:** Commit0 runs can involve 10+ rounds of task assignment. Without condensation, the manager's context would grow indefinitely and hit the model's context limit. The condenser summarizes old events while preserving the initial system instruction (keep_first=4).

**Trade-off:** Summarized events lose detail. The manager may forget specific details of earlier engineer interactions. This is acceptable because the structured event log (`outputs.jsonl`) captures all facts; the manager only needs enough context to make assignment decisions.

### Decision 4: TaskModule interface with 22 methods

**Rationale:** Commit0 and PaperBench are very different tasks (different Docker images, different evaluation methods, different worktree structures, different prompt formats). Rather than cluttering the core with `if is_commit0:` branches everywhere, all task-specific logic is delegated to `TaskModule`. New tasks can be added without touching `Manager` or `SubAgentRunner`.

**Trade-off:** The interface has many methods, making it harder to implement a new task from scratch. The reference implementations (`Commit0Task`, `PaperbenchTask`) mitigate this.

### Decision 5: Prompt templates in YAML files

**Rationale:** Prompts need to be tuned frequently during research. Keeping them in YAML (rather than Python strings or code) makes them easy to edit without touching Python code, and makes version-controlled prompt history readable in diffs.

**Trade-off:** Prompt variables must be formatted with Python's `.format()` at call sites. Template syntax errors (mismatched braces, missing keys) surface at runtime rather than at import time.

---

## Scaling characteristics

**What scales well:**
- Adding more engineers (`--max_subagents`): minimal marginal cost since they share the container.
- Longer iteration limits: no coordination overhead; engineers just run longer.
- New tasks: `TaskModule` encapsulates all task-specific code.

**What doesn't scale well:**
- Very large numbers of engineers (>8): the single-threaded manager becomes a bottleneck; also, git merge conflicts become more frequent with more concurrent branches.
- Very long manager conversations: the condenser helps, but summarization quality degrades with very long histories.
- Tasks requiring cross-engineer synchronization: the current model assumes engineers work on independent subtasks.

## External dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| `openhands-sdk` | 1.11.0 | Agent + Conversation framework |
| `openhands-workspace` | 1.11.0 | Docker workspace management |
| `openhands-tools` | 1.11.0 | Default tool set (shell, file) |
| `litellm` | 1.81.11 | LLM provider abstraction |
| `fire` | 0.7.1 | CLI argument parsing |
| `pyyaml` | 6.0.3 | Prompt template loading |
| `datasets` | 3.0.1 | Commit0 dataset loading |
| `paperbench` | from source | PaperBench judge |
