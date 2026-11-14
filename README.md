# options_pricing
Options Pricing: Black-Scholes, Monte Carlo & Binomial Trees

[![CI](https://github.com/<your-username>/options-pricing/actions/workflows/ci.yml/badge.svg)](https://github.com/<your-username>/options-pricing/actions)
![Python](https://img.shields.io/badge/python-3.9%2B-blue)
![License](https://img.shields.io/badge/license-MIT-green)

A small, tested Python library that prices vanilla options with three independent methods and cross-validates them against each other:

| Method | Option types | Notes |
|---|---|---|
| **Black-Scholes** (closed form) | European call / put | Vectorised over NumPy arrays; benchmark for everything else |
| **Monte Carlo** (GBM) | European call / put | Standard and **antithetic variates**; optional standard error |
| **CRR binomial tree** | European call / put, **American put** | Backward induction with early-exercise check |
| **Greeks** | Call / put | Analytical Delta, Gamma, Vega, Theta, Rho (verified against finite differences) |

The accompanying notebook [`notebooks/options_pricing.ipynb`](notebooks/options_pricing.ipynb) walks through the convergence analysis, variance reduction, Greeks, early-exercise premium and parameter sensitivities.

## Installation

```bash
git clone https://github.com/<your-username>/options-pricing.git
cd options-pricing
pip install -e .            # library only
pip install -e ".[dev]"     # + pytest, matplotlib, pandas, jupyter
```

## Quick start

```python
from options_pricing import (
    black_scholes_call, black_scholes_put, bs_greeks,
    monte_carlo_price, binomial_call, binomial_put,
)

S, K, T, r, sigma = 100, 100, 1.0, 0.05, 0.20

black_scholes_call(S, K, T, r, sigma)             # 10.450584
black_scholes_put(S, K, T, r, sigma)              # 5.573526

# Monte Carlo with antithetic variates, returning the standard error
price, se = monte_carlo_price(
    S, K, T, r, sigma, n_simulations=1_000_000,
    antithetic=True, return_stderr=True,
)

binomial_call(S, K, T, r, sigma, n_steps=1000)    # 10.448584
binomial_put(S, K, T, r, sigma, n_steps=1000, american=True)   # 6.089595

bs_greeks(S, K, T, r, sigma, option_type="call")
# {'Delta': 0.6368, 'Gamma': 0.01876, 'Vega': 37.52, 'Theta': -6.414, 'Rho': 53.23}
```

## Results

Base case: `S = K = 100, T = 1, r = 5%, sigma = 20%`, European call.

| Method | Price | Relative error vs Black-Scholes |
|---|---|---|
| Black-Scholes | 10.450584 | - |
| Standard Monte Carlo (5M paths) | 10.458798 (SE 0.0066) | 0.0786% |
| Antithetic Monte Carlo (5M paths) | 10.449897 (SE 0.0047) | 0.0066% |
| CRR binomial tree (1,000 steps) | 10.448584 | 0.0191% |
| CRR binomial tree (5,000 steps) | 10.450184 | 0.0038% |

Key findings:

- **Monte Carlo error decays as 1/sqrt(N)**: the fitted log-log slope of the standard error is about -0.50 (theory: -0.5).
- **Antithetic variates reduce the estimator variance by about 50% at equal path count** (standard error about 29% smaller). Each antithetic pair-average has about 75% lower variance than a single payoff, but a pair costs two paths, so the net gain at equal budget is about half. The call payoff is monotone in the random draw, which is what makes the pairing effective.
- **Binomial tree error shrinks roughly linearly in the number of steps** (with the usual odd/even oscillation).
- **American put premium**: 6.0896 vs 5.5735 for the European put, an early-exercise premium of about 0.516.
- **Greeks**: analytical values match central finite differences to better than 1e-6.

## Project structure

```
options-pricing/
├── src/options_pricing/
│   ├── black_scholes.py     # closed-form European prices
│   ├── monte_carlo.py       # standard + antithetic Monte Carlo
│   ├── binomial_tree.py     # CRR tree, European and American
│   ├── greeks.py            # analytical Greeks
│   └── _validation.py       # input checks
├── tests/                   # pytest suite
├── notebooks/               # analysis and plots
├── scripts/make_figures.py  # regenerates docs/images
└── docs/images/
```

## Development

```bash
pip install -e ".[dev]"
pytest
python scripts/make_figures.py
```

## Conventions and assumptions

- Prices follow the Black-Scholes model: constant volatility and rate, no dividends, lognormal terminal price.
- `r` is the continuously compounded risk-free rate; `T` is in years.
- Vega is per 1.00 change in volatility, Theta is per year, Rho is per 1.00 change in the rate. Divide by 100 / 365 / 100 for per-vol-point / per-day / per-1% values.
- For Monte Carlo, `n_simulations` is the **total** number of paths. With `antithetic=True` they are generated as `n/2` pairs `(Z, -Z)`, so both estimators use the same budget.
- Without dividends an American call is never exercised early, so only the American **put** needs a tree with early exercise.

## Possible extensions

Dividend yield, implied volatility solver, barrier/Asian options via Monte Carlo, control variates, Crank-Nicolson finite differences for the American put.

## License

[MIT](LICENSE)
