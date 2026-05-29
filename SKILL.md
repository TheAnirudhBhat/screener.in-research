---
name: screener-research
description: Automates Indian equity stock screening and deep-dive research using Screener.in via Playwright browser automation. Run predefined themed screens (Baid compounders, CDMO, power/infra, consumption, precision aero, or custom), extract results, cross-reference across themes, and deep-dive top picks. Use when the user asks to "screen stocks", "find compounders", "run screener", "research Indian equities", "find undervalued gems", or references Screener.in.
---

# Screener Research

Automate Indian equity screening and research via Screener.in using Playwright browser tools.

## Prerequisites

- User must have Screener.in open and logged in via Playwright (`mcp__playwright__browser_navigate` to `https://www.screener.in/login/`, user completes login manually)
- Playwright MCP tools available: `browser_navigate`, `browser_evaluate`, `browser_snapshot`, `browser_click`

## Workflow

### Phase 1: Run Themed Screens

Navigate directly to screen results via URL pattern:
```
https://www.screener.in/screen/raw/?sort=Market+Capitalization&order=desc&query=<URL_ENCODED_QUERY>&latest=on
```

Check `references/screens.md` for predefined screen queries. To run a custom screen, build the query using Screener's ratio names (see references).

### Phase 2: Extract Results

Use `browser_evaluate` with this JavaScript to extract table data:

```javascript
() => {
  const rows = document.querySelectorAll('table tbody tr');
  const results = [];
  rows.forEach((row) => {
    const cells = row.querySelectorAll('td');
    if (cells.length < 5) return;
    const link = cells[1]?.querySelector('a');
    if (!link) return;
    const ticker = link.getAttribute('href')?.split('/company/')[1]?.replace(/\//g,'').replace('consolidated','') || '';
    const name = link.textContent?.trim() || '';
    const cmp = cells[2]?.textContent?.trim();
    const pe = cells[3]?.textContent?.trim();
    const mcap = cells[5]?.textContent?.trim();
    const roce = cells[8]?.textContent?.trim();
    const opm = cells[9]?.textContent?.trim();
    const de = cells[10]?.textContent?.trim();
    results.push(`${ticker}|${name}|${cmp}|${mcap}|${pe}|${roce}|${opm}|${de}`);
  });
  return `COUNT:${rows.length}\n${results.join('\n')}`;
}
```

### Phase 3: Cross-Reference

When running multiple themed screens, track which stocks appear across 2+ screens. These are high-conviction candidates — they pass multiple independent quality filters.

Build a cross-reference table:
```
Ticker | Name | Screens Hit | MCap | Best Theme Fit
```

Prioritize stocks appearing in 3+ screens. Filter out already-held positions (check `latest_snapshot.json`, `us_stocks.json`).

### Phase 4: Deep-Dive Top Picks

For the top 5-8 cross-referenced stocks, navigate to individual company pages:
```
https://www.screener.in/company/<TICKER>/
```

Extract detailed financials via `browser_evaluate`:

```javascript
() => {
  // Get key metrics from the company page
  const getMetric = (label) => {
    const items = document.querySelectorAll('.company-ratios li, .ratios-table tr');
    for (const item of items) {
      if (item.textContent.includes(label)) {
        const val = item.querySelector('.value, td:last-child, .number');
        return val?.textContent?.trim() || '';
      }
    }
    return '';
  };
  
  // Get quarterly results table
  const quarters = [];
  const qTable = document.querySelector('#quarters');
  if (qTable) {
    const headers = Array.from(qTable.querySelectorAll('th')).map(h => h.textContent.trim());
    const rows = qTable.querySelectorAll('tbody tr');
    rows.forEach(row => {
      const label = row.querySelector('td')?.textContent?.trim();
      const vals = Array.from(row.querySelectorAll('td')).slice(1).map(c => c.textContent.trim());
      quarters.push(`${label}: ${vals.join(' | ')}`);
    });
  }
  
  return JSON.stringify({
    url: window.location.href,
    name: document.querySelector('h1')?.textContent?.trim(),
    quarters: quarters.slice(0, 5)
  });
}
```

For each pick, also fan out a WebSearch agent for recent news, analyst ratings, and thesis validation.

### Phase 5: Verdict

For each deep-dived stock, assign a verdict:

- **BUY** — Passes Baid framework (ROIC > cost of capital, moat, reinvestment runway), reasonable valuation, sector tailwind, no red flags
- **WATCH** — Quality business but valuation stretched OR sector tailwind unclear OR needs one more quarter of data
- **AVOID** — Fails Baid framework (weak moat, capital cycle risk, equity dilution, declining margins)

Output a final ranked table with: Ticker, Name, CMP, MCap, PE, ROCE, Theme Fit, Verdict, One-line Rationale.

## Running Custom Screens

To create a custom screen, use any of these Screener.in ratio names in the query:
- Market Capitalization, Current price, Price to Earning, PEG Ratio
- Sales, Sales growth, Sales growth 3Years, Sales growth 5Years
- Profit after tax, Profit growth, Profit growth 3Years, Profit growth 5Years
- OPM (Operating Profit Margin), Return on capital employed, Return on equity
- Average return on equity 5Years, Average return on equity 3Years
- Debt to equity, Current ratio, Interest Coverage Ratio
- Promoter holding, Change in promoter holding, Pledged percentage
- Price to book value, Price to Sales, Price to Free Cash Flow, EVEBITDA
- Dividend yield, EPS, Enterprise Value
- Return over 3months, 6months, 1year, 3years, 5years
- YOY Quarterly sales growth, YOY Quarterly profit growth
- Sales latest quarter, Profit after tax latest quarter

Combine with AND/OR operators and >, < comparisons. Add `&latest=on` to filter for latest results only.

## Baid Framework Quick Reference

From Gautam Baid (Stellar Wealth Partners, "The Joys of Compounding"):

**Compounding Machine Criteria:**
1. ROIC > cost of capital (ROCE >15% as proxy)
2. Strong sustainable competitive advantage (moat)
3. Ample reinvestment opportunity at high returns (low dividend payout)

**Management Quality Test:**
- 5-10yr organic market share growth
- No equity dilution (warrant/QIP issuance = red flag)
- Stable operating margins across cycles

**Position Sizing:**
- 3-5% starter position, average UP only on execution
- Max 15% single stock, max 30% single sector

**Sector Tailwinds (Stage 2 India bull market):**
- Precision engineering / aerospace exports (FTA-driven)
- Innovator CDMO (AI drug discovery bottleneck)
- Power ancillaries / data center infrastructure
- Domestic consumption (tax cuts + rate cuts)

**Avoid:**
- Traditional IT services (AI headcount disruption)
- Hyper-competitive quick commerce (capital cycle trap)
- Silver at retail euphoria peaks
- Capex-heavy stocks in de-rating cycle
