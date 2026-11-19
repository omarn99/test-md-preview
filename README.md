# Syntra / AXIOM — Project Overview

> Complete map of the `.syntra` workspace: an autonomous AI agent ecosystem
> consisting of a Python agent core, a terminal UI, a desktop-automation
> library, a skill-hub backend, and a self-describing agent workspace.

---

## 1. What this project is

This workspace is the development home of **Syntra** (also branded **AXIOM**),
a terminal-based autonomous AI coding assistant. It is not a single
application — it is a constellation of related components:

| Component | Location | Language | Role |
|---|---|---|---|
| **AXIOM agent core** | `Sysdom Library/src/axiom/` | Python ≥3.12 | The agent: LLM clients, tools, memory, compaction, heartbeat, resume |
| **SysDOM library** | `Sysdom Library/sysdom/` | Python ≥3.11 | Cross-platform desktop UI introspection & control ("System DOM") |
| **AXIOM terminal UI** | `ui/` | TypeScript / React / Ink | The TUI the user talks to |
| **Hub server** | `hub-loadtest/hub-server/` + spec docs | Node 24 / Fastify / TS | Backend for a community skill marketplace |
| **Agent workspace** | `workspace/` (= `.syntra` home) | Markdown + SQLite | Persona, rules, memory store, skills, sessions — "where the agent lives" |
| **Compaction system** | `_ref_compaction/`, `compaction_out/`, `eval/` | Python | Context-window compaction engine + A/B quality harness |

The Python package is published under two names in `pyproject.toml`:
`axiom` (agent, entry points `axiom-term` / `axiom`) lives in
`Sysdom Library/src`, while `sysdom` is a separate installable package at
`Sysdom Library/sysdom` (entry via `python -m sysdom`).

---

## 2. Top-level directory map

```
.syntra/
├── PRODUCT.md                  Product brief: terminal AI coding assistant
├── smart-system.md             Design doc: DependencyCache + Store + VirtualRenderer
├── pyproject.toml              Builds package "axiom" from Sysdom Library/src
├── requirements.txt            Runtime deps (anthropic, playwright, pydantic…)
├── config/settings.yaml        db_path, log_level, max_agent_turns
│
├── Sysdom Library/             ★ Main Python project (see §3)
├── ui/                         ★ Terminal UI, axiom-ui (see §4)
├── workspace/                  ★ Agent home: persona/memory/skills (see §5)
│
├── hub-loadtest/               Hub server impl + load-test harness + REPORT.md
├── _ref_compaction/            Reference compaction implementation + tests
├── compaction_out/             Generated compaction output (engine + prompts + tests)
├── eval/                       Working-memory quality harness (A/B fold evaluation)
├── prompts/memory/             Memory-related prompt templates
├── skills/, sessions/, logs/   Runtime data dirs (mostly empty/logs)
├── docs/                       summary-lifecycle-diagnostics.md
├── Zed_Plan/                   Planning notes for markdown rendering components
│
├── HUB_SERVER_SPEC.md / .en.md Implementation spec for the Hub server (Arabic/English)
├── HUB_CONNECTIVITY_*.md       Hub connectivity review & implementation bundle
├── HUB_WEEKLY_INSTALL_TRACKING_SOURCE.md
├── PLATFORM_SUPPORT.md         Windows 11 verified; Linux/macOS partial
├── CODING_SKILLS_GUIDE*.md     Claude Code capability survey (v2.1.88 source)
├── CODING_STYLE_GUIDE (3).md   Coding style guide
│
├── *.py (root)                 One-off maintenance/mutation/test scripts (§8)
├── *.rar                       Archived workspace snapshots (workspace*.rar)
└── *.png, *.log, *.txt         Screenshots, server logs, tsc error dumps
```

Hidden/agent dirs: `.devin/`, `.claude/`, `.kilo/`, `.agents/`,
`.backups/`, plus two mutation-backup dirs (`.browser-render-mutation-backup-*`,
`.insight-tools-mutation-backup-*`). Git repo with a single commit
(`f12e4ec checkpoint: skills UI…`), no remote.

---

## 3. `Sysdom Library/` — the Python core

### 3.1 SysDOM — desktop UI as a queryable tree

`sysdom` (MIT, `sysdom/pyproject.toml`, version 1.0.0) converts live desktop
UI state into a structured tree that can be queried, serialized, and driven
programmatically. Per `README.md`:

- Cross-platform reader layer: Windows (`pywinauto`, `comtypes`), Linux
  (`pyatspi`), macOS — verified on Windows 11 (`PLATFORM_SUPPORT.md`).
