# kanban-loop — Design Document

Status: implementation-ready. Author: Designer role. Audience: Implementer.

This document specifies every file the Implementer must produce. Where the
brief left something open, the choice is recorded inline as `DECISION:` with a
one-line rationale. There are no open questions.

---

## 1. Overview & loop-engineering framing

kanban-loop is a Claude Code plugin that turns a GitHub Projects v2 board into a
self-running development loop. A single **resident Claude Code session** (the
*dispatcher*) runs in a workspace root directory and executes a recurring tick
via `/loop 2m /kanban-tick`. Each tick polls the board, dispatches per-issue
subagents, collects finished subagent results, moves issue statuses, and
enforces safety caps. It is harness-native and deliberately lightweight — a
single long-lived session rather than a separate process per issue.

The dispatcher is the **only** writer of board Status and `state.json`.
Subagents are pure workers: they receive an issue ref + mode, do the work, and
return a single structured JSON object as their final message. They never touch
the board or state.

### 1.1 The five moves

| Move | Where it lives in kanban-loop |
|------|-------------------------------|
| **Discovery** | `kanban-tick` polls the board with one GraphQL query (items whose Status is in a watched column AND that carry the `ai-ready` label). |
| **Handoff** | `kanban-tick` dispatches background subagents (`kanban-implementer` / `kanban-reviewer`) with a compact prompt (issue ref + mode). The subagent's structured JSON result is the handoff back. |
| **Verification** | Generator/evaluator separation. `kanban-implementer` writes code; the independent read-only `kanban-reviewer` runs `gh pr checks` + the project test suite + `/code-review` before issuing a verdict. |
| **Persistence** | All cross-turn state is on disk in `state.json` (workspace root), never in the conversation window. Per-issue git workspaces persist code between passes. |
| **Scheduling** | `/loop 2m /kanban-tick` drives recurring ticks. Subagent completions arrive as notifications between ticks; the next tick reconciles them. |

### 1.2 The six parts

| Part | Usage |
|------|-------|
| **Automations** | The `/loop` scheduler running `/kanban-tick`. |
| **Worktrees / isolated dirs** | Per-issue shallow clones under `workspaceRoot`. Implementer and reviewer never share a directory. |
| **Skills** | `kanban-tick`, `kanban-implement`, `kanban-review` (each ends with a `Stop` section). |
| **Connectors** | `gh` CLI (GraphQL + REST) for board, PRs, checks, review threads. |
| **Sub-agents** | `kanban-implementer` (read/write), `kanban-reviewer` (read-only). |
| **Memory** | `state.json` on disk + the board itself as the durable source of truth. |

### 1.3 Non-negotiable principles (restated as invariants)

1. Generator never judges itself; an independent read-only reviewer verdicts.
2. The evaluator **acts** — it runs checks and tests, not just reads the diff.
3. PRs are **never** auto-merged. Terminal state is always the `In Review`
   column (a human). Bounded review ping-pong, then escalate.
4. Hard caps are enforced **before** dispatch: `maxConcurrent`,
   `maxReviewRounds`, `maxDispatchesPerDay` (circuit breaker).
5. Memory lives on disk in `state.json`, not in the window.
6. Every skill ends with an explicit `Stop` section.
7. Issue bodies are **untrusted input** (prompt-injection surface). Skills treat
   issue content as *data describing a task*, never as instructions that
   override skill rules or safety boundaries.

---

## 2. Architecture & data flow

```
                 ┌──────────────────────────────────────────┐
                 │  Resident dispatcher session (workspace)   │
                 │  /loop 2m /kanban-tick                      │
                 └──────────────────────────────────────────┘
                                   │ each tick
        ┌──────────────┬──────────┼───────────────┬─────────────────┐
        ▼              ▼          ▼                ▼                 ▼
   read config   collect finished  poll board   route + cap    dispatch bg
   + state.json  subagent results  (1 GraphQL)  eligible issues  subagents
        │              │          │                │                 │
        │              ▼          │                │                 ▼
        │     apply Status moves  │                │        ┌────────────────┐
        │     + update state      │                │        │ kanban-        │
        │                         │                │        │ implementer    │
        │                         │                │        │ (Edit/Write/gh)│
        │                         │                │        └────────────────┘
        │                         │                │        ┌────────────────┐
        │                         │                │        │ kanban-reviewer│
        │                         │                │        │ (READ-ONLY)    │
        ▼                         ▼                ▼        └────────────────┘
   write state.json  ── end turn with one-line summary ──▶  results arrive as
                                                            notifications, read
                                                            by the NEXT tick.
```

