# Systematic Pairs Trading

Research on a daily equity pairs trading strategy that uses cointegration to identify potential mean reverting relationships among current S&P 500 constituents.

The strategy compares stocks within the same sector, estimates their long run log price relationship, and trades deviations measured by a rolling z-score. A long spread buys the first stock and shorts the second; a short spread reverses those positions. Position sizes incorporate the estimated hedge ratio.

## Research questions

- Which within sector pairs show evidence of cointegration during the formation period?

- Do those relationships have stable hedge ratios and economically useful mean reversion speeds?

- Does the strategy remain profitable during validation after trading and borrowing costs?

- Does a shortlist selected on validation data retain its performance in a later holdout period?

- How do raw p values and optional multiple testing corrections affect the candidate universe?

- Does grouping by sub industry produce more economically coherent candidates than sector level comparisons?

## Methodology

1. Retrieve current S&P 500 constituents and their GICS sector and sub industry classifications.

2. Download historical adjusted closing prices and volume from Yahoo Finance, with batch downloads and retries for unavailable symbols.

3. Divide the common trading calendar into formation, validation and final holdout periods.

4. Create every unordered pair within each sector, excluding share classes belonging to the same issuer. Sub industry and all market grouping are optional.

5. Estimate each pair's log price regression on formation data and apply the Engle–Granger cointegration test in a fixed alphabetical orientation.

6. Filter candidates using statistical evidence, data completeness, liquidity, return correlation, hedge ratio stability and estimated mean reversion half life.

7. Backtest qualifying pairs during validation, including per leg transaction costs and short borrow fees.

8. Rank qualified pairs and select up to five, with limits on repeated tickers and sector concentration.

9. Evaluate the frozen shortlist during the final holdout and display price charts with execution markers, spread, z-score, dollar equity, net P&L and percentage drawdown.

10. Save the full screening audit, leaderboard, selected pairs, available trade logs and run metadata.

## Statistical screening

The current default uses **raw Engle–Granger p-values at a 5% cutoff**, without multiple-testing correction:

```python
CONFIG = Config(
    group_by="sector",
    fdr_method="none",
    fdr_alpha=0.05,
    top_n=5,
)
```

Set `fdr_method="fdr_bh"` for Benjamini–Hochberg or `fdr_method="fdr_by"` for Benjamini–Yekutieli. In raw mode, the parameter `fdr_alpha` is simply the per test p value cutoff; it does not control the false discovery rate across the search. The output `selection_pvalue` records the value used for filtering and statistical ranking. `qvalue` is blank when correction is disabled.

Other default formation requirements include:

| Check | Default requirement |
|---|---|
| Data history | Complete positive formation prices |
| Median daily dollar volume | At least $20 million for each stock |
| Median adjusted price | At least $5 for each stock |
| ADF compatibility checks | Level p value above 0.05; first difference p value below 0.05 |
| Return correlation | At least 0.40 |
| Hedge ratio | Between 0.10 and 5.00 |
| Estimated half life | Between 2 and 60 sessions |
| Split half hedge-ratio change | At most 50% of the absolute full period estimate |

The ADF checks assess compatibility with the model's integration assumptions; they do not prove that prices are I(1). Half life and hedge stability are additional screening heuristics, not independent confirmations of statistical significance.

## Validation and selection design

The default chronological split assigns 60% of sessions to formation, 20% to validation and 20% to a final holdout. Regression coefficients are estimated during formation and remain fixed throughout evaluation.

Validation candidates must have positive net P&L and Sharpe, at least five closed trades, maximum drawdown no worse than 20%, and positive P&L when transaction costs are doubled.

The default leaderboard combines percentile ranks among qualified pairs:

- 50% validation Sharpe ratio.

- 20% validation drawdown, favoring smaller losses.

- 20% consistency, measured by the fraction of three contiguous validation blocks with positive returns.

- 10% statistical strength, using the configured p value measure.

Alternative ranking modes are `sharpe`, `pnl` and `statistical`. The shortlist allows no repeated ticker and at most two pairs per sector by default. It may contain fewer than five pairs or be empty.

Selection is saved before holdout evaluation.

## Trading and accounting

The spread is `log(Y) - alpha - beta * log(X)`. Signals use a 60 session rolling z-score, with entry at an absolute z-score of 2, mean-reversion exit at 0.5, an extreme-deviation exit at 4, and a maximum holding period of 60 sessions.

Each pair starts with $100,000. Entry dollar weights are proportional to `(1, -beta)`, with 100% gross exposure at entry. Adjusted price units remain fixed during each trade, so exposures subsequently drift. The model charges 5 basis points per traded dollar on each leg and assumes annual short borrow fees of 100 basis points, accrued over calendar days.

## Current research results

The saved experiment used 503 current constituent symbols and a family of 13,920 within sector pairs. Its formation period ran from January 2, 2020 through January 3, 2024.

1. **Raw screening produces a substantial candidate set.** Of the saved pairs, 1,230 passed the raw 5% p value cutoff and 492 passed all formation filters when correction was disabled.

2. **Search-wide correction materially changes the result.** No pairs passed the original Benjamini–Yekutieli screen at 5%. A separate Benjamini–Hochberg calculation also produced no discoveries at 5%.

3. **Economic similarity alone is insufficient.** KO/PEP passed the raw p value cutoff but failed the hedge ratio stability requirement.

## Repository layout

```text
.
|-- microestructuraprecios_ieb_systematic.ipynb
|-- pairs_screener.py
|-- test_screener.py
|-- requirements.txt
|-- live_run/                 # Earlier BY experiment: data, audit and metadata
|-- raw_pvalue_run/           # Raw-mode formation audit from saved statistics
|-- REVIEW_AND_GUIDE.md
|-- RAW_MODE_UPDATE.md
|-- VALIDATION.md
|-- ORIGINAL_FILE_SHA256.txt
`-- README.md
```

The notebook embeds the engine and can also run independently in Google Colab. The Python module is provided for reuse outside the notebook.

## Reproduce the analysis

Create an isolated Python environment. For example, in Windows PowerShell:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
python -m pip install jupyterlab
python -m unittest discover -s . -p "test_screener.py" -v
jupyter lab
```
