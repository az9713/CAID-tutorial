# Extend CAID with agentic features

Five concrete features that address real gaps in the current design. Each is scoped to be additive — no rewrites of existing code — and includes a clear testing strategy that doesn't require a running Docker container or live LLM.

---

## 1. Per-merge test feedback

**Gap:** Tests only run once at the very end. The manager assigns new tasks with no knowledge of whether prior merges broke anything. A regression introduced by engineer_1 might not surface until after engineer_3 has built on top of it.

**What to build:** A `run_scoped_tests()` function that fires inside `collect_and_merge()` after a successful merge. It runs pytest scoped to the files the engineer just modified and injects the pass/fail delta into the `assign_task` prompt.

```python
# core/testing.py
def run_scoped_tests(workspace, repo_dir, files_modified, test_cmd="pytest", test_dir="tests/") -> dict:
    """Run tests related to the modified files and return a structured result."""
    if not files_modified:
        return {"ran": False, "reason": "no files modified"}

    # Build a focused test invocation using pytest's -k flag or file-level targeting
    test_targets = " ".join(
        f"tests/test_{Path(f).stem}.py"
        for f in files_modified
        if Path(f).suffix == ".py"
    )
    cmd = f"cd {repo_dir} && {test_cmd} {test_targets} --tb=no -q 2>&1 || true"
    result = workspace.execute_command(cmd, timeout=120)

    # Parse the summary line: "3 passed, 1 failed in 2.3s"
    summary = _parse_pytest_summary(result.stdout)
    return {"ran": True, "passed": summary["passed"], "failed": summary["failed"], "output": result.stdout[-500:]}
```

The result is then formatted into the `assign_task` prompt:

```yaml
# In prompts/commit0.yaml, append to assign_task template:
<test_feedback>
After merging {engineer_id}'s branch: {test_passed} passed, {test_failed} failed.
{test_summary}
</test_feedback>
```

**Integration point:** `Manager.collect_and_merge()` already returns a `review_result` dict — add `"test_feedback"` to it, then pass it through to `assign_task()`.

**How to test without Docker or LLM:**

```python
# tests/test_scoped_tests.py
from unittest.mock import MagicMock
from core.testing import run_scoped_tests

def test_returns_parsed_counts():
    workspace = MagicMock()
    workspace.execute_command.return_value = MagicMock(
        stdout="3 passed, 1 failed in 2.3s", exit_code=0
    )
    result = run_scoped_tests(workspace, "/workspace/repo", ["minitorch/autodiff.py"])
    assert result["passed"] == 3
    assert result["failed"] == 1

def test_skips_when_no_files():
    result = run_scoped_tests(None, "/workspace/repo", [])
    assert result["ran"] is False
```

---

## 2. Cost budget gate with early stopping

**Gap:** All the cost tracking infrastructure exists — `assign_task_total_cost`, per-engineer `result.cost`, per-phase breakdowns — but nothing uses it to make decisions. Expensive runs silently overrun any intended budget.

**What to build:** A `budget_usd` field on `WorkflowConfig` and a `should_accept_new_tasks()` check before each `assign_task` call. When the remaining budget is below a configurable threshold, the manager stops issuing new assignments and lets running engineers finish their current round.

```python
# In config.py — add to WorkflowConfig:
budget_usd: Optional[float] = None          # None = no limit
budget_stop_threshold: float = 0.10         # stop when <10% budget remains

# In core/budget.py:
def budget_remaining(config, manager, subagent_results) -> Optional[float]:
    """Return remaining USD budget, or None if no budget set."""
    if config.budget_usd is None:
        return None
    spent = (
        manager.analysis_cost
        + manager.delegation_cost
        + manager.assign_task_total_cost
        + manager.final_review_cost
        + sum(r.cost for r in subagent_results)
    )
    return max(0.0, config.budget_usd - spent)

def should_accept_new_tasks(config, manager, subagent_results) -> bool:
    remaining = budget_remaining(config, manager, subagent_results)
    if remaining is None:
        return True
    return remaining > config.budget_usd * config.budget_stop_threshold
```

**Integration point:** In `run_subagents_parallel()`, before calling `manager.assign_task()`:

```python
if not should_accept_new_tasks(config, manager, results):
    print(f"[Budget] Remaining budget below threshold — no new assignments")
    idle_runners.clear()
    continue
```

**How to test without Docker or LLM:**

