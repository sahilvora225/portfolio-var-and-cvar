# Portfolio VaR and Expected Shortfall (CVaR)

Estimating the downside risk of an equity portfolio using **Value-at-Risk (VaR)** and
**Conditional Value-at-Risk (CVaR / Expected Shortfall)**.

## Introduction

This project quantifies how much a weighted stock portfolio could lose under adverse market
conditions. For a portfolio of **62 NSE-listed Indian stocks** (holdings and weights defined in
[portfolio.csv](portfolio.csv), benchmarked against the **Nifty 50** index `^NSEI`), it computes
two complementary tail-risk measures at a **95% confidence level**, on both a daily and an
annualized basis:

- **Value-at-Risk (VaR)** — the loss threshold that is *not* expected to be exceeded with a given
  confidence. A 95% daily VaR of −1.5% means there is roughly a 5% chance of losing more than 1.5%
  of the portfolio's value on any given day.
- **Conditional VaR (CVaR) / Expected Shortfall** — the *average* loss on the days when the loss
  does exceed the VaR threshold, i.e. the expected loss in the worst 5% of cases. CVaR is always at
  least as large as VaR and better captures the severity of tail events.

Both measures are computed using two independent methods — a **parametric** (normal-distribution)
approach and a **historical** (empirical) approach — so the results can be cross-checked.

## How the agenda was achieved

The full analysis lives in [portfolio_analysis.ipynb](portfolio_analysis.ipynb) and follows this
pipeline:

1. **Download market data.** Pull 7 years of OHLC data for every stock in `portfolio.csv` plus the
   Nifty 50 benchmark from Yahoo Finance via `yfinance`. Downloads are chunked into ~700-day
   windows to stay within API limits, with duplicate timestamps removed and failed downloads
   handled gracefully.
2. **Compute daily returns.** Derive daily percentage returns from adjusted close prices for the
   benchmark and every holding.
3. **Back-fill recently-listed stocks.** Some stocks lack a full 7-year history. For each, a linear
   regression of the stock's returns against the Nifty 50 returns (`np.polyfit`) estimates its
   **alpha** and **beta**. Synthetic historical returns are then generated for the missing period
   as `alpha + beta × benchmark_return + residual`, where the residual is sampled from a normal
   distribution fitted to the regression errors. A fixed random seed (`100`) keeps results
   reproducible.
4. **Build portfolio returns.** Combine the individual return series into a single weighted
   portfolio return series using the weights from `portfolio.csv`.
5. **Parametric VaR & Expected Shortfall.** Assuming portfolio returns are normally distributed,
   use the portfolio mean and standard deviation with `scipy.stats.norm` to derive VaR and CVaR at
   95% confidence.
6. **Historical VaR & Expected Shortfall.** Take the empirical 5% quantile of the actual return
   distribution as VaR, and the average of returns beyond that quantile as CVaR — no distributional
   assumption required.
7. **Annualize.** Scale daily figures to an annual horizon using the √252 (trading-day) rule.

### Representative results

| Method      | Daily VaR (95%) | Daily CVaR (95%) | Annual VaR (95%) | Annual CVaR (95%) |
|-------------|-----------------|------------------|------------------|-------------------|
| Parametric  | −1.54%          | −2.50%           | −24.48%          | −39.61%           |
| Historical  | −1.50%          | −2.43%           | −23.79%          | −38.54%           |

(Exact figures will vary slightly with the data download date, since the 7-year window is relative
to the day the notebook is run.)

## Download and run

1. **Clone the repository**

   ```bash
   git clone <repository-url>
   cd portfolio-var-and-cvar
   ```

2. **Prerequisites** — Python 3.x and Jupyter.

3. **Install dependencies** (no `requirements.txt` is bundled):

   ```bash
   pip install numpy pandas scipy matplotlib yfinance jupyter
   ```

4. **Run the notebook**

   ```bash
   jupyter notebook portfolio_analysis.ipynb
   ```

   Run the cells top to bottom.

> **Note:** The notebook needs internet access — it downloads live data from Yahoo Finance at run
> time. To analyze a different portfolio, edit `portfolio.csv`, which has two columns: `Symbol`
> (NSE ticker without the `.NS` suffix) and `Weightage` (decimal weight; the column should sum to
> ~1.0).

## Closing notes

### Assumptions

The results rest on several modeling assumptions worth keeping in mind:

- **Normality of returns.** The parametric method assumes portfolio returns follow a normal
  distribution.
- **Beta-based back-filling.** Stocks that started trading recently and lack a full 7 years of
  history have their earlier returns synthesized using the stock's **beta** relative to the
  Nifty 50 benchmark, rather than using real prices.
- **Normal residuals.** The error/residual term added when generating synthetic returns is itself
  assumed to be normally distributed.
- **Annualization.** Converting daily figures to annual ones via √252 assumes 252 trading days and
  independent, identically distributed daily returns.
- **History as a proxy.** Both methods use the historical return distribution as a stand-in for
  future risk; structural market changes are not modeled.

### Interpreting the results

- A 95% **daily VaR of ≈ −1.5%** means: on roughly 95% of days the portfolio is not expected to
  lose more than 1.5% of its value. On the worst ~5% of days, the **CVaR (≈ −2.5%)** is the
  expected average loss — losses can and do exceed VaR, and CVaR estimates how bad they get.
- The parametric and historical estimates are close, which suggests the normal-distribution
  assumption is reasonable for this portfolio. The parametric method tends to be slightly more
  conservative.
- Holdings whose history was synthesized via beta carry additional **model risk** — their
  contribution to the overall figures should be treated with caution, since it reflects modeled
  rather than realized behavior.