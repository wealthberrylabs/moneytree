# Requirements: Moneytree

**Defined:** 2026-04-30
**Core Value:** A typed thesis on a ticker becomes a structured, defined-risk options trade with human-readable rationale, executed on Alpaca paper — without humans hand-picking strikes.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Interface

- [ ] **INT-01**: Moneytree exposes its pipeline as an MCP server using `FastMCP` (streamable-http stateless, `json_response=True`) with Pydantic-typed tool returns
- [ ] **INT-02**: AionUi (`iOfficeAI/AionUi`) connects as the primary operator UX surface and successfully invokes Moneytree MCP tools
- [ ] **INT-03**: A CLI Runner accepts `ticker + thesis` as the dev/test path that exercises the same pipeline functions as the MCP tools (no parallel code path)
- [ ] **INT-04**: Operator can request a full pipeline run (`run_full_pipeline(ticker, thesis) -> FillResult`) and receive a structured result without manual stage-by-stage invocation

### Pipeline

- [ ] **PIPE-01**: TradingAgents engine runs upstream v0.2.4 unmodified and produces a typed `AgentVerdict { direction, confidence }` (not BUY/SELL/HOLD strings)
- [ ] **PIPE-02**: Options Strategist consumes `AgentVerdict` plus chain + IV pulled via Alpaca MCP and emits a structured `TradeProposal { structure, legs, max_loss, max_profit, breakeven, greeks, thesis_match_score, rationale }`
- [ ] **PIPE-03**: Greeks Risk Manager aggregates portfolio-level Δ Γ ν Θ, runs scenario PnL at ±1σ and ±2σ underlying moves and ±5 vol points, and emits a `RiskDecision { approved, sizing, rejection_reason? }`
- [ ] **PIPE-04**: Greeks Risk Manager hard-rejects threshold breaches (gate, not advisory) — rejected proposals never reach the Portfolio Manager
- [ ] **PIPE-05**: Portfolio Manager submits the approved + sized multi-leg order via Alpaca MCP in a single API call (≤4 legs)
- [ ] **PIPE-06**: Portfolio Manager does not re-decide structure or size — it only executes and persists

### Data

- [ ] **DATA-01**: Chroma stores debate vector memory and is read/written by the TA engine across runs (daily cadence)
- [ ] **DATA-02**: SQLite (single-file at `./data/`) persists fill confirmations and PnL on every order
- [ ] **DATA-03**: Every market-data tool call (news, fundamentals, chain, IV) is date-capped to prevent look-ahead bias

### Safety

- [ ] **SAFE-01**: Margin requirement is pre-checked locally before any order is submitted to the broker (avoid wasting agent turns on rejected orders)
- [ ] **SAFE-02**: Only defined-risk option structures are permitted in v0 (vertical, calendar, iron condor, butterfly) — naked premium is rejected
- [ ] **SAFE-03**: Risk Manager differentiates defined-risk vs naked premium when sizing (sizing math is non-linear in options)
- [ ] **SAFE-04**: All trades route to Alpaca paper account only — no live keys accepted in v0

### Education

- [ ] **EDU-01**: Every trade-related MCP tool return includes a human-readable rationale field describing structure, Greeks, max loss, and breakeven
- [ ] **EDU-02**: Educational/research disclaimer surfaces in code, comments, and operator-visible outputs (inherits TradingAgents disclaimer)

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Lifecycle

- **LIFE-01**: Time-based exits (DTE-trigger or fixed-hold close)
- **LIFE-02**: Position rolling
- **LIFE-03**: Stop-loss state machine
- **LIFE-04**: Per-position state persistence across restarts

### Cadence

- **CAD-01**: Intraday loop separate from daily debate memory
- **CAD-02**: 0DTE / weeklies support
- **CAD-03**: Multi-ticker batch runs

### Verification

- **VERIF-01**: Backtest harness with explicit slippage model (NBBO fills-at-mid is optimistic)
- **VERIF-02**: Replay-mode backtest with date-capped tool calls

### Architecture

- **ARCH-01**: Path B native LangGraph fork (replace trader output schema; migrate Strategist + RM into graph nodes)
- **ARCH-02**: Strategist split into separate `options_analyst` + `options_strategist` per whitepaper
- **ARCH-03**: Borrow operator's LLM session via MCP `ctx.session.create_message` instead of holding own Anthropic key

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Real-money trading | v0 paper-only; safety + compliance bar not yet met |
| Streamlit / Flask / Next.js / custom web UI | AionUi is the UI; avoid duplicating operator surface |
| OpenAI / local / multi-provider LLM router | Locked to Anthropic Claude in v0 (decision 2026-04-30) |
| `poetry` for env management | Locked to `uv` in v0 (decision 2026-04-30) |
| Custom Black-Scholes / binomial Greeks engine | Use Alpaca-provided Greeks; computing locally is non-trivial and out of v0 scope |
| Hand-rolled Alpaca SDK wrapper | Use official `alpacahq/alpaca-mcp-server` instead |
| Naked premium structures | Sizing math + risk gating not designed for unlimited-loss positions |
| Live broker margin computation by Moneytree | Broker enforces margin; Moneytree pre-checks only to avoid wasted turns |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| INT-01 | Phase 3 | Pending |
| INT-02 | Phase 3 | Pending |
| INT-03 | Phase 3 | Pending |
| INT-04 | Phase 3 | Pending |
| PIPE-01 | Phase 2 | Pending |
| PIPE-02 | Phase 2 | Pending |
| PIPE-03 | Phase 2 | Pending |
| PIPE-04 | Phase 2 | Pending |
| PIPE-05 | Phase 2 | Pending |
| PIPE-06 | Phase 2 | Pending |
| DATA-01 | Phase 2 | Pending |
| DATA-02 | Phase 1 | Pending |
| DATA-03 | Phase 1 | Pending |
| SAFE-01 | Phase 2 | Pending |
| SAFE-02 | Phase 2 | Pending |
| SAFE-03 | Phase 2 | Pending |
| SAFE-04 | Phase 1 | Pending |
| EDU-01 | Phase 2 | Pending |
| EDU-02 | Phase 1 | Pending |

**Coverage:**
- v1 requirements: 19 total
- Mapped to phases: 19
- Unmapped: 0

**Per-phase counts:**
- Phase 1 (Foundation): 4 requirements (DATA-02, DATA-03, SAFE-04, EDU-02)
- Phase 2 (Pipeline Core): 11 requirements (PIPE-01..06, DATA-01, SAFE-01, SAFE-02, SAFE-03, EDU-01)
- Phase 3 (Operator Surface): 4 requirements (INT-01, INT-02, INT-03, INT-04)

---
*Requirements defined: 2026-04-30*
*Last updated: 2026-04-30 — traceability mapped during roadmap creation*
