# Subagents

Subagents (engineers) are the parallel worker LLMs that do the actual implementation work. This document covers how they are set up, how they run, how git isolation works, and how the async loop coordinates them.

## What a subagent is

Each engineer is a `SubAgentRunner` (`core/subagent.py`) wrapping an independent OpenHands `Agent` and `Conversation`. Unlike the manager, engineers do not use a condenser — their conversations are typically short (one task, a few dozen events) and they get a fresh context when their `Conversation` is first created.

The engineer's `Agent` uses the same default tool set as the manager (shell, file read/write) but with `system_prompt_kwargs={"cli_mode": True}`.

## Git worktrees

The core isolation mechanism is git worktrees. Each engineer gets:

- A **branch** named `feature/{engineer_id}` branching from the current `HEAD` (the base commit).
- A **worktree** at `/workspace/{worktree_name}` linked to that branch.

```bash
git branch feature/engineer_1 a1b2c3d4
git worktree add /workspace/minitorch_engineer_1_worktree feature/engineer_1
```

Engineers are instructed to work only in their worktree path. Changes in one worktree do not appear in others. The main repo directory remains the manager's domain.

When the manager merges an engineer's work, it runs from the main repo directory:

```bash
cd /workspace/minitorch_repo
git merge feature/engineer_1 --no-edit
```

This is a standard three-way merge. If multiple engineers touch the same file, conflicts are handled via the merge conflict flow (see below).

## The async execution loop

`run_subagents_parallel()` manages all engineers concurrently using Python's asyncio:

```
Initial state: all runners have asyncio tasks created

while tasks or idle_runners:
    done = asyncio.wait(tasks, FIRST_COMPLETED)

    for completed_task in done:
        result = task.result()
        collect_and_merge(result)           # merge their branch

        if merge conflict and rounds remain:
            spawn conflict resolution round

        elif no commit and rounds remain:
            spawn auto-reassign (same task, continue)

        elif rounds remain:
            assignment = manager.assign_task(result)
            for new_subagent in assignment:
                if target is idle: activate them
                if target is inactive: onboard them (create worktree)
                spawn new asyncio task
```

Engineers that hit `max_rounds_chat` are moved to `finished` and never assigned again. Engineers waiting for an assignment are moved to `idle`. The loop exits when there are no active tasks and no idle runners with capacity.

## Rounds of chat

Each engineer has a `max_rounds_chat` limit (default: 2). Each call to `SubAgentRunner.run()` is one round.

Round 1 always sends the `subagent_prompt` template (the initial assignment). Rounds 2+ send the `followup_prompt` template (the next task or continuation instruction).

The manager tracks which round each engineer is on and includes this in the event log (`round_num`).

## The runner lifecycle

```
SubAgentRunner.setup()
    → creates Agent + Conversation
    → records instruction_time

SubAgentRunner.run()  [called once per round]
    → build prompt (first round or followup)
    → conversation.send_message(prompt)
    → conversation.run()   [LLM acts until max_iterations or done]
    → get_commit_info()    [check if a new commit was made]
    → populate result (success/failure, cost, tokens, files)
    → save events to output_logger
    → completed_rounds += 1

SubAgentRunner.update_task(new_subagent)
    → replaces self.subagent with new task
    → does NOT reset the Conversation (preserves history)

SubAgentRunner.cleanup()
    → closes the Conversation
```

The conversation is not reset between rounds. Engineers remember their prior work, which helps them continue mid-implementation when auto-reassigned.

## Success detection

After each `conversation.run()`, the runner checks whether the engineer committed new work:

```python
commit_info = self.get_commit_info()
current_hash = commit_info["hash"]  # 8-char short hash from git log -1

if current_hash == base_commit[:8]:
    result.success = False  # no new commit
else:
    result.success = True   # new commit found
```

If no commit is found, the manager may auto-reassign the same task or the system may try to salvage uncommitted changes from the worktree.

## Merge conflict handling

When a merge conflict occurs:

1. The manager records the conflicted files in `result.conflict_files`.
2. If the engineer has rounds remaining, the `conflict_resolution` prompt is sent:

```
Your branch has merge conflicts with master. Please resolve them:
1. Run: git merge master
2. The following files have conflicts: {conflict_file_list}
3. Open each conflicted file, find <<<<<<< HEAD markers, and resolve
4. After resolving: git add . && git commit -m "Resolve merge conflicts"
```

3. The engineer runs in its own worktree, resolves the conflicts, and commits.
4. The manager retries the merge.

If the engineer has no rounds remaining, the manager force-merges with `git merge -X theirs`, preferring the engineer's changes in all conflicts.

## Background exploration (Commit0 only)

While engineers are running their first round, the manager runs `explore_background()` concurrently via `asyncio`. This pre-reads the next batch of files to prepare richer instructions for the upcoming `assign_task` calls.

The exploration is launched as a separate `asyncio.Task`:

```python
exploration_task = asyncio.create_task(
    manager.explore_background_async(remaining_tasks, running_summary)
)
```

The instant any engineer completes, `manager.cancel_exploration()` is called, which sets `exploration_cancelled = True` and calls `conversation.pause()`. This prevents the manager from burning tokens on exploration when a real decision is needed.

## Onboarding inactive engineers

Engineers are either "active" (have a runner and worktree) or "inactive" (planned by the delegation but not yet started). The manager's `delegation_plan.num_agents` may be less than `max_subagents` if the task doesn't need all slots.

When the manager's `assign_task` response assigns a task to an inactive engineer, the async loop:

1. Gets the current HEAD as the new base commit.
2. Creates the branch and worktree for that engineer.
3. Creates a new `SubAgentRunner` and sets up its `Agent`+`Conversation`.
4. Launches it as a new asyncio task.

This lazy onboarding means small tasks can run with fewer agents, and additional engineers can be recruited mid-run if the manager decides more parallelism is needed.

## Per-engineer event logging

Each engineer's conversation events are saved to `agent_events/{engineer_id}_events.jsonl`. Events include `task_id`, `round_num`, and task-specific extras (e.g., `file_path` for Commit0).

Only new events since the last save are written per round, tracked by `self.last_saved_event_count`.
