# FeatureBoard — Status

**Maintained across sessions.** Update this file at the end of every work
session. Read this first, then `docs/featureboard-design.md` for the full spec.

**Last updated:** 2026-11-01 (design session)

---

## Current state

Design complete, not yet implemented. Repo scaffolded (git init, docs/).

| Piece | State |
|---|---|
| `docs/featureboard-design.md` | ✅ written 2026-11-01 |
| `src/featureboard/*` | not started |
| `tests/*` | not started |

## What this is

Standalone, zero-dep Python package: feature/task lifecycle manager for
local coding agents. Per-project board (`.featureboard/board.json`),
phase gates (unit tests as canonical gate, hard-block on failure with
`--force` override), plan capture + LLM validation, session-start context
injection. Decoupled from any host — integrates via 5 hook points
(design doc §6). LLM abstracted behind `LLMClient` protocol with `MockLLM`
+ `OpenAICompatibleLLM` so it's copy-pasteable into other projects.

## Decisions (log)

- 2026-11-01: Separate repo `~/PycharmProjects/featureboard` (Lucas wants
  it copy-pasteable elsewhere).
- 2026-11-01: Data in-repo at `.featureboard/` (version-controlled).
- 2026-11-01: Gate failure **hard-blocks** task completion; explicit
  `force=True` / `--force` override, recorded in task history.
- 2026-11-01: **One active feature per project** (enforced invariant).
- 2026-11-01: LLM used only for plan validation; abstracted via protocol,
  MockLLM for tests/offline.
- 2026-11-01: Design doc first, implement after Lucas review (his choice).

## Next steps (in order)

1. Lucas reviews `docs/featureboard-design.md` — 4 open questions in §8
   (gate config location, digest builder, task IDs, /plan UX).
2. Implement `store.py` + tests (state machine, one-active, atomic writes).
3. Implement `gates.py` + tests (test/command gates, force override).
4. Implement `llm.py` + `plan.py` + `hooks.py` + tests.
5. `pyproject.toml`, README with the 5 localcoder hook points.
6. Lucas wires the 5 hook points into localcoder (design doc §6).

## Pitfalls / notes

- Host OS Python is 3.9 — use a venv (uv) for anything ≥3.10.
- Target host is localcoder (`~/PycharmProjects/localcoder`): plan mode in
  `agent.py` (read-only tools only), `prompt.py` builds the system prompt,
  `session.py` is the JSONL pattern to mirror.
