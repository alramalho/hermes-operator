# Message Bus

How the slack-like project channels are powered. This file documents (1) how Hermes messaging works **today**, (2) the **new requirements** Operator adds, and (3) the **concrete changes**. Referenced from [spec.md](spec.md).

The headline: we do **not** build a new bus from scratch. Hermes already has a platform-adapter framework, a `send_message` tool, persistent sessions, an SSE event stream, and a kanban event table. The "operator channel" is just a **new platform** plugged into that framework.

---

## 1. How Hermes messaging works today

**The gateway is one persistent async process.** It owns a set of *platform adapters* (Telegram, Discord, Slack, …) and an LRU cache of agent instances keyed by session. Inbound flow ([`gateway/run.py:_handle_message`](hermes-agent/gateway/run.py), ~line 6687+):

```
platform adapter receives a message
  → builds a SessionSource (platform, chat_id, user_id, thread_id)
  → session_store.get_or_create_session(source)        # gateway/session.py
  → load transcript from SQLite
  → invoke the cached AIAgent for that session_key      # self._running_agents
  → LLM runs one turn; reply goes back out via the adapter
```

The LLM is invoked **once per inbound message** (state reloaded from the session store each turn) — this is what "long-running agent" means: a persistent process, not a continuously-thinking loop.

**Platform adapters share one interface** — [`gateway/platforms/base.py`](hermes-agent/gateway/platforms/base.py), `BasePlatformAdapter`:

```python
class BasePlatformAdapter(ABC):
    @abstractmethod
    async def connect(self) -> bool: ...
    @abstractmethod
    async def disconnect(self) -> None: ...
    @abstractmethod
    async def send(self, chat_id: str, content: str,
                   reply_to: Optional[str] = None,
                   metadata: Optional[Dict] = None) -> SendResult: ...
```

**Platforms are extensible without forking.** The `Platform` enum ([`gateway/config.py`](hermes-agent/gateway/config.py)) creates dynamic members via `_missing_()` for unknown names, and plugin platforms register a `PlatformEntry` (with an optional `standalone_sender_fn` and `max_message_length`) in the `platform_registry`. Discord is itself a plugin (`plugins/platforms/discord/adapter.py`).

**Agents send out via the `send_message` tool** ([`tools/send_message_tool.py`](hermes-agent/tools/send_message_tool.py)). The agent calls `send_message(target="<platform>:<chat_id>[:<thread_id>]", message=...)`. Routing ([`_send_via_adapter`, line 469](hermes-agent/tools/send_message_tool.py)):

1. If the gateway is in-process, use the live adapter: `runner.adapters.get(platform).send(...)`.
2. Else (out-of-process, e.g. a spawned worker or cron), use the plugin's registered `standalone_sender_fn`.
3. Else, error.

This matters: **a spawned worker is a separate process from the gateway**, so worker→channel posts go through path (2) — the plugin must register a `standalone_sender_fn`.

**Live output is already streamed over SSE** — [`gateway/platforms/api_server.py`](hermes-agent/gateway/platforms/api_server.py): `POST /v1/runs` returns a `run_id`, `GET /v1/runs/{run_id}/events` is an SSE stream of lifecycle events (text deltas, tool calls, completion). This is the basis for UI push.

**The kanban board already has an event + notification substrate** — [`hermes_cli/kanban_db.py`](hermes-agent/hermes_cli/kanban_db.py):
- `task_events(id, task_id, run_id, kind, payload, created_at)` — every status transition is appended.
- `kanban_notify_subs(task_id, platform, chat_id, thread_id, …, last_event_id)` — subscriptions with a cursor.
- `claim_unseen_events_for_sub(...)` — atomically reads new events past the cursor and advances it.

So a "task status → channel" feed is mostly a **poller over existing tables**, not new plumbing.

---

## 2. New requirements

1. **One channel per project** (slack-like), human ↔ orchestrator, project-scoped history.
2. **Worker → channel** progress posts (the Dev/Marketer/assistant posting "done X, starting Y").
3. **Live status feed**: kanban transitions appear in the channel automatically.
4. **UI push**: the web UI updates in real time.
5. **Human ↔ profile tunnel**: a direct chat with a specific profile (e.g. Dev), bypassing the orchestrator, channel-only (no Telegram), auto-closing after ~1h idle.

---

## 3. Design & concrete changes

### 3.1 The operator channel = a new platform adapter
Implement `plugins/platforms/operator/adapter.py` as a `BasePlatformAdapter` and register a `PlatformEntry` with a `standalone_sender_fn`.

- **Outbound** (`send`): POST to the operator backend, which persists the message (operator SQLite) and pushes it to the UI over SSE. `chat_id` == project/channel id; `thread_id` optional (for tunnels).
- **Inbound**: when the user types in the UI, the operator backend feeds the gateway message handler a `SessionSource(platform="operator", chat_id=<project>, user_id=<user>, thread_id=<optional profile/tunnel>)` — exactly like Telegram delivering a message. The orchestrator (the agent bound to that session) replies via its own `send`.

