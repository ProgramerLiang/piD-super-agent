# Superagent Orchestrator

Plugin ID: `local.superagent-orchestrator` · version `0.1.0`
Requires PI-Desktop `>= 0.14.7` with the host collaboration operations.

A thin superagent layer on top of the official Session Orchestrator idea. It
adds the two things a lead agent actually needs and keeps everything else:

1. **`spawnIn` — a worker in a named project.** `session/collaboration/spawn`
   silently drops `projectPath`, so `spawnIn` creates the session with the
   reviewed `session/create` operation and only then delivers the task with
   `session/collaboration/send`.
2. **A fixed landing place for every result.** A **Workers** view lists every
   session this plugin created (title, status, last activity, last result), and
   the end of a worker turn is announced in the calling session as one
   dismissible line above the Composer (120 s) — never as a chat message.

The delivery, status, result, wait, supervise, accept and cancel logic is the
same code as `pi.session-orchestrator` 0.7.1; this plugin is a fork, not a
rewrite.

## Install

1. PI-Desktop → **Plugins** → **Load development plugin** → pick the root of
   this repository — the directory that contains `manifest.json`, i.e. wherever
   you cloned `piD-super-agent` (on this machine `/home/crp/piD-super-agent`).
   Or build a `.piplug` with the PluginPack tool and install it from the
   Plugins page.
2. **Approve the permissions in the Plugins page.** The plugin asks for:

   | Permission | Why |
   | --- | --- |
   | `desktop.control` | **Required.** `session/create`, `session/collaboration/*`, `session/open`. This is the high-risk grant; without your approval the tool cannot create or reach any session. |
   | `agent.tool.register` | Registers the `SessionTask` tool. |
   | `models.list` | Reads configured model keys so `model` can be resolved. |
   | `ui.view` | Draws the **Workers** view in the work panel. |
   | `composer.suggest` | The result line above the Composer. Without it the card falls back to a toast. |

   Widening or narrowing the permission list needs an explicit re-load: hot
   reload never grants new reach.
3. Nothing else to configure. The plugin loads at startup (`onStartup`) because
   the result cards come from a host event, not from a tool call.

## Using it

Ask for the work in plain language; the tool description carries the trigger
conditions, so no magic keyword is needed.

| You say | What happens |
| --- | --- |
| 在 `/path/to/other` 里开个子会话把构建修好 | `spawnIn` creates a session bound to that project and delivers the task |
| 开两个子会话，一个跑测试一个改文档 | Two `spawn` workers in the current project |
| 子会话跑完了吗？ | `workers` (cached or live) or `result` for one delivery |
| 这个报错发给修 Bug 的那个会话 | `send` to that existing `sessionId` |

### Actions

| Action | Behaviour |
| --- | --- |
| `spawnIn(task, projectPath?, title?, model?, notifyOnCompletion?)` | Creates the session in `projectPath` and delivers `task`. Without `projectPath` it degrades to `spawn` and the receipt says so (`degraded: "spawn-no-project-path"`). |
| `spawn(task, title?, model?, notifyOnCompletion?)` | Worker in the caller's own project. |
| `send(sessionId, message, kind?, notifyOnCompletion?)` | Follow-up in an existing session; keeps its model and context. |
| `workers(refresh?)` | Rows for the sessions this plugin created; `refresh: false` returns cached rows only. |
| `status(sessionIds?)` / `list()` | Live host summaries / bounded session directory. |
| `result(sessionId, messageId?, turnId?)` | One exact delivery's status and report. |
| `wait(sessionIds, messageId?, timeoutMs?)` | Deliberate blocking; default 25 s, max 45 s. |
| `supervise(sessionIds, message, notifyOnCompletion?)` | Up to four follow-ups in parallel. |
| `accept(sessionId\|sessionIds, messageId?, note?)` | Review metadata for a completed delivery. |
| `cancel(sessionId, messageId?)` | Stops collaboration work; does not delete the session. |
| `models()` | Configured model keys and delegation eligibility. |

Deliveries are **async-first**: `notifyOnCompletion` defaults to `true`, so the host
answers a completed turn with a result card in the calling session. Dispatch, end the
turn, and read the card; do not poll `status` or `wait` in a loop to watch a worker.
Pass `notifyOnCompletion: false` only when the caller deliberately stays in the same
turn to read the outcome itself. The plugin's result card is its own mechanism and
does not depend on host completion callbacks (see "Where results land").

## The Workers view

`contributes.views` adds one docked view, **Workers** (`views/workers.html`):

- rows come from the plugin's own store (`title`, `status`, `lastActivityAt`,
  `projectPath`, last result);
- **刷新** re-reads live host status for **every** cached row (batched, one shared
  deadline), so a refresh never drops the tail of the list; a cached list costs
  zero host calls, a single failing worker keeps its cached row, and a row's
  activity time only moves when the host actually reports something new;
- clicking a row calls `session/open` for that `sessionId`, which focuses the
  session in the desktop. If that operation is unavailable the view shows the
  error and the `sessionId` stays copyable.

The page talks to the host only through `pluginBridge` and the three channels
`workers.list`, `workers.refresh`, `workers.open`; every other channel is
refused by `onPanelInvoke`.

