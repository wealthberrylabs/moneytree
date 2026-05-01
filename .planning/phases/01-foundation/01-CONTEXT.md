# Phase 1: Foundation - Context

**Gathered:** 2026-04-30
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 1 delivers the scaffolded Python package that everything else builds on: typed Pydantic contracts, an Alpaca-MCP client wrapper (paper-only + date-capped), SQLite trade-log persistence, and educational disclaimer plumbing. **No agent logic, no LLM calls, no MCP tool registration yet** — those come in Phase 2 and Phase 3 respectively. This phase produces the typed-contract substrate the pipeline consumes.

Requirements covered: **DATA-02, DATA-03, SAFE-04, EDU-02** (plus the Pydantic-model success criterion that's a Phase 1 deliverable shared with all later phases).

</domain>

<decisions>
## Implementation Decisions

### Console-script + entrypoint
- **D-01:** Single `moneytree` console script declared in `pyproject.toml` `[project.scripts]`. No `moneytree-mcp` separate binary.
- **D-02:** Subcommand structure: `moneytree mcp` (start MCP server), `moneytree run <ticker> "<thesis>"` (CLI dev/test path), `moneytree positions` (list positions). Use **`typer`** (Pydantic-friendly, type-hint-driven) — exact lib choice is implementation detail, but the user-facing shape is locked.
- **D-03:** AionUi MCP config will reference `command: "moneytree"`, `args: ["mcp"]`, `type: "stdio"` — locks the AionUi-facing contract Phase 3 must respect.

### Disclaimer plumbing (EDU-02)
- **D-04:** Module-level docstring at the top of `moneytree/__init__.py` carries the disclaimer text. This is the source for `LICENSE`/`NOTICE` cross-reference.
- **D-05:** Banner string emitted at CLI startup (`moneytree run`/`moneytree positions`) and MCP server init (`moneytree mcp`); identical string both surfaces. Format: 3 lines — name + version + disclaimer.
- **D-06:** Operator-visible Pydantic models (`AgentVerdict`, `TradeProposal`, `RiskDecision`, `FillResult`, `Position`) each include a `disclaimer: str` field with default value populated from a single source-of-truth constant `moneytree.const.DISCLAIMER`. AionUi will render this field on every output.
- **D-07:** Wording (single source of truth): `"Educational/research only. Not financial advice. Inherits the TradingAgents disclaimer."` — matches existing `README.md` and memory.

### Date-cap enforcement (DATA-03)
- **D-08:** Required keyword arg `date_cap: date` on every Alpaca-MCP wrapper function (`get_option_chain(*, date_cap, ...)`, `get_option_snapshot(*, date_cap, ...)`, `place_option_market_order(*, date_cap, ...)`). Missing arg = `TypeError` at call site. No defaults. Fail-loud.
- **D-09:** Pipeline orchestrator (`run_full_pipeline` — built in Phase 2/3) accepts a single `as_of: date` and threads it down through every wrapper call. Symmetric with TradingAgents' `ta.propagate(ticker, date=as_of)`.
- **D-10:** Date type is `datetime.date` (not `datetime`, not `str`). Wrappers internally translate to whatever Alpaca MCP expects.

### Paper-only enforcement (SAFE-04)
- **D-11:** `MoneytreeAlpacaClient` wrapper class has **only** the paper code path. Constructor does NOT accept a `paper: bool` flag — paper is **structural**, not configurable.
- **D-12:** Startup gate in `moneytree.__main__` validates `ALPACA_PAPER_TRADE` env var equals `"True"` before any subcommand runs. Raises `RuntimeError` with an explicit operator-visible message if missing/false. Both `moneytree mcp` and `moneytree run` route through this gate.
- **D-13:** No `ALPACA_LIVE_*` env vars are read or referenced in v0 code. Belt + suspenders: structural wrapper + env-var gate. No live keys can flow even if env is tampered.

### Claude's Discretion
[Pure implementation detail — not user-decision-shaped. Captured here so the planner doesn't re-ask.]

- **Package layout:** `src/moneytree/` per `uv init --lib` convention.
- **Persistence stack:** raw `sqlite3` stdlib + a tiny migration script in `moneytree/persistence/migrations/`. LangGraph `AsyncSqliteSaver` shares the same DB file in Phase 2 (one DB file at `./data/moneytree.db`, two-or-more tables).
- **CLI library:** `typer` (Pydantic-friendly, decorator-based, type-hint-driven).
- **Lint/format:** `ruff` (single tool — formatter + linter). **Type-check:** `mypy` strict on public API surface.
- **Test framework:** `pytest` + `pytest-asyncio` (Alpaca MCP wrappers are async).
- **Pydantic version:** v2. `model_config = ConfigDict(strict=False)` for ergonomic coercion.
- **Field naming:** snake_case (Python idiomatic). `model_dump_json(by_alias=False)` default.
- **Project metadata:** `name="moneytree"`, `version="0.0.1"`, `license="Apache-2.0"`. Dependency groups: `dev` (ruff, mypy, pytest, pytest-asyncio).
- **Entry point declaration:** `moneytree = "moneytree.__main__:app"` (typer app).
- **SQLite path:** `./data/moneytree.db` (per repo-portability convention; `data/` is gitignored).
- **Banner format:** plain text, 3 lines, ASCII only — render predictably in AionUi and stdio.

</decisions>

<specifics>
## Specific Ideas

- "Symmetric with TA `propagate(ticker, date)`" — keep terminology aligned with upstream so anyone who knows TradingAgents recognizes Moneytree's date-cap convention.
- The `disclaimer` field on Pydantic models is the **most important** EDU-02 surface — AionUi renders structured outputs as JSON in chat, so embedding the disclaimer in the model itself guarantees the operator sees it on every response (banners may be lost to scrollback).
- Wrapper class name `MoneytreeAlpacaClient` (not `AlpacaClient`) — disambiguates from any future direct-SDK code path that may exist post-v0.
- "Structural + startup gate" feels like over-engineering for a v0, but the cost is ~10 lines and the upside is "no possible code path that could submit a live order in v0."

</specifics>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project-level scope + decisions
- `.planning/PROJECT.md` — locked v0 stack + scope decisions, constraints, requirements summary
- `.planning/REQUIREMENTS.md` — DATA-02, DATA-03, SAFE-04, EDU-02 wording (this phase's reqs)
- `.planning/ROADMAP.md` §"Phase 1: Foundation" — goal + 5 success criteria the planner must satisfy
- `CLAUDE.md` — repo-level architecture summary, known traps (look-ahead, sizing non-linearity, margin pre-check), Path A vs Path B framing

### Stack-component API references (ctx7-verified 2026-04-30)
- `~/.claude/projects/-Users-MacBook-Developer-moneytree/memory/reference_stack_components.md` — verified API shapes for uv, Pydantic, Alpaca MCP, TradingAgents, LangGraph, Chroma, AionUi. Especially: Alpaca MCP function signatures (`place_option_market_order`, `get_option_snapshot`, env vars `ALPACA_API_KEY` / `ALPACA_SECRET_KEY` / `ALPACA_PAPER_TRADE`); option symbol format `AAPL251121C00190000`; AionUi McpServer config shape `{name, command, args[], env, type:'stdio'}`.

### Project memory (decision history)
- `~/.claude/projects/-Users-MacBook-Developer-moneytree/memory/project_v0_decisions.md` — env=uv, LLM=Anthropic Claude, Path A wrapper start, entry-only lifecycle (locks Phase 1 to NOT modify TA upstream)
- `~/.claude/projects/-Users-MacBook-Developer-moneytree/memory/project_aionui_integration.md` — stdio MCP transport (corrected from streamable-http), FastMCP, tool surface design

### Source-of-truth design doc
- `~/Downloads/moneytree-whitepaper.pdf` — full design rationale (April 2026, Randy Ellis, Wealthberry Labs). The Pydantic model field structure must align with the whitepaper's typed-contract intent.

### Visual reference
- `architecture.html` (repo root) — current architecture diagram showing the Phase 1 substrate slot

### External docs (fetch via Context7 if needed during planning)
- `/astral-sh/uv` — `uv init`, `uv add`, `uv lock`, `uv sync` workflow
- `/pydantic/pydantic` — `BaseModel`, `Field`, `model_dump`, `model_json_schema`
- `/alpacahq/alpaca-mcp-server` — async function signatures, env-var configuration

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets

**None — greenfield.** Repo currently contains only planning docs (`.planning/`), `CLAUDE.md`, `architecture.html`, `LICENSE`, `NOTICE`, `README.md`, `.gitignore`. No Python code yet.

### Established Patterns

- **Doc style:** All docs use Markdown with embedded code blocks for stack snippets. Phase 1 should establish the parallel pattern in code: Pydantic models with `Field(description="...")` so operator-visible JSON Schema carries the same level of clarity the docs carry.
- **Disclaimer text:** Already canonicalized in `README.md` and project memory — Phase 1 must reuse the exact wording (D-07).
- **Repo layout:** `.gitignore` already excludes `data/`, `.venv/`, `__pycache__/`, secrets — Phase 1's SQLite path and venv land in already-ignored territory.

### Integration Points

- **AionUi config format:** Phase 1 establishes the `command: "moneytree"` + `args: ["mcp"]` shape that Phase 3 will hand to AionUi's `mcpService.syncMcpToAgents`.
- **TradingAgents:** Phase 1 does NOT install or import TA yet (Path A wrapper consumes it from Phase 2). But the `AgentVerdict` Pydantic model must mirror TA's `decision = {action, confidence, reasoning}` shape so Phase 2 can map it directly without a translation layer.
- **Alpaca MCP:** Wrapper class is the seam. Wrapper accepts `date_cap` per call (D-08) and exposes the subset Strategist + PM need: `get_option_chain`, `get_option_snapshot` (Greeks live here), `place_option_market_order`. Don't expose more than v0 needs.

</code_context>

<deferred>
## Deferred Ideas

- **v2 Pydantic models** for `LIFE-*` (lifecycle), `CAD-*` (cadence), `VERIF-*` (backtest), `ARCH-*` (Path B) — out of v0 scope per `.planning/REQUIREMENTS.md`.
- **Live-trading code path** — explicitly forbidden in v0 (SAFE-04, D-11, D-13).
- **Multi-tenant Chroma path config** — TA owns Chroma, Moneytree shares its path; multi-tenant deferred.
- **Migration framework beyond ad-hoc script** — defer until schema churn warrants it; v0 has at most 2 tables (fills + LangGraph checkpoints).
- **`langchain-anthropic` install** — not Phase 1; Phase 2 brings it in when the TA wrapper actually invokes the LLM.
- **MCP tool registration** — Phase 3 territory; Phase 1 only defines the Pydantic models the tools will return.

</deferred>

---

*Phase: 01-foundation*
*Context gathered: 2026-04-30*