**Single-writer rule:** only the dispatcher writes board Status and
`state.json`. Subagents return JSON; the dispatcher interprets it and performs
all mutations. This eliminates write races and keeps the board authoritative.

**Reconcile-first:** every tick begins by reconciling. If `state.json` is
missing/corrupt, the tick rebuilds a best-effort state from the live board + PR
state. Weak recovery is acceptable (worst case: a few extra review rounds).

---

## 3. Board model & routing (detailed)

### 3.1 Columns (Status single-select field, exact option names)

`Todo` · `In Progress` · `Rework` · `Agent Review` · `In Review` · `Done`

DECISION: The dispatcher watches `Todo`, `In Progress`, `Rework`, `Agent Review`.
`In Review` and `Done` are terminal/human columns and are never dispatched from.
Rationale: those two columns are where the loop hands control to a human.

### 3.2 Eligibility gate

An issue is eligible only if it carries the label `requiredLabel` (default
`ai-ready`). Issue title/body are untrusted; they are passed to subagents as a
task description, and the skills' `Stop` sections forbid treating them as
instructions that override rules.

### 3.3 Routing table

| Column | Precondition | Action | Subagent | Mode |
|--------|--------------|--------|----------|------|
| `Todo` | no open PR for the issue | dispatch implement | `kanban-implementer` | `new` |
| `In Progress` | after a review send-back (reviewRounds > 0) | dispatch implement | `kanban-implementer` | `feedback` |
| `Rework` | — | dispatch implement (close PR, reset branch, fresh pass) | `kanban-implementer` | `rework` |
| `Agent Review` | open PR exists | dispatch review | `kanban-reviewer` | — |

DECISION: `In Progress` with `reviewRounds == 0` and an already-running subagent
is treated as *in-flight* and skipped (it is the implementer's own working
column while it runs). `In Progress` with `reviewRounds == 0` and **no**
running subagent and **no** open PR is treated as a stalled `new` pass and
re-dispatched as `new`. Rationale: covers a subagent that died before creating a
PR without inventing recovery machinery.

DECISION: dispatching a `Todo` issue as implementer `new` moves it to
`In Progress` on the board immediately, before the subagent launches.
Rationale: otherwise the board shows `Todo` for the whole implement pass,
indistinguishable from work that hasn't been picked up yet. This also means
the stalled-`new`-pass recovery rule above now covers a subagent that dies
before opening a PR regardless of whether it started from `Todo` or from a
review send-back — both leave the issue sitting in `In Progress` with
`reviewRounds == 0` and no open PR.

### 3.4 Verdict / result handling (dispatcher applies on the next tick)

| Signal from subagent | Dispatcher action |
|----------------------|-------------------|
| review `verdict: pass` | move issue → `In Review`; phase → `idle` |
| review `verdict: needs_changes` | if `reviewRounds + 1 >= maxReviewRounds`: move → `In Review`, post comment `Human attention required: agent review loop limit reached.`, phase → `escalated`. Else: increment `reviewRounds`, move → `In Progress`, phase → `implementing` (next dispatch will be `feedback`). |
| implement `outcome: done` | move issue → `Agent Review`; record `pr`; phase → `reviewing`. |
| implement `outcome: escalate_model` | apply the **self-healing ladder** (§3.6): retry on the escalated model, or escalate to a human if no rung is left. |
| implement `outcome: needs_decision` | resolve the decision (§3.6): AskUserQuestion when `interactiveDecisions`, else comment the question + move → `In Review`. |
| implement `outcome: blocked` | move → `In Review`; post comment `Human attention required: <note>.`; phase → `escalated`. |
| any subagent recorded `running` but no result and no live subagent | reconcile: clear the in-flight fields, leave issue where the board has it, re-evaluate next tick (see §11). |
| running subagent stalled (watchdog: `elapsed > stallTimeoutMinutes`) | `TaskStop` it, then apply the **self-healing ladder** (§3.6). |

DECISION: `reviewRounds` counts **completed** review→send-back cycles. The cap
default 3 means at most 3 send-backs; the 3rd `needs_changes` escalates to
`In Review` rather than looping a 4th implement pass. Rationale: matches "when
reviewRounds reaches the cap → escalate."

### 3.6 Self-healing (watchdog + model escalation + decisions)

The dispatcher recovers a subagent that is *alive but stuck*, not just one that
died. This is a scoped exception to §11's "no recovery daemons" stance: it lives
**inside the tick** (no separate process), keyed off state and wall-clock time.

