# Hermes Operator

# Problem

Hermes is powerful but hard to operate across many simultaneous projects.
Projects become stale because the user cannot quickly see project state,
agent state, pending tasks, blockers, and required approvals.

Telegram is too noisy as the main interface. Kanban exists but does not
provide enough project-level visibility. Memory exists but project-level
memory boundaries depend on provider behavior. With crons and multiple projects, it becomes nearly impossible to manage several ongoing tasks.
On the other hand, native kanban and dashboards often end up with too many boards created and forgotten, tasks created and forgotten, and items stuck on blocked for days.

# Goal

Create a lightweight wrapper called 'hermes operator' on top of Hermes that lets one user manage
many commercial & personal projects, and ensures they keep progressing.

# Solution

Two-fold: a UI for managing + an opinionated Hermes backend integration for cron/task management. Each project runs a small set of Hermes profiles — an **Orchestrator** (the single human↔agents bridge) plus worker agents (commercial → Dev + Marketer; personal → a general assistant) — working a per-project kanban board. Workers do the work; the orchestrator routes.

## UI
A slack like ui that has one channel per project, allows user to see state of the project simply by looking at what agents saying. One project can have many agents that communicate.

Stack: Vite + React SPA, web-first for speed and to validate the UI fast; wrap later (Capacitor for iOS) once it proves out. Single user — each person self-hosts for themselves. The operator runs as one fixed user identity (a single `user_id`), so session keys and tunnels are stable.

Agents communicate through an operator-owned message bus (not native Hermes). Channels are project-scoped; messages persist in the operator DB. Any new message — human or agent — triggers the orchestrator. Worker agents post progress to the channel and can be @-tagged; tagging a profile that isn't currently running (like dev) should open a temporary (1h closed on inactivity) tunnel from human -- profile. This is a 'orchestrator' bypass when the human wants more control, and exchange should only be visible in the intiative channel (no telegram)

> Note: We will be using sqlite for the operator db

Message bus design — how Hermes messaging works today, the new requirements, and the exact changes — lives in **[message_bus.md](message_bus.md)**. Summary: UI push rides the gateway's existing **SSE**; the operator channel is a **new Hermes platform adapter**; workers post back with the existing `send_message` tool targeting `operator:<channel_id>` (channel id carried in `task.metadata`); the live status feed is driven by polling the existing `task_events` table.


On the left:
- channels. One per project. Projects can be commercial and non commercial.

On the center:
- On top: The project 'header'. It shoudl be basic, for now only project name + icon. The header is **read-only**; clicking the name/icon opens the project in the right pane. Project-level configuration is edited there (see Project header configuration below).
- Main communication UI (slack like). Here we can see user's messages and agents messages. Agents with different profiles will have different names.

On the right:
This is a panel that is only shown if you click either the agent or the project. It expands to show both agent or project tasks.

