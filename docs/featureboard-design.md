# FeatureBoard — Design

**Status:** Design (proposed, not implemented)
**Last updated:** 2026-11-01
**Repo:** `~/PycharmProjects/featureboard` (standalone, copy-pasteable)

---

## 1. Problem

Local coding agents (localcoder, or any OpenAI-compatible agentic loop) have
no durable memory of *what they are working on*. Across sessions:

- The agent doesn't know which feature is active or what the next task is.
- Plans produced in a plan session evaporate when the session ends.
- There is no enforced checkpoint that "this task is actually done" — the
  model says done, and done it is, even if tests fail.

**Idea:** a small, standalone package — **featureboard** — that owns the
feature/task lifecycle for a project:

1. **Task storage** — per project, per feature, with states.
2. **Phase gates** — declarative checkpoints per task (unit tests are the
   canonical gate); a task cannot be marked done until its gate passes.
3. **Plan capture** — a plan session's output (plan markdown + summary) is
   persisted and attached to the feature it spawns.
4. **Session-start context** — every new session, the host asks featureboard
   for "active feature + next task" and injects it into the system prompt.

**Decoupling constraint (the whole point):** featureboard has **zero
dependency on localcoder** (or any host). It is a plain-Python package with a
small public API and a hook interface. The host (localcoder today, anything
else tomorrow) implements the hooks. Copy the package into another project
and it works — the only integration surface is the hook interface in §6.

---

## 2. Design constraints

