# Roadmap: Moneytree

## Overview

Moneytree is a fork of TradingAgents v0.2.4 that adds an options layer. The journey converts a typed thesis on a ticker into a structured, defined-risk options trade executed on Alpaca paper. We start by laying the typed contracts and external client wiring (Foundation), then implement the four-stage pipeline that turns a verdict into a fill (Pipeline Core), then expose the pipeline as an MCP server consumable by AionUi while keeping a CLI dev/test path (Operator Surface). Path A (wrapper around upstream TradingAgents) is the starting point; the end-to-end loop is intended to ship in a weekend.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation** - uv scaffold, Pydantic contracts, Alpaca-MCP client wiring, disclaimer plumbing
- [ ] **Phase 2: Pipeline Core** - TA verdict + Options Strategist + Greeks Risk Manager + Portfolio Manager end-to-end
- [ ] **Phase 3: Operator Surface** - FastMCP server, CLI Runner sharing pipeline functions, AionUi integration verified

## Phase Details

### Phase 1: Foundation
**Goal**: A scaffolded Python package with typed contracts, an Alpaca-MCP client, persistence wiring, and educational disclaimer plumbing — ready to be consumed by the pipeline.
**Depends on**: Nothing (first phase)
**Requirements**: DATA-02, DATA-03, SAFE-04, EDU-02
**Success Criteria** (what must be TRUE):
  1. A developer can `uv sync` and `uv run` the package; the project layout is established and importable.
  2. Pydantic models `AgentVerdict`, `TradeProposal`, `RiskDecision`, and `FillResult` are defined and round-trip serializable for use as MCP tool returns.
  3. An Alpaca-MCP client wrapper is callable from Python, configured for paper trading only (live keys raise an explicit error), and every market-data helper accepts and enforces a `date_cap` argument. (Note: TradingAgents' own `propagate(ticker, date)` already date-caps the analyst stages upstream; this wrapper covers the Strategist + PM call sites that bypass TA.)
  4. A SQLite trade-log file is created at `./data/` on first use with a schema for fill confirmations and PnL.
  5. The educational/research disclaimer appears in the package's top-level module docstring, in the CLI/MCP banner string, and is attached to every operator-visible structured output.
**Plans**: TBD

### Phase 2: Pipeline Core
**Goal**: A function `run_full_pipeline(ticker, thesis) -> FillResult` runs the four-stage pipeline (TA verdict → Strategist proposal → Risk gate + sizing → PM submit + persist) end-to-end against Alpaca paper.
**Depends on**: Phase 1
**Requirements**: PIPE-01, PIPE-02, PIPE-03, PIPE-04, PIPE-05, PIPE-06, DATA-01, SAFE-01, SAFE-02, SAFE-03, EDU-01
**Success Criteria** (what must be TRUE):
  1. Given a ticker and thesis, the upstream TradingAgents engine runs unmodified via `ta.propagate(ticker, date_cap)` and the wrapper extracts a typed `AgentVerdict { direction, confidence, reasoning }` from the returned `decision` dict (mapping `action → direction`, carrying `reasoning` forward into the trade rationale); Chroma read/write is owned by TA — Moneytree configures TA's path and does not spawn a second client.
  2. The Options Strategist consumes the verdict plus Alpaca-MCP chain + IV (date-capped) and returns a `TradeProposal` whose structure is one of {vertical, calendar, iron condor, butterfly} with `legs ≤ 4`, `max_loss`, `max_profit`, `breakeven`, `greeks`, `thesis_match_score`, and a human-readable `rationale`; naked-premium proposals are rejected at this stage.
  3. The Greeks Risk Manager returns a `RiskDecision` containing aggregated Δ Γ ν Θ, scenario PnL at ±1σ / ±2σ underlying and ±5 vol points, an explicit `approved` flag with sizing, and a `rejection_reason` when threshold breaches occur; rejected proposals never reach the Portfolio Manager.
  4. For an approved proposal, the Portfolio Manager passes a local margin pre-check, submits the multi-leg order via Alpaca MCP in one API call, and persists the fill + PnL to SQLite — without re-deciding structure or size.
  5. The returned `FillResult` includes a human-readable rationale describing structure, Greeks, max loss, and breakeven, suitable for direct display to the operator.
**Plans**: TBD

### Phase 3: Operator Surface
**Goal**: The pipeline is exposed as a FastMCP server (the production operator path), AionUi connects to it and successfully invokes its tools, and a CLI Runner shares the same pipeline functions for dev/test.
**Depends on**: Phase 2
**Requirements**: INT-01, INT-02, INT-03, INT-04
**Success Criteria** (what must be TRUE):
  1. A FastMCP server starts via a `moneytree-mcp` console-script entrypoint, runs **stdio transport by default** (so AionUi can spawn it as a subprocess), optionally also exposes `streamable-http` for HTTP clients, and registers tools `run_thesis`, `propose_trade`, `assess_risk`, `submit_trade`, `list_positions`, and `run_full_pipeline` — each with Pydantic-typed returns from Phase 1.
  2. AionUi (`iOfficeAI/AionUi`) connects to the running server, lists Moneytree's tools, and successfully invokes `run_full_pipeline(ticker, thesis)` end-to-end against Alpaca paper, receiving a structured `FillResult`.
  3. A CLI Runner accepts `ticker` and `thesis` positional arguments and exercises the same in-process pipeline functions as the MCP tools (no parallel implementation), printing the same human-readable rationale.
  4. The operator can issue a single `run_full_pipeline(ticker, thesis)` call from either AionUi or the CLI and receive a complete `FillResult` without manual stage-by-stage invocation.
**Plans**: TBD
**UI hint**: yes

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation | 0/TBD | Not started | - |
| 2. Pipeline Core | 0/TBD | Not started | - |
| 3. Operator Surface | 0/TBD | Not started | - |