- `SysNode` data model + `SysTree` query engine (CSS-like selectors,
  XPath-like lookup) — see `sysdom/core/{node,tree,selectors,extractors,
  geometry,identity,watcher,actions}.py`.
- `SysSerializer` → SysML XML / JSON / compact summaries (`screen.sysml`,
  `full_snapshot.sysml`, `compact_snapshot.sysml` are sample outputs at root).
- `SysActions` for click/type/drag/key/window ops; `SysWatcher` for
  diff-based change events; OCR vision fallback; stable element IDs.
- CLI (`python -m sysdom`): `snapshot`, `watch`, `running-apps`,
  `list-instances`, `inspect-ui`.
- Agent-facing tool layer in `sysdom/tools/` (dispatcher, mouse, keyboard,
  locator, uia_backend, screenshot, scroll, window, wait…) — this is what the
  AXIOM `sysdom_tool`/`hwnd_tool`/`browser` tools wrap.
- Extras: `[windows,automation,ocr]`, `[linux,…]`, `[macos,…]`.

### 3.2 AXIOM — the agent (`src/axiom/`)

Package layout (all under `Sysdom Library/src/axiom/`):

| Subpackage | Contents | Purpose |
|---|---|---|
| `ai/` | `factory.py`, `capabilities.py`, `gemini_client.py`, `fireworks_client.py` | LLM provider abstraction (Gemini, Fireworks, Anthropic via `anthropic` dep) |
| `core/` | `agent.py`, `agent_restored.py`, `compaction.py`, `compaction_archive.py`, `memory.py`, `evaluator.py`, `observer.py`, `proactive.py`, `self_modify.py`, `system_context.py`, `history_window.py`, `call_effect.py`, `creativity.py` | Agent loop, compaction wiring, self-modification, evaluation |
| `memory/` | `store.py`, `retrieve.py`, `rank.py`, `entities.py`, `normalize.py`, `redact.py`, `resolve.py`, `verify.py`, `consolidate.py`, `capture.py`, `inject.py`, `pack.py`, `stem.py`, `ids.py`, `paths.py`, `migrations/` | Typed memory store (BM25/trigram ranking — no vectors), migrations |
| `pipeline/` | `pipeline.py` | Turn pipeline |
| `prompts/templates/` | `compaction/`, `context/`, `execution/`, `headers/`, `insight/`, `notifications/`, `nudges/`, `persona/`, `repair/`, `response/`, `resume/`, `skills/`, `thinking/` + `response_protocol.md`, `thought_rule.*` | All prompt templates (incl. the `notifications/waiting.md` open in the IDE) |
| `terminal/` | `main.py`, `cli`, `executor.py`, `shell.py`, `dialect.py`, `config.py` | `axiom-term` CLI entry point |
| `tools/` | `base.py`, `filesystem.py`, `bash_tool.py`, `browser.py`, `browser_cursor/`, `code_runner.py`, `memory_tool.py`, `memory_tools.py`, `heartbeat_tool.py`, `hwnd_tool.py`, `sysdom_tool.py`, `app_control.py`, `avatar_tool.py`, `act/`, `insight/`, `insight_tool.py`, `my_history_tool.py`, `skill_reference.py`, `terminal_tool.py`, `git_status_tool.py`, `act_mode_tools.py`, `creativity_tool.py` | Tool implementations exposed to the model |
| `resume/` | `controller.py`, `detector.py`, `merger.py`, `store.py`, `config.py` | Session resume/continuation |
| `Heartbeat/` | `heartbeat.py` | Background scheduler/heartbeat tasks |
| `server.py`, `skills_cli.py`, `main.py` | — | HTTP server, `axiom` skills CLI, package main |
| `response_kinds.py`, `thinking_levels.py` | — | Response taxonomy & thinking-level config |

Notable root artifacts: `axiom_memory.sqlite` (the typed memory DB —
schema documented in `workspace/AGENTS.md`), `config/` (`settings.yaml`:
`db_path`, `log_level`, `max_agent_turns`), `tests/` (4.8 MB suite),
`examples/`, `docs/`, `ui-features/`, `workspace/` (agent instance copy),
`logs/`, `dist/`, `sysdom.egg-info/`, `screen*.sysml` snapshots, JSON test
fixtures (`test_*compaction*.json`, `test_boundary_scenarios.json`).

