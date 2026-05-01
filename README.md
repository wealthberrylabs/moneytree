# Moneytree

> A typed thesis on a ticker becomes a structured, defined-risk options trade with human-readable rationale, executed on Alpaca paper — without humans hand-picking strikes.

**Status:** Greenfield · Phase 1 (Foundation) starting · v0 paper-only

**Disclaimer:** Educational / research only. Not financial advice. Inherits the [TradingAgents](https://github.com/TauricResearch/TradingAgents) disclaimer.

---

## What this is

Moneytree is a fork of [TradingAgents](https://github.com/TauricResearch/TradingAgents) (multi-agent LLM trading framework, v0.2.4) that adds a **native options layer**. TradingAgents only emits `BUY/SELL/HOLD` on equities. Moneytree extends the LangGraph pipeline so a directional thesis is converted into a structured options position (vertical, calendar, iron condor, butterfly) with defined risk, Greeks-aware sizing, and broker execution via [Alpaca paper trading](https://alpaca.markets/) through the official `alpacahq/alpaca-mcp-server` MCP server.

The operator types a ticker and a thesis into [AionUi](https://github.com/iOfficeAI/AionUi) (the operator UX surface). AionUi calls Moneytree's MCP tools. Moneytree drives the four-stage pipeline and returns a structured fill result.

## Architecture (v0)

```
AionUi (operator UX, MCP client)
  └── Moneytree MCP Server (FastMCP, stdio default)
        ├── TradingAgents Engine        — direction + confidence
        ├── Options Strategist          — chain + IV → trade proposal
        ├── Greeks Risk Manager         — hard gate + sizing
        └── Portfolio Manager           — multi-leg submit + persist
              └── Alpaca MCP Paper      — execution
              └── SQLite                — fill + PnL log
```

Full diagram: [`architecture.html`](./architecture.html). Design source-of-truth: `moneytree-whitepaper.pdf` (April 2026, Randy Ellis, Wealthberry Labs).

## Stack

| Layer | Choice |
|---|---|
| Env / packaging | [`uv`](https://github.com/astral-sh/uv) |
| Agent graph | [LangGraph](https://github.com/langchain-ai/langgraph) (extends upstream TradingAgents) |
| LLM provider | Anthropic Claude (Sonnet 4.6 / Opus 4.7) via `langchain-anthropic` |
| Operator UX | [AionUi](https://github.com/iOfficeAI/AionUi) (MCP client) |
| Operator surface | [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk) — FastMCP, stdio transport |
| Broker | [Alpaca MCP Server](https://github.com/alpacahq/alpaca-mcp-server) — paper trading, options level 3 |
| Persistence | Chroma (debate vector memory, owned by TA upstream) + SQLite (fill + PnL log) |

## v0 scope

**In:** entry-only lifecycle, daily cadence, defined-risk structures, single ticker per run, Path A (wrapper around upstream TradingAgents).

**Out:** real-money trading, exits / rolls / stops, 0DTE / weeklies / intraday loop, backtest harness, multi-ticker batch, custom Black-Scholes engine, web UI.

See [`.planning/REQUIREMENTS.md`](./.planning/REQUIREMENTS.md) for the full v1 / v2 / out-of-scope breakdown and [`.planning/ROADMAP.md`](./.planning/ROADMAP.md) for phase structure.

## Roadmap

| Phase | Goal | Reqs |
|---|---|---|
| 1 — Foundation | uv scaffold, Pydantic contracts, Alpaca-MCP wrapper, SQLite, disclaimer plumbing | 4 |
| 2 — Pipeline Core | `run_full_pipeline` end-to-end on Alpaca paper (verdict → proposal → risk gate → submit + persist) | 11 |
| 3 — Operator Surface | FastMCP server, AionUi integration verified, CLI Runner shares pipeline functions | 4 |

## Known traps (built into the design)

- **Look-ahead bias**: every market-data tool call (news, fundamentals, chain, IV) is date-capped. TradingAgents Issue #203 documents the upstream framework pulling current info into backtests; IV is forward-looking so this matters more for options.
- **Daily cadence ≠ 0DTE**: TradingAgents runs once per ticker per day. 0DTE / weeklies need an intraday loop with separate memory — deferred to v2.
- **Sizing is non-linear in options**: `$250 max loss on a vertical ≠ $250 of long premium that vol-crushes overnight`. The Risk Manager distinguishes defined-risk structures from naked premium.
- **Margin checks happen at the broker**: Alpaca enforces level requirements, but Moneytree pre-checks before sending — failed orders waste agent turns.

## Contributing

Wealthberry Labs is open to collaboration. Commits and PRs are public-readable. Keep contributions inside the whitepaper scope; surface scope-creep proposals as issues first.

## License

Apache License 2.0 — see [`LICENSE`](./LICENSE) and [`NOTICE`](./NOTICE). Inherits the upstream TradingAgents license.