No change to the gateway core — it already dispatches by adapter.

### 3.2 Worker → channel posting (reuse `send_message`)
Workers already get rich env at spawn ([`kanban_db.py:_default_spawn`, ~5655](hermes-agent/hermes_cli/kanban_db.py)): `HERMES_KANBAN_BOARD`, `HERMES_KANBAN_TASK`, `HERMES_KANBAN_RUN_ID`. They need to know **which channel** to post to. Cleanest: the dispatcher stamps the channel onto the task at creation, and the worker reads it.

```diff
# operator: when the orchestrator creates a task, attach the channel
  kanban_create(
      title=...,
      assignee="dev",
+     metadata={"operator_channel": channel_id},   # already a free-form dict on the task
  )
```

The worker then posts with the **existing** tool — no change to `send_message`:

```python
# inside the worker (its SOUL/profile instructs it to post progress)
send_message(target=f"operator:{operator_channel}", message="Pushed feature X, running tests…")
```

For this to work out-of-process, register the operator platform's `standalone_sender_fn` (same hook Discord uses) so [`_send_via_adapter`](hermes-agent/tools/send_message_tool.py) path (2) resolves:

```python
# plugins/platforms/operator/__init__.py
platform_registry.register(PlatformEntry(
    name="operator",
    standalone_sender_fn=_operator_standalone_send,   # POSTs to operator backend
    max_message_length=0,
))
```

### 3.3 Status feed (poll the existing event table)
A gateway background watcher polls `task_events` via `claim_unseen_events_for_sub()` and posts formatted transitions to the operator channel (and, per the spec's noise rules, only user-actionable ones to Telegram). On task creation the operator adds a notify-sub:

```python
# when a task is created
add_notify_sub(task_id, platform="operator", chat_id=channel_id)
# watcher loop (every ~2s)
_, _, events = claim_unseen_events_for_sub(conn, task_id=tid, platform="operator", chat_id=channel_id)
for e in events:
    operator_adapter.send(chat_id=channel_id, content=render(e))
```

### 3.4 UI push
The UI subscribes over **SSE** (existing pattern in `api_server.py`). The operator backend emits a channel-scoped event stream; the React app renders messages + status events as they arrive. (SSE chosen over websockets because the infra already exists and the flow is server→client.)

### 3.5 Human ↔ profile tunnel
A tunnel is just a **separate session lineage** for the same user+channel, bound to a specific profile instead of the orchestrator. The minimal, backwards-compatible change is an **optional `profile` segment in the session key** ([`gateway/session.py:build_session_key`, ~600](hermes-agent/gateway/session.py)):

```diff
 def build_session_key(source, group_sessions_per_user=True,
-                      thread_sessions_per_user=False):
+                      thread_sessions_per_user=False, profile=None):
     ...
     key_parts = ["agent:main", platform, source.chat_type, ...]
+    if profile:
+        key_parts.append(f"profile:{profile}")
     return ":".join(key_parts)
```

Absent `profile` → byte-identical key to today (no migration; keys are opaque strings in the session store). With `profile="dev"` → a distinct session that loads the Dev profile. The operator opens a tunnel by routing the user's messages to that keyed session, and closes it after 1h idle (timeouts are already configurable). Callsites to thread the param through: [`gateway/run.py` ~2162, ~2345](hermes-agent/gateway/run.py).

This is the one change that touches Hermes core, so it goes **upstream as a PR** (the repo is forked already). `CONTRIBUTING.md` wants Conventional Commits + tests under `tests/gateway/test_session.py` + cross-platform; the change is non-breaking and self-contained, so it's a realistic PR (`feat(gateway): optional profile segment in session keys`).

---

## 4. Change summary

| Component | Today | Operator change | Touches |
|---|---|---|---|
| Project channel | platform adapters (telegram/discord/…) | **new** `operator` adapter (plugin) | plugin only |
| UI push | SSE for `/v1/runs` | reuse SSE, channel-scoped | operator backend |
| Worker → channel | `send_message` + plugin `standalone_sender_fn` | register operator sender; dispatcher stamps `task.metadata.operator_channel` | plugin + dispatcher call-site |
| Status feed | `task_events` + `kanban_notify_subs` + `claim_unseen_events_for_sub` | watcher polls + posts | operator backend |
| Inbound routing | `SessionSource` → session → agent | feed `platform="operator"` source | plugin only |
| Profile tunnel | session key has no profile segment | **optional** `profile` segment (backwards-compatible) | Hermes core → upstream PR |

**Net new vs reused:** new = operator adapter + operator backend (SQLite + SSE) + status-feed watcher + dispatcher metadata stamp. Reused = `send_message`, sessions, SSE, `task_events`/notify-subs. Core change = one (optional session-key segment, upstream PR).