- **Zero new deps.** Pure stdlib: `json`, `pathlib`, `subprocess`, `dataclasses`.
  (localcoder's ethos: "read it in an afternoon".)
- **LLM is optional and abstracted.** featureboard needs an LLM for exactly
  one thing: *plan validation* (§5.3). It talks to a `LLMClient` protocol,
  never to a concrete client. Ships with:
  - `MockLLM` — canned responses, for tests and offline use.
  - `OpenAICompatibleLLM` — thin adapter over any OpenAI-compatible
    endpoint (omlx, OpenAI, vLLM, …). The host can also pass its *own*
    client object if it already has one (duck-typed against the protocol).
- **Data lives in the project repo** at `.featureboard/` (version-controlled,
  travels with the project, auditable in git). Path is configurable; a
  `~/.featureboard/projects/<hash>/` mode is a one-line change if Lucas
  prefers it later.
- **One active feature per project** (Lucas decision, 2026-11-01). Keeps
  resume logic trivial: "the" active feature is unambiguous.
- **Gate failure hard-blocks** (Lucas decision, 2026-11-01): a task whose
  gate fails cannot be marked done, except via explicit `force=True`
  (CLI: `--force`), which is recorded in the task's history as an override.

---

## 3. Data model

Single JSON file: `.featureboard/board.json` (atomic write: tmp + rename).

```jsonc
{
  "version": 1,
  "project": "my-project",                 // basename of repo root
  "active_feature": "health-endpoint",     // null if none
  "features": {
    "health-endpoint": {
      "name": "health-endpoint",
      "title": "Add /health endpoint",
      "status": "active",                  // planned | active | done | shelved
      "created": "2026-11-01T12:00:00Z",
      "plan": {                            // null until a plan is captured
        "markdown": "## Plan\n1. ...",     // full plan text from plan session
        "summary": "One-paragraph digest", // for prompt injection (cheap)
        "validated": true,                 // plan validation pass ran
        "validated_at": "2026-11-01T12:05:00Z"
      },
      "tasks": [
        {
          "id": "t1",
          "title": "Add /health route to app.py",
          "status": "done",                // pending | in_progress | done | blocked
          "gate": {                        // null = no gate (manual completion)
            "kind": "test",                // "test" | "command"
            "target": "tests/test_health.py",   // file path (kind=test)
            "command": null                // shell command (kind=command)
          },
          "gate_state": "passed",          // null | passed | failed
          "gate_output_tail": "3 passed in 0.12s",
          "history": [
            {"at": "...", "event": "created"},
            {"at": "...", "event": "gate_failed", "detail": "1 failed"},
            {"at": "...", "event": "completed", "forced": false}
          ]
        }
      ]
    }
  }
}
```

State machine:

```
feature:  planned ──activate──▶ active ──complete──▶ done
                 └────────shelve──────────────────▶ shelved
task:     pending ──start──▶ in_progress ──gate passes──▶ done
                     │  gate fails
                     ▼
                  blocked ──gate re-passes / force──▶ done
```

- `complete_feature` requires **all tasks done** (or force).
- Activating a feature deactivates the previous one (status → `planned` if it
  had pending tasks, `done` if all done).

---

## 4. Public API (what the host calls)

```python
from featureboard import FeatureBoard

board = FeatureBoard(cwd=Path("/path/to/project"))   # reads .featureboard/board.json

# --- lifecycle -----------------------------------------------------------
board.create_feature(name, title, plan_markdown=None, tasks=[...]) -> Feature
board.activate_feature(name) -> Feature              # enforces one-active
board.complete_feature(name, force=False) -> Feature
board.shelve_feature(name) -> Feature

# --- tasks ---------------------------------------------------------------
board.start_task(feature, task_id) -> Task
board.complete_task(feature, task_id, force=False) -> Task
    # runs the task's gate first; raises GateBlocked if it fails and not force
board.add_task(feature, title, gate=None) -> Task

# --- plans ---------------------------------------------------------------
board.capture_plan(feature, markdown, summary=None) -> Plan
    # summary defaults to first paragraph; validation is separate (§5.3)

# --- context for the host ------------------------------------------------
board.resume_context() -> str
    # "Active feature: health-endpoint — next task: t2 'Write tests'
    #  (gate: tests/test_health.py, last state: failed). 2 of 5 tasks done."
    # "" if no active feature. This string goes into the system prompt.
board.next_task(feature=None) -> Task | None
```

All mutations are atomic (tmp + rename) and append to the task/feature
`history` array — the board file is its own audit log.

---

## 5. Components

### 5.1 `store.py` — `FeatureBoard`
The class above. Owns the JSON file, the state machine, and the one-active
invariant. No I/O beyond the board file. No LLM. No subprocess.

### 5.2 `gates.py` — `GateRunner`
Runs a gate, returns a verdict. Two kinds:

- **`test`** — `target` is a test file (or dir). Runner executes the project's
  test command scoped to it. Configured per-project in `.featureboard/config.toml`:
  ```toml
  [gates.test]
  command = "python -m pytest {target} -q"   # {target} substituted
  timeout = 300
  ```
  Default when no config: `python -m pytest {target} -q`.
- **`command`** — arbitrary shell command, run in the project root.

```python
@dataclass
class GateResult:
    passed: bool
    output_tail: str      # last ~2KB of combined stdout/stderr
    duration: float
    error: str | None     # non-None if the gate itself couldn't run
```

A gate that *can't run* (missing pytest, timeout) is `passed=False` with
`error` set — it blocks, but the message tells the host why.

### 5.3 `plan.py` — plan capture + validation
- `capture_plan` stores markdown + summary (summary = first paragraph, or
  LLM-generated if the host wants a better one).
- `validate_plan(board, feature, llm, codebase_digest)` — the "prompt
  validate" step Lucas described: a **read-only** LLM pass that checks the
  plan against a digest of the actual codebase (file tree + key signatures,
  produced by the *host* — featureboard doesn't parse code) and returns
  `PlanVerdict {ok: bool, issues: [str]}`. Issues like "plan references
  `app.py` which doesn't exist" or "plan touches 14 files — likely out of
  scope". The host decides what to do with a failing verdict (typically:
  show issues, ask the user, re-plan).

### 5.4 `llm.py` — the LLM abstraction
```python
class LLMClient(Protocol):
    def chat(self, messages: list[dict], tools: list | None = None) -> dict: ...
    # returns an OpenAI-style message dict: {"role": "assistant", "content": ...}

class MockLLM:            # canned/scripted responses; for tests + offline
    def __init__(self, responses: list[str] | None = None): ...

class OpenAICompatibleLLM:   # base_url, api_key, model — same shape as localcoder's client
    def __init__(self, base_url, api_key, model, timeout=120): ...
```

featureboard uses the LLM **only** in `validate_plan` (and optionally for
summary generation). Everything else is deterministic. A host that already
has a client (localcoder's `OpenAICompatibleClient`) can pass it in directly
if its `chat()` matches the protocol — otherwise wrap it in 5 lines.

### 5.5 `hooks.py` — the seam (what the host implements)
```python
class FeatureBoardHooks(Protocol):
    def on_session_start(self, board: FeatureBoard) -> str: ...
        # default impl: return board.resume_context()
    def on_plan_complete(self, board, feature, plan_markdown) -> None: ...
        # default: board.capture_plan + validate_plan
    def on_task_complete(self, board, feature, task) -> None: ...
        # default: no-op (gate already ran inside complete_task)
```

The host calls these at its natural seams. featureboard ships default
implementations, so a host can opt in with **zero code** for the basics and
only override what it needs.

---

## 6. Hook points — what Lucas needs to implement in localcoder

This is the complete integration surface. featureboard does not touch
localcoder; these are the edits on the localcoder side:

1. **`prompt.py` — system prompt injection** (the big one)
   In `build_system_prompt`, append `board.resume_context()` (or the
   host's `on_session_start` output). This is how every new session knows
   the active feature + next task. ~5 lines.

2. **`agent.py` — `complete_task` tool**
   Add a mutating tool `complete_task(feature, task_id, force=false)` that
   calls `board.complete_task(...)`. Mutating ⇒ goes through the existing
   approval gate + checkpoint, and the gate runs inside it. The tool result
   is the `GateResult` rendered as text, so the model sees test output and
   can react to failures. ~20 lines + schema entry.
   (Optional: a `start_task` tool, or let the model just call
   `complete_task` — the board auto-starts on complete. Recommend: only
   `complete_task` in v1, fewer tools for a 27B to juggle.)

3. **`agent.py` — plan-mode completion**
   When a plan-mode session ends (user exits plan mode / says "turn this
   into a feature"), call `on_plan_complete` with the plan text. The
   default hook captures + validates; on a failing verdict the host shows
   the issues and asks the user (this is the "are you ready to turn this
   into a feature set?" confirmation Lucas described).

4. **`cli.py` — slash commands**
   `/feature [name]` — list/activate features; `/tasks` — show the active
   feature's task list with gate states; `/gate <task_id>` — re-run a gate
   manually; `/plan` — capture the current plan-mode text as the feature's
   plan. All thin wrappers over the board API.

5. **`config.toml` — one section**
   ```toml
   [featureboard]
   enabled = true
   data_dir = ".featureboard"        # or "~/.featureboard/projects"
   ```

That's it. Nothing else in localcoder changes.

---

## 7. Package layout

```
featureboard/
├── pyproject.toml          # name=featureboard, zero deps, python>=3.10
├── src/featureboard/
│   ├── __init__.py         # exports FeatureBoard, GateRunner, LLMClient, MockLLM, ...
│   ├── store.py            # FeatureBoard + Feature/Task/Plan dataclasses
│   ├── gates.py            # GateRunner, GateResult
│   ├── plan.py             # capture_plan, validate_plan, PlanVerdict
│   ├── llm.py              # LLMClient protocol, MockLLM, OpenAICompatibleLLM
│   └── hooks.py            # FeatureBoardHooks + defaults
├── tests/
│   ├── test_store.py       # state machine, one-active, atomicity, history
│   ├── test_gates.py       # pass/fail/timeout/missing-runner, force override
│   ├── test_plan.py        # capture, summary, validation w/ MockLLM
│   └── test_hooks.py       # defaults, resume_context formatting
└── docs/featureboard-design.md   # this file
```

**Copy-paste story:** `pip install ./featureboard` (or vendor `src/featureboard/`
into the host repo) + the five hook points in §6. Works with any host that
has (a) a system prompt to inject into, (b) a tool registry, (c) a shell.

---

## 8. Open questions (await Lucas)

1. **Gate config location** — `.featureboard/config.toml` in-repo (my
   default) vs. a global `~/.featureboard/config.toml` with per-project
   overrides? In-repo wins for copy-paste-ability; global wins if the test
   command is the same everywhere.
2. **`validate_plan` codebase digest** — the host must supply it. For
   localcoder, the natural source is the RAC Tier 1 index (M5) when it
   lands; until then, a plain `find`-style file tree + `ast` signatures of
   top-level modules. Should featureboard ship a *basic* digest builder
   (stdlib `ast`, ~50 lines) so it works standalone, or stay host-supplied?
   I'd ship the basic one — it makes the package self-contained.
3. **Task IDs** — `t1, t2, …` per feature (my default) vs. UUIDs. Short IDs
   are friendlier for a 27B to reference in tool calls.
4. **`/plan` confirmation UX** — after validation, does the host auto-ask
   "turn this into a feature?" (Lucas's described flow), or only when the
   user explicitly says so? I'd default to asking, since plan mode is
   already an explicit intent signal.

---

## 9. Non-goals (v1)

- No cross-project views / dashboards.
- No dependencies between tasks (DAGs) — ordered list only.
- No multi-user / locking — single agent, single machine.
- No embeddings/LLM for task *execution* — the LLM is only in plan
  validation. Task work is done by the host agent.
