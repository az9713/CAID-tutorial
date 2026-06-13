# What is CAID?

CAID is a multi-agent framework for running software engineering benchmarks: a central manager LLM breaks a task into subtasks and delegates them to multiple engineer LLMs working in parallel, each isolated in its own git worktree.

## The problem it solves

Most LLM benchmark evaluations use a single agent that must complete an entire task sequentially. For large codebases or complex research reproduction tasks, this creates two problems:

- **Context limits**: one agent accumulates a massive conversation history as it works through many files or steps.
- **No parallelism**: one agent can only work on one thing at a time, making wall-clock time proportional to task count.

CAID addresses both by splitting work across multiple agents running simultaneously. The manager only ever needs context about planning and coordination; the engineers only ever need context about their specific subtask.

## How it works

At the highest level, CAID runs a 10-step pipeline:

```
Docker workspace starts
       │
       ▼
Manager scans the task (explores codebase or paper)
       │
       ▼
Manager creates a delegation plan (which engineer gets which subtask)
       │
       ▼
Git worktrees are created (one per engineer, from same base commit)
       │
       ▼
Engineers run in parallel — each in their own worktree
       │
       ▼
As each engineer finishes a round, manager collects + merges their branch
       │
       ▼
Manager assigns the next subtask (or marks engineer finished)
       │
       ▼
Manager runs a final review over all merged work
       │
       ▼
Evaluation runs (pytest or LLM judge)
```

The manager and all engineers share a single Docker container. Isolation between engineers is achieved by git worktrees, not separate containers.

## The two benchmark tasks

CAID ships with two concrete task implementations:

**Commit0** — given a Python repository with all function bodies replaced by `pass` stubs, implement the functions so that the pytest suite passes. The manager analyzes which files have stubs and their dependencies, then delegates file-by-file (or function-by-function) to engineers.

**PaperBench** — given a machine-learning research paper, reproduce its experiments and submit a `reproduce.sh` script. The manager reads the paper and rubric, delegates sub-experiments to engineers, and an LLM judge scores the final submission against the rubric.

## What CAID is not

- **Not a general purpose agent framework.** CAID is a research evaluation harness. The "engineers" are OpenHands agents, not arbitrary LLMs.
- **Not a CI/CD system.** There is no persistent server. Each run is a one-shot script invocation.
- **Not container-per-agent.** All agents share the same Docker container; isolation is git-level, not OS-level.

## How the pieces fit together

A typical Commit0 run looks like this end to end:

1. `run_infer.py` parses CLI args and builds a `WorkflowConfig`.
2. A Docker workspace is started and the repository is cloned inside it.
3. The `Manager` agent analyzes the repo (`scan_and_analyze`) and produces a `DelegationPlan`.
4. The manager creates git branches and worktrees for each engineer.
5. `SubAgentRunner` instances are created — one per engineer — and launched concurrently via `asyncio`.
6. As engineers complete rounds, the manager collects their branches, merges them, and assigns the next task.
7. After all engineers finish, the manager runs a final review pass.
8. Pytest runs against the merged codebase; results are saved to `outputs/`.

See [workflow](../concepts/workflow.md) for a step-by-step breakdown of this pipeline.