Docs: `README.md` (SysDOM), `CHANGELOG.md`, `CONTRIBUTING.md`, `AUTHORS.md`,
`SECURITY.md`, `LICENSE`, `Tools sysdom.md`,
`professional-comments-guide.md`.

---

## 4. `ui/` — AXIOM terminal UI (`axiom-ui`)

`ui/package.json`: **"AXIOM professional terminal UI — Ink-based,
TypeScript, physics animations"**. React 18 + a vendored `ink`
(`vendor/ink`), `tsx` runtime, `vitest` tests, `ws` for sockets.
Scripts: `dev`/`start` (`tsx src/index.tsx`), `test`, `typecheck`.

`ui/src/` structure:

- `App.tsx`, `index.tsx`, `client.ts`, `enhanced-ui.ts`, `theme.ts`,
  `constants/`, `colors/` — app shell & theme.
- `components/` — large Ink component library: messages, markdown,
  `pasted-text/` (indexed paste feature, documented in `INDEX.md` +
  `PASTED_TEXT_FIX` docs), `session/skills/AddSkillWizard.tsx` (the Hub
  client surface), plus dirs for agents, browser, commands, context, cost,
  dialogs, diff, fileedit, filesystem, git, heartbeat, input, insight,
  layout, mcp, memory, notifications, panels, permissions, questions,
  sandbox, screens, select, session, settings, shell, status.
- `smart-system/` — implementation of `smart-system.md`:
  `dependencyCache.ts`, `store.tsx`, `springStore.ts`, `virtualRenderer.ts`,
  `vim.ts` — named-deps memo cache, `useSyncExternalStore`-style pub/sub
  store, IntersectionObserver-style virtual rendering with node recycling.
- `compaction/` — TS mirror of the Python compaction engine with vitest
  coverage: `compactPolicy.ts`, `compactionState.ts`, `runCompaction.ts`,
  `compactionText.ts`, contract/wiring tests.
- `engines/`, `terminal/`, `hooks/`, `context/`, `utils/`, `questions/`,
  `react-compiler-cache/` (output of React Compiler the smart-system doc
  replaces).
- Versioned snapshots: `v0.2.0.json`, `one.json`.

`ui/claude-code-main/claude-code-main/` — a vendored copy of the **Claude
Code** source tree (Ink-based CLI: `cli/`, `commands/`, `components/`,
`QueryEngine.ts`, `memdir/`, `keybindings/`, `screens/`, `remote/`,
`plugins/`…). Its `src/` was excluded from this survey per request; it
appears to be reference/source material for the UI rather than the running
app.

Other `ui/` content: docs in Arabic (`INDEX.md`, `HUB_CONNECTION.md`,
`UI_ENHANCEMENTS.txt`, `USAGE_EXAMPLES.txt`, `scroll-selection-*.md`,
`add-skill-terminal-grid.md`), `AddSkillWizard.tsx` at root, `add-skill-ui/`
(186K design mockups), `Large · Step 1 — hub (search)-html/` (HTML mockup),
`scripts/`, `docs/`, `workspace/`, `vendor/` (vendored ink), `ui.png`,
`tsc-errors*.txt` diagnostics, `test-paste.mjs`, `scroll-selection-backup/`.

---

## 5. `workspace/` — the agent's home (`.syntra` instance)

`WORKSPACE-GUIDE.md` states it plainly: *"`.syntra/` is not a project
folder… It's where you live."* The agent's persona is **"Johnny Lawrence"**.

Base persona/rule files (compiled into `SYSTEM-PROMPTS.md`):

| File | Role |
|---|---|
| `SOUL.md` | Voice/tone: blunt, witty, brief; memory-aware behavior; kills corporate filler |
| `AGENTS.md` | Operating rules: filesystem tool JSON API, memory vocabulary (`kind`/`predicate`/`entity_key`), session lifecycle, escalation |
| `IDENTITY.md`, `USER.md`, `TOOLS.md`, `HEARTBEAT.md`, `molty-prompt.md`, `ROOMS.md`, `HOME-HISTORY.md` | Identity, user profile, tool docs, heartbeat state, live overrides, chat rooms |
| `SYSTEM-PROMPTS.md`, `AXIOM-SKILLS.md`, `SKILLS-GUIDE.md` | Compiled system prompt & skill docs |
| `WORKSPACE-GUIDE.md` | This workspace's own map (the file described above) |

Memory architecture (from `AGENTS.md`):