![Right pane showing an agent's tasks](assets/mockup_agents.png)
![Right pane showing the project's tasks](assets/mockup_projects.png)

### Project header configuration
Opened by clicking the project name/icon (the header itself is read-only). Holds the project-level config: name, icon, archetype (commercial/personal), the enabled agents and their per-agent model override, and the project's artifacts. Editing here is what the orchestrator otherwise does conversationally during setup.

## Project Tasks & Integration with Hermes Kanban
Hermes already supports projects, with boards, and tenants. 
We will leverage the internal kanban board in an opinionated manner. We will totally hijack the board control. And we will have one board per project (also referred to as initiative). 

The goal of this is to be the backbone of task status management, but it must naturally integrate with the rest of the application.

We adopt Hermes' 9 native statuses as-is (triage, todo, scheduled, ready, running, blocked, review, done, archived) — custom statuses would require a fork, not worth it. We integrate with the rest of the app via the board's event stream (tailing `task_events`). Native dispatch already spawns `hermes -p <profile>` per `ready`+assigned task and filters by assignee, so **assignee == profile** gives us a profile-scoped task queue for free — no always-on worker pool needed.

## Setting up the project
This is arguably the most important part, because in the setup is where we will answer the most basic questions about the project. Understand at what level it is (idea, already has material like code, website), what ARTIFACTS does it have, and setup everything that we need to maintain it. This will not be immediate, this will be a conversation. Our setup agent should have access to tools to manage initiatives/projects. It should also define which agents, and the task lifecycle.

The orchestrator owns setup (it is the one that manages initiatives). Setup must be idempotent, keyed by project slug: re-running reconciles profiles / board / artifacts instead of duplicating.

**Creation trigger.** User clicks *New Project* in the UI → picks an **archetype** (commercial / personal) → the orchestrator opens a setup conversation in the new project channel. The archetype seeds defaults: *commercial* → dev + marketer agents; *personal* → a single general-assistant agent. (QA is a future role; for now QA work merges into Dev.) The setup conversation then probes maturity (idea / has-code / live), captures artifacts (librarian role), and provisions the board, the role profiles, and the heartbeat cron. All keyed by slug, so re-running is safe.

### Artifacts
By artifacts i mean any external valuable resource that is worth explictly noting. This is important because many of the projects I will be using this are coding projects, so they have for example code in github and code running in a VPS. This is important information because in a web dev / app project there will be at least one 'Dev' agent that needs everything to be able to deliver.

**Framing.** Artifacts are the project's **pinned, typed, guaranteed-injected registry of the external resources agents must have to operate** — the deliberate counterpart to evictable memory. Where `INITIATIVE.md` holds the project's *narrative* (bounded, agent-curated, allowed to forget), artifacts hold its *load-bearing nouns*: always present (never evicted/compressed), typed (so they're machine-actionable), and secret-bearing (which memory must never be). They live in a `project_artifacts` table in the operator DB: `{project, kind, ref, notes, secret_ref?, state, created_at, updated_at}`, `kind` ∈ {github_repo, vps, domain, db, social_handle, api_key, …}.

**Lifecycle.**
- *Create* — captured during the setup conversation (orchestrator has an artifact CRUD tool) and editable later via orchestrator or the UI project header.
- *Consume* — non-secret metadata (repo URL, host, domain) is injected into task context / `INITIATIVE.md`; secrets are resolved at runtime by the dispatcher and injected as **env vars at worker spawn** (so `git`/`ssh` just work), never stored raw.
- *Update* — orchestrator/UI edits; reflected on the next task.
- *State* — `active` → (project archived) retained read-only → (project deleted) purged incl. secret references.
- *Validation (future)* — heartbeat can ping artifacts (repo reachable? VPS up?) and flag staleness.

**Nothing is hardcoded — requirements are discovered.** Two non-hardcoded paths populate the registry:
- *Just-in-time (safety net)*: a worker that hits a missing requirement does not fail — it calls native `kanban_block` with a clear free-form reason ("cannot clone repo — need the GitHub repo URL and push access"). The block bubbles up the `task_events` stream to the orchestrator, which **parses** the reason into a structured `{category, needs}` (see Task lifecycle) and either self-resolves (creates/links the artifact) or escalates to the user as a `question`; on resolution it unblocks and the dispatcher re-spawns the worker with the artifact now available.
- *Setup-time proposal*: from what you describe, the orchestrator *suggests* likely requirements ("a web app with code → Dev will want a repo + a deploy target?") — a conversational proposal, never a rule.

The registry therefore grows into the exact set of resources the project's agents have proven to need.

**Orchestrator is the librarian.** It owns persisting load-bearing info so it is never lost: *reactively* creating/linking artifacts when a requirement block bubbles up, and *proactively* promoting durable typed facts it sees in conversation ("the repo is github.com/x/y") into artifacts. Routing rule: typed/load-bearing facts → artifacts; narrative facts → `INITIATIVE.md`.

**Secrets backend is pluggable** (must run self-hosted on a VPS, so no hardcoding to a desktop keychain): an artifact stores a `secret_ref` (a name), and a `SecretsBackend` interface resolves it — OS keychain (macOS/libsecret) for local, an encrypted file / `.env` / external vault for VPS. The dispatcher resolves refs → env vars at spawn.

### Worker workspace
Hermes runs on the host machine and its file/terminal tools operate on the **real filesystem** (scoped to the process cwd, with path-traversal guards — not a hard sandbox). Kanban tasks carry `workspace_kind` ∈ {`scratch`, `dir`, `worktree`} and a `workspace_path`; the dispatcher sets `HERMES_KANBAN_WORKSPACE` and the worker `cd`s into it, after which `git`/`ls`/`grep`/edits operate there naturally.

The **workspace is the project's working root, not a repo**: `workspace_kind="dir"`, `workspace_path` = a per-project dir. **Repos are artifacts** — a project can have N `github_repo` artifacts (frontend, backend, …), each checked out **as a subdir of the workspace** by convention, with its absolute path recorded on the artifact. Non-repo material (deploy scripts, generated assets, scratch) lives in the workspace alongside them. v1: the dispatcher always drops Dev at the **project root** and the task text points it at the relevant repo(s) — since tools take absolute paths and aren't sandboxed, Dev navigates between checkouts freely (per-task repo pinning is a later optimization). For each repo the user names at setup, the orchestrator creates a `github_repo` artifact and ensures it's cloned into the workspace; idea-stage projects start empty and accrue repos as they're created. (`worktree` kind — per-branch parallel work — is available later; not v1.) Caveat: the workspace is a coordination surface, not a security boundary — keeping a worker inside its repos is prompt/trust, not enforcement.

## Agent profiles
This should not be hardcoded, but a default can be provided. Hermes provides profiles, so it would be good to integrate natively with them. Perhaps via a plugin.
Each project has its own set of agents, each agent being one profile. For now we won't support many agents with the same profile.

Profiles are **global** (one per role: orchestrator, dev, marketer) and reused across all projects — config, toolsets and `SOUL.md` identity live once in the profile. The **project-local component is resolved at runtime**: a spawned worker reads `HERMES_KANBAN_BOARD` (set by the dispatcher) to scope its memory and channel posting to the right project. Per-project model and any project-specific guidance are carried on the task / project record, not the profile. Operator ships the role profiles; users can add/import their own.

Projects come in two archetypes, each starting with a minimal role set:
- **Commercial** (e.g. tracking.so — a web/iOS app at ~10€MRR with code + a VPS, to scale): orchestrator + **Dev / CTO** + **Marketer / CEO**. QA is a future role — merges into Dev for now.
- **Personal**: orchestrator + a single **general assistant** for life-admin automation (reply email, check Vinted / Kleinanzeigen listings, calendar, web research), mostly cron-driven recurring chores. It still sits behind the orchestrator and uses a board — mild overkill for one profile, but we avoid special-casing in v1 since the dispatch/routing already exists for the commercial path.

The **Orchestrator** is a special profile with awareness of all initiatives; it is the sole bridge between the human and the worker agents. This avoids race conditions — only the orchestrator reads the user's messages, and it decides to clarify, do nothing, or create task(s). If a request is underspecified (unclear initiative, vague problem) it asks; if well-specified it creates kanban task(s) on the right board, assigned to the right agent. One request can yield several chained tasks (e.g. a new feature → a code task, then an evaluation task).

Each agent (profile) has a subset of the whole 90+ skills and 100+ tools provided by the hermes agent. Each profile is basically a running hermes instance (similar to existing profiles).

Tool/skill split per role:
- Orchestrator: kanban (orchestrator mode), send_message, delegate, memory, todo, approval, interrupt. No browser/terminal/file/codegen/generation. Its prompt must be assertive — zero drift, every task correctly placed on the board — reinforced via both the profile prompt and tool descriptions.
- Dev/CTO: terminal, file_*, browser_*, code_execution, web; github/software-development/devops/mlops/data-science skills; kanban (worker).
- Marketer/CEO: browser, web, image/video gen, send_message; social-media/email/creative/research skills; kanban (worker). No terminal/codegen.
- General assistant (personal): browser/web + research; email (`email/himalaya` IMAP read+reply, or `productivity/google-workspace`) + calendar (`google-workspace`) + send_message; memory, todo, cronjob, maps, notion/obsidian, mcp; kanban (worker); file + terminal behind approval. No codegen/devops/github, no marketing/social, no media generation.

This tools and skills should also be configurable (All loadable skills and tools listed, and as easy as clicking a 'switch')

Default model: gpt-5.5, medium effort, all roles — overridable per (project, agent) in the UI agent page. Because profiles are global, the override is stored on the operator project record and carried per task via kanban's native `model_override`. Available models are loaded from Hermes providers: a small operator endpoint aggregates each provider's `fetch_models()` (cached) since Hermes has no single list-all-models endpoint. Per-profile/per-task tool-usage analytics is deferred (see Future optimizations).

> Note! Board events (task created / status change) stream in full to the project channel UI. Telegram only carries orchestrator clarifications, user-actionable blocks, and orchestrator→user messages — not every status change (avoids the original noise problem).


### Agent to task orchestration 
This orchestration is subtle and depends on 
- Initiative definition (goals, artifacts, constraints, requirements)
- Orchestrator
- Agent profiles

That said, the lifecycle of a task will need to be properly documented, in order to make sure the orchestrator knows how to best route, and reroute!

Chained tasks (e.g. Dev → QA) use native kanban parent/child links; Hermes promotes a child to `ready` only once its parents are `done`. Wiring this is the orchestrator's responsibility — it sets the links and packs each task with all context the spawned worker will need, since workers don't see the live channel. Handoff artifacts (Dev output → QA input) pass via kanban comments / project memory.

## Task lifecycle
Tasks live on the kanban board and use its native statuses (a block uses native `blocked`). Status changes must be clearly visible in the UI (via the `task_events` stream) and notify the orchestrator, so it can keep the user informed and head off staleness.

Blocks carry a **category** so the UI can distinguish them: `question` (needs a human answer/decision → ❓) vs `requirement` (needs an artifact/access → ⚠️). We do not pollute `kanban_block` (it stays native free-text `reason`); instead the **orchestrator parses** the free-form block reason into `{category, needs}` and stores that operator-side, keyed by task_id. The UI reads operator state to render the right card icon; the orchestrator routes requirement blocks to artifact creation. Caveat: the category resolves one orchestrator turn after the block (not instant), and parsing is LLM-judgment (recoverable).

Task card states (right pane): ⚠️ requirement block, ❓ question (answerable inline), ▶️ running/steerable (with a type box), green check done.

![Task card states](assets/tasks.png)

## Consuming messages
The orchestrator is the **sole addressee** of channel messages (see its role above) — workers post to the channel but never consume user input directly. The user reaches a worker only by the orchestrator queueing it a task, or via the direct tunnel.

The orchestrator is a **single long-running process consuming one queue** of inbound items across all projects (channel messages, agent messages, Telegram, heartbeats). Mechanically, "long-running" = a persistent gateway process that re-invokes the orchestrator LLM **once per inbound item**, with state reloaded from the session store each turn (it is not a continuously-thinking loop). Each item is handled with context scoped to its **initiative**, so a 20-message clarification thread on project A doesn't pollute project B — history is keyed per (user, initiative). A bare "you there?" with no initiative is not a special path — the orchestrator just answers/clarifies.

Workers may be shown recent channel history as **read-only** context but cannot post back, except when directly @-mentioned (the tunnel exception). When a new message targets a worker, the orchestrator classifies it: **additive** ("after that, also do Y") → queued as a follow-up task / pending, delivered when the worker is idle; **corrective** ("no, use credential X instead") → interrupts the running worker and re-tasks it. **v1 ships queue + interrupt**; only **steer** (inject without stopping) is deferred. Mechanism in **[worker_control.md](worker_control.md)**.

**Interactive controls are orchestrator-brokered.** A worker's channel post is read-only — it can show inline status (e.g. `Blocked`) but carries no buttons. Accept/Reject and inline answers live only on (1) the **orchestrator's** messages and (2) **task cards** in the right pane; clicking one resolves an orchestrator-mediated decision, which then unblocks / re-tasks the worker. A worker surfaces a needed decision by **blocking** (`question`/`requirement`) — the orchestrator brokers it (resolves itself or relays to the human). The UI's `clarify`/`approval` resolvers are bound to the **orchestrator↔human** leg, preserving the single front door. Worker-originated dangerous-command approvals are left to **Hermes' native approval policy** (smart auto-approve of low-risk commands, configurable) — the operator doesn't intervene, keeping workers from blocking needlessly. The only exception to the single front door is the explicit human↔profile **tunnel**, where direct interaction with a worker is the intent.

## Project memory
Three layers, all live at once: `USER.md` (global user profile), `MEMORY.md` (per-profile agent notes — kept as-is), and a **new `INITIATIVE.md`** (per-project facts/state). Added via a Hermes **memory provider plugin** (no fork); the two existing layers are untouched. Project scoping comes from `HERMES_KANBAN_BOARD` (workers) or the session→initiative map (orchestrator); the orchestrator gets the right `INITIATIVE.md` injected per turn. Concurrency is already safe natively (lock + read-before-write). Full design — current Hermes memory, the provider plugin, parallel rendering, concrete code — lives in **[memory.md](memory.md)**.

## Staleness
One major point of impact is project staleness. Every project should have an heartbeat cron automatically setup. This heartbeat will be responsible for every 12h verifying whether progress has been made or not. Basic points of interest are recent project scoped message history + project memory + artifacts.

The heartbeat triggers the orchestrator (not a separate evaluator) to assess current state. It's a **cron on the orchestrator profile** (Hermes cron is per-profile); the fired item enters the same orchestrator queue as channel/Telegram messages.

I/O:
- **Input** (the cron prompt, per active project): recent project-scoped message history + task states (counts by status, and how long anything's been `blocked`/stale) + `INITIATIVE.md` + artifacts + time-since-last-progress. The prompt tells it to evaluate and decide an action.
- **Output** (a decision per project — *silence is first-class*): **silent** (healthy progress — the default, emits nothing), **nudge** (all tasks done & user idle → reach out), **act** (create/unblock a task to move things forward), or **archive** (after ~2–3 days of user non-response → status → archived). Any user-facing output goes to the channel + Telegram per the noise rules.

Being allowed to do nothing most of the time is what makes the heartbeat a staleness *guard* rather than a noise source.

## Telegram
Telegram serves as human -- orchestrator communication channel. So will essentially just be the agent replying to clarify, or if a task is created, a default rule based message is sent to the user with a link that opens the UI on the right project, and right task right away (we can use query parameter w/ task ID)

Telegram is **optional**: the operator is fully functional with the UI alone — Telegram is a nice on-the-go add-on. When configured, it centralizes / duplicates all human↔orchestrator comms and is bidirectional — the orchestrator can initiate (clarifications, blocks, staleness nudges). We use one Telegram forum with one topic per project, **auto-created on project creation** (via the Telegram Bot API; the returned `thread_id` is mapped to the project); inbound messages route to the right project via `thread_id` (natively supported). Inbound in the general chat (no topic) is not a special path — the orchestrator handles it as a normal clarifying turn. Orchestrator→user messages appear both in the project channel and Telegram (accepted duplication).

## Future optimizations (not for now)
Deferred to keep v1 simple. Recorded so we don't lose the use cases.

- **Steer a running worker** (mid-run message injection): inject a message into a worker mid-task so it reads before its next step, without restarting. Use case: the user adds a constraint while the worker is already running and we don't want to lose progress. Net-new in Hermes — the agent loop must drain a message queue between tool calls (interrupt today is stop-only). See [worker_control.md](worker_control.md).
- **Task wrap-up learnings**: on task completion the worker reflects and records reusable shortcuts via Hermes' `on_session_end` hook. Use case: remember optimization paths so similar future tasks are faster. Open: which profile decides what to persist, and project `INITIATIVE.md` vs a separate "playbook"; keep distinct from any profile-level self-optimization.
- **Project-scoped profile customization**: per-project guidance that doesn't fit in memory (e.g. specific dev instructions for this codebase). Since profiles are global, this needs a per-(project, role) overlay injected at runtime.
- **Tool-usage analytics**: count which tools each profile uses per task to surface usage patterns over time. Net-new — raw `tool_calls` live in Hermes' `messages` table but there's no rollup; a nightly aggregation job would feed a UI view. Not v1.


