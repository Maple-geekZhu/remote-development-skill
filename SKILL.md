---
name: remote-shadow-development
description: Use when taking over a project that has a local workspace, an SSH-accessible remote machine, a remote shadow directory, long-running remote jobs, or local-to-remote code synchronization.
---

# Remote Shadow Development

## Overview

Use this workflow to make local code edits safely while validating and running long jobs on an SSH-accessible remote machine. The core rule is to keep authority, state, and evidence explicit: know which tree is canonical, sync intentionally, run bounded smoke tests before long jobs, and leave reproducible commands/logs.

## Intake Checklist

Before editing, establish these facts from the user, handoff docs, repo scripts, or read-only inspection:

- local workspace path and active branch;
- remote SSH alias or host;
- remote canonical project path, if any;
- remote shadow/work directory where Codex may write;
- environment activation commands for each task;
- sync direction and script, if one exists;
- paths that must not be synced or edited, such as `data/`, `outputs/`, `logs/`, checkpoints, model caches;
- frozen experiments or results that must not be rerun;
- success criterion: code-only, smoke test, full run instructions, commit/push, or merge.

If any item would materially change the work, state the assumption before acting. Do not invent remote paths or environments when they can be inspected.

## Standard Workflow

1. Inspect local state:
   - `git status --short`
   - `git branch --show-current`
   - relevant handoff/docs/scripts
2. Preserve user work:
   - treat uncommitted changes as user-owned;
   - do not reset, delete, or overwrite unrelated files;
   - if a task needs isolation, use a separate worktree or remote shadow directory.
3. Modify locally when possible:
   - use normal repo tests for logic;
   - keep large generated artifacts out of git unless explicitly required.
4. Sync intentionally:
   - prefer the repo-provided sync script;
   - otherwise use `rsync`/archive with explicit excludes for large/generated paths;
   - never reverse-sync remote outputs over source code without a deliberate plan.
5. Validate remotely before long runs:
   - run syntax checks or dry-runs first;
   - run a bounded smoke test using `--max-rows`, a tiny fixture, or a short subset;
   - verify output counts, required artifacts, logs, and exit codes.
6. Provide long-running commands only after smoke passes:
   - use `nohup` or a job manager;
   - write a PID file;
   - write logs to predictable paths;
   - include `tail -f`, `ps -p`, and post-run summary commands.
7. Finish with evidence:
   - list changed files;
   - report local tests and remote smoke commands with exit status;
   - commit/push only when requested or when it is the established project workflow.

## Remote Command Pattern

Use one remote shell script for multi-step jobs. It is easier to review, restart, and log than a long inline command.

```bash
cd /remote/shadow/project
cat > /tmp/run_job.sh <<'SH'
set -euo pipefail
cd /remote/shadow/project
source /path/to/activate /path/to/env

mkdir -p logs outputs/run_name

echo "[1/4] bounded or full step"
python -u scripts/task.py ... > logs/task.log 2>&1

echo "[2/4] evaluate"
python -u scripts/eval.py ...

echo "[3/4] acceptance checks"
python - <<'PY'
# check counts, required files, statuses, and no forbidden tokens
PY
SH

nohup bash /tmp/run_job.sh > logs/run_name.nohup.log 2>&1 &
echo $! > logs/run_name.pid
tail -f logs/run_name.nohup.log
```

For parallel subtasks, redirect each child process to its own log and `wait` before dependent steps:

```bash
python -u step_a.py > logs/step_a.log 2>&1 &
pid_a=$!
python -u step_b.py > logs/step_b.log 2>&1 &
pid_b=$!
wait "$pid_a"
wait "$pid_b"
```

If a child fails, print the tail of its log before exiting.

## Smoke Test Standard

A smoke test passes only when all relevant conditions are true:

- command exits with status 0;
- it uses the same code path and environment as the full job;
- it writes the same classes of artifacts as the full job;
- counts match the bounded subset;
- parsers/evaluators run, not just inference;
- logs contain progress or enough information to diagnose stalls;
- no hidden labels, forbidden IDs, or frozen outputs are consumed unless explicitly authorized.

If smoke exposes a parser or orchestration issue, fix the smallest code/script path and rerun smoke. Do not hand off a full `nohup` command from a failed smoke.

## Sync Rules

- Prefer project-provided scripts such as `push-to-server.ps1`.
- If writing a new sync command, exclude `data/`, `outputs/`, `logs/`, `.git/`, checkpoints, caches, downloaded models, and other large/generated directories unless the user explicitly scopes them in.
- Confirm the remote target with `pwd`/`ls` before modifying it.
- Keep remote generated outputs in remote `outputs/` or another agreed location.
- Do not make the remote canonical source unless the user says so.

## Common Mistakes

| Mistake | Correct behavior |
|---|---|
| Running full training/eval before smoke | Run a bounded smoke through the same code path first. |
| Waiting silently on a long job | Use logs, PID files, and status commands. |
| Mixing local and remote edits | Edit in one canonical place, then sync intentionally. |
| Trusting that output exists | Check counts, schemas, status fields, and required artifacts. |
| Re-running frozen experiments | Treat frozen results as read-only unless explicitly reopened. |
| Giving a huge one-liner | Generate a remote script with step labels and logs. |

## Final Report Template

Keep the final handoff concrete:

```text
Changed:
- file/path: purpose

Verified:
- local: command -> exit 0
- remote smoke: command/path -> exit 0, N rows/artifacts checked

Remote:
- project: /remote/shadow/project
- output/logs: outputs/... and logs/...

Long run:
- nohup command
- PID/log commands
- post-run summary command
```
