# Add a new task

Add a new benchmark or software engineering task to the CAID workflow by implementing the `TaskModule` interface.

This guide assumes familiarity with the [workflow](../concepts/workflow.md) and [key concepts](../overview/key-concepts.md).

## What this guide accomplishes

After following these steps, your task will be runnable via:

```bash
uv run python run_infer.py --task my_task --model litellm_proxy/...
```

## Prerequisites

- A Docker image or base image that contains your task's dependencies
- A dataset or data directory that the workspace will load
- Familiarity with the `Commit0Task` (`tasks/commit0.py`) or `PaperbenchTask` (`tasks/paperbench.py`) as reference implementations

---

## 1. Create the task file

Create `tasks/my_task.py`. Start with a config dataclass and a class that extends `TaskModule`:

```python
from dataclasses import dataclass
from typing import Optional
from tasks.base import TaskModule
from config import SubAgent, SubAgentResult


@dataclass
class MyTaskConfig:
    # Docker image for the workspace container
    docker_image: str = "my-task-image:latest"
    # Path to task data
    data_dir: str = "data/my_task"
    # Output directory (set by run_infer.py)
    output_dir: str = "outputs"


class MyTask(TaskModule):
    def __init__(self, config: Optional[MyTaskConfig] = None, **kwargs):
        self.config = config or MyTaskConfig(**kwargs)
        self.task_data = None
```

---

## 2. Implement the six required methods

### get_docker_image()

Return the Docker image name used to start the workspace container.

```python
def get_docker_image(self):
    return self.config.docker_image
```

### get_work_dir()

Return the working directory path inside the container where the main repo or submission directory lives. Git commands during the worktree phase run from this directory.

```python
def get_work_dir(self):
    return "/workspace/my_repo"
```

### get_workspace_config()

Return a dict that `run_infer.py` uses to construct the workspace. Use `"server_image"` for a pre-built image, or `"base_image"` + `"target"` for a dev build.

```python
def get_workspace_config(self):
    return {"server_image": self.config.docker_image}
```

### load_task_data()

Load task-specific data from disk. Store it on `self` for use in later methods.

```python
def load_task_data(self):
    import json
    with open(f"{self.config.data_dir}/tasks.json") as f:
        self.task_data = json.load(f)
    return self.task_data
```

### setup_workspace(workspace)

Called after the Docker container is running. Clone repos, install dependencies, upload files, etc. Use `workspace.execute_command()` for shell commands.

```python
def setup_workspace(self, workspace):
    workspace.execute_command(
        "cd /workspace && git clone /host/my_repo my_repo",
        timeout=120
    )
    workspace.execute_command(
        "cd /workspace/my_repo && pip install -e .",
        timeout=300
    )
```

### evaluate(workspace)

Run evaluation after all agents finish. Return a dict with results.

```python
def evaluate(self, workspace):
    result = workspace.execute_command(
        "cd /workspace/my_repo && python evaluate.py",
        timeout=600
    )
    return {
        "score": float(result.stdout.strip()),
        "exit_code": str(result.exit_code),
        "output": result.stdout,
    }
```

---

## 3. Implement manager integration methods

These methods customize how the manager interacts with your task. See `tasks/base.py` for default implementations — override only what you need.

### get_prompt_format_args(config)

Return variables for the manager's system prompt template.

```python
def get_prompt_format_args(self, config):
    return {
        "max_agents": config.max_subagents,
        "data_dir": self.config.data_dir,
    }
```

### build_subagent(engineer_id, primary_task, all_tasks)

Create a `SubAgent` from the manager's delegation. The `primary_task` is a `SubAgentTask` from the delegation plan.

```python
def build_subagent(self, engineer_id, primary_task, all_tasks):
    from config import SubAgent
    return SubAgent(
        engineer_id=engineer_id,
        task_id=primary_task.task_id,
        task_node_id=primary_task.task_node_id,
        requirements=primary_task.requirements,
        instruction=primary_task.instruction,
        submission_path=self.get_work_dir(),
    ), None
```

### get_worktree_name(engineer_id)

Return the worktree directory name inside `/workspace/`.

```python
def get_worktree_name(self, engineer_id):
    return f"my_repo_{engineer_id}_worktree"
```

### build_completed_task_summary(result, task_status)

Return a text summary of a completed task for the `assign_task` prompt context.

```python
def build_completed_task_summary(self, result, task_status):
    return (
        f"Task {result.task_id}: {task_status}\n"
        f"Requirements: {result.requirements[:200]}\n"
        f"Files modified: {', '.join(result.files_modified)}"
    )
```

