# Manager agent

The manager is the central LLM agent that coordinates the entire workflow. It is a single `Agent`+`Conversation` pair that runs sequentially — all phases share the same conversation context.

## What it is

`Manager` (`core/manager.py`) wraps an OpenHands `Agent` and drives it through four distinct phases: scan, delegate, assign (repeated), and final review. Each phase sends a different prompt and then calls `conversation.run()` to let the LLM act.

## Why it exists as its own subsystem

The manager needs to maintain a coherent picture of the entire task across time: what has been analyzed, which tasks are delegated, what each engineer completed, and what remains. A single persistent conversation provides that continuity.

The alternative — creating a new agent for each decision — would lose context between decisions and require prompt-stuffing the entire state every time. The LLMSummarizingCondenser handles the context growth problem instead.

## How it works

### Initialization

```python
Manager(
    llm=llm,
    workspace=workspace,
    task=task_module,
    config=workflow_config,
    output_logger=output_logger,
    prompts=prompts,
)
```

Prompts are loaded from `prompts/{task}.yaml` at startup. Each method uses a specific key from that YAML.

### Phase 1: scan_and_analyze()

Sends `prompts["scan_analysis"]` to the manager. The manager uses shell tools inside the Docker container to explore the task. After the run completes, `task_module.build_analysis_from_state()` extracts structured data from the conversation events.

Cost and token usage are recorded as deltas:
```python
metrics_before = extract_conversation_metrics(self.conversation)
# ... run ...
self.analysis_cost = metrics_after["cost"] - metrics_before["cost"]
```

### Phase 2: delegate_tasks()

Sends `prompts["task_delegation"]` formatted with `max_agents`. The manager outputs a `delegation_plan` JSON block. `extract_json_from_events()` scans all events for this block.

If no valid JSON is found, `fallback_delegation()` distributes tasks evenly across engineers without LLM involvement.

The parsed plan becomes `self.delegation_plan: DelegationPlan`.

### Phase 3: assign_task() (called repeatedly)

Called after every engineer round completes. Sends `prompts["assign_task"]` with:

- Which engineer just completed and their task status (`success` / `failed` / `recovered`)
- A summary of all running, idle, inactive, and finished engineers
- The list of completed tasks so far

The LLM responds with an `assign_task` JSON block specifying which engineer(s) to assign next and what task. The manager validates all assignments:

- Rejects assignments to running engineers
- Rejects assignments to finished engineers
- Rejects duplicate agent or task assignments in the same response

This method is called sequentially (not concurrently) because the manager's conversation is single-threaded. The async loop in `run_subagents_parallel` handles concurrency at the engineer level, not the manager level.

### Phase 4: final_review_all()

Sends `prompts["manager_final_review_all"]` with a summary of all engineers' outcomes and a list of unmerged worktrees. The manager reviews the merged codebase, fixes integration issues, and commits any repairs.

This phase runs with a reduced `max_iterations` cap (default 30) to avoid cost overrun.

## Context management

The manager uses `LLMSummarizingCondenser`:

```python
condenser = LLMSummarizingCondenser(
    llm=condenser_llm,
    max_size=200,
    keep_first=4,
)
```

When the conversation exceeds 200 events, the condenser summarizes events 5 through N, keeping the first 4 verbatim. The first 4 events contain the system instruction and initial task context, which must not be summarized away.

This prevents context overflow on long Commit0 runs with many assign_task cycles.

## Cost and time tracking

The manager tracks cost and time separately for each phase:

| Field | Phase |
|-------|-------|
| `analysis_cost / analysis_tokens` | scan_and_analyze |
| `delegation_cost / delegation_tokens` | delegate_tasks |
| `assign_task_total_cost / assign_task_total_tokens` | all assign_task calls combined |
| `review_total_cost / review_total_tokens` | collect_and_merge reviews |
| `exploration_cost / exploration_tokens` | background exploration (Commit0) |
| `final_review_cost / final_review_tokens` | final_review_all |

All phase costs are accumulated into `cost.json` at the end of the run.

## Branch merging

`collect_and_merge()` is called after every engineer round. It tries three strategies in order:

1. **Branch merge**: `git merge {branch_name} --no-edit`
2. **Force theirs** (last round only): `git merge {branch_name} -X theirs`
3. **Worktree commit+merge**: if the engineer has uncommitted changes, the manager commits them on their behalf, then merges

If a conflict is detected and the engineer has rounds remaining, the conflict files are sent back to the engineer for resolution. If the engineer has no rounds left, `force_theirs` is used as a last resort.

## Event logging

After each phase, `save_events()` serializes the new conversation events to `agent_events/manager_events.jsonl`. Each event gets `engineer_id = "manager"` and a `phase` label for analysis.

## Interaction with other subsystems

| System | Interaction |
|--------|-------------|
| `SubAgentRunner` | Manager creates `SubAgent` configs; runners execute them |
| `run_subagents_parallel` | Calls `manager.assign_task()` and `manager.collect_and_merge()` from the async loop |
| `OutputLogger` | Manager logs events, instructions, and reviews |
| `TaskModule` | Manager delegates all task-specific logic (prompt args, subagent building, evaluation) |

## Common gotchas

**Manager runs out of iterations before delegation completes.** Increase `--max_iterations`. Delegation can require many tool calls if the repository is large.

**Delegation JSON not found.** The fallback triggers — this is non-fatal. Check `delegations.json` to see what the fallback produced. If the fallback is consistently wrong, check whether the `task_delegation` prompt is being sent (look for it in `manager_events.jsonl`).

**assign_task returns empty assignments.** The manager decided no task should be assigned right now. Usually means it is waiting for dependencies. The idle engineers will be polled again when another engineer completes.
