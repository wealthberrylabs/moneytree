# Moneytree

## What This Is

Moneytree is a fork of [TradingAgents](https://github.com/TauricResearch/TradingAgents) (multi-agent LLM trading framework, v0.2.4) that adds a native options layer. An operator types a directional thesis on a ticker; a LangGraph agent debate produces a typed verdict; an Options Strategist converts it into a structured options position (vertical, calendar, iron condor, etc.) with defined risk, Greeks-aware sizing, and execution via Alpaca paper trading. Educational/research tool, not financial advice.

## Core Value

A typed thesis on a ticker becomes a structured, defined-risk options trade with human-readable rationale, executed on Alpaca paper — without humans hand-picking strikes.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] CLI Runner accepts ticker + operator-typed thesis (dev/test path)
- [ ] Moneytree exposes pipeline as MCP server (FastMCP + Pydantic tool returns; streamable-http stateless)
- [ ] AionUi connects as primary operator UX surface, consumes Moneytree MCP
- [ ] TradingAgents engine produces typed `direction + confidence` (not BUY/SELL/HOLD strings)
- [ ] Options Strategist consumes verdict + chain + IV via Alpaca MCP, emits structured `TradeProposal` (legs, max_loss, max_profit, breakeven, Greeks, thesis_match_score)
- [ ] Greeks Risk Manager aggregates Δ Γ ν Θ, runs scenario PnL at ±1σ / ±2σ underlying moves and ±5 vol points, hard-rejects breaches, emits `RiskDecision` with sizing
- [ ] Portfolio Manager submits multi-leg order via Alpaca MCP and writes fill + PnL to SQLite
- [ ] Every order produces human-readable rationale (structure, Greeks, max loss, breakeven) — education goal
- [ ] All market-data tool calls date-capped (anti look-ahead, including chain pulls and IV reads)
- [ ] Margin pre-check before broker submission (avoid broker-side rejection wasting agent turns)
- [ ] Educational disclaimer surfaced in code, comments, and operator-visible outputs

### Out of Scope

- Real-money trading — Alpaca paper only in v0
- Exit logic, rolls, stop-loss state machines — entry-only lifecycle in v0; operator closes manually in Alpaca UI
- 0DTE / weeklies / intraday loop — daily cadence only in v0; whitepaper warns daily memory store is wrong for short DTE
- Backtest harness — deferred; NBBO fills-at-mid are optimistic and slippage model needs explicit treatment before any backtest is trustworthy
- Multi-ticker per run — single ticker per invocation in v0
- Native LangGraph fork (Path B) — start with Path A wrapper around upstream TradingAgents; migrate piece-by-piece after end-to-end loop is proven
- Streamlit / Flask / Next.js / custom web UI — AionUi is the UI
- Strategist split into separate `options_analyst` + `options_strategist` nodes — collapsed into one node for v0; split later if prompt grows unwieldy
- Custom Black-Scholes / binomial Greeks engine — use Alpaca-provided Greeks where available

## Context

- Fork of TradingAgents v0.2.4 (Tauric Research). Upstream is Python + LangGraph.
- Source of truth for scope: `moneytree-whitepaper.pdf` (April 2026, Randy Ellis, Wealthberry Labs), local copy at `~/Downloads/moneytree-whitepaper.pdf`.
- Greenfield code-wise as of 2026-04-30. Repo is public-readable; Wealthberry Labs is open to collaboration.
- TradingAgents only emits BUY/SELL/HOLD on equities. Moneytree's value-add is the options conversion layer plus a Greeks-aware risk gate.
- AionUi (iOfficeAI/AionUi) is an Electron+React multi-agent cowork app that already consumes MCP — chosen as operator UX so we don't hand-build a web frontend in v0.
- TradingAgents' Issue #203 documents the framework pulling current info into backtests; Moneytree must cap dates more aggressively because IV is forward-looking.

## Constraints

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

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Path A wrapper as starting point (not Path B native fork) | Weekend-shippable end-to-end loop; migrate piece-by-piece after proven | — Pending |
| `uv` for env management | Fast, single binary, modern lockfile semantics | — Pending |
| Anthropic Claude as LLM provider | Strong reasoning for debate; native MCP fit; tool-use solid | — Pending |
| Entry-only lifecycle in v0 | Simplest end-to-end loop; defer exits / rolls / stops until entry proven | — Pending |
| AionUi as operator UX surface | Already consumes MCP; symmetric with Alpaca MCP; avoids hand-building a web UI | — Pending |
| Moneytree exposed as MCP server via FastMCP + Pydantic | Typed tool boundaries; auditable per-call args/returns; clean MCP-in / MCP-out architecture | — Pending |
| Strategist as single collapsed node (not split per whitepaper) | OK for v0; split into `options_analyst` + `options_strategist` later if prompt grows unwieldy | — Pending |
| Daily cadence only in v0 | Daily debate-memory store is wrong for 0DTE; defer intraday loop | — Pending |
| Date-cap every market-data tool call | TradingAgents Issue #203 look-ahead bias; IV is forward-looking so worse for options | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-30 after initialization*