### get_single_agent_info(workspace, config, prompts)

Return `(header_text, user_instruction, log_content)` for single-agent mode.

```python
def get_single_agent_info(self, workspace, config, prompts):
    instruction = prompts.get("single_agent_instruction", "").format(
        work_dir=self.get_work_dir(),
    )
    return (
        "Single Agent: My Task",
        instruction,
        {"task": "my_task", "work_dir": self.get_work_dir()},
    )
```

---

## 4. Implement subagent runner methods

These methods customize what happens when engineers run.

### create_subagent_result(subagent)

Create the initial result object for a subagent.

```python
def create_subagent_result(self, subagent):
    return SubAgentResult(
        engineer_id=subagent.engineer_id,
        task_id=subagent.task_id,
        task_node_id=subagent.task_node_id,
        requirements=subagent.requirements,
    )
```

### populate_success_result(result, runner, commit_info)

Fill in result fields after a successful run.

```python
def populate_success_result(self, result, runner, commit_info):
    result.success = True
    result.commit_hash = commit_info.get("hash", "")
    result.commit_message = commit_info.get("message", "")
    result.files_modified = runner.get_modified_files()
    result.git_diff = runner.get_git_diff()
```

### Other required methods

The following must be implemented — see `tasks/base.py` for signatures and `tasks/paperbench.py` for examples:

| Method | Purpose |
|--------|---------|
| `get_followup_prompt_args(subagent)` | Variables for `followup_prompt` template |
| `get_run_start_log_lines(subagent)` | Log lines printed at run start |
| `get_event_serialization_extras(subagent)` | Extra fields in event JSONL |
| `get_print_summary_lines(result, commit_info)` | Commit summary log lines |
| `get_new_task_print_lines(subagent)` | Log lines when new task assigned |
| `get_onboard_names(engineer_id)` | `(branch_name, worktree_name)` tuple |
| `get_completion_print_lines(result)` | Log lines after engineer completes |
| `get_log_agent_response_kwargs(result)` | kwargs for `output_logger.log_agent_response()` |
| `get_conflict_instruction_args(subagent, conflict_files, workspace, repo_dir)` | Variables for `conflict_resolution` template |
| `get_execution_summary_lines(results)` | Summary lines at the end of all runs |

---

## 5. Create prompt templates

Create `prompts/my_task.yaml`. The minimum set of keys required:

```yaml
user_instruction: |
  (Used as the manager's system prompt suffix in multi-agent mode)
  You are a software engineering manager...

scan_analysis: |
  (Sent to the manager to explore the task)
  Start by exploring...

assign_task: |
  (Sent after each engineer completes a round)
  {engineer_id} has completed round {completed_round}...

subagent_prompt: |
  (Sent to the engineer at the start of round 1)
  You are a software engineer...

followup_prompt: |
  (Sent to the engineer at the start of rounds 2+)
  Here is your next task...

conflict_resolution: |
  (Sent when a merge conflict needs resolution)
  Your branch has merge conflicts...

auto_reassign: |
  (Sent when an engineer didn't commit — continue same task)
  Continue finishing your previous task...

manager_final_review_all: |
  (Sent for the final review phase)
  All engineers have completed their work...

single_agent_instruction: |
  (Used in single-agent mode — entire task in one prompt)
  You are a software engineer...
```

See `prompts/commit0.yaml` and `prompts/paperbench.yaml` for complete examples with all template variables.

---

## 6. Register the task

Add your task to `tasks/__init__.py`:

```python
from tasks.my_task import MyTask
```

Add it to the task factory in `core/utils.py`'s `build_task_module()`:

```python
def build_task_module(task, **kwargs):
    if task == "commit0":
        ...
    elif task == "paperbench":
        ...
    elif task == "my_task":
        from tasks.my_task import MyTask, MyTaskConfig
        config = MyTaskConfig(**{k: v for k, v in kwargs.items() if hasattr(MyTaskConfig, k)})
        return MyTask(config=config)
    else:
        raise ValueError(f"Unknown task: {task}")
```

---

## 7. Verify

Run in single-agent mode first to check workspace setup and evaluation:

```bash
uv run python run_infer.py --task my_task --nomulti_agent --max_iterations 20 --model litellm_proxy/...
```

Check that:
- The Docker container starts and `setup_workspace` runs without error
- The manager receives a sensible task description
- `evaluate()` returns a results dict
- Output files appear in `outputs/my_task/`

Then run multi-agent:

```bash
uv run python run_infer.py --task my_task --max_subagents 2 --model litellm_proxy/...
```

Check `delegations.json` to verify the manager's delegation plan looks correct.
