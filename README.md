# Cyber-Bench - Tasks, Runs & Headroom Evaluation

**Vendor:** PilotCrew AI &nbsp;•&nbsp; **Domain:** Web Security / CTF / Memory-Vulnerability &nbsp;•&nbsp; **Format:** Cyber-Bench, packaged as Harbor / Terminal-Bench task directories.

Cyber-Bench packages **100 cybersecurity tasks** across six categories — Web CTF, Web-5 combined, Memory Vulnerability, Binary Exploitation, Reverse Engineering, and Cryptographic Attack — as self-contained Harbor / Terminal-Bench task directories, runs frontier coding agents against them, and scores submitted flags. This repo holds the **task packages**, the **agent run artifacts**, and a **headroom analysis** of where **`gemini-3.1-pro-preview`** (agent harness **`terminus-2`**) still fails.

---

## Repository layout

| Path | Contents |
|---|---|
| `final_tasks/` | The **100 runnable task definitions** (one folder per task)
| `final_runs/` | Agent **run artifacts** per task — mostly `terminus-2` model runs plus `oracle` reference runs. |
| `final_runs/gemini/` | `gemini-3.1-pro-preview` / `terminus-2` failure artifacts supplied separately: `task002-libxml2-stack-overflow/`, `task004-binutils-as/`, and `web_5_l1_combined/` (one combined Web-5 run). |
| `REPORT.pdf` | The reference benchmark report — methodology, per-model results, per-task failure analysis. |
| `REPORT2.pdf`| **This evaluation's Gemini headroom analysis** (see below). |

---

## Prerequisites

- Docker Engine with **Docker Compose V2**
- **Harbor CLI**
- A Python virtual environment for repo checks
- **OpenRouter API key** in `.env` for model runs

```bash
source .venv/bin/activate      # before running Python / Harbor commands
```

### Task images
Web-5 task Dockerfiles are self-contained: each package builds its `main` agent container from `python:3.12` and installs the recon toolchain (`curl`, `wget`, `nmap`, `nc`, `dig`, `jq`, `git`, `tmux`, `asciinema`). There is no separate `cyberbench/attacker:latest` prebuild. Memory-vul tasks use their task-specific public `n132/arvo:*` vulnerable base images.

---

## The benchmark environment

Each task runs as a multi-container Docker range:

- **Host runner** — starts the bundle, manages the budget, records transcripts, scores flags.
- **Attacker container (`main`)** — the only shell the model controls; recon/exploit commands run here.
- **Target gateway** — exposes stable URLs (e.g. `http://target:8102/`) and forwards to the right challenge container.
- **Challenge containers** — the real vulnerable applications.
- **Scorer** — a deterministic `submit_flag` exact-match verifier (`verifier/reward.txt` = `1.0` pass / `0.0` fail).

---

## Running tasks

### Oracle checks (validate wiring, solution scripts, verifiers — not model capability)

```bash
source .venv/bin/activate

# All tasks
harbor run --path final_tasks --agent oracle --force-build \
  --job-name all_tasks_oracle --jobs-dir jobs/oracle

# One combined Web-5 task / one individual service / one memory-vul task
harbor run --path final_tasks/web_5_l0                       --agent oracle --force-build
harbor run --path final_tasks/co2/l0                         --agent oracle --force-build
harbor run --path final_tasks/task002-libxml2-stack-overflow --agent oracle --force-build
```

### Model runs

```bash
set -a; source .env; set +a            # load OPENROUTER_API_KEY etc.

# One task with terminus-2 through OpenRouter
harbor run --path final_tasks/web_5_l0 \
  --agent terminus-2 --model openrouter/google/gemini-3.1-pro-preview

# Full set
harbor run --path final_tasks \
  --agent terminus-2 --model openrouter/google/gemini-3.1-pro-preview \
  --job-name all_tasks_gemini --jobs-dir jobs/models -n 2
```
---

## Run-artifact layout

A typical run folder (`final_runs/<task>/<trial>/`) contains:

```
result.json              # task_name, agent_info (name/model), agent_execution times, exception_info
config.json              # agent config (agent.name = "terminus-2" or "oracle")
job.log                  # harness log ("Sending keys: [...]" = each shell command)
agent/
  terminus_2.pane        # raw terminal capture of the agent session
  trajectory.json        # structured steps: per-step analysis/plan + tool_calls (keystrokes)
  episode-N/             # per-step prompt/response/debug
verifier/
  reward.txt             # 1.0 (solved) or 0.0 (failed)
  details.json
```

- **`agent.name = "terminus-2"`** → a model run (a real Gemini attempt). **`agent.name = "oracle"`** → the reference-solution run proving the task is solvable (not a model result).
- "Commands" in the analysis = tool-call invocations in `trajectory.json` (or `job.log` "Sending keys" count); "Runtime" = `agent_execution` wall-clock. A run ending exactly at the budget records an `AgentTimeoutError`.

```bash
# Reward for a run
cat final_runs/gemini/task002-libxml2-stack-overflow/*/verifier/reward.txt

# What the agent typed, in order
grep "Sending keys" final_runs/gemini/task004-binutils-as/*/job.log

# Model + outcome
python3 - <<'PY'
import json, glob
r = json.load(open(glob.glob("final_runs/gemini/task002-libxml2-stack-overflow/*/result.json")[0]))
print(r["agent_info"]["name"], r["agent_info"]["model_info"]["name"], r.get("exception_info",{}).get("exception_type"))
PY
``
