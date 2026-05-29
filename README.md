# screener-research

A Claude Code skill that turns [Screener.in](https://www.screener.in) into a programmable equity research engine via browser automation.

## What it does

**Screen → Extract → Cross-reference → Deep-dive → Verdict** — all within Screener.in.

Instead of manually building queries and tabbing between company pages, this skill:

1. **Runs themed screens** — predefined or custom queries using Screener's 40+ financial ratios
2. **Extracts structured data** — parses result tables via JavaScript evaluation
3. **Cross-references** — finds stocks passing multiple independent screens (higher conviction)
4. **Deep-dives companies** — pulls 12 years of financials, quarterly trends, peer comparisons, shareholding patterns, and filings from Screener's company pages
5. **Auto red-flag scanner** — detects declining ROCE, rising debt, negative CFO, promoter selling, OPM compression
6. **Peer + valuation context** — PE vs Industry PE, peer ranking on PE and ROCE, relative valuation signals
7. **Outputs verdicts** — ranked shortlist with BUY / WATCH / AVOID based on your framework

## What data is available

Screener.in provides **10 structured sections per company** — enough for full fundamental analysis without leaving the site:

| Data | History | What you get |
|------|---------|-------------|
| Key Ratios | Current | MCap, PE, ROCE, ROE, EPS, 52W H/L, Book Value, Industry PE |
| Pros/Cons | Current | Auto-generated red/green flags |
| Peer Comparison | Current | 9 peers with PE, ROCE, FII%, debt, 6mo return |
| Quarterly Results | 13 quarters | Sales, expenses, OPM, PAT, EPS per quarter |
| Annual P&L | 12 years | Revenue, profit, margins + compounded growth rates |
| Balance Sheet | 12 years | Equity, debt, reserves, fixed assets |
| Cash Flow | 12 years | CFO, investing, financing activities |
| Financial Ratios | 12 years | ROCE, debtor days, inventory days, payable days |
| Shareholding | 12 quarters | Promoter, FII, DII, public % with trends |
| Documents | 100+ filings | BSE/NSE announcements, earnings transcripts |

All data is from BSE/NSE filings — authoritative, not estimated.

## Auto red-flag scanner

The company page extractor automatically detects these warning signals:

| Red Flag | What it checks | Threshold |
|----------|---------------|-----------|
| ROCE declining | Last 3 years of ROCE from ratio history | Consecutive decline |
| Debt rising | Borrowings trend from balance sheet | >50% growth over 3 years |
| Negative CFO | Operating cash flow from cash flow statement | Latest year < 0 |
| Promoter selling | Shareholding pattern trend | >1.5pp decline over recent quarters |
| OPM compression | Operating profit margin from quarterly results | >3pp drop over 4 quarters |

Stocks with 2+ red flags default to AVOID. The extractor also returns peer ranking (PE and ROCE rank among 9 peers) and valuation context (PE vs Industry PE premium/discount).

## Screen caching & diff-aware re-screening

Re-running a screen compares against the cached previous run:

```
Screen: baid-compounders (last run: 14 days ago)
Results: 42 → 45 (+3)
NEW (3): TICKER_A, TICKER_B, TICKER_C ← auto deep-dive
DROPPED (0): none
RETAINED (42): stable
```

New entrants get automatic deep-dive priority. Caches persist at `references/cache/<screen-name>.json`. A persistent watchlist at `references/watchlist.json` tracks entry triggers across sessions.

## Portfolio-aware screening

When a portfolio snapshot exists, the skill:
- **Excludes held stocks** from deep-dive (marks them `HELD` in output)
- **Flags sector correlation** ("Same sector as ARTEMISMED — adds concentration")
- **Suggests role-bucket fit** ("NEULANDLAB → growth bucket, TDPOWERSYS → cyclicals")

Works standalone if no portfolio file exists.

## Predefined screens

These are starting points — not mandates. Use your own framework and criteria.

| Screen | What it finds | Filters |
|--------|--------------|---------|
| `baid-compounders` | Quality compounders | ROCE >15%, 5yr growth >15%, OPM >15%, low debt |
| `cdmo-innovator` | High-margin pharma CDMOs | 5yr growth >20%, OPM >25%, ROCE >20% |
| `power-infra` | Power/data center plays | 3yr growth >25%, ROCE >15% |
| `consumption` | Domestic consumption | Established quality, ROE >15%, steady growth |
| `precision-aero` | Aerospace/defense | High growth, OPM >15%, ROCE >15% |
| `deep-value` | Low PE + quality | PE <20, ROCE >20%, 5yr growth >15% |
| `quality-midcap` | Consistent midcaps | Avg ROE 5Y >18%, low debt, low pledge |

Custom screens work with any of Screener's ratio names — see `references/screens.md`.

## Dependencies

Requires **one** browser automation MCP:

- **[Playwright MCP](https://github.com/anthropics/mcp-playwright)** (recommended) — headless, doesn't interfere with main browser
- **[Claude-in-Chrome MCP](https://github.com/anthropics/claude-in-chrome)** — uses your existing Chrome session

### Setup

1. Install a browser automation MCP in Claude Code
2. Navigate to `https://www.screener.in/login/` in the automation browser
3. Log in manually (Screener requires authentication)
4. Say "run screener" or "screen stocks" or "find compounders"

## How it works

```
"Find undervalued CDMO stocks"
            │
            ▼
   ┌────────────────────┐
   │  Run screen query   │  Screener.in/screen/raw/?query=...
   │  on Screener.in     │
   └────────┬───────────┘
            │
            ▼
   ┌────────────────────┐
   │  Extract table      │  browser_evaluate() → structured data
   │  (ticker, PE, ROCE) │
   └────────┬───────────┘
            │
            ▼
   ┌────────────────────┐
   │  Cross-reference    │  Stocks in 2+ screens = high conviction
   │  across themes      │
   └────────┬───────────┘
            │
            ▼
   ┌────────────────────┐
   │  Deep-dive top 5-8  │  /company/TICKER/ → 12yr financials,
   │  on Screener        │  peers, shareholding, quarterly trends
   └────────┬───────────┘
            │
            ▼
   ┌────────────────────┐
   │  Ranked output      │  BUY / WATCH / AVOID + rationale
   │  with verdicts      │
   └────────────────────┘
```

## Example frameworks

The skill is framework-agnostic. Here are some approaches users have applied with it:

| Framework | Key screen criteria | Source |
|-----------|-------------------|--------|
| Quality compounding | ROCE >15%, stable margins, no dilution, reinvestment runway | Gautam Baid, Saurabh Mukherjea |
| GARP | PEG <1.5, earnings acceleration, reasonable PE vs growth | Peter Lynch |
| Coffee-can | 10yr revenue CAGR >10%, ROCE >15%, MCap >100 Cr | Marcellus |
| Deep value | PE <15, PB <2, healthy balance sheet, catalyst | Graham/Greenblatt |
| Momentum + quality | 6mo return >20%, ROCE >15%, profit growth >20% | O'Neil/CAN SLIM adapted |
| Thematic | Sector-specific + company fundamentals | Varies by theme |

## License

MIT