```python
# tests/test_budget.py
from core.budget import should_accept_new_tasks, budget_remaining
from config import WorkflowConfig

def make_manager(analysis_cost=0.0, assign_cost=0.0):
    m = MagicMock()
    m.analysis_cost = analysis_cost
    m.delegation_cost = 0.0
    m.assign_task_total_cost = assign_cost
    m.final_review_cost = 0.0
    return m

def test_no_budget_always_accepts():
    config = WorkflowConfig(budget_usd=None)
    assert should_accept_new_tasks(config, make_manager(), []) is True

def test_stops_when_below_threshold():
    config = WorkflowConfig(budget_usd=1.00, budget_stop_threshold=0.10)
    # spent $0.95 of $1.00 — only $0.05 left (5% < 10% threshold)
    manager = make_manager(analysis_cost=0.50, assign_cost=0.45)
    assert should_accept_new_tasks(config, manager, []) is False

def test_continues_when_above_threshold():
    config = WorkflowConfig(budget_usd=1.00, budget_stop_threshold=0.10)
    manager = make_manager(analysis_cost=0.20, assign_cost=0.30)  # $0.50 spent
    assert should_accept_new_tasks(config, manager, []) is True
```

---

## 3. Dependency-aware task re-ordering

**Gap:** The `remaining_tasks` list in `DelegationPlan` already carries a `depends_on` field (list of file paths), but nothing enforces it. The manager decides ordering by LLM reasoning alone, which means it can accidentally assign a task whose dependencies haven't been merged yet.

**What to build:** A pure-Python `resolve_ready_tasks()` function that filters `remaining_tasks` to only those whose `depends_on` files are all present in the set of merged files. Inject this filtered list (not the full remaining list) into the `assign_task` prompt.

```python
# core/dependency.py
from config import SubAgentTask

def resolve_ready_tasks(remaining_tasks: list, merged_files: set) -> list:
    """Return tasks whose dependencies are all satisfied by merged_files."""
    ready = []
    for task in remaining_tasks:
        depends_on = getattr(task, "depends_on", []) or []
        if all(dep in merged_files for dep in depends_on):
            ready.append(task)
    return ready

def collect_merged_files(subagent_results) -> set:
    """Build the set of all files merged so far."""
    files = set()
    for result in subagent_results:
        if result.merged:
            files.update(result.files_modified or [])
            if result.file_path:
                files.add(result.file_path)
    return files
```

**Integration point:** In `Manager.assign_task()`, replace the prompt's remaining tasks section:

```python
merged = collect_merged_files(all_completed)
ready_tasks = resolve_ready_tasks(
    self.delegation_plan.remaining_tasks, merged
)
# Format ready_tasks into prompt instead of self.delegation_plan.remaining_tasks
```

**How to test without Docker or LLM:**

```python
# tests/test_dependency.py
from core.dependency import resolve_ready_tasks, collect_merged_files

def make_task(task_id, depends_on):
    t = MagicMock()
    t.task_id = task_id
    t.depends_on = depends_on
    return t

def test_returns_tasks_with_no_deps():
    tasks = [make_task("a", []), make_task("b", ["x.py"])]
    ready = resolve_ready_tasks(tasks, merged_files=set())
    assert [t.task_id for t in ready] == ["a"]

def test_unlocks_when_dep_merged():
    tasks = [make_task("b", ["x.py"])]
    ready = resolve_ready_tasks(tasks, merged_files={"x.py"})
    assert [t.task_id for t in ready] == ["b"]

def test_blocks_when_dep_missing():
    tasks = [make_task("c", ["x.py", "y.py"])]
    ready = resolve_ready_tasks(tasks, merged_files={"x.py"})
    assert ready == []
```

---

## 4. Engineer confidence signals

**Gap:** Engineers commit silently. The manager has no signal about whether an engineer was confident or uncertain, so it gives all merged work equal weight in the final review — potentially missing shaky implementations.

**What to build:** Instruct engineers to append a structured JSON block to their commit message. The manager parses this after merge and uses it to prioritize final review attention.

```yaml
# Append to subagent_prompt in prompts/commit0.yaml:
Before committing, append a confidence report to your commit message:
```json
{{"confidence": "high|medium|low", "concerns": "brief description or empty string"}}
```
Commit message format:
  First line: description of what you implemented
  Blank line
  {{"confidence": "...", "concerns": "..."}}
```

```python
# core/confidence.py
import json, re

def parse_engineer_confidence(commit_message: str) -> dict:
    """Extract confidence JSON from the end of a commit message."""
    match = re.search(r'\{[^{}]*"confidence"[^{}]*\}', commit_message or "", re.DOTALL)
    if not match:
        return {"confidence": "unknown", "concerns": ""}
    try:
        data = json.loads(match.group())
        return {
            "confidence": data.get("confidence", "unknown"),
            "concerns": data.get("concerns", ""),
        }
    except json.JSONDecodeError:
        return {"confidence": "unknown", "concerns": ""}

def flag_low_confidence(subagent_results) -> list:
    """Return results where confidence is low or unknown."""
    return [r for r in subagent_results
            if r.confidence in ("low", "unknown")]
```

Low-confidence submissions are highlighted in `manager_final_review_all`:

