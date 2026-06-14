# WSL setup problems & fixes (from the CAID PaperBench setup)

What went wrong getting CAID running under WSL2 on this Windows machine, what actually fixed each issue, and which fixes are permanent.

> **Status:** The painful one — the **network MTU corruption — is now permanently fixed** (a boot hook in `/etc/wsl.conf`, verified). It applies to **every** WSL project, current and future, and survives reboots. The other two issues are a platform choice (use WSL) and a scripting technique (file-based scripts) — see Problems 1–2.

---

## The permanent global fix (read this first)

### The issue it solves

WSL2's `eth0` came up with **MTU 1430** (the Windows host link is effectively ~1430, typical of a VPN/PPPoE). At that MTU, **large TLS transfers get fragmented and corrupted** — you see `cannot decrypt peer's message` / SSL `bad record mac`, or git `early EOF`. Small requests (a few packets) survive; large ones (a 45 MB wheel = tens of thousands of packets) almost always hit a corrupted packet and fail. This made `uv sync` / `pip` / `git clone` unusable for any non-trivial Python project — they all choke on a big wheel like `pyarrow` or `torch`.

### The fix

One line, added to the **existing** `/etc/wsl.conf` (appended under `[boot]`, preserving `systemd=true` and the default user):

```ini
[boot]
command = ip link set dev eth0 mtu 1280
systemd=true
```

Applied with (one `sudo`, once):
```bash
sudo sed -i '/^\[boot\]/a command = ip link set dev eth0 mtu 1280' /etc/wsl.conf
# then, from Windows PowerShell:
wsl --shutdown
```

### How it addresses the issue

- **Lowers the MTU to 1280** (the IPv6 minimum — safe everywhere). Packets are now small enough to traverse the ~1430 host link without fragmentation, so they stop corrupting. Large TLS records arrive intact → no more `bad record mac`.
- **`/etc/wsl.conf [boot] command` runs as root on every WSL boot**, so the MTU is set automatically each time — no manual `ip link` command, no re-doing it after a reboot. **Permanent and global** to all distros/projects.
- **Must be in NAT mode (the default), not `networkingMode=mirrored`.** In mirrored mode `eth0` mirrors the Windows adapter and *ignores* the WSL-side MTU change (the value stayed 1430 — that's why an interactive `sudo ip link set` appeared to do nothing earlier). NAT mode gives WSL its own `eth0` whose MTU actually takes effect.

### Verified

After the fix + restart: `ip link show eth0` → `mtu 1280`, and the exact 47 MB `pyarrow` wheel that corrupted on nearly every prior attempt downloaded clean: `HTTP 200, 48,863,122 bytes in 17.7s`. The per-project wheel-staging workaround (below) is no longer needed.

> If 1280 ever still flakes, the cause is NIC offload rather than pure MTU — also run `sudo ethtool -K eth0 tso off gso off gro off` and add it to the boot command.

---

## Problem 1 — Native Windows can't run the agent (Unix-only stdlib)

**Symptom:** On native Windows, `run_infer.py` crashed at tool registration:
```
ModuleNotFoundError: No module named 'fcntl'
```
and the PaperBench judge crashed on `import resource`, and the terminal tooling on `import pty`.

**Root cause:** `openhands-tools` (`subprocess_terminal.py`) imports `fcntl`/`pty` at module load, and the judge's `nanoeval` imports `resource`. These are **Unix-only** Python stdlib modules with no Windows equivalent. They're imported eagerly, before any OS-conditional logic runs.

**Fix:** Run the whole thing in **WSL2 Ubuntu**, where these modules are native.

**Permanent / general?** This is a platform choice, not a patch. It's permanent for CAID and applies to **any** project whose host-side Python depends on Unix-only modules (most agent/terminal frameworks, anything using `pty`, `fcntl`, `termios`, `resource`, `os.fork`). On Windows, such projects belong in WSL or a Linux container.

---

## Problem 2 — Driving WSL from Windows corrupted multi-line scripts

**Symptom:** Commands like `wsl.exe -d Ubuntu -- bash -lc '<multi-line script>'` intermittently ran with **empty variables** (`$HOME` empty, assigned vars empty). One such corruption turned a copy into an effective `rsync / /` that churned the entire root filesystem.

**Root cause(s):**
- Passing a **multi-line, single-quoted** script as an argv string through `wsl.exe` is fragile — quoting/newline handling intermittently mangles it.
- `bash -lc` (login shell) vs `bash -c` (non-login) differ: login shells set `$HOME`/PATH from profile; non-login `bash -c` does **not** have `~/.local/bin` on PATH, so `uv` was "command not found."

**Fix (a technique, not a config):**
1. **Write the script to a file**, then run it by path — a single clean argument can't be mangled:
   ```bash
   wsl.exe -d Ubuntu bash -c "tr -d '\r' < /mnt/c/path/script.sh > /tmp/script.sh && bash /tmp/script.sh"
   ```
   (`tr -d '\r'` strips CRLF that Windows editors add — CRLF breaks bash with `$'\r': command not found`.)
2. **Guard every destructive script** so a corrupted env aborts instead of running wild:
   ```bash
   set -uo pipefail
   : "${HOME:?HOME empty - aborting}"
   [ -d "$SRC" ] || { echo "FATAL: source missing"; exit 1; }
   ```
3. **Use absolute tool paths** (`$HOME/.local/bin/uv`) instead of relying on PATH under `bash -c`.
4. Always **target the distro explicitly** (`wsl.exe -d Ubuntu`) — after `wsl --shutdown`, a bare `wsl.exe` can attach to the wrong distro (e.g. `docker-desktop`) where `$HOME` is empty.

**Permanent / general?** It's a reusable discipline for **any** non-interactive WSL automation from Windows. Nothing is persisted to the system; you just follow the pattern every time.

---

## Problem 3 — WSL network corrupts large downloads (the hard one)

**Symptom:** Small downloads succeed; large ones (e.g. the 45 MB `pyarrow` wheel) fail every time with:
```
cannot decrypt peer's message
error decoding response body   (a.k.a. SSL "bad record mac")
```
`uv sync` could never complete because `pyarrow` corrupted on every retry. Even index-metadata fetches (`grpcio`) intermittently corrupted.

**Root cause:** **MTU mismatch.** The effective link MTU is ~1430 (typical of a VPN/PPPoE on the Windows host). Large TLS records get fragmented and corrupted; small ones fit and survive. The signature is "small = fine, large = `bad record mac`."

**What did NOT work:**
- `sudo ip link set dev eth0 mtu 1280` while in **mirrored** networking mode — in mirrored mode `eth0` mirrors the Windows adapter and **ignores** the WSL-side MTU change (the value stayed 1430).
- Switching to `networkingMode=mirrored` in `.wslconfig` — made things less stable; **reverted**.
- `--find-links` to a local wheel — uv's resolver still chose the index URL and re-downloaded.
- Pre-`uv pip install` then `uv sync` — `uv sync` is transactional and re-downloaded `pyarrow` anyway.

**What actually worked (a per-project WORKAROUND, not a fix):**
1. Reverted to **NAT** networking (removed `networkingMode=mirrored` from `C:\Users\<you>\.wslconfig`; `wsl --shutdown`). In NAT mode `eth0` is WSL's own interface (private `172.x` IP) whose MTU *can* be set — but we still didn't lower it persistently.
2. **Stage the oversized wheels from the Windows side** (where the network is fine) and install them by **direct wheel path** so uv never fetches them over the flaky link:
   ```bash
   # Windows (reliable network): download the exact Linux wheel
   curl -sSL -o caid_wheels/pyarrow-XX.whl https://files.pythonhosted.org/.../pyarrow-...manylinux...x86_64.whl
   # WSL: install by direct path BEFORE the dependent install, so it's already satisfied
   uv pip install --python <venv>/bin/python /mnt/c/.../caid_wheels/pyarrow-XX.whl
   ```
3. **Retry loop** for everything else: `uv pip install -r req.txt` in a loop. uv caches each clean download and skips installed packages, so intermittent successes accumulate until a pass completes.

**This wheel-staging is a per-project workaround — and it is no longer needed**, because the **permanent MTU fix at the top of this document is now applied**. With MTU pinned at 1280, large wheels download cleanly and `uv sync` just works. The staged wheels were deleted after the permanent fix was verified; if you ever revert the fix and need them, re-download per the steps above.

---

## Current state of this machine (as of setup)

| Setting | Location | Current value | Permanent? |
|---|---|---|---|
| Networking mode | `C:\Users\simon\.wslconfig` | NAT (default) | reverted to default |
| MTU boot hook | `/etc/wsl.conf` | **`command = ip link set dev eth0 mtu 1280`** | ✅ applied + verified |
| `eth0` MTU | runtime | **1280** — large transfers now clean | ✅ permanent (survives reboots) |
| Staged wheels | (was `Downloads\caid_wheels\`) | **deleted** after fix verified | not needed |
| WSL project | `~/CAID_cmu` (WSL2) | main + judge venvs built, `.env` Linux paths | ✅ working |

**Bottom line:** CAID works, and the network is now **permanently fixed** — verified by downloading the 47 MB `pyarrow` wheel clean (`HTTP 200, 48,863,122 bytes`) that used to corrupt every time. Future WSL projects will no longer hit the corruption; the wheel-staging workaround is no longer required.

---

## Helper scripts produced during setup

These live in `C:\Users\simon\Downloads\` and encode the working patterns:
- `wsl_copy.sh` — guarded copy from `/mnt/c` into WSL
- `wsl_findlinks_install.sh` / `wsl_install_retry.sh` — retry-loop installs over the flaky link
- `wsl_judge_setup.sh` — judge venv with pre-staged `pyarrow`
- `wsl_fix_env.sh` — rewrite `.env` to Linux paths
- `wsl_smoke.sh` — the `uv run --no-sync` smoke test