**Watchdog.** Each `subagent == "running"` issue records `dispatchedAt` and the
background `subagentId`. On every tick, before routing, the dispatcher computes
`elapsed = now - dispatchedAt`; if `elapsed > stallTimeoutMinutes` (config,
default 20) it `TaskStop`s the subagent and feeds it into the ladder below.
DECISION: this is a **wall-clock timeout**, not true progress detection — a
background subagent emits no incremental progress signal, so "no progress in N
minutes" is only observable as "running longer than N minutes." Honest naming
over an illusion of liveness tracking.

**Model-escalation ladder.** Triggered by a watchdog stall **or** an implementer
`escalate_model` result (the implementer self-assesses task difficulty *before*
doing work and can ask for a stronger model). `attempts` increments on each
(re)dispatch and resets when the issue changes column. Decide in order:

1. `attempts >= maxAttempts` (default 3) → hand to a human (comment + `In Review`,
   phase `escalated`). Hard backstop against thrash.
2. Implementer on the base model (`models.implementer`, e.g. `sonnet`) → record
   `model = models.escalated` in state, then re-dispatch same issue/mode; the
   dispatch step reads the stored `model` and launches on it (e.g. `opus`).
3. Implementer already escalated → no higher rung → hand to a human.
4. Reviewer (opus, no ladder) → re-dispatch (same model); repeated stalls are
   bounded by rule 1 (`maxAttempts`).

DECISION: the ladder is exactly one rung (sonnet→opus), then a human. Rationale:
"basically sonnet, opus only when difficulty shows"; unbounded escalation has no
higher model to reach and would only burn tokens. An `escalate_model` from an
already-escalated implementer is treated as `blocked`. Re-dispatches are real
dispatches (count against `maxConcurrent` / `maxDispatchesPerDay`).

**Decisions.** `needs_decision` is for a genuine human judgment call (a one-line
`question` + 2–4 `options`), distinct from `blocked`. When config
`interactiveDecisions` is true the tick surfaces it via **AskUserQuestion**,
records the answer as an issue comment, and re-dispatches with `decision="…"`.
When false (default, for unattended loops) it posts the question to the issue and
moves → `In Review`. DECISION: the loop runs unattended under `/loop`, so a human
is usually absent; interactivity is opt-in and always has an async fallback — the
value is in *structuring* the ambiguity, not in the prompt itself.

### 3.5 Board mutation mechanics (GraphQL)

The tick needs `projectId`, the Status `fieldId`, and the `optionId` for each
column name. Fetch these once per tick (cheap) and hold in memory for that
tick's mutations.

Metadata fetch (org shown; use `user(login:)` when `ownerType == "user"`):

```graphql
query($owner:String!, $number:Int!){
  organization(login:$owner){
    projectV2(number:$number){
      id
      field(name:"Status"){
        ... on ProjectV2SingleSelectField {
          id
          options { id name }
        }
      }
    }
  }
}
```

Status move mutation (used by §3.4 result handling and by step 10's
`Todo` → `In Progress` dispatch move):

```graphql
mutation($project:ID!,$item:ID!,$field:ID!,$option:String!){
  updateProjectV2ItemFieldValue(input:{
    projectId:$project, itemId:$item, fieldId:$field,
    value:{ singleSelectOptionId:$option }
  }){ projectV2Item { id } }
}
```

Plain issue comment (escalation):

```bash
gh issue comment <num> --repo <owner/repo> --body "Human attention required: ..."
```

---

## 4. Repo layout (recap — Implementer builds all of these)

```
kanban-loop/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── skills/
│   ├── kanban-tick/SKILL.md
│   ├── kanban-implement/SKILL.md
│   └── kanban-review/SKILL.md
├── agents/
│   ├── kanban-implementer.md
│   └── kanban-reviewer.md
├── schemas/
│   └── state.schema.json
├── examples/
│   ├── kanban-loop.config.json
│   └── settings.json
├── docs/
│   └── DESIGN.md          ← this file
└── README.md
```

Skill names keep the `kanban-` prefix so unqualified invocations are
`/kanban-tick`, `/kanban-implement`, `/kanban-review`.

---

## 5. Skill specifications

Each `SKILL.md` uses Claude Code skill frontmatter: `name`, `description`, and
optional `allowed-tools`. Keep bodies lean and high-signal. Every skill ends
with a `## Stop` section.

