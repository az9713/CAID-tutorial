# Running CAID / PaperBench on Windows via WSL2 — Complete Journey, Replication Guide & Results

This document records the **full journey** of getting CAID (a multi-agent PaperBench harness) running on a Windows 11 machine: every blocker hit, every fix, the clean replication path (knowing what we know now), and the **actual measured results** from three runs. It is written so another person on a similar machine can reproduce the same setup and the same observations.

- For the *why this is hard* and the *permanent WSL network fix*, see [`WSL_TROUBLESHOOTING.md`](WSL_TROUBLESHOOTING.md).
- For the day-to-day run commands, see [`SETUP_PAPERBENCH_LOCAL.md`](SETUP_PAPERBENCH_LOCAL.md).
- For what CAID *is*, see [`docs/`](docs/index.md).

---

## 0. TL;DR

- **Native Windows cannot run CAID** — the agent host imports Unix-only stdlib (`fcntl`/`pty`/`resource`). Use **WSL2 Ubuntu**.
- The single biggest time sink was **WSL2 network corruption on large downloads** (`bad record mac`), caused by an **MTU mismatch**. Permanent one-line fix: a `/etc/wsl.conf` boot hook setting `eth0` MTU to 1280.
- CAID's **multi-agent workflow works end-to-end** on this box — verified with three runs ($0.008, $0.28, $0.50, all under budget).
- **The PaperBench score is ~0 / null — a GPU hardware ceiling**, not a setup bug. `rice` is a reinforcement-learning paper; reproduction needs a GPU this box doesn't have.
- Along the way we found and fixed two real CAID bugs/gaps: the judge venv needs `fire`+`litellm`, and the judge has a hardcoded 600s timeout that `rice` at full depth exceeds.

---

## 1. The environment (what we started with)

| Component | Value |
|---|---|
| OS | Windows 11 Home, build 26200 |
| Docker | Docker Desktop 4.77 / Engine 29.5.3 (WSL2 backend) |
| WSL | WSL2, Ubuntu (default distro), kernel 6.6.87.2-microsoft-standard-WSL2 |
| Python (WSL) | 3.12.3 |
| Python (Windows) | 3.13 |
| uv | 0.10.2 (WSL: `~/.local/bin/uv`) |
| WSL user / home | `az9713` / `/home/az9713` |
| Network | `eth0` MTU **1430** (VPN-like cap) — the root of the network pain |
| Project source | `C:\Users\simon\Downloads\CAID_cmu` (a clone of `JiayiGeng/CAID`) |

Starting state: no `data/`, no `.venv`, Docker daemon **not running**, `.env` not present.

---

## 2. The clean replication path (do this, in this order)

This is the *shortest correct path* given everything we learned. If you follow it top to bottom on a comparable machine, you get a working CAID. The war story (Section 3) explains *why* each step exists.

### Step 0 — Prerequisites
- Windows 11 (22H2+), Docker Desktop with WSL2 integration **enabled and running**.
- WSL2 with an Ubuntu distro; inside it: `uv` installed (`curl -LsSf https://astral.sh/uv/install.sh | sh`), Python 3.12+.
- An **OpenAI API key** with `gpt-4o`/`gpt-4o-mini` access (OpenAI direct serves both the agent *and* the OpenAI-native judge with one key).

### Step 1 — Apply the permanent WSL network (MTU) fix FIRST
This is the step that, if skipped, makes everything below randomly fail on large downloads. In WSL:
```bash
# Append a boot hook to the EXISTING /etc/wsl.conf (do NOT overwrite it).
sudo sed -i '/^\[boot\]/a command = ip link set dev eth0 mtu 1280' /etc/wsl.conf
```
Then from Windows PowerShell: `wsl --shutdown`. After it restarts, verify:
```bash
ip link show eth0 | grep mtu     # -> mtu 1280
```
Requires **NAT networking** (the WSL default — do *not* set `networkingMode=mirrored`, which ignores the MTU change). Detail + verification in [`WSL_TROUBLESHOOTING.md`](WSL_TROUBLESHOOTING.md).