- `instance/axiom_memory.sqlite` — typed store; one row per claim with
  `kind` (`episode|fact|rule|skill_note`), `predicate`, `entity_key`,
  `trust`, `importance`, `confidence`, supersession fields, `recall_count`.
  Resolved from project root. **BM25 + trigram keyword ranking — no vector
  search.** Three surfaces: pinned prefix rows, `<memory_context>` block,
  and pull tools `memory_search`/`memory_get`/`memory_note`; citations as
  `[m:xxxx]`.
- `instance/memory/YYYY-MM-DD.md` — daily notes written by the turn
  distiller; `memory_compaction` heartbeat task folds old notes into
  `compacted.md`.
- `sessions/{sessionId}.json` — per-session snapshots; `resume/` package
  handles continuation.
- `skills/` — flat skills (`TERM.md`, `HEARTBEAT.md`, `SELF-AWARENESS.md`,
  `SELF_MODIFY.md`, `desktop-automation*.md`) and folder skills
  (`axiom-browser/` with SKILL.md+actions/captcha/recipes/scraping,
  `desktop-automation/`, `memory/` with references, `skill-notes/`).
  Folder skills load only on an exact `/<name>` message; companions via
  `skill_reference` tool.
- `Config/`, `logs/`, `.backups/` (auto-backups before edits),
  `browser_*_probe.py` scripts + cursor screenshot artifacts.

`instance/` mirrors the base files — it's the live copy auto-populated at
startup.

---

## 6. Hub server — community skill marketplace

- **Spec**: `HUB_SERVER_SPEC.md` (Arabic) + `HUB_SERVER_SPEC.en.md`.
  Proposed v1: Node 24 + TypeScript + Fastify + Zod, PostgreSQL + migrations,
  S3-compatible package storage, REST `/v1`, public reads, Bearer-token
  writes, `verified` trust set by admins only, uploads scanned but never
  executed. Consumer: `ui/src/components/session/skills/AddSkillWizard.tsx`
  via a `HubClient` (`status/search/detail/popular/recentlyUpdated`).
- **Implementation used for testing**: `hub-loadtest/hub-server/` — real
  Fastify+TS server (`src/{app,server}.ts`, `routes/`, `services/`, `db/`,
  `plugins/`, `schemas/`, `config/`, `types/`).
- **Load test**: `hub-loadtest/run-loadtest.mjs`, `bench-server.mjs`,
  `runs/`, and `REPORT.md` (2026-10-28): ~4,950 RPS sustainable on an
  18-skill catalog (P95 ~110 ms); degrades at 1k–10k skill catalogs; global
  rate limit 100 req/min per IP; documented a bug where `@fastify/rate-limit`
  429s surface as HTTP 500 via `src/plugins/errors.ts`.
- **Maintenance script**: `apply_hub_server_changes.py` patches files in a
  **separate checkout** at `F:\server request trae\hub-server` (outside this
  workspace).
- `HUB_CONNECTIVITY_IMPLEMENTATION_BUNDLE.md`, `HUB_CONNECTIVITY_REVIEW.md`,
  `HUB_WEEKLY_INSTALL_TRACKING_SOURCE.md` — connectivity design/review docs.

---

## 7. Compaction & evaluation

Context compaction keeps long tasks inside the model window in 4 layers
(`_ref_compaction/COMPACTION_INTEGRATION.md`):

1. **Structural** — truncate tool results at creation; collapse repeats.
2. **Clearing** — replace old tool results with placeholders (last 4 kept).
3. **Summary** — incremental structured summary via model call.
4. **Emergency** — clear harder / truncate / summarize without model.

Preserved by code (not the model): verbatim user messages, exact
identifiers (paths/URLs/errors/hashes), changed files, on-disk JSONL
transcript archive. Defaults: clear@55%, summarize@72%, target 30%,
emergency >92% of window.

- `generate_compaction.py` (root) — stdlib-only generator that emits the
  engine (`src/axiom/pipeline/compaction.py`), prompt templates, and tests.
- `_ref_compaction/` and `compaction_out/` — reference vs. generated
  outputs, each with `src/` + `tests/` + the integration doc.
- Wired into `axiom/core/compaction.py` + `compaction_archive.py` and
  mirrored in `ui/src/compaction/` (TS).
- `eval/` — working-memory quality harness (`fixture.py`, `probe.py`,
  `run.py`, `results/`): A/B test of full history vs. `Compactor.prepare()`
  output, scoring fact retention per zone; JSON result snapshots dated
  2026-11-06.
