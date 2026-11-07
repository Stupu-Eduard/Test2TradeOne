# Test2Trade

Search millions of indicator-based trading strategies on historical data, then statistically filter out the ones that only look good by chance.

> **Status:** early design phase. Nothing described below is implemented yet.

---

## 1. Problem

Testing many indicator combinations on the same price history will always produce strategies with a great backtest. Most of them are noise: the more combinations you try, the better the best one looks, even on random data. Existing tools handle the search well but leave the multiple-testing problem to the user.

| Tool | Search | Overfitting handling |
|---|---|---|
| [QuantConnect](https://www.quantconnect.com/docs/v2/cloud-platform/optimization/strategies) | Grid / Euler search on cloud nodes | Parameter charts, manual judgement |
| [StrategyQuant X](https://strategyquant.com/blog/robustness-tests-and-analysis/) | Genetic programming over indicator blocks | Out-of-sample, Monte Carlo, walk-forward (run manually) |
| [vectorbt](https://github.com/polakowo/vectorbt) | Vectorized parameter sweeps (Numba) | None built in |

## 2. Goal

Build a system that:

1. Generates and backtests a very large space of indicator strategies (e.g. MACD with specific periods, RSI with specific thresholds, and combinations of them).
2. Counts every trial it runs.
3. Uses that count to decide which strategies are statistically credible, and reports "nothing survives" when that is the truth.

**Thesis claim:** a trial-count-aware validation pipeline selects strategies that perform measurably better on unseen data than picking the best raw backtest.

## 3. Scope

**In scope**
- Indicator-based strategies, long-only first, single asset per strategy.
- Crypto first (BTC, ETH, then more), then XAUUSD and other markets.
- Timeframes: 1h, 4h, 1d (lower ones only if compute allows).
- Realistic costs: fees, slippage / spread.

**Out of scope**
- User-uploaded Python code, LLM strategy generation, news filters.
- Live trading and order execution.
- Machine-learning models as strategies.

## 4. Pipeline

```
Raw data ──► Validate & clean ──► Parquet store
                                      │
                 ┌────────────────────┴────────────────────┐
                 │ Train / holdout split (holdout locked)  │
                 └────────────────────┬────────────────────┘
                                      ▼
 1. Define search space ─► 2. Vectorized screen (all N trials, logged in a ledger)
                                      ▼
 3. Validate top K:  CPCV (purged + embargo) · PBO · Deflated Sharpe Ratio (uses N)
                                      ▼
 4. Robustness:  parameter-neighborhood stability · trade-shuffle Monte Carlo
                 · cost stress · other assets / timeframes
                                      ▼
 5. One-shot test on locked holdout ─► ranked report (possibly empty)
```

**Search space.** Indicators (e.g. SMA/EMA, MACD, RSI, Bollinger Bands, ATR, ADX, Stochastic), each with parameter ranges, combined into entry/exit rules (AND/OR, up to 2–3 indicators). Small spaces use grid search; large spaces use random or genetic search. Every evaluated configuration is stored in the trial ledger.

**Validation methods** (from López de Prado and co-authors)
- [Deflated Sharpe Ratio](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=2460551): corrects Sharpe for selection bias under multiple testing and non-normal returns.
- Probability of Backtest Overfitting (PBO): how often the in-sample winner ranks below median out of sample.
- Combinatorial Purged Cross-Validation (CPCV): gives a distribution of out-of-sample results instead of one path; in one [synthetic comparison](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4686376) it beat walk-forward.

## 5. Data

Zero-cost, local, no live API calls. Data is downloaded once from public archives and stored as Parquet. Raw vendor files are not committed to the repository; only download scripts are.

| Market | Source | Notes |
|---|---|---|
| Crypto | [Binance public data](https://github.com/binance/binance-public-data) (`data.binance.vision`) | Official daily/monthly archives, klines from 1s to 1mo, no API key. Spot timestamps are in microseconds from 2025 onward. |
| XAUUSD (and other FX / metals) | [Dukascopy](https://www.dukascopy-node.app/) historical feed via open-source downloaders | Free tick and candle data with bid/ask, which allows realistic spread modelling. |
| Cross-check | [HistData.com](https://github.com/philipperemy/FX-1-Minute-Data) 1-minute FX | Second source to compare against Dukascopy. |

No free source is perfect, so the project includes a **data quality stage**: duplicate and gap detection, spike/outlier detection, timezone and DST normalization, trading-session handling for XAUUSD, and a per-dataset quality report. Cross-source comparison (Dukascopy vs HistData) is used to estimate error. Check each source's terms of use before publishing any derived dataset.

Storage: Parquet files are the source of truth. PostgreSQL holds metadata only (datasets, search configs, jobs, trial ledger summaries, results).

## 6. Evaluation (how the thesis proves it works)

1. **No-edge test.** Run the full pipeline on synthetic random-walk data. A naive "best Sharpe" selection will find strong-looking strategies. The pipeline should reject all of them. Measure the false-positive rate for both.
2. **Holdout comparison.** On real data, compare the naive pick against the pipeline's picks on the locked holdout (return, Sharpe, drawdown).
3. **Transfer test.** Search on one asset (BTC) and test survivors on others (ETH, then XAUUSD) to see whether validated strategies generalize.
4. **Compute.** Report trials per second and scaling when the grid is split across workers.

## 7. Planned architecture

Everything is Python, with no separate backend language.

- **API:** FastAPI (REST): dataset registry, search jobs, results.
- **Compute:** Numba (or vectorbt) for the vectorized backtest, plus the validation statistics (numpy, scipy, pandas). Workers take shards of the parameter grid, so the search runs in parallel on one machine first (`multiprocessing`, later a queue such as Celery/RQ/Dask) and on cloud machines if needed.
- **Storage:** Parquet (market data) and PostgreSQL (metadata and results).
- **Frontend:** undecided. A simple results dashboard (ranked strategies, equity curves, validation metrics) is enough for the thesis.

## 8. Roadmap

1. Data: downloaders, validation, quality reports, Parquet store (Binance first).
2. Vectorized backtest engine with fees and slippage; verify against a simple reference implementation.
3. Search space definition and trial ledger.
4. Validation: DSR, PBO, CPCV.
5. Robustness tests and holdout evaluation.
6. Synthetic no-edge experiment and real-data experiments.
7. XAUUSD and other markets.
8. API and dashboard.
9. Parallel / cloud execution.

## 9. Open decisions

- Own Numba engine vs vectorbt.
- Short selling and multi-asset strategies (after long-only works).
- Frontend technology.
