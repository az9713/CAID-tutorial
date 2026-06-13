# Key concepts

Definitions for every term used across the CAID documentation.

---

**AnalysisResult** — The structured output of the manager's scan phase. For Commit0, it contains `functions_by_file`, `implementation_order`, and `blocking_dependencies`. For PaperBench, it contains the `task_tree` and `leaf_tasks`. Populated by `Manager.scan_and_analyze()`.

**assign_task** — One of the manager's recurring operations. After an engineer completes a round, `Manager.assign_task()` sends a prompt to the manager LLM that decides which subtask to give the engineer next. The LLM responds with an `assign_task` JSON block.

**background exploration** — A Commit0-specific optimization: while engineers are running their first round, the manager concurrently explores the remaining unassigned files to prepare richer instructions for the next assignment. Cancelled the moment any engineer completes. See [subagents](../concepts/subagents.md).

**base commit** — The git commit that all engineer branches diverge from. Recorded at worktree creation time so the manager can compute diffs (`git diff {base_commit}..HEAD`) and know exactly what each engineer changed.

**Commit0** — A benchmark where all Python function bodies are replaced with `pass`. The task is to implement them so the test suite passes. See the [Commit0 task](https://commit-0.github.io/).

**conflict resolution round** — When a `git merge` produces a conflict, the engineer is sent a special prompt instructing them to resolve the conflicts in their worktree and recommit. This consumes one of the engineer's available rounds.

**delegation plan** — A JSON document the manager produces during `delegate_tasks()`. Contains `first_round` tasks (immediately assigned) and `remaining_tasks` (queued for later). Saved to `delegations.json`.

**DelegationPlan** — The Python dataclass (`config.py`) holding the parsed delegation plan: `num_agents`, `reasoning`, `first_round_tasks`, and `remaining_tasks`.

**Docker workspace** — The container managed by `openhands-workspace`. All agents execute commands inside this container. CAID uses either `DockerWorkspace` (from a pre-built server image) or `DockerDevWorkspace` (built from source with a base image).

**engineer** — An instance of `SubAgentRunner`. Each engineer runs an OpenHands `Agent` with a `Conversation` object. Engineers are identified by `engineer_id` strings like `"engineer_1"`, `"engineer_2"`, etc.

**finished** — Status of an engineer that has exhausted `max_rounds_chat`. Finished engineers are never assigned new tasks.

**git worktree** — A linked working tree created by `git worktree add`. Each engineer gets one worktree at `/workspace/{engineer_id}_worktree` on branch `feature/{engineer_id}`. Changes in one worktree do not affect others. See [subagents](../concepts/subagents.md#git-worktrees).

**idle** — Status of an engineer that has completed a round and is waiting for an assignment, but has not yet reached `max_rounds_chat`. Idle engineers are polled in the async loop and assigned tasks when the manager decides they should continue.

**LiteLLM** — The model abstraction layer used by CAID. Model identifiers follow the LiteLLM format, e.g., `litellm_proxy/neulab/gpt-5-mini`. All LLM calls go through `openhands.sdk.LLM`, which internally uses LiteLLM.

**LLMSummarizingCondenser** — An OpenHands SDK component attached to the manager agent. When the conversation grows beyond `max_size` events, the condenser summarizes old turns to keep context within token limits. Configured with `max_size=200, keep_first=4`.

**Manager** — The central coordinator LLM agent (`core/manager.py`). Runs sequentially (one active conversation), but orchestrates engineers asynchronously. Responsible for scanning, delegation, task assignment, branch merging, and final review.

**max_rounds_chat** — Maximum number of task assignments an engineer can receive across its lifetime. Each round consists of one `SubAgentRunner.run()` call. Default: 2.

**merge methods** — How the manager integrates an engineer's work. Three strategies in descending preference:
1. `branch_merge` — clean `git merge {branch_name}`.
2. `worktree_commit_merge` — uncommitted changes are committed by the manager, then merged.
3. `none` — no changes found; engineer's work is discarded.

**OpenHands SDK** — The agent execution framework (`openhands-sdk`) that provides `Agent`, `Conversation`, `LLM`, and workspace utilities. CAID builds on top of this SDK.

**output directory** — The directory where all results are written. Auto-generated as `outputs/{task}/{model}/{identifier}/{mode}/{params}/` unless `--output_dir` is specified. See [output files reference](../reference/output-files.md).

**PaperBench** — A benchmark where the task is to reproduce ML research paper experiments. The manager reads the paper PDF/markdown and rubric, delegates sub-experiments to engineers, and an LLM judge scores the submission. See [PaperBench](https://arxiv.org/pdf/2504.01848).

**round** — One execution of `SubAgentRunner.run()`. An engineer completes a round when its `Conversation.run()` call returns (either by hitting `max_iterations` or finishing the task). Each round is tracked by `round_num` in events and results.

**SubAgent** — The config dataclass (`config.py`) representing an assigned task for a specific engineer: `engineer_id`, `task_id`, `file_path`, `functions_to_implement`, `worktree_path`, `branch_name`, `base_commit`, `instruction`, etc.

**SubAgentResult** — The result dataclass produced after a round completes. Tracks `success`, `commit_hash`, `files_modified`, `cost`, `prompt_tokens`, `completion_tokens`, `duration_seconds`, `merged`, `merge_method`.

**SubAgentRunner** — The wrapper class (`core/subagent.py`) that manages one engineer's OpenHands `Agent` and `Conversation` across multiple rounds. Each runner persists the conversation so engineers maintain context from prior rounds.

**task module** — A concrete implementation of the `TaskModule` abstract interface. Encapsulates all task-specific logic: Docker image, workspace setup, prompt formatting, subagent building, and evaluation. Currently: `Commit0Task` and `PaperbenchTask`.

**TaskModule** — The abstract base class (`tasks/base.py`) that defines the interface every task must implement. See [add a task](../guides/add-a-task.md).

**TaskNode** — A node in PaperBench's rubric tree. Leaf nodes are individual scoreable tasks. The manager decomposes the tree to create per-engineer assignments.

**worktree_name** — The directory name of an engineer's worktree inside the container. For Commit0: `{repo_name}_{engineer_id}_worktree`. For PaperBench: `submission_{engineer_id}`.

**WorkflowConfig** — The top-level configuration dataclass (`config.py`). Holds `model`, `subagent_model`, `manager_max_iterations`, `max_subagents`, `subagent_max_iterations`, `max_rounds_chat`, and `output_dir`. See [configuration reference](../reference/configuration.md).