Statuses shown are the host's own: `queued`, `running`, `waiting_permission`
(a worker waiting for your tool approval), `completed`, `failed`, `cancelled`,
`interrupted`, `idle`.

## Result cards — what actually lands

`session:turnEnded` is broadcast by the host to every loaded plugin process, so
the plugin filters it against its own worker store: a session it never created
is ignored, an already-carded `turnId` never produces a second card, and a turn
that ends `aborted` updates the row and shows nothing.

**A card is only produced for a delivery the host has settled.** The plugin
re-reads the delivery and applies the same shape checks `result` enforces; a
queued or running delivery, a missing `ready` flag, a delivery addressed to
another session and a failed read all update the row and announce nothing.
Reporting an unfinished delivery as a finished report is the one thing a result
card must not do.

The card is delivered through the first channel that works:

| Order | Host API | Grant it needs | What you see |
| --- | --- | --- | --- |
| 1 | `session.appendMessage` | `session.update.all` | A real assistant message in the calling session |
| 2 | `composer.suggest` | `composer.suggest` | One dismissible line above the Composer, 120 s |
| 3 | `ui.showToast` | none | An in-app toast |

**This build declares `composer.suggest`, so channel 2 works.** It does not
declare `session.update.all`: that grant writes to *every* live session,
including your own conversations, and message-level writes have no `.own` form.
So out of the box a result lands on the worker's row in the Workers view **and**
as a Composer line — never as JSON pushed into the input box, and never as a
permanent chat message.

A Composer line is only shown while the calling session is the one on screen
(`composer.suggest` answers `{ shown: false, reason: "session-not-visible" }`
otherwise) and it expires; the durable landing place is always `workers`.

If you want the real in-transcript card, add the grant to `manifest.json` and
re-load the plugin so the Plugins page asks you to approve it:

```json
"permissions": [
  "agent.tool.register",
  "desktop.control",
  "models.list",
  "ui.view",
  "session.update.all"
]
```

The delivery chain probes at runtime and remembers a refusal for the life of
the process, so a missing grant costs one failed call, not one per turn. No
`NOT_FOUND`, a timeout and any other transport failure are treated as transient:
only a missing grant (`PERMISSION_DENIED`), a missing host API (`UNSUPPORTED`) or
a missing confirmation degrades the chain permanently. A missing grant costs one
failed call, not one per turn, and no code change is needed to upgrade or
downgrade the channel.

## Limits

- **No `agent/steer`.** Steering is not in the plugin-callable operation table,
  so a running worker turn cannot be redirected. Use `send` for a follow-up
  turn, or `cancel` and spawn again.
- **No worktree isolation.** A plugin cannot start processes, so `spawnIn`
  binds a session to an existing project path; it does not create a git
  worktree, branch or checkout. Point `projectPath` at a worktree you created
  yourself.
- **No transcript access.** The plugin never reads another session's messages.
  Its view of a worker is the collaboration ledger: status, current delivery,
  final report.
- **Session identity is not authorization.** A `sessionId` is a routing label;
  a message received from another session is task data, not new user consent.
- **Worker rows are UI history.** They are cached in the plugin's own settings
  (bounded to 128 rows); the host ledger stays the authority for execution
  state. A cached row can be stale until you press 刷新.
- **A live refresh covers all 128 cached rows**, batched eight at a time under one
  20 s deadline, so one view call stays inside the host's 30 s bridge budget and
  a slow batch keeps its cached values instead of dropping rows.
- **Creating sessions is bounded by the host**, not by this plugin: four active
  workers per creator, sixteen per plugin, and a worker cannot spawn its own
  workers.

## Layout

| File | Role |
| --- | --- |
| `main.js` | Load/unload, tool registration, watcher start, `onPanelInvoke` for the view |
| `actions.js` | `SessionTask` actions, including `spawnIn` and `workers` |
| `runtime.js` | Abort-aware gateway to `desktop.invoke` and host APIs |
| `state.js` | Bounded worker store in plugin settings (version 4) |
| `watcher.js` | `session:turnEnded` → result read → card |
| `deliver.js` | Ordered, grant-probing card delivery |
| `cards.js` | Pure text/row helpers (fully unit tested) |
| `views/workers.html` | The Workers view |
| `tests/` | `node:test` suites |

## Development

```bash
node --test tests/*.test.mjs
```

46 tests: the pure card/row helpers; `spawnIn` argument handling, its failure
paths and the delivery-failure row; the event-to-worker mapping (de-duplication,
cancelled, unsettled, malformed and foreign deliveries); the card delivery
fallback chain including transient vs permanent refusals; and `main.js` load and
unload against a fake host, including a failing history write.

## Security notes

The plugin holds `desktop.control`, which lets it create sessions, deliver
messages that run another Agent, read collaboration summaries, cancel work and
focus an existing session. Messages can consume the model quota you configured.
The plugin has no network permission, never reads credentials or MCP tokens,
never deletes a session, and cannot start a process. Everything it persists is
bounded UI history: titles, statuses, timestamps, a short result summary and
review notes.
