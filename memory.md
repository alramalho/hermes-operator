# Memory

How project memory works. Documents (1) Hermes memory **today**, (2) the **new requirements** (three parallel layers), and (3) the **provider-plugin design** that adds the project layer without forking. Referenced from [spec.md](spec.md).

Headline: Hermes already has two memory layers and a **plugin extension point** for memory. We add a third layer (`INITIATIVE.md`) as a plugin — the two existing layers keep working untouched, and all three are live at the same time.

---

## 1. How Hermes memory works today

**Two file-backed layers**, both under `get_memory_dir()` = `HERMES_HOME/memories/` ([`tools/memory_tool.py:56`](hermes-agent/tools/memory_tool.py)):

| File | Purpose | Char limit |
|---|---|---|
| `MEMORY.md` | the agent's own notes — environment facts, conventions, quirks learned | 2,200 |
| `USER.md` | the user profile — preferences, communication style | 1,375 |

The single `memory` tool mutates them by `target` ([`_path_for`, line 247](hermes-agent/tools/memory_tool.py)):

```python
@staticmethod
def _path_for(target: str) -> Path:
    mem_dir = get_memory_dir()
    if target == "user":
        return mem_dir / "USER.md"
    return mem_dir / "MEMORY.md"
```

**Loading = frozen snapshot at session start.** `load_from_disk()` reads both files once and bakes them into the system prompt ([`memory_tool.py:133`](hermes-agent/tools/memory_tool.py)). Mid-session writes go to disk immediately but **don't change the running prompt** (prefix-cache stability); the snapshot refreshes on compression or next session.

**Writes are already concurrency-safe.** Every `add/replace/remove` takes an `fcntl` lock, **re-reads under the lock**, writes atomically (`os.replace`), and refuses on external drift ([`memory_tool.py:298+`, `516+`](hermes-agent/tools/memory_tool.py)) — i.e. read-before-write is built in.

**Memory is pluggable.** There is a `MemoryProvider` ABC ([`agent/memory_provider.py:42`](hermes-agent/agent/memory_provider.py)) discovered/registered from `plugins/memory/<name>/`. Key methods:

```python
class MemoryProvider(ABC):
    def is_available(self) -> bool: ...
    def initialize(self, session_id, **kwargs): ...      # gets hermes_home, platform, agent_identity, ...
    def system_prompt_block(self) -> str: ...            # static contribution to the prompt
    def prefetch(self, query, *, session_id) -> str: ... # per-turn recall (fresh each turn)
    def get_tool_schemas(self) -> list: ...              # extra tools this provider exposes
    def handle_tool_call(self, tool_name, args, **kwargs): ...
    def on_session_end(self, messages): ...
    def on_memory_write(self, action, target, content, metadata): ...  # mirror built-in writes
```

**Constraint:** the `MemoryManager` allows the built-in store **plus exactly one external provider** ([`agent/memory_manager.py:268`](hermes-agent/agent/memory_manager.py)). Our operator provider is that one external provider.

---

## 2. New requirements

Three layers, **all live simultaneously** for any agent in a project:

| Layer | File | Scope | Who writes |
|---|---|---|---|
| User | `USER.md` | **global** — one per user, shared across every project/agent | any agent (e.g. Marketer learns user's age) |
| Agent | `MEMORY.md` | **per-profile** — a role's own cross-project notes | that profile |
| Initiative | `INITIATIVE.md` | **per-project** — facts/state/decisions for one initiative | any of the project's agents + orchestrator |

Scoping signal — *which* initiative an `initiative` write belongs to:
- **Worker**: `HERMES_KANBAN_BOARD` env var (set by the dispatcher per spawn).
- **Orchestrator**: the current turn's session→initiative mapping (it's multi-project in one process).

---

## 3. Design — operator memory provider (no fork)

A plugin at `plugins/memory/operator/` implements `MemoryProvider` and **only** owns the new `INITIATIVE.md` layer. `USER.md` and `MEMORY.md` stay with the built-in store, unchanged.

**Project-dir resolution:**
```python
def _initiative_dir(self):
    board = os.getenv("HERMES_KANBAN_BOARD")          # worker path
    if not board:
        board = self._session_initiative()            # orchestrator path (session→initiative)
    return OPERATOR_HOME / "projects" / board
```

**Layer is injected in parallel with the built-in block:**
- Workers (single-project, frozen ok): `system_prompt_block()` returns the project's `INITIATIVE.md`, rendered next to the built-in `MEMORY.md`/`USER.md` block at session start.
- Orchestrator (multi-project, must be fresh): `prefetch(query, session_id)` returns the *current* initiative's `INITIATIVE.md` each turn — so switching projects between turns swaps the initiative context without rebuilding the frozen prompt.

**Writes go through a dedicated tool** the provider exposes via `get_tool_schemas()`:
```
initiative_memory(action=add|replace|remove, content=...)   # always resolves to the project file
```
The built-in `memory` tool keeps `target=memory|user`. So an agent has both: `memory` for personal/user notes, `initiative_memory` for project facts — three layers, two tools, no ambiguity.

**Reusing the safe-write mechanics.** The bounded store's locking/drift logic lives in `MemoryStore`, but its path is hardwired to `get_memory_dir()`. One small wrinkle to pick:
- (a) **tiny refactor** (upstream-friendly): let `MemoryStore` accept a base dir, then the provider instantiates one pointed at `_initiative_dir()` and gets locking/limits/snapshot for free; or
- (b) **reimplement** a ~100-line bounded store in the plugin reusing the same `fcntl` lock helper.
Recommend (a) — it's a non-breaking parameter and keeps one implementation of the safe-write path.

**Char limit:** give `INITIATIVE.md` its own budget (suggest ~2,500–3,000 chars) — project state is denser than an agent's personal notes.

---

## 4. How the three layers render together

At an agent turn the system prompt contains, in parallel:
```
## Memory (MEMORY.md)        ← built-in store, per-profile
## User (USER.md)            ← built-in store, global
## Initiative (INITIATIVE.md)← operator provider, per-project (frozen for workers / per-turn for orchestrator)
```
Nothing is dropped or overwritten; the provider is purely additive.

---

## 5. Change summary

| Concern | Today | Operator change | Touches |
|---|---|---|---|
| User layer | `USER.md`, global path | unchanged (keep global) | none |
| Agent layer | `MEMORY.md`, per-profile | unchanged | none |
| Initiative layer | — | **new** `INITIATIVE.md` via memory provider plugin | plugin only |
| Project scoping | n/a | env (`HERMES_KANBAN_BOARD`) or session→initiative | plugin only |
| Safe writes | fcntl + reload + atomic + drift | reuse; let `MemoryStore` accept a base dir | tiny upstream refactor (optional) |
| Provider slot | builtin + 1 external | operator provider takes the external slot | config: `memory.provider: operator` |

**Constraint to remember:** only one external memory provider at a time — if a future provider (e.g. Honcho) is wanted alongside, they'd have to be merged into one provider.