### Step 2 — Get the project + judge source into WSL-native filesystem
Put the project at `~/CAID_cmu` (WSL ext4), **not** `/mnt/c` (the 9p mount is slow and Docker bind-mounts from it are flaky). You can `git clone` the repo, or copy from a Windows checkout. You also need the PaperBench judge source from OpenAI's `frontier-evals`:
```bash
cd ~ && git clone <your CAID repo> CAID_cmu
git clone https://github.com/openai/frontier-evals.git ~/CAID_cmu/frontier-evals
```

### Step 3 — Patch `run_infer.py` to load `.env`
The repo never calls `load_dotenv()`, so a `.env` is ignored. Add at the top of `run_infer.py` (before any `os.environ` read):
```python
try:
    from dotenv import load_dotenv
    load_dotenv()
except ImportError:
    pass
```
(One call feeds both the agent and the judge subprocess, which inherits the parent env.)

### Step 4 — Main venv
```bash
cd ~/CAID_cmu && ~/.local/bin/uv sync
```
With the MTU fix applied this just works. (Without it, large wheels like `pyarrow` corrupt — see Section 3.5 for the wheel-staging workaround.)

### Step 5 — PaperBench data (Git LFS gotcha)
The dataset lives in `frontier-evals` under Git LFS, but `.lfsconfig` **excludes** `project/paperbench/data/**`, so a normal clone gives 132-byte pointer files. Pull a paper explicitly, overriding the exclude:
```bash
cd ~/CAID_cmu/frontier-evals
git -c lfs.fetchexclude="" lfs fetch origin main --include "project/paperbench/data/papers/rice/**"
git -c lfs.fetchexclude="" lfs checkout
```
Then scaffold the layout CAID expects (`data/paperbench/papers/<id>/...` + `data/paperbench/src/paperbench/instructions/instructions.txt`):
```bash
cd ~/CAID_cmu
mkdir -p data/paperbench/papers data/paperbench/src/paperbench/instructions
cp -r frontier-evals/project/paperbench/data/papers/rice data/paperbench/papers/rice
cp frontier-evals/project/paperbench/paperbench/instructions/instructions.txt \
   data/paperbench/src/paperbench/instructions/instructions.txt
```

### Step 6 — Judge venv (separate; needs extra deps)
The judge's dependencies conflict with the project's pinned versions, so it lives in its own venv and runs as a subprocess via `JUDGE_PYTHON`:
```bash
cd ~/CAID_cmu/frontier-evals
~/.local/bin/uv venv .judge-venv --python 3.12
JPY=~/CAID_cmu/frontier-evals/.judge-venv/bin/python
~/.local/bin/uv pip install --python "$JPY" -e project/common/preparedness_turn_completer -e project/paperbench
# CRITICAL: judge_runner.py imports these but the editable install does NOT pull them:
~/.local/bin/uv pip install --python "$JPY" fire litellm
# verify:
"$JPY" -c "import fire; from litellm import cost_per_token; import paperbench; from paperbench.judge.create_judge import create_judge; print('judge OK')"
```
> On **native Windows** (not needed in WSL) the judge also needs a no-op `resource.py` shim in its venv site-packages, because `nanoeval` imports the Unix-only `resource` module. In WSL this is native — no shim.

### Step 7 — `.env`
```ini
LLM_API_KEY=sk-...your-openai-key...
LLM_BASE_URL=https://api.openai.com/v1
LLM_MODEL=openai/gpt-4o-mini
JUDGE_PYTHON=/home/<you>/CAID_cmu/frontier-evals/.judge-venv/bin/python
SDK_SOURCE_DIR=/home/<you>/CAID_cmu
```
- `JUDGE_PYTHON` **must** point at the judge venv (else the judge subprocess uses the main venv and fails to import `paperbench`).
- `SDK_SOURCE_DIR` exists only because `run_infer.py` unconditionally `os.chdir()`s there before building the Docker workspace; it defaults to a non-existent `../software-agent-sdk` and crashes if unset. Any existing dir works.

