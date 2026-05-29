# Worker Control

How the orchestrator controls already-running workers when a new message arrives mid-task. This is **control** (lifecycle), distinct from **transport** ([message_bus.md](message_bus.md)). Referenced from [spec.md](spec.md).

When the user says something while a worker is running, the orchestrator classifies it:
- **Additive** ("after that, also do Y") → **queue** a follow-up (a new/linked task or a pending note delivered when the worker is idle).
- **Corrective** ("no — use credential X instead") → **interrupt** the running worker and re-task it.
- **Steer** (inject a hint mid-run *without* stopping, worker keeps its progress) → **deferred** (see §3).

---

## 1. How interruption works in Hermes today

Interruption is an **in-process** flag ([`tools/interrupt.py:39`](hermes-agent/tools/interrupt.py)):
```python
def set_interrupt(active: bool, thread_id: int | None = None): ...
def is_interrupted() -> bool: ...   # tools check this between steps; True → abort
```
The API server stops an in-process run with `agent.interrupt()` ([`api_server.py` stop-run handler](hermes-agent/gateway/platforms/api_server.py)). **But a kanban worker is a separate subprocess** ([`kanban_db.py:_default_spawn`](hermes-agent/hermes_cli/kanban_db.py)) — the gateway/orchestrator cannot reach its in-process flag. That's the gap to close.

Workers do share state with the orchestrator through one channel: the kanban DB (`HERMES_KANBAN_DB`, injected at spawn).

---

## 2. v1 design — queue + interrupt (plugin-only, no core fork)

### 2a. Queue (additive) — trivial
The orchestrator creates a follow-up task (optionally child-linked to the running one) or appends a pending note to the task. No new mechanism — uses `kanban_create` / comments. Delivered when the worker picks up its next task.

### 2b. Interrupt (corrective) — a shared flag + a worker-side poller
Because the worker shares the kanban DB, we cross the process boundary through it:

1. **Flag store** — a new row keyed by `run_id`:
   ```sql
   CREATE TABLE IF NOT EXISTS worker_interrupts (
       run_id  INTEGER PRIMARY KEY,
       task_id TEXT NOT NULL,
       reason  TEXT,
       created_at INTEGER NOT NULL
   );
   ```
2. **Orchestrator writes the flag** via a small tool (`interrupt_worker(run_id, reason)`), then re-tasks (a fresh task with the corrected instructions).
3. **Worker-side poller** — a daemon thread started by the operator plugin *inside the worker process* watches the table for its own `run_id` and flips the in-process flag:
   ```python
   main_ident = threading.main_thread().ident
   def _poll():
       while True:
           if _has_interrupt(os.environ["HERMES_KANBAN_DB"], os.environ["HERMES_KANBAN_RUN_ID"]):
               set_interrupt(True, main_ident)   # existing between-step check then aborts
               return
           time.sleep(1.5)
   threading.Thread(target=_poll, daemon=True).start()
   ```

This is **plugin-only**: `set_interrupt` is importable, the DB is shared, and the worker already checks `is_interrupted()` between tool calls. Effort: a table, an orchestrator tool, and a ~30–50-line poller thread. No change to Hermes' agent loop.

**Latency:** interruption lands at the worker's next between-step check (≤ ~poll interval + current tool runtime). Good enough — "stop and redo" tolerates a second or two.

---

## 3. Deferred — steer (mid-run injection)

Steer = deliver a message to a *running* worker that it reads and incorporates **without restarting**, preserving progress. Unlike interrupt (a boolean abort), this needs a **message queue drained inside the agent loop** between tool calls and re-injected into the conversation — that's a change to Hermes' core run loop, not a plugin. Deferred until the queue+interrupt model proves insufficient. Use case retained: user adds a constraint mid-run and we don't want to throw away in-flight work.

---

## 4. Change summary

| Behavior | v1? | Mechanism | Touches |
|---|---|---|---|
| Additive → queue | ✅ | follow-up / linked task / pending note | orchestrator logic |
| Corrective → interrupt | ✅ | `worker_interrupts` flag + worker poller flipping in-process flag | plugin + small DB table |
| Steer (inject, no stop) | ❌ future | message queue drained in the agent run loop | Hermes core |
