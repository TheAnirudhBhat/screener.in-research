# Predefined Screener.in Queries

URL pattern: `https://www.screener.in/screen/raw/?sort=Market+Capitalization&order=desc&query=<ENCODED>&latest=on`

## baid-compounders
Gautam Baid compounding machine criteria: high ROCE, consistent growth, low leverage, profitable.
```
Return on capital employed > 15 AND Sales growth 5Years > 15 AND Profit growth 5Years > 15 AND OPM > 15 AND Debt to equity < 0.5 AND Market Capitalization > 500 AND Market Capitalization < 50000
```

## cdmo-innovator
High-margin pharma/chem CDMO businesses partnering with Western innovator pharma.
```
Sales growth 5Years > 20 AND OPM > 25 AND Return on capital employed > 20 AND Market Capitalization > 1000 AND Market Capitalization < 50000 AND Debt to equity < 0.5
```

## power-infra
Power ancillaries, data center infra, energy equipment — recent high growth (3yr focus since theme is newer).
```
Sales growth 3Years > 25 AND Profit growth 3Years > 25 AND Return on capital employed > 15 AND OPM > 12 AND Market Capitalization > 1000 AND Market Capitalization < 50000
```

## consumption
India domestic consumption pivot — established businesses benefiting from tax cuts + rate cuts.
```
Sales growth 5Years > 12 AND Profit growth 5Years > 12 AND OPM > 10 AND Return on capital employed > 15 AND Return on equity > 15 AND Debt to equity < 0.5 AND Market Capitalization > 5000 AND Market Capitalization < 100000
```

## precision-aero
Precision engineering, aerospace exports, defense electronics — FTA-driven outsourcing.
```
Sales growth 5Years > 20 AND Profit growth 5Years > 25 AND OPM > 15 AND Return on capital employed > 15 AND Debt to equity < 1 AND Market Capitalization > 500 AND Market Capitalization < 50000
```

## deep-value
Low PE + high ROCE + growth — potential Baid "pay low PE in decline? No — pay low PE in early growth stage."
```
Price to Earning < 20 AND Return on capital employed > 20 AND Sales growth 5Years > 15 AND Profit growth 5Years > 15 AND Market Capitalization > 1000 AND Debt to equity < 0.5
```

## quality-midcap
Quality midcaps with consistent returns and low debt — potential core portfolio holdings.
```
Average return on equity 5Years > 18 AND Sales growth 5Years > 12 AND OPM > 15 AND Debt to equity < 0.3 AND Market Capitalization > 5000 AND Market Capitalization < 50000 AND Pledged percentage < 5
```

## Available Screener.in Ratio Names

### Most Used
Market Capitalization, Current price, Price to Earning, PEG Ratio, Dividend yield, Price to book value, Price to Sales, Price to Free Cash Flow, EVEBITDA, Enterprise Value, EPS, Return on capital employed, Return on assets, Return on equity, Debt to equity, Current ratio, Interest Coverage Ratio, OPM, Promoter holding, Change in promoter holding, Pledged percentage, Industry PE, Earnings yield

### Growth
Sales, Sales growth, Sales growth 3Years, Sales growth 5Years, Profit after tax, Profit growth, Profit growth 3Years, Profit growth 5Years, Sales latest quarter, Profit after tax latest quarter, YOY Quarterly sales growth, YOY Quarterly profit growth

### Historical
Average return on equity 5Years, Average return on equity 3Years, Return over 1year, Return over 3years, Return over 5years, Return over 3months, Return over 6months

### Operators
AND, OR, >, <, +, -, *, /

### Tips
- Add `&latest=on` to filter for companies with latest quarter results only
- Sort by any column: `&sort=<Column+Name>&order=desc`
- Combine with OR for sector-specific: `(Industry = Pharmaceuticals OR Industry = Chemicals)`
- Custom ratios can be created at https://www.screener.in/ratios/
