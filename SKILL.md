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
  const num = (s) => parseFloat(String(s).replace(/,/g,'')) || 0;
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

  // --- Auto Red-Flag Scanner ---
  const redFlags = [];

  // 1. ROCE trend (from ratio history table)
  const roceRow = Array.from(document.querySelectorAll('#ratios table tr')).find(r =>
    r.querySelector('td')?.textContent?.includes('ROCE'));
  if (roceRow) {
    const cells = Array.from(roceRow.querySelectorAll('td')).slice(1).map(c => num(c.textContent));
    const recent3 = cells.slice(-3);
    if (recent3.length === 3 && recent3[2] < recent3[0] && recent3[1] < recent3[0])
      redFlags.push(`ROCE declining: ${recent3.join('% → ')}% (last 3 years)`);
  }

  // 2. Debt/equity trend (from balance sheet)
  const bsRows = document.querySelectorAll('#balance-sheet table tr');
  const debtRow = Array.from(bsRows).find(r =>
    r.querySelector('td')?.textContent?.match(/^Borrowings/i));
  if (debtRow) {
    const cells = Array.from(debtRow.querySelectorAll('td')).slice(1).map(c => num(c.textContent));
    const recent3 = cells.slice(-3);
    if (recent3.length === 3 && recent3[2] > recent3[0] * 1.5)
      redFlags.push(`Debt rising: borrowings grew ${Math.round((recent3[2]/recent3[0]-1)*100)}% over last 3 years`);
  }

  // 3. Negative FCF (from cash flow)
  const cfoRow = Array.from(document.querySelectorAll('#cash-flow table tr')).find(r =>
    r.querySelector('td')?.textContent?.match(/Cash from Operating/i));
  if (cfoRow) {
    const cells = Array.from(cfoRow.querySelectorAll('td')).slice(1).map(c => num(c.textContent));
    const lastCFO = cells[cells.length - 1];
    if (lastCFO < 0) redFlags.push(`Negative operating cash flow: ₹${lastCFO} Cr (latest year)`);
  }

  // 4. Promoter selling (from shareholding)
  const shRows = document.querySelectorAll('#shareholding table tr');
  const promoRow = Array.from(shRows).find(r =>
    r.querySelector('td')?.textContent?.match(/Promoters/i));
  if (promoRow) {
    const cells = Array.from(promoRow.querySelectorAll('td')).slice(1).map(c => num(c.textContent));
    const recent4 = cells.slice(-4);
    if (recent4.length >= 2 && recent4[recent4.length-1] < recent4[0] - 1.5)
      redFlags.push(`Promoter stake declining: ${recent4[0]}% → ${recent4[recent4.length-1]}% (last ${recent4.length} quarters)`);
  }

  // 5. OPM compression (from quarterly results)
  const qRows = document.querySelectorAll('#quarters table tr');
  const opmRow = Array.from(qRows).find(r =>
    r.querySelector('td')?.textContent?.match(/OPM/i));
  if (opmRow) {
    const cells = Array.from(opmRow.querySelectorAll('td')).slice(1).map(c => num(c.textContent));
    const recent4 = cells.slice(-4);
    if (recent4.length === 4 && recent4[3] < recent4[0] - 3)
      redFlags.push(`OPM compressing: ${recent4[0]}% → ${recent4[3]}% (last 4 quarters)`);
  }

  // --- Valuation Context ---
  const pe = num(ratios['Stock P/E'] || ratios['Price to Earning']);
  const indPE = num(ratios['Industry PE']);
  const eps = num(ratios['EPS']);
  const valuation = {};
  if (pe > 0) valuation.pe = pe;
  if (indPE > 0) {
    valuation.industryPE = indPE;
    valuation.peVsIndustry = pe > 0 ? Math.round((pe / indPE - 1) * 100) + '% ' + (pe > indPE ? 'premium' : 'discount') : null;
  }

  // --- Peer Ranking ---
  const peerRows = tableData('peers', 10);
  let peerRank = null;
  if (peerRows.length > 1) {
    const header = peerRows[0].split('|');
    const peIdx = header.findIndex(h => h.match(/P\/E|PE/i));
    const roceIdx = header.findIndex(h => h.match(/ROCE/i));
    const nameIdx = header.findIndex(h => h.match(/Name/i));
    if (peIdx > -1 && roceIdx > -1) {
      const peers = peerRows.slice(1).map(r => {
        const c = r.split('|');
        return { name: c[nameIdx]||c[0], pe: num(c[peIdx]), roce: num(c[roceIdx]) };
      }).filter(p => p.pe > 0);
      const byPE = [...peers].sort((a,b) => a.pe - b.pe);
      const byROCE = [...peers].sort((a,b) => b.roce - a.roce);
      const companyName = document.querySelector('h1')?.textContent?.trim();
      const peRank = byPE.findIndex(p => p.name.includes(companyName?.split(' ')[0])) + 1;
      const roceRank = byROCE.findIndex(p => p.name.includes(companyName?.split(' ')[0])) + 1;
      peerRank = { totalPeers: peers.length, peRank: peRank || null, roceRank: roceRank || null,
        cheapestPeer: byPE[0]?.name, cheapestPE: byPE[0]?.pe,
        bestROCE: byROCE[0]?.name, bestROCEVal: byROCE[0]?.roce };
    }
  }

  return JSON.stringify({
    ratios, pros, cons, redFlags, valuation, peerRank,
    peers: peerRows,
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

The extractor now returns three additional fields:

- **`redFlags`**: Array of auto-detected warning strings. Empty = clean bill of health. Checks:
  - ROCE declining over last 3 years
  - Borrowings grew >50% over 3 years
  - Negative operating cash flow (latest year)
  - Promoter stake down >1.5pp over recent quarters
  - OPM compressed >3pp over last 4 quarters

- **`valuation`**: `{pe, industryPE, peVsIndustry}` — instant context on whether the stock trades at a premium or discount to its sector.

- **`peerRank`**: `{totalPeers, peRank, roceRank, cheapestPeer, bestROCE}` — where this company sits among its Screener peers on PE and ROCE.

#### Interpreting Red Flags

Use `redFlags` as a mandatory first-pass filter before deep analysis:

| Red Flag | Severity | Action |
|----------|----------|--------|
| ROCE declining | High | Check if cyclical trough or structural erosion |
| Debt rising >50% | High | Verify capex-driven (OK if ROCE rising) vs. stress |
| Negative CFO | Critical | Almost always disqualifying unless turnaround story |
| Promoter selling >1.5pp | Medium | Check if pledge release or genuine exit |
| OPM compressing >3pp | Medium | Check if input cost spike (temporary) or pricing power loss |

A stock with 2+ red flags should default to AVOID unless the user's framework explicitly accounts for turnarounds.

#### Peer Comparison Guidance

Use `peerRank` to contextualize valuation:
- PE rank 1-3 out of 9 peers = relatively cheap
- ROCE rank 1-3 = quality leader in the sector
- Best combo: low PE rank + high ROCE rank = potential mispricing
- If PE > Industry PE by >30% AND ROCE rank is bottom half → valuation stretched

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

## Screen Caching & Diff-Aware Re-Screening

Persist screen results to disk so re-runs highlight what changed. This avoids re-researching stocks already evaluated and surfaces new entrants.

### Cache Format

After running a screen, save results to `references/cache/<screen-name>.json`:

```json
{
  "screen": "baid-compounders",
  "query": "Return on capital employed > 15 AND ...",
  "runDate": "2026-05-29",
  "resultCount": 42,
  "tickers": ["TICKER1", "TICKER2", ...],
  "results": [
    {"ticker": "TICKER1", "name": "Company Name", "cmp": "1234", "mcap": "5000", "pe": "25", "roce": "22", "opm": "18", "de": "0.1"}
  ]
}
```

### Diff-Aware Re-Screening

When re-running a previously cached screen:

1. Load previous cache from `references/cache/<screen-name>.json`
2. Run the screen fresh on Screener.in
3. Compute diff:
   - **New entrants**: tickers in fresh results but not in cache → flag as `NEW` (prioritize for deep-dive)
   - **Dropped**: tickers in cache but not in fresh results → flag as `DROPPED` (check why — deteriorating fundamentals?)
   - **Retained**: in both → flag as `RETAINED` (stable quality signal)
4. Update cache file with fresh results
5. Present diff summary before deep-diving

```
Screen: baid-compounders (last run: 2026-05-15, 14 days ago)
Results: 42 → 45 (+3)
NEW (3): TICKER_A, TICKER_B, TICKER_C ← prioritize deep-dive
DROPPED (0): none
RETAINED (42): all prior stocks still passing
```

New entrants get automatic deep-dive priority — a stock freshly qualifying for a quality screen is a stronger signal than one that's been there for months.

### Watchlist File

Maintain a persistent watchlist at `references/watchlist.json`:

```json
{
  "lastUpdated": "2026-05-29",
  "entries": [
    {
      "ticker": "NEULANDLAB",
      "addedDate": "2026-05-29",
      "sourceScreen": "cdmo-innovator",
      "verdict": "WATCH",
      "entryTrigger": "₹14,000-15,000",
      "rationale": "CDMO play, ROCE 30%+, PE 51x stretched",
      "redFlags": [],
      "lastChecked": "2026-05-29"
    }
  ]
}
```

When running screens, cross-reference against the watchlist:
- Watchlist stocks that appear in new screens → **reinforced conviction** (mention in output)
- Watchlist stocks that drop from screens → **check for deterioration** (flag for review)
- Watchlist stocks hitting entry triggers → **alert** (promote to action)

## Portfolio-Aware Screening

When the user has a portfolio (e.g., `latest_snapshot.json`, `us_stocks.json`), screening should be context-aware.

### Exclude Already-Held Stocks

After extracting screen results, check against the user's holdings. Mark held stocks distinctly:

```
TICKER | Name | CMP | MCap | PE | ROCE | Status
ARTEMISMED | Artemis Medicare | ₹278 | 2500 | 42 | 18% | HELD (413 sh, 16.5%)
NEULANDLAB | Neuland Labs | ₹15200 | 7500 | 51 | 30% | NEW
```

Held stocks should not be deep-dived again (unless the user explicitly asks). Focus research time on NEW opportunities.

### Correlation Check

When shortlisting new stocks, check if they're in the same sector or have similar risk factors as existing holdings:

- If the user holds ARTEMISMED (hospitals) and the screen surfaces MEDANTA (also hospitals) → flag: "Same sector as ARTEMISMED — adds concentration, not diversification"
- If the user holds ACUTAAS (CDMO) and the screen surfaces NEULANDLAB (also CDMO) → flag correlation

### Role-Bucket Fit

If the user's portfolio has defined roles (growth, cyclicals, core, etc.), suggest which bucket a new stock fits:

```
NEULANDLAB → fits "growth" bucket (CDMO + high ROCE + expansion phase)
TDPOWERSYS → fits "cyclicals" bucket (capex/infra cycle play)
MEDANTA → fits "growth" bucket BUT correlates with ARTEMISMED (same sector)
```

### Implementation

To make screening portfolio-aware, read the user's snapshot at the start of the session:

1. Check if `latest_snapshot.json` exists in the user's memory directory
2. If it does, extract: `holdings[].ticker`, `holdings[].weight`, `holdings[].role`
3. Build a held-tickers set and a sector map
4. Pass this context through all screen result processing

If no portfolio file exists, skip portfolio-aware features silently — the skill works standalone too.
