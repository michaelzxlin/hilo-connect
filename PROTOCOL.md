# Hilo Connector Protocol — v0 (draft)

**Status:** v0 draft, 2026-07-05 (additive: threaded-reply semantics, 2026-07-12).
This document will be published openly (decision D8):
the protocol is public so any agent harness can plug into a Hilo node without our
involvement; the product stays closed. Wire format stability rules are in §7.

**One line:** the minimum an agent must expose to be *manageable* — four surfaces,
adopted incrementally as **capability levels L0→L3**. Everything is deliberately
low-tech: append-only JSONL files and file-drop conventions on the same machine,
because any harness — from a cron script to a full IDE agent — can implement those.

---

## 1. Model

A **node** (the `hilo-node` daemon) manages the agents on one machine. Each agent
exposes up to four surfaces to the node:

| Surface | Direction | What it carries |
|---|---|---|
| **Events** | agent → node | facts about work: jobs, incidents, decision requests |
| **Inbox** | node → agent | operator messages, decision results, policy updates |
| **State** | node reads | brain/memory files, reports, health signals |
| **Lifecycle** | node acts | spawn turns, watchdog, restart |

An adapter declares which surfaces it implements — its **capability level**:

- **L0 — observed.** Events only. Anything that can append a JSON line qualifies.
  Buys: the employee file, incident tracking, heartbeat/dead-man detection.
- **L1 — managed.** + Inbox. Operator desk messages and decision results reach the
  agent. Buys: tickets, escalation round-trips, correction propagation.
- **L2 — operated.** + State reads and node-spawned turns. Buys: live workspace
  conversations, brain reviews, memory curation, hygiene lint on its files.
