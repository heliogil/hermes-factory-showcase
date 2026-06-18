# Hermes Factory — Multi-Agent AI Factory

> Production AI factory running on a VPS. Autonomous agents process task packets
> through a **cron → ephemeral Docker executor → closer/reviewer** pipeline,
> with TDD enforcement, multi-pass execution, and automatic retry on failure.

## Architecture

```
Cron (*/5 min per agent)
    │
    ▼
factory-cycle.sh <agent>
    │
    ├─ flock (one execution per line — prevents races)
    ├─ reads oldest packet from agent-bus/inbox/<agent>/
    │
    ▼
hermes-agent-executor-core.sh
    │
    ├─ validates packet frontmatter
    ├─ docker run --rm hermes-agent-codegraph   ← ephemeral, destroyed after each packet
    │      ├─ MiniMax M3 as brain
    │      ├─ multi-pass: understand → plan → execute
    │      ├─ writes result.md to outbox/
    │      └─ bind mounts: memory/ state/ work/ infra/
    │
    ▼
closer (hermes-closer.sh, runs every minute)
    │
    ├─ pass_.py    — reads result.md, moves packet to done/
    ├─ review.py   — on NEEDS_REVISION: auto-creates <task>-R1 in builder inbox
    └─ deferred.py — escalates stale packets via Jarvis DM (>1h WARN, >4h ALERT)
```

## Agents

| Agent | Line | Role |
|---|---|---|
| **Ergon** | line-a | Builder — implements code, infra, scripts |
| **Argus** | line-a | Reviewer — TDD verification, blast-radius check |
| **Hermes** | line-h | Orchestrator — decomposes specs into packets |
| **Caliope** | line-b | Strategist — content strategy, weekly retro |
| **mkt-strategist** | line-b | Reads metrics → calendar → MKT-BRIEF packets |
| **mkt-creative** | line-b | Consumes briefings → generates copy |

## Packet Format

```markdown
---
Status: ready
Task: ERGON-PCB-101
Owner: Ergon
Priority: P1
Project: pc-builder-br
---

# Task title

## TDD obrigatorio — RED antes de GREEN
1. Write failing test first
2. Run, confirm RED
3. Implement minimum
4. Run, confirm GREEN

## Entregas
- file.py
- All tests GREEN

## Done =
- pytest passes
- Argus approved
```

## Key Patterns

### Ephemeral executors
Each packet runs in a fresh `docker run --rm` container. No state leaks between
tasks. Bind mounts for `memory/`, `state/`, `work/` persist across executions.

### Multi-pass execution
Complex packets use `multi_pass: true`. The agent emits `===HERMES-PHASE-RESULT===`
between phases (understand / plan / execute), with state in `work/<TASK_ID>/state.json`.
Each phase is one cron tick — prevents timeout on long tasks.

### Auto-reassignment on failure
`closer/review.py` reads `Status: needs_revision` from Argus result and
automatically creates `<TASK_ID>-R1` in the builder inbox. Cap: 5 rounds.

### Metrics ledger
Every packet transition is appended to `state/ledger/factory-ledger.jsonl`:
```json
{"ts": "2026-06-10T...", "root_task": "ERGON-PCB-101", "event": "builder_failed", "line": "line-a"}
```
`tools/factory_metrics.py` computes first-pass rate, block rate, throughput.

## Tech Stack

- **LLM**: MiniMax M3 (flat-rate subscription, zero per-token cost)
- **Executor image**: `hermes-agent-codegraph` — MiniMax M3 + graphify + code-review-graph
- **Orchestration**: bash + flock (no Kubernetes, no Celery — intentionally minimal)
- **Storage**: flat files (JSONL packets, markdown results, JSON state)
- **Code graph**: graphify 0.8.14 — semantic search over SharpAnalysis codebase

## Running a packet manually

```bash
# Place a packet in the inbox
cp my-task.md /opt/hermes-agent-executor/agent-bus/inbox/ergon/

# Trigger one cycle immediately (normally runs every 5 min via cron)
/usr/local/bin/factory-cycle.sh ergon
```

---

*Part of a personal venture studio stack. See also:
[jarvis-showcase](https://github.com/heliogil/jarvis-showcase)*