DECISION: Skills declare `allowed-tools` narrowly where it clarifies posture but
rely primarily on the agent definition + workspace `settings.json` for actual
permission enforcement. Rationale: agent `tools` allowlists are the real gate;
skill `allowed-tools` documents intent.

The three skills are implemented in `skills/kanban-tick/SKILL.md`,
`skills/kanban-implement/SKILL.md`, and `skills/kanban-review/SKILL.md` — those
files are the source of truth. This section records only what is not restated
there: the authoritative behavioral contracts live in §3 (routing §3.3, verdicts
§3.4, self-healing §3.6), the state contract in §7, and the config contract in
§8; design rationale is consolidated in §14. Line-count budgets (soft gate,
§12): tick ≤ 230, implement ≤ 90, review ≤ 55.

### 5.1 Subagent result contracts

Each worker returns exactly one JSON object as its final message; the dispatcher
parses it per §3.4 and performs all board/state mutations itself.

Implementer (`kanban-implement`):
```json
{"outcome": "done"|"blocked"|"escalate_model"|"needs_decision", "pr": <number|null>, "note": "<one line>", "question": "<one line>", "options": ["...","..."]}
```
`question`/`options` are present only for `needs_decision`.

Reviewer (`kanban-review`):
```json
{"verdict": "pass"|"needs_changes", "findings": <int>, "pr": <number>, "note": "<one line>"}
```

### 5.2 GraphQL and Stop sections

The board poll, project-metadata fetch, and status-move mutation are specified in
`skills/kanban-tick/SKILL.md` (mutation mechanics also in §3.5). The unresolved
review-thread fetch/reply/resolve flow (implementer `feedback` mode) is in
`skills/kanban-implement/SKILL.md`. Each skill file ends with its own `## Stop`
section — that is the operative copy of the safety boundaries.

---

## 6. Agent definitions

Claude Code subagent definitions live in `agents/*.md` with YAML frontmatter,
then a system-prompt body. The dispatcher launches them by name via the Agent
tool (`subagent_type`) with a prompt string.

DECISION: the implementer is pinned to `sonnet` (the base implementer model) and
is escalated to `opus` only on demand via the §3.6 ladder — cheap by default,
strong when difficulty actually shows. The reviewer is pinned to `opus`: review
quality (the evaluator half of generator/evaluator separation) is where
correctness is won, so it always runs on the strong model; its cost is instead
controlled by its read-only posture and the bounded review rounds. The escalated
implementer model is `config.models.escalated`; the tick passes the chosen model
to the Agent tool per dispatch, overriding the agent's pinned default.

The two agent definitions live in `agents/kanban-implementer.md` (read/write,
pinned `sonnet`) and `agents/kanban-reviewer.md` (read-only, pinned `opus`). The
reviewer's frontmatter deliberately omits `Edit`/`Write` — the read-only posture
is enforced by the `tools` allowlist, not just by instruction.

The dispatch prompts (what the tick passes each worker, including the optional
`model=` and `decision="…"` the implementer may receive) are specified in
`skills/kanban-tick/SKILL.md` step 11. DECISION: the config values a worker needs
(`workspaceRoot`, `branchPrefix`, `reviewEffort`, `model`) are passed inline in
the dispatch prompt, not re-read from config by the worker — subagents run in
their own workspace dirs and should not depend on locating the dispatcher's
config; the dispatcher owns config.

---

## 7. state.json — JSON Schema

