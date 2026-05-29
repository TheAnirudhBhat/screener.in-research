---
name: screener-research
description: Automates Indian equity stock screening and deep-dive research using Screener.in via Playwright browser automation. Run predefined or custom screens, extract results, cross-reference across themes, and deep-dive companies using Screener's financials (12yr P&L, balance sheet, cash flow, ratios, shareholding, peers). Use when the user asks to "screen stocks", "find stocks", "run screener", "research Indian equities", "find compounders", "find undervalued stocks", or references Screener.in.
---

# Screener Research

Automate Indian equity screening and company research via Screener.in using Playwright browser tools.

## Prerequisites

- Screener.in open and logged in via Playwright (navigate to `https://www.screener.in/login/`, user completes login)
- Playwright MCP tools: `browser_navigate`, `browser_evaluate`, `browser_snapshot`, `browser_click`
- Alternative: Claude-in-Chrome MCP tools work similarly

## Workflow

### Phase 1: Screen

Navigate to screen results via URL:
```
https://www.screener.in/screen/raw/?sort=Market+Capitalization&order=desc&query=<URL_ENCODED_QUERY>&latest=on
```

Add `&latest=on` to filter for companies with latest quarter results only. See `references/screens.md` for predefined queries and all available ratio names.

Users can provide their own framework — the predefined screens are starting points, not mandates. Any combination of Screener's 40+ ratios works.

### Phase 2: Extract Screen Results

Use `browser_evaluate` with this JavaScript:

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

### Phase 3: Cross-Reference (optional)

When running multiple screens, track stocks appearing in 2+ screens. Multi-theme hits = higher conviction. Build a cross-reference table:
```
Ticker | Name | Screens Hit | MCap | Best Theme Fit
```

### Phase 4: Deep-Dive via Screener Company Pages

Navigate to: `https://www.screener.in/company/<TICKER>/consolidated/`

Screener provides **10 structured data sections** per company — enough for a full fundamental analysis without leaving the site:

| Section | DOM ID | Data | Use |
|---------|--------|------|-----|
| Top Ratios | `.company-ratios li` | MCap, CMP, PE, ROCE, ROE, EPS, 52W H/L, Industry PE, Book Value | Quick snapshot |
| Pros/Cons | `.pros li`, `.cons li` | Auto-generated flags | Instant red/green signals |
| Peer Comparison | `#peers` | 9 peers with CMP, PE, ROCE, FII%, Debt, 6mo return | Relative valuation |
| Quarterly Results | `#quarters` | 13 quarters: Sales, OPM, PAT, EPS | Trend + margin trajectory |
| Annual P&L | `#profit-loss` | 12 years + compounded growth + ROE history | Long-term track record |
| Balance Sheet | `#balance-sheet` | 12 years: equity, debt, reserves, assets | Leverage + dilution check |
| Cash Flow | `#cash-flow` | 12 years: CFO, investing, financing | FCF quality + capex intensity |
| Ratios | `#ratios` | 12 years: ROCE, debtor days, inventory days | Moat durability over cycles |
| Shareholding | `#shareholding` | 12 quarters: promoter/FII/DII/public % | Smart money tracking |
| Documents | `#documents` | BSE/NSE filings, earnings transcripts | Latest announcements |

#### Company Page Extractor

Use this comprehensive `browser_evaluate` to pull all key data in one call:

```javascript
() => {
  const clean = (s) => s?.textContent?.trim().replace(/\s+/g,' ') || '';
  const tableData = (id, maxRows) => {
    const rows = [];
    document.querySelectorAll(`#${id} table tr`).forEach((row, i) => {
      if (maxRows && i >= maxRows) return;
      rows.push(Array.from(row.querySelectorAll('td, th')).map(c => clean(c)).join('|'));
    });
    return rows;
  };

  // Top metrics
  const ratios = {};
  document.querySelectorAll('.company-ratios li, #top-ratios li').forEach(li => {
    const name = li.querySelector('.name')?.textContent?.trim();
    const value = li.querySelector('.value, .number')?.textContent?.trim().replace(/\s+/g,'');
    if (name) ratios[name] = value;
  });

  // Pros & Cons
  const pros = Array.from(document.querySelectorAll('.pros li')).map(li => clean(li));
  const cons = Array.from(document.querySelectorAll('.cons li')).map(li => clean(li));

  return JSON.stringify({
    ratios,
    pros,
    cons,
    peers: tableData('peers', 10),
    quarters: tableData('quarters', 14),
    profitLoss: tableData('profit-loss', 15),
    balanceSheet: tableData('balance-sheet', 12),
    cashFlow: tableData('cash-flow', 8),
    ratioHistory: tableData('ratios', 8),
    shareholding: tableData('shareholding', 7),
    about: document.querySelector('.about p, .company-profile p')?.textContent?.trim()?.substring(0, 500)
  });
}
```

This returns structured JSON with the full fundamental picture. Parse it to:
1. Check margin trajectory (quarterly OPM trend)
2. Verify growth consistency (annual P&L compounded rates)
3. Assess leverage (balance sheet debt trend)
4. Evaluate FCF quality (CFO vs PAT, capex as % of CFO)
5. Track smart money (FII/DII trend in shareholding)
6. Compare valuation vs peers (PE, ROCE relative ranking)

### Phase 5: Verdict

For each researched stock, assign a verdict based on the user's investment framework. Common frameworks include:

- **Quality compounding**: ROIC > cost of capital, moat, reinvestment runway (Baid, Mukherjea)
- **GARP**: Growth at reasonable price — PEG <1.5, earnings acceleration
- **Deep value**: Low PE/PB with catalyst, margin of safety
- **Momentum + quality**: RS rank + fundamentals filter
- **Thematic**: Sector tailwind + company positioning (e.g., China+1 for CDMO)

Output a final table: Ticker, Name, CMP, MCap, PE, ROCE, Theme Fit, Verdict, Rationale.

## Tips

- Screener's "Show all Ratios" button on the screen creation page lists every available field
- Custom ratios can be created at `https://www.screener.in/ratios/`
- For sector-specific screens, browse 50K+ community screens at `https://www.screener.in/screens/`
- The `&latest=on` flag filters for companies with the most recent quarter results — always use it
- Screener data is from BSE/NSE filings — authoritative, no estimation
