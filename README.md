# Black-Litterman Portfolio Optimizer

Portfolio construction on the NIFTY 50 universe, comparing classical
mean-variance optimization against the Black-Litterman framework. Ten years of
daily prices are pulled from Yahoo Finance, market-implied equilibrium returns
are recovered by reverse optimization, and subjective views are blended in to
produce a posterior allocation.

## Why Black-Litterman

Markowitz optimization on historical sample means is unstable. Small errors in
estimated returns produce large swings in the optimal weights, and the result is
usually concentrated in whichever handful of assets happened to perform best in
the sample.

Black-Litterman starts from the other end. Instead of estimating returns and
deriving weights, it takes the market's own weights as given and asks what
returns would justify them - the equilibrium returns. Those become a prior.
Investor views are then blended in, weighted by how confident you are in each,
and the posterior returns feed the optimizer.

## Results

Ten years of daily prices, 2015-01-01 to 2025-01-01. Risk-free rate 6.5%,
risk aversion coefficient lambda = 3.3799 recovered from the market proxy,
tau = 0.025.

| Portfolio | Expected return | Volatility | Sharpe |
|---|---|---|---|
| Markowitz (max Sharpe) | 30.27% | 18.19% | 1.334 |
| Black-Litterman posterior | 23.93% | 14.57% | 1.230 |

### Reading that table

The Black-Litterman portfolio has a *lower* in-sample Sharpe ratio, and that is
the expected outcome rather than a failure.

The Markowitz number is produced by optimizing directly on ten years of sample
means. It is high precisely because the optimizer is free to load up on whatever
performed best over that window - BAJAJ-AUTO at 25%, SUNPHARMA at 15%, TRENT at
14%, with most of the fifty names at zero. That concentration is the estimation
error being fitted, not skill, and it is the well-documented failure mode the
Black-Litterman model was built to address.

The posterior allocation is far more diversified and carries roughly 3.6
percentage points less volatility. Its lower in-sample Sharpe is the cost of not
chasing the sample; the argument for it is out-of-sample stability, which this
notebook does not test.

## Method

### 1. Data pipeline

NIFTY 50 constituents are read from `ind_nifty50list.csv`, and ten years of
daily adjusted closes plus market capitalizations are pulled from Yahoo Finance
via `yfinance`. Returns and the covariance matrix are computed with NumPy and
pandas.

### 2. Mean-variance optimization

`scipy.optimize.minimize` with SLSQP, subject to weights summing to one and
long-only bounds. Two objectives are solved: maximum Sharpe ratio, and minimum
volatility at each target return to trace the efficient frontier.

### 3. Reverse optimization

Rather than estimating expected returns, the market's weights are taken as
optimal and the implied returns recovered:

```
lambda = (market_return - risk_free_rate) / market_variance
Pi     = lambda * Sigma * w_market
```

CAPM betas are computed as `cov(asset, market) / var(market)` and used alongside
the equilibrium excess returns. An equal-weight portfolio stands in for the
market here; a cap-weighted proxy would be the more faithful choice and is noted
below.

### 4. Views

Three views are encoded in the `P` matrix with their magnitudes in `Q`:

| View | Type |
|---|---|
| RELIANCE outperforms | absolute |
| INFY outperforms | absolute |
| HDFCBANK outperforms ICICIBANK | relative |

The third is what makes the framework worth using - Black-Litterman handles
relative views, where one asset beats another, which plain mean-variance cannot
express at all.

Confidence is set through `Omega = tau * P * Sigma * P^T`, so a view on a
volatile asset is automatically held with less confidence than one on a stable
asset.

### 5. Posterior

The blended returns come from the standard formula:

```
E(R) = [(tau*Sigma)^-1 + P^T * Omega^-1 * P]^-1 * [(tau*Sigma)^-1 * Pi + P^T * Omega^-1 * Q]
```

Those posterior returns then go back through the same SLSQP optimizer.

## Running it

```bash
pip install -r requirements.txt
```

Then run `Black_Litterman.ipynb` top to bottom. It downloads its own price data;
`ind_nifty50list.csv` is committed so the universe is reproducible even as index
membership changes.

## Known issues

- One ticker (`MM.NS`) fails to download with a timezone error from Yahoo
  Finance and is dropped from the universe. The traceback is visible in the
  committed notebook output.
- A `NameError` on `eff_vol` appears in one frontier cell from an earlier
  variable name; the results below it computed correctly, but the cell wants
  cleaning up on the next run.
- The market portfolio is approximated with equal weights rather than actual
  free-float market capitalization. Since the equilibrium returns Pi are derived
  directly from those weights, a cap-weighted proxy would change the prior
  materially.
- Views and their confidences are hand-set. There is no sensitivity analysis
  showing how the posterior moves as tau or the view magnitudes change, which is
  the most useful thing to add next.
- Everything is in-sample. No backtest, no walk-forward, no out-of-sample test -
  so the stability argument for Black-Litterman is asserted here, not
  demonstrated.
- Long-only with no transaction costs, no turnover constraint and no rebalancing
  schedule.

## Summary

This project implements the Black-Litterman framework on the NIFTY 50 universe
by integrating market equilibrium with subjective investor views. Historical
price data and market capitalizations are retrieved automatically from Yahoo
Finance, enabling systematic computation of equilibrium returns, covariance
structure and posterior allocations. The workflow combines reverse optimization
of implied returns with view-adjusted blending, and the resulting portfolios are
compared against the classical mean-variance benchmark across return, volatility
and Sharpe ratio.

## References

- Black, F. and Litterman, R. (1992), "Global Portfolio Optimization",
  Financial Analysts Journal
- Idzorek, T. (2005), "A Step-by-Step Guide to the Black-Litterman Model"
- Markowitz, H. (1952), "Portfolio Selection", Journal of Finance

## Author

Pal Ajay Ramsagar - github.com/ajaypal5117