- **L3 — owned.** + Lifecycle. Buys: watchdogs, auto-respawn, session management,
  the live channel (the agent's own session hears workspace messages).

## 2. Events (agent → node) — L0

Append one JSON object per line to the agent's **outbox**:

```
<agent-home>/events/outbox.jsonl
```

```json
{"ts": "2026-07-05T00:40:12+02:00", "agent": "mary", "type": "incident",
 "payload": {"kind": "channel_dead", "detail": "poller pid gone"}}
```

- `ts` — ISO-8601 with timezone. `agent` — roster name. `type` — see below.
  `payload` — object; unknown keys are preserved.
- **Delivery: at-least-once.** The node tails byte offsets and dedups on insert;
  truncation/rotation is detected (offset > size ⇒ re-read from 0). Writers must
  append with a single O_APPEND write per line (atomic on POSIX).
- Reference emitter: `connector/emit_event.sh` (shell, wraps malformed payloads,
  truncates oversized ones, never fails the caller).

**v0 event types** (in production today): `job_completed`, `incident`
(payload.kind + detail), `needs_decision` (item_id, title, source_report),
`counterpart_action` (who, what — the training log), `decision_result` (node-
emitted receipt). **Reserved names** (documented intent, not yet consumed):
`mission_progress`, `artifact_produced`, `heartbeat`, `cost`.

## 3. Inbox (node → agent) — L1

Two complementary drops, both written by the node at delivery time (no LLM in the
path):

1. **Machine inbox** — append-only JSONL the harness can poll:
   `<agent-home>/inbox/decisions.jsonl`
   carrying `decision_result` entries (operator messages, approve/reject results),
   each pointing at its human-readable twin.
2. **Handoff note** — a markdown file the agent reads like any work item:
   `<agent-home>/handoffs/<date>-hilo-message-<slug>-from-<operator>.md`

An L1 adapter's only obligation: the agent eventually reads its inbox/handoffs at
the start of a working session and acts per its own conventions.

## 4. State (node reads) — L2

The node reads, never writes, the agent's tree:

- **Identity/brain:** `CLAUDE.md` (or harness equivalent) — rendered as commentable
  sections in the employee file.
- **Memory index:** `projects/<slug>/memory/MEMORY.md` (+ `MEMORY_*.md` volumes),
  byte-budget-linted.
- **Work journal:** `reports/*.md` — parsed into reviews; their Tier-2 blocks become
  pending operator decisions. `memory/daily/*.md` — the track record.
- **Health:** harness-specific liveness signals (reference: tmux session presence +
  channel poller pid). Adapters declare their own "alive" probe.

**Turns:** an L2 adapter provides a **turn command** — a headless one-shot the node
invokes to have the agent answer a conversation message with full context
(reference: `claude -p` via `workspace/agent_turn.sh`, timeout-guarded, output
parsed for reply + produced artifacts). `HILO_TURN_CMD` overrides it for drills.

## 5. Lifecycle (node acts) — L3

- **Watchdog:** periodic liveness probe + respawn (reference:
  `agent_watchdog.sh` under launchd; respawn emits an `incident` of kind `respawn`
  which the node records as auto-healed).
- **Restart:** operator-triggered restart from the roster (emits `manual_restart`).
- **Live channel:** the agent's own session hears workspace messages via a channel
  plugin (reference: `channel/channel_server.py`, an MCP server polling the org
  store) — replies post back as the agent, comment-reply blocks thread into
  document margins.
- **Threaded replies:** workspace conversations carry Slack-model threads. A reply
  posted as the agent — including the connected-agent API's `POST /api/agent/reply`
  (this repo's CLI: `hilo message send --thread/-t <message_id> …`) — MAY carry an
  optional `thread_root_id`: the id of **any message in the target thread**. The
  node normalizes it server-side to the thread's public root, so callers need not
  distinguish a root from a reply. An anchor that cannot host a public thread (a
  private reminder, a deleted message, an id from another conversation) is dropped
  and the reply posts to the main conversation flow instead — never an error. A
  threaded reply is thread-only: it renders in the thread panel and does not bump
  the conversation's recency or unread state.

## 6. Transports

- **v0 (normative):** append-only JSONL files + file-drop conventions, same
  machine, POSIX permissions as the security boundary. The node runs as the same
  user (or a user with read access to agent homes).
- **v0.1 (reserved):** a local UNIX socket on the node offering the same four
  surfaces as RPC for harnesses that prefer push over files, and for multi-machine
  worker nodes relaying over the gateway tunnel. Not yet implemented; the file
  contract stays supported indefinitely.

## 7. Versioning & compatibility

- Optional `protocol_version` field on every event (absent ⇒ `"0"`).
- Within a major version: **additive only** — new event types, new payload keys.
  Consumers MUST ignore unknown types/keys; producers MUST NOT change the meaning
  of existing fields.
- Reserved names become normative only when a consumer ships.
- Breaking changes ⇒ new major version; nodes support N and N-1 for a full
  deprecation cycle.

## 8. Security notes

- Events and inbox lines MUST NOT carry secrets (tokens, keys, passwords) — the
  node's store renders them in an operator UI and may sync excerpts to documents.
- The trust boundary in v0 is the machine account: file permissions, not crypto.
  Cross-machine transport (worker nodes) rides the node↔gateway tunnel and is out
  of scope for this document.
- The node is auditable: every consumed event lands in the org store's `events`
  table verbatim.

## 9. Adapters

### 9.1 Claude Code — the L3 reference (certified, v1)

| Surface | Implementation |
|---|---|
| Events | `emit_event.sh` sourced by job wrappers → `events/outbox.jsonl` |
| Inbox | `handoffs/*.md` + `inbox/decisions.jsonl`, read at session start |
| State | `CLAUDE.md`, `memory/MEMORY.md`, `reports/`, `memory/daily/` |
| Turns | `claude -p` one-shots with the agent's config dir (`agent_turn.sh`) |
| Lifecycle | tmux session + watchdog respawn + MCP channel plugin |

This is exactly how the first production fleet (6 agents, one node) runs today.

### 9.2 Certification wave 2 — Codex, OpenClaw, Hermes (decision D9)

Expected entry levels, to be validated during certification — treat as targets,
not claims:

- **Codex:** headless one-shot runs exist → turn command feasible; file
  conventions map directly. Target **L2**, L3 once a session/watchdog story is
  proven.
- **OpenClaw:** long-running gateway daemon with its own session management →
  events/inbox via a small sidecar; lifecycle likely via its own API. Target
  **L1 → L3** progressively.
- **Hermes:** open-weights harness; start at **L0/L1** through the generic JSONL
  contract (a sidecar script is enough), grow with what its runtime exposes.

The certification bar for any adapter: a scripted drill proving each claimed
surface against a disposable node home — same style as this repo's `test/drill-*`.

### 9.3 Everything else — the long tail

The contract is deliberately implementable by the client's own agents: point a
coding agent at this document and the target harness, and have it write the
sidecar — that's the supported path, and it's how the installer's agent-install
flow is designed (shipped alongside the node installer). L0 is a few lines of
shell. If it can append JSON to a file, it can be managed.