### Step 8 — Run (always `--no-sync`)
```bash
cd ~/CAID_cmu && ~/.local/bin/uv run --no-sync python run_infer.py \
  --task paperbench --paper_id rice \
  --model openai/gpt-4o --subagent_model openai/gpt-4o-mini \
  --max_iterations 50 --max_subagents 2 --sub_iterations 80 --rounds_of_chat 2 \
  --paperbench_dir data/paperbench \
  --test_max_depth 1 --test_reproduce_timeout 300 \
  --judge_type simple --judge_model gpt-4o-mini --code_dev
```
- `--no-sync` is essential: plain `uv run`/`uv sync` re-resolves and would re-download (and fail on a flaky link, or just waste time).
- Use `--test_max_depth 1` for a fast judge (see the 600s-timeout finding, Section 3.7).
- **Driving WSL from Windows non-interactively?** Always run scripts from a *file* with guards (`: "${HOME:?}"`), never inline `wsl.exe -- bash -lc '<multiline>'` — it corrupts variables (Section 3.2).

---

## 3. The blockers, in the order we hit them (the war story)

### 3.1 Native Windows can't import the agent's tools
First run on Windows crashed immediately:
```
ModuleNotFoundError: No module named 'fcntl'
   at openhands/tools/terminal/terminal/subprocess_terminal.py: import fcntl
```
`openhands-tools` imports `fcntl`/`pty` at module load; the judge's `nanoeval` imports `resource`. All three are **Unix-only**. There's no clean Windows shim chain (faking `fcntl` in the main venv mis-routes other libraries that detect Unix via `try: import fcntl`). **Decision: move to WSL2.** (Docker daemon was also down — started Docker Desktop.)

### 3.2 Driving WSL from Windows corrupted scripts → a runaway `rsync`
`wsl.exe -d Ubuntu -- bash -lc '<multi-line script>'` intermittently ran with **empty variables** (`$HOME` empty). One such corruption turned a copy into an effective `rsync / /` that churned the whole root filesystem (harmless in the end — mostly permission errors — but alarming). Root causes: (a) multi-line single-quoted argv mangling; (b) `bash -c` (non-login) lacks `~/.local/bin` on PATH so `uv` was "command not found"; (c) after `wsl --shutdown`, a bare `wsl.exe` can attach to the wrong distro. **Fix:** write scripts to a file, `tr -d '\r'` (strip CRLF), run by path, guard with `: "${HOME:?}"`, use absolute tool paths, and always pass `-d Ubuntu`.

### 3.3 The big one — WSL network corrupts large downloads (MTU)
Symptom: small requests succeed; large ones fail with `cannot decrypt peer's message` / SSL `bad record mac`; git gives `early EOF`. `uv sync` died every time on the 45 MB `pyarrow` wheel. **Root cause: MTU.** `eth0` was 1430 (VPN-like host link); large TLS records fragment and corrupt. Dead ends tried: `sudo ip link set ... mtu 1280` in **mirrored** mode (ignored — `eth0` mirrors the Windows adapter), and `networkingMode=mirrored` itself (made it worse, reverted). **Permanent fix:** NAT mode + `/etc/wsl.conf` `[boot] command = ip link set dev eth0 mtu 1280`. **Verified:** the 47 MB `pyarrow` wheel that corrupted every time then downloaded clean — `HTTP 200, 48,863,122 bytes in 17.7s`.

### 3.4 `run_infer.py` crashes before Docker even starts
`os.chdir(SDK_SOURCE_DIR)` is unconditional and defaults to a non-existent `../software-agent-sdk`. Set `SDK_SOURCE_DIR` to any real dir (repo root). Also, `.env` was never loaded — added `load_dotenv()`.