- Root mutation/test scripts (`_mutate_*.py`, `_new_compact_history.py`,
  `_check_compact_keep.py`, `_probe_split_ask.py`, `_test_*.py`,
  `notify_ab_probe.py`, `memory_test.txt`, `codex_chat_backup.txt`) —
  scratch probes used while developing compaction/protocol behavior.

---

## 8. Other directories & files

- `prompts/memory/` — memory prompt templates (empty at survey time).
- `docs/summary-lifecycle-diagnostics.md` — diagnostics notes.
- `Zed_Plan/` — planning folders for markdown rendering in the editor UI
  (heading-components, markdown-layout, markdown-rendering(+fix),
  syntax-colors).
- `sessions/`, `skills/` (root) — runtime dirs, currently empty; the live
  ones are under `workspace/` and `Sysdom Library/workspace/`.
- `logs/` (`server.log`, `_server_8427.log`, `logs/compaction/`) — runtime
  logs incl. compaction traces.
- `chrome_profile/` (root + inside `Sysdom Library/`) — Playwright/browser
  automation profiles (~90 MB each).
- `*.rar` archives — snapshots of `workspace` variants (incl. Arabic-named
  `workspaceسند*.rar`); `workspa2ce.rar` typo-named.
- `tsc-errors.txt`, `tsc-errors2.txt`, `screenshot.png`, `ui.png` —
  diagnostics/screenshots.
- `AddSkillWizard.tsx`, `Create a login page` (0-byte file), assorted
  `_*.py` probes — scratch artifacts.
- `check_syntax.py`, `fix_heartbeat.py` — AST-based repair scripts for
  `src/axiom/Heartbeat/heartbeat.py`.
- `.pytest_cache`, `.ruff_cache`, `__pycache__` — tool caches; ruff config
  in `Sysdom Library/pyproject.toml` (line-length 100, E/F/I/UP/B/SIM),
  strict mypy for Python ≥3.12.

---

## 9. Architecture at a glance

```
User ──► axiom-ui (Ink/React TUI, ui/)
           │  WS/HTTP, HubClient → /v1
           ▼
        AXIOM agent (Sysdom Library/src/axiom)
           ├── ai/        LLM providers (Anthropic, Gemini, Fireworks)
           ├── core/      agent loop, compaction, self_modify, evaluator
           ├── memory/    typed SQLite store + BM25 retrieval
           ├── tools/     filesystem/bash/browser/sysdom/hwnd/memory/…
           ├── terminal/  axiom-term CLI
           ├── Heartbeat/ background tasks (memory compaction, reviews)
           └── resume/    session snapshots & continuation
           │
           ▼
        SysDOM (sysdom/) ── desktop UI tree → screenshots/OCR → control
           │
           ▼
        workspace/ (.syntra) — SOUL/AGENTS/SYSTEM-PROMPTS +
                    instance/axiom_memory.sqlite + skills/ + sessions/

        Hub server (Fastify/TS + Postgres) ── skill marketplace
                    ▲
                    └── consumed by AddSkillWizard in the UI
```

Key cross-cutting ideas:

- **Everything is a file the agent can read** — persona, rules, skills,
  memory notes — compiled into `SYSTEM-PROMPTS.md` with `{{VAR}}`
  templating; stable sections cached, dynamic files re-injected per turn.
- **Memory is typed, not vector** — claims with kind/predicate/entity_key,
  trust-weighted BM25 ranking, citation handles `[m:xxxx]`, supersession
  instead of duplication.
- **Compaction is layered and code-guarded** — model summarization is the
  last resort; identifiers/user messages/files survive by construction.
- **UI performance is engineered** — smart-system (named-dep cache,
  pub/sub store, virtualized rendering) replaces React Compiler output.

---

## 10. Notable observations

- Single git commit, no remote — this is a working snapshot, not a
  maintained repo history.
- Heavy duplication by design: `Sysdom Library/workspace/` vs
  `workspace/`; `_ref_compaction/` vs `compaction_out/`; `ui/` vs
  `Sysdom Library/ui-features/`; Arabic-named `.rar` workspace archives.
- Several scratch/probe scripts at root (`_*.py`, `memory_test.txt`,
  `codex_chat_backup.txt`) document live A/B experiments — treat as
  historical artifacts, not production code.
- `HUB_SERVER_SPEC.md` describes a *proposed* server; the only real
  implementation found is the isolated test copy under `hub-loadtest/`,
  while `apply_hub_server_changes.py` targets an external checkout at
  `F:\server request trae\hub-server`.
- `ui/claude-code-main` is vendored Claude Code source used as reference;
  its `src/` was intentionally excluded from this survey.
