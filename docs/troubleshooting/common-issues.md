# Common issues

Top issues encountered when running CAID, ordered by frequency.

---

## Docker container fails to start

**Cause:** Docker daemon is not running, image pull fails, or insufficient disk space.

**Fix:**

```bash
# Verify Docker is running
docker ps

# If "Cannot connect to the Docker daemon":
# macOS: open Docker Desktop
# Linux: sudo systemctl start docker

# Check disk space (need ~5-20 GB free)
df -h

# Pull the image manually to see the actual error
docker pull <image-name-from-get_docker_image()>
```

**If that doesn't work:** Check `run_{timestamp}.log` for the exact error from `DockerWorkspace` or `DockerDevWorkspace`. Stale containers from prior runs sometimes block port binding — run `cleanup_stale_containers()` manually or `docker ps -a` and remove stopped containers.

---

## LLM API errors (400, 401, 429)

**Cause:** Missing or invalid API credentials, model name mismatch, or rate limits.

**Fix:**

```bash
# Verify environment variables are set
echo $LLM_BASE_URL
echo $LLM_API_KEY

# Test the connection directly
curl -H "Authorization: Bearer $LLM_API_KEY" $LLM_BASE_URL/models
```

For 429 (rate limit): reduce `--max_subagents` so fewer concurrent requests are made, or add retry configuration to your LiteLLM proxy.

For 400 with a model name error: verify the model identifier matches what your proxy supports. The default `litellm_proxy/neulab/gpt-5-mini` is proxy-specific — change it to match your setup.

---

## Manager produces no delegation JSON

**Symptom:** `WARNING: No delegation JSON found, using fallback...` in logs. `delegations.json` shows an evenly-distributed fallback plan instead of a reasoned one.

**Cause:** The manager ran out of iterations during the scan or delegation phase before producing the JSON block.

**Fix:**

```bash
# Increase manager iterations
uv run python run_infer.py --task commit0 --max_iterations 80 ...
```

Also check `agent_events/manager_events.jsonl` — if the manager's last few events show it was still exploring the repository when iterations ran out, the scan phase consumed too many iterations. Consider reducing scan depth in the `scan_analysis` prompt.

---

## Engineers fail to commit (success=false)

**Symptom:** `WARNING: No new commit detected` in engineer logs. Engineer results show `success: false`.

**Cause:** The engineer ran out of iterations before finishing implementation, or got confused and stopped without committing.

**Fix:**

Increase subagent iterations:
```bash
uv run python run_infer.py ... --sub_iterations 100 ...
```

Or increase rounds so the engineer gets a retry:
```bash
uv run python run_infer.py ... --rounds_of_chat 3 ...
```

Check `agent_events/{engineer_id}_events.jsonl` to see what the engineer was doing when it stopped. Common causes: got stuck in a test loop, was confused about which file to modify, or hit an LLM error mid-run.

**Note:** Engineers that don't commit are not necessarily wasted — the manager attempts to salvage uncommitted changes via `worktree_commit_merge`. Check the `merge_method` in `outputs.jsonl` to see if recovery succeeded.

---

## Merge conflicts

**Symptom:** `Merge conflict detected for {branch}` in manager logs. `outputs.jsonl` shows `merge_method: "conflict"`.

**Cause:** Two engineers modified the same file.

**Resolution:** The async loop automatically sends a `conflict_resolution` prompt to the affected engineer (if they have rounds remaining). The engineer resolves the conflicts in their worktree and recommits.

If the engineer has no rounds remaining, the manager uses `git merge -X theirs` (force-accept the engineer's version). This is a last resort and may discard work from the other engineer.

**Prevention:** The manager's delegation prompt instructs it to assign non-overlapping files. If conflicts are frequent, check `delegations.json` — if two engineers were assigned the same file, the manager's scan may have missed the overlap. Increase `--max_iterations` so the manager has more time to plan.

---

## PaperBench judge fails to run

**Symptom:** `grade.json` has `judge_score: null` or evaluation throws an exception.

**Cause:** The `paperbench` and `preparedness-turn-completer` packages are not installed, or `JUDGE_PYTHON` is not set.

**Fix:**

```bash
# Install judge packages
cd frontier-evals
uv pip install -e "project/paperbench"
uv pip install -e "project/preparedness_turn_completer"
cd ..

# Set the judge Python path if needed
export JUDGE_PYTHON=$(which python)
```

Also verify `reproduce.sh` exists in the submission directory. If `repro_script_exists: false` in `grade.json`, the engineers did not produce a reproduction script — the judge will score 0 regardless.

---

## Run fails with `SDK_SOURCE_DIR` error

**Symptom:** Error about `software-agent-sdk` directory not found.

**Cause:** `DockerDevWorkspace` (used by some task configurations) requires the OpenHands SDK source directory to build the container.

**Fix:**

```bash
export SDK_SOURCE_DIR=/path/to/software-agent-sdk
```

Or use `DockerWorkspace` (pre-built image) instead by returning a `server_image` key from `get_workspace_config()` rather than `base_image`.

---

## Output directory already exists

**Symptom:** Run writes output files on top of a previous run's files.

**Cause:** The auto-generated output path is deterministic — same parameters produce the same path. A re-run overwrites previous results.

**Fix:** Specify a custom output directory:

```bash
uv run python run_infer.py ... --output_dir outputs/my-experiment-v2
```

Or rename the old output directory before re-running.

---

## Manager uses too many tokens / run is too expensive

**Cause:** Long scan phases, many assign_task rounds, or final review going deep.

**Fix options:**

1. Use a cheaper model for subagents:
   ```bash
   --subagent_model litellm_proxy/neulab/cheaper-model
   ```

2. Reduce engineer iteration limit:
   ```bash
   --sub_iterations 30
   ```

3. Reduce rounds of chat (fewer retries):
   ```bash
   --rounds_of_chat 1
   ```

4. Check `cost.json` to find the expensive phase, then tune that specific phase's parameters.

---

## Background exploration is always cancelled

**Symptom:** `exploration_cancelled: true` in all `background_exploration` events in `outputs.jsonl`.

**Cause:** This is expected behavior when engineers finish quickly. Background exploration is cancelled the moment any engineer completes, so it only runs to completion when all engineers are slower than the exploration.

This is not an error. If you want the manager to explore more before engineers finish, increase `--sub_iterations` so engineers take longer, or reduce `--max_iterations` for exploration (not currently a separate parameter — it uses the manager's global limit).