File: `schemas/state.schema.json` (draft 2020-12).

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/<owner>/kanban-loop/schemas/state.schema.json",
  "title": "kanban-loop state",
  "type": "object",
  "additionalProperties": false,
  "required": ["date", "dispatchedToday", "issues"],
  "properties": {
    "date": {
      "type": "string",
      "pattern": "^\\d{4}-\\d{2}-\\d{2}$",
      "description": "Local date of the current daily window (YYYY-MM-DD)."
    },
    "dispatchedToday": {
      "type": "integer",
      "minimum": 0,
      "description": "Dispatches issued during the current date window."
    },
    "issues": {
      "type": "object",
      "description": "Keyed by '<owner/repo>#<num>'.",
      "propertyNames": { "pattern": "^[^/]+/[^#]+#\\d+$" },
      "additionalProperties": { "$ref": "#/$defs/issueState" }
    }
  },
  "$defs": {
    "issueState": {
      "type": "object",
      "additionalProperties": false,
      "required": ["phase", "pr", "reviewRounds", "subagent", "workspace", "updatedAt"],
      "properties": {
        "phase": {
          "type": "string",
          "enum": ["implementing", "reviewing", "escalated", "idle"]
        },
        "pr": { "type": ["integer", "null"], "minimum": 1 },
        "reviewRounds": { "type": "integer", "minimum": 0 },
        "subagent": { "type": ["string", "null"], "enum": ["running", null] },
        "subagentId": { "type": ["string", "null"], "description": "Background task id for the watchdog's TaskStop; null when idle." },
        "dispatchedAt": { "type": ["string", "null"], "format": "date-time", "description": "Dispatch time; basis for the stall timeout; null when idle." },
        "attempts": { "type": "integer", "minimum": 0, "description": "Consecutive dispatches for the current work unit; resets on column change." },
        "model": { "type": ["string", "null"], "description": "Model of the current/last dispatch; encodes the escalation rung; null when idle." },
        "workspace": { "type": "string" },
        "updatedAt": { "type": "string", "format": "date-time" }
      }
    }
  }
}
```

The four self-healing fields (`subagentId`, `dispatchedAt`, `attempts`, `model`)
are **not** in `required`, so old state files and reconciled state stay valid;
the tick treats them as null/0 when absent. See `schemas/state.example.json` for
a full instance (a running opus reviewer, a mid-escalation opus implementer, and
an escalated-to-human issue).

---

## 8. Example runtime config

File: `examples/kanban-loop.config.json` (copied to the workspace root at setup;
final field names).

```json
{
  "owner": "acme",
  "ownerType": "organization",
  "projectNumber": 12,
  "statusField": "Status",
  "requiredLabel": "ai-ready",
  "workspaceRoot": "./workspaces",
  "branchPrefix": "ai/",
  "caps": {
    "maxConcurrent": 3,
    "maxReviewRounds": 3,
    "maxDispatchesPerDay": 20
  },
  "reviewEffort": "high",
  "stallTimeoutMinutes": 20,
  "maxAttempts": 3,
  "models": {
    "implementer": "sonnet",
    "escalated": "opus"
  },
  "interactiveDecisions": false
}
```

Field notes for the README config section:
- `ownerType`: `"organization"` selects `organization(login:)` in GraphQL;
  `"user"` selects `user(login:)`.
- `projectNumber`: the number in the project URL (`/projects/<N>`).
- `statusField`: single-select field name; must be `"Status"` for the locked
  column model.
- `workspaceRoot`: dir (relative to cwd) holding per-issue clones.
- `branchPrefix`: branch name prefix; branch = `<branchPrefix><issueNumber>`.
- `reviewEffort`: passed to `/code-review` (e.g. `low|medium|high`).
- `stallTimeoutMinutes`: watchdog wall-clock timeout; a subagent running longer
  is stopped and re-dispatched via the §3.6 ladder (default 20).
- `maxAttempts`: hard backstop on consecutive dispatches per work unit before
  handing to a human (default 3).
- `models.implementer` / `models.escalated`: the base and escalated implementer
  models (default `sonnet` / `opus`). The reviewer stays pinned to `opus`.
- `interactiveDecisions`: when true, `needs_decision` is surfaced via
  AskUserQuestion; when false (default) it is posted to the issue and routed to
  `In Review`. Leave false for unattended loops.

---

## 9. Plugin manifest & marketplace

### 9.1 `.claude-plugin/plugin.json`
```json
{
  "name": "kanban-loop",
  "description": "Turn a GitHub Projects v2 board into a self-running, loop-engineered development loop: a resident Claude Code dispatcher polls the board, dispatches implement/review subagents, and moves issue status — never auto-merging.",
  "version": "0.1.0",
  "author": { "name": "kanban-loop" }
}
```
DECISION: rely on Claude Code's convention-based discovery of `skills/`,
`agents/` (no explicit path arrays in plugin.json) to keep the manifest minimal;
add explicit arrays only if `claude plugin validate` requires them. Rationale:
minimal manifest, fewer places to drift.

### 9.2 `.claude-plugin/marketplace.json` (self-hosting, single plugin)
```json
{
  "name": "kanban-loop",
  "owner": { "name": "kanban-loop" },
  "plugins": [
    {
      "name": "kanban-loop",
      "source": "./",
      "description": "Loop-engineered GitHub Projects board runner for Claude Code."
    }
  ]
}
```
Install path (README): `/plugin marketplace add <owner>/kanban-loop` then
`/plugin install kanban-loop@kanban-loop`. Local dev: `/plugin marketplace add
./kanban-loop`.

---

## 10. `examples/settings.json` (workspace-root permission allowlist)

The allowlist ships in `examples/settings.json`. DECISION: `defaultMode:
"acceptEdits"` so the resident loop is non-interactive for edits, plus a tight
`allow` list — `Bash(gh:*)`, `Bash(git:*)`, `Read`/`Write`/`Edit`, `Agent`, and
(for self-healing) `TaskStop` and `AskUserQuestion` — with no blanket `Bash(*)`.
A `deny` block refuses `gh pr merge` and the force-push variants outright, so even
a prompt-injected subagent cannot merge or force-push. Rationale: matches the
brief; the README warns this is a powerful posture that belongs only in a
dedicated workspace, and that target-repo branch protection is the real safety
net.

---

## 11. Edge cases & failure handling

Design stance: trade fault-tolerance for lightness. Default resolution is
"reconcile from the live board + PRs on the next tick." No recovery *daemons* —
the one active-recovery mechanism (the §3.6 watchdog) is a scoped exception that
runs **inside the tick**, not as a separate process, preserving the
single-resident-session model.

| Case | Resolution |
|------|------------|
| **Restart mid-flight** (resident session killed) | On restart, first tick reconciles: state.json (if present) may show `subagent: running` for subagents that no longer exist. Since no result arrived, the tick clears the in-flight fields, keeps the board column as-is, and re-routes normally next tick. In-flight code survives in the workspace/branch. |
| **Subagent stalled (alive but stuck)** | Watchdog (§3.6): if `elapsed > stallTimeoutMinutes`, `TaskStop` the subagent and apply the escalation ladder — implementer sonnet→opus, then a human; reviewer re-dispatched on the same model. All paths bounded by `maxAttempts`. |
| **Stall caused by infra hang** (network/`gh`), not difficulty | Escalating the model will not fix it, but the ladder is bounded (one rung, then a human) so the cost is capped and a human is always reached. Accepted tradeoff — the tick cannot distinguish a hung subagent from a hard one. |
| **state.json missing/corrupt** | Start empty state, `reconcileFull=true`: rebuild `issues` from the board columns + `gh pr list` by branch. Best-effort phase/PR inference; reviewRounds resets to 0 (worst case: a few extra review rounds). |
| **state / board divergence** | Board is authoritative for Status. The tick reads the live column and routes from it; state only informs `reviewRounds`, `subagent`, `pr`. If they disagree, follow the board. |
| **Subagent died without result** | `subagent:"running"` but no completion notification and no live subagent → clear `subagent`, do nothing else this tick. Next tick re-routes from the current column (e.g. an implementer that died in `Todo`/`In Progress` gets re-dispatched; a reviewer that died in `Agent Review` gets re-dispatched). Idempotent because the workspace/branch/PR persist. |
| **PR closed by a human mid-loop** | Detected when routing/reconciling (`gh pr list --state open` returns nothing for the branch). Treat the issue as no-open-PR: if in `Agent Review`, there is nothing to review → leave for the human (do not re-open). If in `Todo`, dispatch `new`. DECISION: never re-open a human-closed PR; if a column implies a PR should exist but none does, and the branch has commits, dispatch does not force a new PR in review columns — it only creates PRs in implement modes. Rationale: respect the human action. |
| **Issue moved by a human mid-flight** | Board is authoritative. Next tick routes from the new column. If a subagent is still running for the old intent, its result is applied against §3.4 relative to where the issue now is; if that no longer makes sense (e.g. issue moved to `Done`/`In Review`), the dispatcher applies the result only as a state update and does not override a human's terminal column. DECISION: never move an issue **out of** `In Review` or `Done`. Rationale: those are human columns. |
| **Day rollover** | If `state.date != today`: set date, reset `dispatchedToday=0`. Issues/phases untouched. |
| **`gh` auth failure** | Any `gh` call failing auth → the tick reports `gh auth failed — run 'gh auth login'` in its one-line summary and ends the turn without mutations. No state write beyond a possible date rollover. The next tick retries. |
| **`ai-ready` label removed mid-flight** | Issue drops out of the poll (eligibility gate). Any in-flight subagent finishes; its result is recorded in state but no board move is made for a now-ineligible issue. It simply stops being routed. |
| **Duplicate dispatch risk** | Guarded by `subagent == "running"` in state + single-writer ticks (ticks do not overlap because `/loop` serializes). No locking needed. |
| **maxConcurrent reached** | Skip further dispatch this tick; routed-but-skipped issues are re-evaluated next tick (no queue). |
| **maxDispatchesPerDay reached** | Skip all dispatch; set a `dailyCapReached` note; mention in summary; post nothing to the board. Resets at rollover. |

---

## 12. Validation plan (runnable without a live board)

The team can validate every deliverable offline:

1. **JSON validity.** `jq empty schemas/state.schema.json`,
   `jq empty examples/kanban-loop.config.json`,
   `jq empty examples/settings.json`,
   `jq empty .claude-plugin/plugin.json`,
   `jq empty .claude-plugin/marketplace.json`.
2. **Schema self-check.** Validate the schema against its meta-schema and
   validate the example instance (§7) against the schema. If `ajv-cli` is
   available: `ajv validate -s schemas/state.schema.json -d /tmp/state.example.json`.
   Otherwise a minimal Python `jsonschema` snippet (documented in README
   troubleshooting) — this is a check step, not a shipped file.
3. **Skill frontmatter parse.** For each `skills/*/SKILL.md`, confirm the YAML
   frontmatter block parses and contains `name` + `description`
   (e.g. pipe the block through `yq`/`python -c` or `claude plugin validate`).
   Confirm each skill body contains a `## Stop` section (grep).
4. **Agent frontmatter parse.** For each `agents/*.md`, confirm `name`,
   `description`, `tools` parse; confirm the reviewer's `tools` list contains no
   `Edit`/`Write`.
5. **Plugin validate.** Run `claude plugin validate .` (or the current
   equivalent) if available; fix reported issues.
6. **Line-count budget check.** `wc -l` each SKILL.md — a soft gate against
   bloat. Targets were raised to absorb the §3.6 self-healing logic: tick
   ≤ 230, implement ≤ 90, review ≤ 55.
7. **Paper dry-run of one tick** against a fictional 3-issue board:

   Board state:
   - `acme/web#1` in `Todo`, label `ai-ready`, no PR.
   - `acme/web#2` in `Agent Review`, PR #20 open, `reviewRounds=0`.
   - `acme/web#3` in `In Progress`, `reviewRounds=1`, subagent not running,
     PR #21 open (came back from a review send-back).

   Expected tick behavior (walk each skill by hand and confirm):
   - Reconcile: no running subagents → nothing to collect.
   - Poll returns all three (all `ai-ready`, all in watched columns).
   - Route: #1 → implement `new`; #2 → review; #3 → implement `feedback`.
   - Caps: `maxConcurrent=3`, `dispatchedToday` well under 20 → dispatch all 3.
   - state.json after: each issue `subagent:"running"`, phases
     `implementing/reviewing/implementing`, `dispatchedToday += 3`.
   - Summary line: `tick: polled 3 / eligible 3 / dispatched 3 (impl 2, review 1) / inFlight 3 / today 3/20`.
   - Next tick (simulate results): #2 review returns `pass` → move #2 to
     `In Review`; #1 implement returns `done` pr=22 → move #1 to `Agent Review`;
     #3 implement returns `done` → move #3 to `Agent Review`. Confirm no issue is
     ever moved to `Done` or auto-merged.
   - Simulate a 3rd `needs_changes` on some issue → confirm escalation path
     (move to `In Review` + the exact comment string).

8. **Paper dry-run of self-healing (§3.6):**
   - **Stall + escalate:** an implementer on `sonnet` with
     `dispatchedAt` 25 min ago, `stallTimeoutMinutes=20`, `attempts=1` →
     watchdog `TaskStop`s it, re-dispatches the same issue/mode on `opus`,
     `attempts=2`, `model=opus`, fresh `dispatchedAt`; `dispatchedToday += 1`.
   - **Escalated stall → human:** the same issue stalls again on `opus` →
     comment `Human attention required: implementer stalled on opus …`, move →
     `In Review`, phase `escalated`.
   - **escalate_model:** a fresh `sonnet` implementer returns `escalate_model`
     → immediate re-dispatch on `opus` (no wait for the timeout).
   - **needs_decision, unattended:** `interactiveDecisions=false` +
     `needs_decision` → question posted to the issue, move → `In Review`.
   - **maxAttempts backstop:** any role reaching `attempts >= maxAttempts` →
     human, regardless of role/model.

Passing 1–8 means the plugin is structurally correct and the loop logic is sound
on paper; a live board is only needed for end-to-end confirmation.

---

## 13. README table of contents (1–2 sentence notes per section)

The Implementer writes `README.md` (English) with these sections:

1. **What it is** — Loop-engineering framing; the 5-moves / 6-parts mapping
   table (reuse §1.1–1.2). One paragraph on the resident-dispatcher model and
   the "never auto-merge; terminal state is In Review" promise.
2. **How the loop behaves** — The routing table (§3.3), verdict handling (§3.4),
   caps, and escalation. This is the mental model section.
3. **Prerequisites** — Claude Code version with `/loop` + subagents; `gh` CLI
   authenticated; board setup (the six columns + `ai-ready` label). Explicitly
   note **no custom fields needed** — only the built-in `Status` single-select.
4. **Install** — Plugin marketplace add from the repo
   (`/plugin marketplace add <owner>/kanban-loop` → `/plugin install
   kanban-loop@kanban-loop`), or local (`/plugin marketplace add ./kanban-loop`).
5. **Configuration** — `kanban-loop.config.json` field-by-field (reuse §8 notes).
6. **Startup** — `cd` into the workspace root; place `kanban-loop.config.json`;
   set up `.claude/settings.json` from `examples/settings.json`
   (acceptEdits + `Bash(gh:*)` + `Bash(git:*)`); run `claude`; then
   `/loop 2m /kanban-tick`.
7. **state.json reference** — The schema fields and an annotated example (§7).
   Note it is disk memory, auto-created, and safe to delete (rebuilds by
   reconcile).
8. **Security notes** — Untrusted issue bodies (prompt-injection surface) and how
   the skills' Stop sections defend; the `ai-ready` label gate; and the real
   safety net: branch protection + required human review on target repos.
9. **Operational discipline** — Read a sample of PRs daily (comprehension rot);
   respect caps (token blowout); the human checkpoint at `In Review` is
   permanent by design.
10. **Limitations & evolution path** — Local `/loop` stops when the machine
    sleeps; future: move the tick to cloud scheduling / CI; the tick logic can
    later be extracted to deterministic code without changing the skills.
11. **Troubleshooting** — `gh auth` failures; corrupt/missing state.json; issues
    not being picked up (label / column / eligibility); caps tripped; how to run
    the §12 validation checks.

---

## 14. Decisions log (consolidated)

- **DECISION:** Watch only `Todo`, `In Progress`, `Rework`, `Agent Review`;
  `In Review`/`Done` are human terminal columns, never dispatched from.
- **DECISION:** `In Progress` + `reviewRounds==0` + no running subagent + no PR →
  re-dispatch as `new` (covers a dead pre-PR subagent without recovery machinery).
- **DECISION:** `reviewRounds` counts completed send-back cycles; the
  `maxReviewRounds`-th `needs_changes` escalates rather than looping again.
- **DECISION:** Fetch project/field/option metadata once per tick, hold in memory.
- **DECISION:** Board poll uses one 100-item page; paginate only on `hasNextPage`.
- **DECISION:** Implementer only resolves review threads it addressed this pass.
- **DECISION:** Implementer pinned to `sonnet`, escalated to `opus` on demand
  (§3.6); reviewer pinned to `opus` (correctness-critical evaluator half). Tune
  reviewer cost via `reviewEffort`, not a cheaper model.
- **DECISION:** Self-healing lives inside the tick (watchdog wall-clock timeout +
  `TaskStop` + one-rung sonnet→opus ladder + `maxAttempts` backstop), not a
  separate daemon — a scoped exception to "no recovery daemons".
- **DECISION:** Difficulty is discovered, not pre-declared: escalation is driven
  by an implementer `escalate_model` self-assessment and by watchdog stalls, not
  by a human-set difficulty label (humans often cannot judge hardness upfront).
- **DECISION:** `needs_decision` is interactive only when `interactiveDecisions`
  is set; the default (unattended) path posts the question to the issue and
  routes to `In Review`. Interactivity is opt-in with an async fallback.
- **DECISION:** Subagents receive `workspaceRoot`/`branchPrefix`/`reviewEffort`
  inline in the dispatch prompt; the dispatcher owns config.
- **DECISION:** Minimal `plugin.json` relying on convention-based discovery; add
  explicit path arrays only if `claude plugin validate` demands them.
- **DECISION:** `settings.json` uses `acceptEdits` + tight `Bash(gh:*)` /
  `Bash(git:*)` allowlist (no blanket `Bash(*)`).
- **DECISION:** Never re-open a human-closed PR; never move an issue out of
  `In Review`/`Done`. The human checkpoint is inviolable.
- **DECISION:** Skills declare narrow `allowed-tools` for documentation; the
  agent `tools` allowlist + workspace `settings.json` are the real enforcement.
