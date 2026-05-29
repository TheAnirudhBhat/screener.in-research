# screener-research

A Claude Code skill that automates Indian equity stock screening and deep-dive research using [Screener.in](https://www.screener.in) via browser automation.

## What it does

Turns Screener.in into a programmable research engine. Instead of manually building queries and comparing results across tabs, this skill:

1. **Runs themed screens** — predefined queries based on Gautam Baid's compounding framework (ROCE, growth, margins, leverage)
2. **Extracts structured data** — parses Screener's result tables into ticker-level data via JavaScript evaluation
3. **Cross-references across themes** — finds stocks that pass multiple independent quality filters (e.g., a stock that shows up in both "CDMO innovator" and "Baid compounder" screens is higher conviction)
4. **Deep-dives top picks** — navigates to individual company pages for detailed financials, then fans out WebSearch agents for recent news and analyst ratings
5. **Outputs ranked verdicts** — BUY / WATCH / AVOID with one-line rationale for each pick

## Predefined screens

| Screen | What it finds | Key filters |
|--------|--------------|-------------|
| `baid-compounders` | Baid-framework quality compounders | ROCE >15%, 5yr sales+profit CAGR >15%, OPM >15%, D/E <0.5 |
| `cdmo-innovator` | High-margin pharma/chem CDMOs | Sales growth 5Y >20%, OPM >25%, ROCE >20% |
| `power-infra` | Power ancillaries, data center infra | 3yr sales+profit growth >25%, ROCE >15% |
| `consumption` | Domestic consumption beneficiaries | Established quality (MCap >5K Cr), ROE >15%, steady growth |
| `precision-aero` | Aerospace exports, defense electronics | High growth (5yr profit >25%), OPM >15% |
| `deep-value` | Low PE + high quality | PE <20, ROCE >20%, 5yr growth >15% |
| `quality-midcap` | Consistent quality midcaps | Avg ROE 5Y >18%, low debt, low pledge |

Custom screens can be built using any of Screener.in's 40+ available ratios.

## Dependencies

Requires **one** of the following browser automation MCP servers:

- **[Playwright MCP](https://github.com/anthropics/mcp-playwright)** (recommended) — `mcp__playwright__browser_navigate`, `browser_evaluate`, `browser_snapshot`
- **[Claude-in-Chrome MCP](https://github.com/anthropics/claude-in-chrome)** — `mcp__claude-in-chrome__navigate`, `javascript_tool`, `get_page_text`

The skill works with either. Playwright is preferred because it runs headless and doesn't interfere with your main browser.

### Setup

1. Install Playwright MCP or Claude-in-Chrome MCP in your Claude Code config
2. Navigate to `https://www.screener.in/login/` in the automation browser
3. Log in manually (Screener requires authentication for full data)
4. Invoke the skill: say "run screener", "screen stocks", "find compounders", etc.

## How it works

```
User: "find undervalued CDMO stocks"
                │
                ▼
    ┌─────────────────────┐
    │  Navigate to         │
    │  Screener.in/screen/ │
    │  with CDMO query     │
    └──────────┬──────────┘
               │
               ▼
    ┌─────────────────────┐
    │  browser_evaluate()  │
    │  Extract table data  │
    │  (ticker, CMP, PE,   │
    │   ROCE, OPM, D/E)   │
    └──────────┬──────────┘
               │
               ▼
    ┌─────────────────────┐
    │  Cross-reference     │
    │  with other screens  │
    │  (multi-theme hits   │
    │   = high conviction) │
    └──────────┬──────────┘
               │
               ▼
    ┌─────────────────────┐
    │  Deep-dive top picks │
    │  • Company page data │
    │  • WebSearch agents  │
    │  • Analyst ratings   │
    └──────────┬──────────┘
               │
               ▼
    ┌─────────────────────┐
    │  Ranked output       │
    │  BUY / WATCH / AVOID │
    │  with rationale      │
    └─────────────────────┘
```

## Investment framework

The screens are built on Gautam Baid's compounding machine criteria (from *The Joys of Compounding*):

- **ROIC > cost of capital** — ROCE >15% as proxy
- **Sustainable competitive advantage** — stable margins across cycles
- **Reinvestment runway** — organic growth without equity dilution
- **Sector tailwinds** — government stimulus alignment, structural demand shifts
- **Position sizing** — 3-5% starter, max 15% single stock, max 30% single sector

### Stage 2 bull market sectors (India, 2026)

Per Baid's FWS116 podcast:
- Precision engineering / aerospace exports (FTA-driven)
- Innovator CDMO (AI drug discovery bottleneck shifts to manufacturing)
- Power ancillaries / data center infrastructure
- Domestic consumption (tax cuts + RBI rate cuts)

### Sectors to avoid
- Traditional IT services (AI headcount disruption)
- Hyper-competitive quick commerce (capital cycle trap)
- Capex-heavy stocks in de-rating cycle

## License

MIT
