# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project: Moneytree

Moneytree is a fork of [TradingAgents](https://github.com/TauricResearch/TradingAgents) (multi-agent LLM trading framework, Tauric Research, v0.2.4) that adds a native options layer. TradingAgents only emits BUY/SELL/HOLD on equities — Moneytree extends the LangGraph agent graph so a directional thesis is converted into a structured options position (vertical, calendar, iron condor, etc.) with defined risk, Greeks-aware sizing, and broker execution via Alpaca.

The repo is currently greenfield (no code yet). The full design rationale lives in `moneytree-whitepaper.pdf` (April 2026, Randy Ellis, Wealthberry Labs).

**Disclaimer in code, comments, and outputs:** educational/research only. Not financial advice. The fork inherits TradingAgents' disclaimer.

## Architecture

Moneytree extends the TradingAgents LangGraph pipeline. Existing nodes (4 analysts, bull/bear researchers) stay; the trader and risk manager are replaced/augmented.

### Pipeline (current diagram)

```
CLI Runner (ticker + thesis)
  -> TradingAgents Engine
       <-> Chroma (debate memory, read/write)
       <-  Market Data Feeds (news + fundamentals)
       <-> LLM Provider (analyst debate)
       -> direction + confidence
  -> Options Strategist
       <-  Alpaca MCP Paper (chain + IV)
       <-> LLM Provider (structure pick)
       -> trade proposal
  -> Greeks Risk Manager
       -> approved + sizing
  -> Portfolio Manager
       -> Alpaca MCP Paper (submit order)
       -> SQLite Trade Log (fill + PnL)
```

Operator entrypoint is a **CLI Runner**. Inputs are a **ticker plus a user-supplied thesis** — the thesis is not derived, the operator types it. TradingAgents emits `direction + confidence`, not just BUY/SELL/HOLD; the strategist consumes that as a typed verdict.

### Persistence

- **Chroma** — vector store for debate memory. Read/written by TradingAgents Engine across runs. Used for the "memory" mechanic upstream TradingAgents already exposes; do not introduce a second memory store for daily-cadence work.
- **SQLite** — local trade log. Portfolio Manager writes fills + PnL after Alpaca confirms. Path/filename TBD when scaffolding; keep it a single file in the repo root or `./data/` for portability.
- **Alpaca MCP Paper** — chain/IV reads (Strategist) and order submission (PM). Same MCP server, two call sites.

### Node responsibilities (strict)

- **Options Strategist** — pulls chain + IV via Alpaca MCP, picks structure via LLM, emits a structured `trade proposal`: structure type, leg list, max loss, max profit, breakeven, current Greeks, thesis-match score. No execution. The whitepaper splits this into `options_analyst` + `options_strategist`; the current diagram collapses both into one node — OK for v0, split later if the strategist prompt gets unwieldy.
- **Greeks Risk Manager** — aggregates portfolio delta/gamma/vega/theta. Runs scenario PnL at ±1σ and ±2σ underlying moves and at ±5 vol points. Rejects breaches (hard gate, not advisory). **Sizing happens here**: emits `approved + sizing` to PM, not just an approval flag.
- **Portfolio Manager** — receives approved+sized proposal, submits multi-leg order via Alpaca MCP, writes fill + PnL to SQLite. Does not re-decide structure or size.

### Two implementation paths (stage by stage, not parallel)

- **Path A — light wrapper.** Run upstream TradingAgents unmodified. Take its directional output as a thesis. Hand to a standalone `options_strategist.py` that calls Alpaca MCP. Ship in a weekend. **This is the starting point.**
- **Path B — native fork.** Modify the LangGraph graph, replace the trader's output schema with the structured trade proposal, migrate strategist + risk manager into graph nodes. **End state.**

Default to Path A until the end-to-end loop is proven, then migrate piece by piece.

## Backend: Alpaca

- Data + execution backend is **Alpaca** (paper trading by default, free, options enabled at the highest level).
- Options orders go through the official `alpacahq/alpaca-mcp-server` MCP server — the entire options stack is exposed as MCP tools, so agents place trades by calling tools rather than a hand-written SDK wrapper.
- Multi-leg orders (up to 4 legs, e.g. iron condor) are a single API call.
- Historical options data from Alpaca is NBBO quotes — **fills-at-mid in backtests are optimistic**; slippage assumptions matter more than for equities.

## Known traps (call these out when relevant)

- **Look-ahead bias**: TradingAgents issue #203 documents the framework pulling current info into backtests. Worse for options because IV is forward-looking. Cap the date in every tool call, including chain pulls and IV reads.
- **Daily cadence is wrong for short DTE**: TradingAgents runs once per ticker per day. 0DTE / weeklies need an intraday loop with separate memory — don't reuse the daily memory store for short-dated.
- **Position sizing is non-linear in options**: $250 max loss on a vertical ≠ $250 of long premium that vol-crushes overnight. The risk manager must distinguish defined-risk structures from naked premium.
- **Margin checks happen at the broker** (Alpaca enforces level requirements), but pre-check before sending — failed orders waste agent turns.

## Stack assumptions (until decided otherwise)

- **Python** — TradingAgents is Python; the fork stays Python.
- **LangGraph** for the agent graph (mature in 2026, prompt patterns public).
- **Alpaca MCP server** for chain ingestion + execution. Run locally with paper keys.
- **uv** or **poetry** for env management — pick one when scaffolding; do not mix.

## When working in this repo

- This is a research / open-source project. Wealthberry Labs is open to collaboration. Commits and PRs should be public-readable.
- The whitepaper is the source of truth for scope. If a feature is not in the whitepaper or a follow-up note, ask before adding it — Randy is allergic to scope creep on this project.
- Every code path that proposes or sends an options order must produce a human-readable rationale (structure, Greeks, max loss, breakeven). Education is one of the three explicit goals.
- When the repo gains code, replace this section's "greenfield" notes with real commands (build, test, lint, run-an-agent-on-a-ticker, run-the-MCP-server). Do not invent commands before the tooling is chosen.

<!-- GSD:project-start source:PROJECT.md -->
## Project

**Moneytree**

Moneytree is a fork of [TradingAgents](https://github.com/TauricResearch/TradingAgents) (multi-agent LLM trading framework, v0.2.4) that adds a native options layer. An operator types a directional thesis on a ticker; a LangGraph agent debate produces a typed verdict; an Options Strategist converts it into a structured options position (vertical, calendar, iron condor, etc.) with defined risk, Greeks-aware sizing, and execution via Alpaca paper trading. Educational/research tool, not financial advice.

**Core Value:** A typed thesis on a ticker becomes a structured, defined-risk options trade with human-readable rationale, executed on Alpaca paper — without humans hand-picking strikes.

### Constraints

- **Tech stack**: Python with `uv` for env/dependency management (not poetry, do not mix). Disclaimer: TradingAgents upstream uses pip — fork uses `uv` regardless.
- **LLM provider**: Anthropic Claude (Sonnet 4.6 / Opus 4.7) via `langchain-anthropic`. Not OpenAI, not local, not multi-provider router. Why: strong reasoning for debate; native MCP fit; tool-use reliability.
- **Agent graph**: LangGraph. Why: mature 2026, prompt patterns public.
- **Broker**: Alpaca paper trading; options level 3 enabled.
- **Execution surface**: official `alpacahq/alpaca-mcp-server` MCP server. Multi-leg orders (≤4 legs, e.g. iron condor) are a single API call.
- **Operator surface**: AionUi (`iOfficeAI/AionUi`) consuming Moneytree MCP. No web UI in v0.
- **MCP server SDK**: official `/modelcontextprotocol/python-sdk` v1.12.4+, `FastMCP` API, Pydantic-typed tool returns, `streamable-http` stateless transport with `json_response=True`.
- **Persistence**: Chroma (debate vector memory, read/write across runs); SQLite single file at `./data/*.db` (fill + PnL trade log; final path TBD when scaffolding).
- **Risk model**: defined-risk structures only in v0; Greeks Risk Manager is a hard gate (rejects breaches, not advisory); pre-check margin before broker submit.
- **Disclaimer**: educational/research disclaimer required in code, comments, and operator-visible outputs. Inherits TradingAgents disclaimer.
- **Path discipline**: scope is locked to whitepaper. Anything outside whitepaper or follow-up notes requires explicit ask before adding (Randy is allergic to scope creep on this project).
<!-- GSD:project-end -->

<!-- GSD:stack-start source:STACK.md -->
## Technology Stack

Technology stack not yet documented. Will populate after codebase mapping or first phase.
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
