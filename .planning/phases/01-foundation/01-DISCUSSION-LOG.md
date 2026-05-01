# Phase 1: Foundation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-30
**Phase:** 01-foundation
**Areas discussed:** Console-script + entrypoint naming, Disclaimer style + frequency, Date-cap default behavior, Paper-only enforcement style

---

## Console-script + entrypoint naming

| Option | Description | Selected |
|--------|-------------|----------|
| Single `moneytree` with subcommands | `moneytree mcp`, `moneytree run TICKER "thesis"`, `moneytree positions`. Modern CLI shape (kubectl/gh/uv). AionUi config: command=moneytree args=[mcp]. One pyproject script entry. | ✓ |
| Two separate scripts | `moneytree-mcp` for the MCP server, `moneytree` for CLI runner. Two pyproject script entries. AionUi config: command=moneytree-mcp args=[]. | |
| Short alias `mt` | `mt mcp`, `mt run TICKER "thesis"`. Faster to type. May collide with existing tools. Less self-explanatory in AionUi config. | |

**User's choice:** Single `moneytree` with subcommands (Recommended)
**Notes:** Locks AionUi MCP config shape to `command: moneytree, args: [mcp], type: stdio`. Subcommand naming established for future surface growth (`positions`, etc.). Implementation lib (`typer`) is Claude's discretion; user-facing shape is the locked decision.

---

## Disclaimer style + frequency

| Option | Description | Selected |
|--------|-------------|----------|
| Banner + Pydantic field | Banner at startup (CLI + MCP init), shared `disclaimer: str` field on every operator-visible Pydantic model (TradeProposal, RiskDecision, FillResult), module docstring + LICENSE/README. Forces AionUi to render it on every output. | ✓ |
| Maximum frequency | Above + comments at top of every source file + log line on every order + every CLI command prints it. Loudest. Cluttered. | |
| Minimalist | Module docstring + LICENSE + README only. No runtime output pollution. Cleanest but easiest to miss in operator UI. | |
| Banner + CLI footer | Banner at startup, footer line only in CLI render. No Pydantic field. AionUi/MCP consumers rely on banner. Lighter than option 1. | |

**User's choice:** Banner + Pydantic field (Recommended)
**Notes:** The Pydantic-field choice means AionUi (which renders structured JSON in chat) will surface the disclaimer on every output, not just at startup where it could be lost in scrollback. Single source-of-truth constant (`moneytree.const.DISCLAIMER`) prevents wording drift; matches existing README phrasing.

---

## Date-cap default behavior

| Option | Description | Selected |
|--------|-------------|----------|
| Required arg, fail-loud | Every Alpaca wrapper takes `date_cap: date` as required keyword arg. Missing it = TypeError. Symmetric with TA's `propagate(ticker, date)`. Hard to forget. Verbose at call sites but unambiguous. | ✓ |
| Default to today + WARN log | `date_cap: date = field(default_factory=date.today)`. WARN log every time default fires. Convenient but logs are easy to miss. Risk: silent look-ahead. | |
| ContextVar-injected via decorator | Set `as_of` once at pipeline entry; `@requires_date_cap` decorator pulls from contextvar. DRY. But hidden control flow — harder to audit per-call. | |
| Hybrid: required at entry, contextvar internally | Pipeline entry (`run_full_pipeline`) requires `as_of`. Downstream wrappers receive via contextvar set by entry. Best of both: loud at boundary, quiet internally. | |

**User's choice:** Required arg, fail-loud (Recommended)
**Notes:** Trade-off accepted: more verbose call sites (every chain/IV/order call passes `date_cap=as_of`) in exchange for compile-time-ish safety. TA's `propagate(ticker, date)` already requires date, so Moneytree wrappers stay symmetric. ContextVar pattern explicitly rejected because hidden control flow makes audits harder for a safety-critical concern.

---

## Paper-only enforcement style

| Option | Description | Selected |
|--------|-------------|----------|
| Structural + startup gate | Wrapper class has NO live-key code path — `paper=True` is the only constructor mode. Plus: startup gate validates `ALPACA_PAPER_TRADE=True` env var or refuses to launch. Belt + suspenders. Cheapest insurance for v0. | ✓ |
| Startup gate only | Validate env var at process start; refuse if missing/False. If env tampered later, no protection. Simpler code. | |
| Per-call check | Every Alpaca wrapper function validates env var before each call. Loudest. Verbose. Slowest. | |
| Structural only (no env check) | Wrapper class only has paper code path. Code-level guarantee. No env validation — trust the wrapper. Cleanest but no defense-in-depth. | |

**User's choice:** Structural + startup gate (Recommended)
**Notes:** Both layers fit in ~10 lines of code. v0 paper-only is a hard safety bar — accepting slight redundancy in exchange for "no possible code path that could submit a live order." `ALPACA_LIVE_*` env vars are not read or referenced anywhere in v0 code (D-13).

---

## Claude's Discretion

User did not call these out explicitly, but Phase 1 has several pure-implementation choices that don't require user vision and are captured in CONTEXT.md `<decisions>`:

- Package layout (`src/moneytree/`)
- Persistence stack (raw `sqlite3` + tiny migration script; LangGraph `AsyncSqliteSaver` shares the DB file)
- CLI library (`typer`)
- Lint/format (`ruff`) + type-check (`mypy` strict)
- Test framework (`pytest` + `pytest-asyncio`)
- Pydantic v2 with non-strict coercion default
- Project metadata (name=`moneytree`, version=`0.0.1`, license=`Apache-2.0`)
- SQLite path (`./data/moneytree.db`)
- Banner format (3 lines, ASCII-only)

## Deferred Ideas

None surfaced during discussion that aren't already captured in `.planning/REQUIREMENTS.md` v2 section. CONTEXT.md `<deferred>` records the standing v0/v2 split: lifecycle, cadence, backtest harness, Path B native fork, multi-tenant Chroma, full migration framework, `langchain-anthropic` install (Phase 2), MCP tool registration (Phase 3).
