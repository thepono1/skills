# quant_v4 Trading Block Library

Ready-to-paste MDX for trading-domain plan blocks, built **only from blocks the
stock renderer already ships** (`diagram`, `mermaid`, `table`, `callout`,
`data-model`, `json-explorer`, `custom-html`). These render TODAY in local-files
mode — no core fork required. For a fully native React block (`<RiskGate>` etc.)
see `agent-native` fork branch `feat/quant-blocks`.

Authoring rules carried over from the catalog: never hard-code hex/rgb — use
`--wf-*` tokens; never set fonts; capitalized components must self-close or have a
closing tag; block ids must be unique.

---

## 1. Risk Gate — Kelly → Circuit Breaker sequence

The CLAUDE.md invariant: Kelly check FIRST (negative ⇒ zero allocation), circuit
breaker SECOND (per-session global), proceed only if both pass. Render as a
`diagram` plus a `risk` callout that names the live thresholds.

```mdx
<Diagram id="risk-gate-seq" data={{
  caption: "Order sizing gate — sequence is fixed: Kelly then circuit breaker.",
  html: `<div class="diagram-panel">
    <div class="diagram-card">1 · Kelly (per-trade)<div class="diagram-muted">fractional 1–5% · negative ⇒ 0</div></div>
    <div class="diagram-pill">pass ▶</div>
    <div class="diagram-card">2 · Circuit Breaker (per-session)<div class="diagram-muted">win-rate loss threshold · 5% daily stop</div></div>
    <div class="diagram-pill">pass ▶</div>
    <div class="diagram-card">3 · Execute<div class="diagram-muted">re-check price ±0.5% · szDecimals · $10 min</div></div>
  </div>`,
  css: `.diagram-panel{display:flex;gap:.6rem;align-items:center;flex-wrap:wrap}
    .diagram-card{padding:.6rem .8rem;border:1px solid var(--wf-border);border-radius:8px;background:var(--wf-surface)}
    .diagram-muted{color:var(--wf-muted);font-size:.8em;margin-top:.25rem}
    .diagram-pill{color:var(--wf-muted)}`
}} />

<Callout id="risk-gate-note" data={{ tone: "risk", body: "A **Kelly pass never overrides a tripped circuit breaker**. Negative Kelly skips the trade without consulting the breaker. Hard stop max 2%, trailing activates at +0.5% (trails 2% from peak). Source: `kaleo/risk_engine.py`, `kaleo/circuit_breaker.py`, `kaleo/position_sizing.py`." }} />
```

---

## 2. Backtest Metrics — Sharpe / DSR / PSR / Drawdown card

Use a `table` for the headline stats (scannable, diff-able across plan versions).
Add a `callout` for the deflation caveat so reviewers read significance, not just
the raw Sharpe.

```mdx
<Table id="bt-metrics" data={{
  density: "compact",
  columns: ["Metric", "Value", "Gate", "Pass?"],
  rows: [
    ["WF OOS Sharpe", "1.21", "> 1.0", "✅"],
    ["DSR (deflated)", "0.60", "> 0.5", "✅"],
    ["PSR", "0.94", "> 0.90", "✅"],
    ["Max Drawdown", "-8.3%", "< 15%", "✅"],
    ["Trades (OOS)", "412", "> 100", "✅"]
  ]
}} />

<Callout id="bt-caveat" data={{ tone: "warning", body: "Report **DSR/PSR, not raw Sharpe** — raw Sharpe ignores multiple-testing inflation. Numbers must come from the walk-forward harness output, never from memory. Label in-sample vs out-of-sample explicitly." }} />
```

---

## 3. Position Snapshot — live HL state

Live positions are JSON from the Hyperliquid `clearinghouseState` endpoint. Use
`json-explorer` for the raw payload (collapsible) and a `table` for the human
read. NEVER inline a capital figure from memory — paste the live fetch or label
it cached-with-timestamp.

```mdx
<Table id="pos-snapshot" data={{
  columns: ["Field", "Value", "Source"],
  rows: [
    ["Wallet", "0x4F62…1597", "live"],
    ["Equity", "$<live-fetch>", "HL clearinghouseState"],
    ["Open position", "ETH short (CLMM hedge)", "assetPositions"],
    ["As of", "<timestamp>", "fetch time"]
  ]
}} />

<Json id="pos-raw" data={{
  title: "clearinghouseState (live payload)",
  collapsedDepth: 1,
  json: { marginSummary: { accountValue: "<paste live>" }, assetPositions: [] }
}} />
```

---

## 4. Strategy State Machine — regime / flip / carry-gate

State transitions read clearer as textual grammar, so `mermaid` is the right
call (the one block where Mermaid beats a spatial diagram).

```mdx
<Mermaid id="strat-sm" data={{
  caption: "Cross-DEX funding strategy state machine.",
  source: `stateDiagram-v2
    [*] --> Flat
    Flat --> Armed: carry_gate pass (PSR/DSR ok)
    Armed --> Long: funding flip + Kelly>0
    Armed --> Short: funding flip + Kelly>0
    Long --> Flat: circuit_breaker trip OR flip_gate
    Short --> Flat: circuit_breaker trip OR flip_gate
    Flat --> Halted: dd_halt (5% daily)
    Halted --> Flat: session reset`
}} />
```

---

## How to use in a plan

1. `export AGENT_NATIVE_PLANS_MODE=local-files`
2. `/visual-plan` (it reads this file + the quant_v4 grounding overlay in SKILL.md).
3. Paste the blocks above into `plans/<slug>/plan.mdx`, fill live values.
4. `npx @agent-native/core@latest plan local check --dir plans/<slug>`
5. `npx @agent-native/core@latest plan local serve --dir plans/<slug> --kind plan --open`