```yaml
# Append to manager_final_review_all prompt:
<low_confidence_submissions>
The following engineers flagged uncertainty — prioritize reviewing these:
{low_confidence_summary}
</low_confidence_submissions>
```

**How to test without Docker or LLM:**

```python
# tests/test_confidence.py
from core.confidence import parse_engineer_confidence

def test_parses_high_confidence():
    msg = 'Implement forward pass\n\n{"confidence": "high", "concerns": ""}'
    result = parse_engineer_confidence(msg)
    assert result["confidence"] == "high"

def test_parses_low_with_concern():
    msg = 'Partial impl\n\n{"confidence": "low", "concerns": "unclear backward pass"}'
    result = parse_engineer_confidence(msg)
    assert result["concerns"] == "unclear backward pass"

def test_missing_json_returns_unknown():
    result = parse_engineer_confidence("Implement autodiff")
    assert result["confidence"] == "unknown"

def test_malformed_json_returns_unknown():
    result = parse_engineer_confidence('Done\n\n{bad json}')
    assert result["confidence"] == "unknown"
```

---

## 5. Cross-engineer diff injection on task assignment

**Gap:** When engineer_2 is assigned a file that depends on work engineer_1 just merged, engineer_2 gets no context about what engineer_1 actually implemented — only what the stubs *were* supposed to do. If engineer_1 made design choices (renamed arguments, changed return types, picked different data structures), engineer_2 can only discover this by re-exploring the repo.

**What to build:** A `build_dependency_context()` function that, at assignment time, computes a diff of the dependency files between the base commit and the current HEAD, and appends a summary to the engineer's instruction.

```python
# core/context_injection.py
def build_dependency_context(workspace, repo_dir, depends_on: list, base_commit: str) -> str:
    """Return a diff summary of dependency files since base_commit."""
    if not depends_on or not base_commit:
        return ""

    sections = []
    for file_path in depends_on:
        cmd = (
            f"cd {repo_dir} && "
            f"git diff {base_commit}..HEAD -- {file_path} --stat 2>/dev/null"
        )
        stat_result = workspace.execute_command(cmd, timeout=30)
        if not stat_result.stdout.strip():
            continue  # file unchanged since base

        diff_cmd = (
            f"cd {repo_dir} && "
            f"git diff {base_commit}..HEAD -- {file_path} -U3 2>/dev/null | head -80"
        )
        diff_result = workspace.execute_command(diff_cmd, timeout=30)
        sections.append(
            f"### Changes in {file_path} (merged by another engineer):\n"
            f"```diff\n{diff_result.stdout.strip()}\n```"
        )

    if not sections:
        return ""

    return (
        "\n\n<dependency_context>\n"
        "The following dependency files were recently implemented by other engineers. "
        "Use these implementations (not the original stubs) as your reference:\n\n"
        + "\n\n".join(sections)
        + "\n</dependency_context>"
    )
```

**Integration point:** In `Manager.assign_task()`, after building `subagent`:

```python
if subagent.depends_on and self.subagent.base_commit:
    dep_context = build_dependency_context(
        self.workspace, self.repo_dir,
        subagent.depends_on, first_base_commit
    )
    subagent.instruction += dep_context
```

**How to test without Docker or LLM:**

```python
# tests/test_context_injection.py
from unittest.mock import MagicMock, call
from core.context_injection import build_dependency_context

def make_workspace(stat_output="1 file changed", diff_output="+ def forward(): ..."):
    ws = MagicMock()
    ws.execute_command.side_effect = [
        MagicMock(stdout=stat_output, exit_code=0),   # stat call
        MagicMock(stdout=diff_output, exit_code=0),   # diff call
    ]
    return ws

def test_injects_diff_when_file_changed():
    ws = make_workspace()
    result = build_dependency_context(ws, "/repo", ["minitorch/autodiff.py"], "abc123")
    assert "dependency_context" in result
    assert "autodiff.py" in result

def test_returns_empty_when_file_unchanged():
    ws = MagicMock()
    ws.execute_command.return_value = MagicMock(stdout="", exit_code=0)
    result = build_dependency_context(ws, "/repo", ["minitorch/autodiff.py"], "abc123")
    assert result == ""

def test_returns_empty_with_no_dependencies():
    result = build_dependency_context(None, "/repo", [], "abc123")
    assert result == ""
```

---

## Implementation order

Start with **2 (budget gate)** and **3 (dependency re-ordering)** — both are pure Python with zero external dependencies and can be fully tested and merged without touching Docker or LLM infrastructure. Then **4 (confidence signals)** adds a prompt change plus a pure-Python parser. Then **1 (test feedback)** and **5 (diff injection)** which require mocking the workspace. None of the five require changes to the `TaskModule` interface or the async coordination loop structure.
