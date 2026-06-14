# CAID — Centralized Asynchronous Isolated Delegation

A multi-agent workflow where a central manager LLM delegates software engineering tasks to parallel engineer subagents, each working in an isolated git worktree.

---

## Documentation

| Section | What's inside |
|---------|--------------|
| [Overview](overview/what-is-this.md) | What CAID is, the problem it solves, how it works |
| [Key concepts](overview/key-concepts.md) | Glossary of every term used across the docs |
| [Prerequisites](getting-started/prerequisites.md) | Exact software requirements and verify commands |
| [Quickstart](getting-started/quickstart.md) | Running your first experiment in under 15 minutes |
| [Workflow](concepts/workflow.md) | The 10-step pipeline from boot to evaluation |
| [Manager agent](concepts/manager.md) | Scan, delegate, assign, review — the manager's full lifecycle |
| [Subagents](concepts/subagents.md) | Git worktrees, parallel runners, rounds, and merge strategies |
| [Add a task](guides/add-a-task.md) | How to implement the TaskModule interface for a new benchmark |
| [Extend with agentic features](guides/extend-with-agentic-features.md) | Five additive features with implementation sketches and unit tests |
| [Interpret output](guides/interpret-output.md) | Reading cost.json, outputs.jsonl, grade.json, and patch.diff |
| [Configuration reference](reference/configuration.md) | All WorkflowConfig fields and CLI arguments |
| [Output files reference](reference/output-files.md) | Every output file: format, location, when produced |
| [System design](architecture/system-design.md) | Architecture decisions, data flows, async model |
| [Troubleshooting](troubleshooting/common-issues.md) | Docker failures, merge conflicts, LLM errors |

> **New here?** Start with [what is this](overview/what-is-this.md), then follow the [quickstart](getting-started/quickstart.md).