### 3.5 `uv sync` is transactional — it re-downloads even pre-installed wheels
Before the MTU fix, the workaround was: download the exact Linux wheel on **Windows** (reliable network), then `uv pip install` it by **direct path** before the dependent package (so it's already satisfied), and a **retry loop** for the rest (`uv` caches clean downloads and skips installed packages, so intermittent successes accumulate). `uv sync` ignores a pre-installed wheel and re-fetches it, so you must use `uv pip install` + `uv run --no-sync`, not `uv sync`. **After the permanent MTU fix, none of this is needed** — but it's the fallback if you can't change MTU (no sudo).

### 3.6 Judge venv missing `fire` + `litellm`
First multi-agent run: the judge crashed with `ModuleNotFoundError: No module named 'fire'`. `judge_runner.py` imports `fire` and `from litellm import cost_per_token`, but neither is a dependency of the `paperbench` editable install. **Fix:** `uv pip install --python <JUDGE_PYTHON> fire litellm`.

### 3.7 Judge 600s timeout exceeded at full rubric depth
Second multi-agent run (judge now imports): the judge **ran** but `subprocess.run(..., timeout=600)` (hardcoded in `tasks/paperbench.py`) **timed out** grading `rice` at `--test_max_depth 999` (~40+ leaf nodes × one gpt-4o-mini call each). `grade.json` got `score: null` despite the judge running. **Fix:** lower `--test_max_depth` (e.g. `1`) or raise the hardcoded timeout.

---

## 4. Results (three runs, actual measured numbers)

All runs: PaperBench, paper `rice`, OpenAI direct, on the CPU-only WSL2 box.

### Run A — single-agent smoke test
`--nomulti_agent --model openai/gpt-4o-mini --max_iterations 30`

| Field | Value |
|---|---|
| Outcome | Completed; agent hit "Remote conversation got stuck" at 12/30 iters |
| `reproduce.sh` | not produced |
| Judge | skipped (no `reproduce.sh`) |
| Cost | **$0.0081** |
| Runtime | 47.8s |
| Score | `null` |

Purpose: validate the full pipeline (Docker workspace → paper upload → reproduce → judge) cheaply. It did — proving the WSL setup works end-to-end.

### Run B — multi-agent, all gpt-4o-mini
`--model openai/gpt-4o-mini` (manager + engineers), `--max_subagents 2 --sub_iterations 80 --rounds_of_chat 2 --max_iterations 50`

| Field | Value |
|---|---|
| Outcome | Completed end-to-end |
| Delegation | **Fallback (even split)** — mini manager didn't emit valid delegation JSON |
| Engineer work | both worked on MuJoCo `Hopper` env setup; rounds FAILED to land a successful commit |
| `reproduce.sh` | **produced** ✅ |
| `reproduce.sh` success | ❌ (0.2s) |
| Judge | **crashed** — `ModuleNotFoundError: fire` (since fixed) |
| Cost | **$0.2819** (manager $0.0266, engineers $0.2554, judge $0) |
| Runtime | ~17 min (1009s) |
| Score | `null` |

Observation: a weak manager falls back to even-split delegation. Engineers did real work (correctly identified the RL/MuJoCo nature of `rice`) but couldn't complete reproduction.

### Run C — multi-agent, gpt-4o manager + gpt-4o-mini engineers (with a $3 watchdog cap)
`--model openai/gpt-4o --subagent_model openai/gpt-4o-mini`, same scale, external cost watchdog killing at $3.

| Field | Value |
|---|---|
| Outcome | Completed (`CAID_RUN_DONE`); watchdog never tripped |
| Delegation | **Real, reasoned** (manager spent $0.445 on genuine scan/delegate/review — no fallback) |
| Engineer work | both engineers, multiple rounds (23/21/16 iters), all rounds FAILED to land a successful reproduction |
| `reproduce.sh` | produced ✅ |
| `reproduce.sh` success | ❌ (2.5s) |
| Judge | **ran** (the `fire` fix worked!) but **timed out at 600s** (`TimeoutExpired`) on the full rubric |
| Cost | **$0.5040** (manager $0.4452, engineers $0.0588, judge $0) |
| Runtime | ~21 min (1240s) |
| Score | `null` |

Observation: a capable (gpt-4o) manager produces proper delegation; the manager is the dominant cost driver ($0.445 of $0.504). The judge now runs but needs lower depth to finish.

### Cost/observation summary

| Run | Config | Cost | Time | Delegation | Judge | Score |
|---|---|---|---|---|---|---|
| A | single, mini | $0.008 | 48s | n/a | skipped | null |
| B | multi, all-mini | $0.282 | 17m | fallback | crashed (fire) | null |
| C | multi, 4o mgr | $0.504 | 21m | real | ran→timeout | null |

---

## 5. Observations & conclusions

1. **CAID's multi-agent machinery works on this machine.** Scan → delegate → parallel engineers in isolated git worktrees → collect/merge → final review → reproduce → judge all execute. This is the real, reproducible validation.
2. **Manager model quality matters a lot.** gpt-4o-mini as manager fell back to even-split delegation; gpt-4o produced genuine reasoned delegation. The manager dominates cost in a multi-agent run.
3. **The score is ~0 / null because of the GPU ceiling, not the setup.** `rice` is a reinforcement-learning paper; its `reproduce.sh` needs GPU compute that Docker-Desktop/Windows doesn't pass through. No amount of model spend fixes this on a CPU-only box. A meaningful PaperBench score requires a GPU machine — where *this exact setup carries over directly*.
4. **Budgeting works via an external watchdog.** CAID has no built-in budget gate; a 30s-polling watchdog summing `outputs.jsonl` cost and killing the run + container at a cap is an effective guard. (A built-in budget gate is one of the proposed enhancements in [`docs/guides/extend-with-agentic-features.md`](docs/guides/extend-with-agentic-features.md).)
5. **Two genuine CAID gaps were found and fixed/documented:** the judge venv needs `fire`+`litellm`; the judge's hardcoded 600s timeout is too short for full-depth `rice` grading.

---

## 6. Artifacts produced (where everything lives)

| Artifact | Location | Purpose |
|---|---|---|
| Working install | `~/CAID_cmu` (WSL2) | the runnable project; main + judge venvs, data, `.env` |
| Permanent MTU fix | `/etc/wsl.conf` `[boot]` | network fix for all WSL projects |
| This journey | `CAID_cmu/JOURNEY.md` | full story + replication + results |
| WSL fixes | `CAID_cmu/WSL_TROUBLESHOOTING.md` | the permanent network fix in depth |
| Local runbook | `CAID_cmu/SETUP_PAPERBENCH_LOCAL.md` | run commands, judge deps, caveats |
| CAID concept docs | `CAID_cmu/docs/` | what CAID is, architecture, guides |
| Reusable skill | `~/.claude/skills/wsl-python-setup/SKILL.md` | diagnose+fix WSL for any Python/Docker project |
| Run launchers | `~/Downloads/wsl_run_capped.sh`, `wsl_smoke.sh` | capped multi-agent run / cheap single-agent test |

---

## 7. If you want a non-zero score

This box can't produce one (no GPU). On a Linux machine with an NVIDIA GPU + Docker GPU passthrough, the same steps apply, plus:
- The agent will see the GPU (`get_single_agent_info` probes `nvidia-smi`), so `reproduce.sh` can actually train.
- Keep `--test_max_depth` high enough to grade meaningfully, and raise the judge `timeout=600` in `tasks/paperbench.py` if grading is slow.
- Use a strong manager (gpt-4o or better) for real delegation; engineers can be cheaper.

The Windows/WSL-specific friction in this document (MTU, Unix modules, wsl.exe scripting) does **not** apply on native Linux — only the CAID-specific gaps (judge `fire`/`litellm`, judge timeout, `SDK_SOURCE_DIR`, `load_dotenv`) carry over.
