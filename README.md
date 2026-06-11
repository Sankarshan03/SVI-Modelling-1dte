
# SPX Options SVI Modeling (0/1/2 DTE)

A complete pipeline for fitting the **SVI (Stochastic Volatility Inspired)** volatility smile to short‑dated SPX options (0, 1, and 2 days to expiry).  
The notebook loads 1‑minute options data, computes forward prices, extracts out‑of‑the‑money (OTM) options, calculates implied volatilities using the Black‑76 model, fits the SVI parameters, and checks for butterfly arbitrage.

---

## Table of Contents

- [Strategy / Model Overview](#strategy--model-overview)
- [Data Requirements](#data-requirements)
- [Setup & Installation](#setup--installation)
- [Methodology](#methodology)
- [How to Run](#how-to-run)
- [Output & Results](#output--results)
- [Customisation](#customisation)
- [Limitations & Future Extensions](#limitations--future-extensions)
- [License](#license)

---

## Strategy / Model Overview

The **SVI (Stochastic Volatility Inspired)** model is a parametric form for the implied volatility smile. It is widely used because it is:

- **Arbitrage‑free** when parameters satisfy certain constraints,
- **Parsimonious** (only 5 parameters),
- **Capable of capturing** the skew, smile, and wings of the volatility surface.

For options with very short time to expiry (0–2 days), the SVI model provides a smooth, arbitrage‑free interpolation/extrapolation of implied volatilities across strikes.

**Total implied variance** is modelled as:

w(k) = a + b * ( ρ*(k - m) + sqrt((k - m)^2 + σ²) )


where `k = ln(K/F)` is log‑moneyness and `w(k) = σ_IV² * T`.

The parameters are:
- `a` – minimum total variance,
- `b` – wing slope,
- `ρ` – skew,
- `m` – location of the minimum,
- `σ` – curvature.

The model is fitted to **OTM options** (calls with `K > F`, puts with `K < F`) to ensure a unique, stable fit.

---

## Data Requirements

The notebook loads a **pickle file** with the following structure:

python
data = {
    "meta": dict,
    "spot": pd.DataFrame,
    "options": pd.DataFrame
}


### 1. Options DataFrame

| Column       | Type    | Description |
|--------------|---------|-------------|
| `ts_utc`     | datetime | UTC timestamp (naive) |
| `ts_et`      | datetime | Eastern Time timestamp (naive) |
| `underlying` | str      | `SPX` or `SPXW` |
| `expiry`     | date     | Expiration date |
| `dte`        | int      | Days to expiry (0,1,2) |
| `strike`     | float    | Strike price |
| `right`      | str      | `C` (call) or `P` (put) |
| `open`       | float    | 1‑minute open price |
| `high`       | float    | 1‑minute high price |
| `low`        | float    | 1‑minute low price |
| `close`      | float    | 1‑minute close price |
| `volume`     | int      | Volume traded |

### 2. Spot DataFrame

| Column   | Type      | Description |
|----------|-----------|-------------|
| `ts_utc` | datetime  | UTC timestamp |
| `ts_et`  | datetime  | Eastern Time timestamp |
| `close`  | float     | Synthetic SPXW close price |

**Important notes** (from metadata):
- Synthetic spot is derived from options via put‑call parity – **not** a market quote.
- For DTE 1 and 2, both SPX (AM‑settled monthly) and SPXW (PM‑settled weekly) exist. The notebook uses `SPXW`.
- DTE 0 rows are all SPXW.
- Greeks, bid‑ask, open interest are **not** included.

---

## Setup & Installation

### Prerequisites
- Python 3.8 or higher
- Recommended: Google Colab, Jupyter Notebook, or local Python environment

### Install Dependencies

bash
pip install pandas numpy scipy matplotlib openpyxl


### Project Structure

Place the pickle file in your Google Drive (or local directory) with the following path:
```
/content/drive/MyDrive/spx-data/spx_0123dte_6mo.pkl
```

If you change the path, update the `open()` statement inside the notebook.

---

## Methodology

### 1. Snapshot Selection
- Trading day: `2026-01-15`
- Expiry: `2026-01-16` (Friday, 1 DTE)
- Snapshot time: `12:00:00 ET`

### 2. Forward Price

```
F = S * exp(r * T)
```

- `S` = synthetic spot at the snapshot minute.
- `r` = 10‑year Treasury yield (assumed **4.5%** for demonstration – should be fetched live).
- `T` = time to expiry in years (calendar days / 365.25).

### 3. Implied Volatility (Black‑76)

For each OTM option, solve:

```
Price = Black_76(F, K, T, r, σ, right)
```

using `scipy.optimize.brentq`.

### 4. SVI Fit

Minimise the squared residuals between market total variance `w_market = σ_IV² * T` and the SVI function `w(k)`.

Bounds:
- `b ≥ 0`
- `-0.999 < ρ < 0.999`
- `σ ≥ 0`

Initial guess: `[a=0.04, b=0.2, ρ=-0.3, m=0.0, σ=0.1]`

Optimiser: `scipy.optimize.least_squares`

### 5. Butterfly Arbitrage Check

- Generate a dense grid of strikes.
- Compute SVI total variance → IV → call prices (Black‑76).
- Compute second derivative of call price with respect to strike (`d²C/dK²`) via finite differences.
- If `d²C/dK² < 0` (with tolerance `1e-6`), a butterfly static arbitrage exists.

---

## How to Run

### Option 1: Run in Google Colab (recommended)

1. Upload the pickle file to your Google Drive at the expected path.
2. Open the notebook in Colab.
3. Run all cells sequentially.

### Option 2: Run locally

1. Change the file path to your local path.
2. Install dependencies.
3. Execute the notebook as usual.

**Expected runtime**: less than 10 seconds (data loading is the heaviest part).

---

## Output & Results

### 1. Console Output

```
Synthetic spot = 6974.69
Risk‑free rate (10y Treasury) = 4.50%
Time to expiry (years) = 0.002738
Forward price = 6975.55

Computed IVs for 37 OTM options

SVI parameters:
a  = -0.001431
b  = 0.018732
ρ  = -0.175986
m  = -0.006947
σ  = 0.079119

RMSE between market IV and SVI-fitted IV: 0.002560

Butterfly arbitrage exists: False
```

### 2. Plots

- **Spot price over time** – full time series of synthetic SPXW.
- **Implied volatility vs. strike** – market OTM points and SVI‑fitted curve.
- **SVI‑fitted total variance vs. log‑moneyness** – fit quality.
- **Butterfly check** – call prices and second derivative over strike grid.

### 3. Interpretation

- **RMSE = 0.00256** → excellent fit.
- **No butterfly arbitrage** → the fitted SVI smile is statically arbitrage‑free.
- **Negative ρ (-0.176)** → slight downward skew (puts slightly more expensive than calls).
- **m ≈ -0.007** → minimum variance very close to at‑the‑money.

---

## Customisation

| What to change | Where to change |
|----------------|-----------------|
| Snapshot date, expiry, time | Cell 4: `day`, `exp`, `snap_time` |
| Risk‑free rate | Cell 5: `r = 0.045` |
| Filter by DTE | Cell 4: add `& (options['dte'] == 1)` |
| Initial SVI parameters | Cell 8: `x0 = [...]` |
| Bounds for SVI fit | Cell 8: `bounds = (lower, upper)` |
| OTM filter (e.g., include near‑the‑money) | Cell 7: modify `otm_mask` |

---

## Limitations & Future Extensions

| Limitation | Suggested improvement |
|------------|------------------------|
| Risk‑free rate is hard‑coded (4.5%) | Fetch real‑time 10‑year Treasury yield from an API (e.g., FRED) |
| Synthetic spot is used | Use market spot if available, or compare both |
| Only one snapshot analysed | Wrap in a loop over multiple days to study parameter stability |
| No calendar spread arbitrage check | Extend to test for dynamic arbitrage |
| No out‑of‑sample validation | Split data into fit / test sets |


## References

- Gatheral, J. (2004). *A Parsimonious Arbitrage‑Free Implied Volatility Parameterization*.
- Black, F. (1976). *The pricing of commodity contracts*.
- `scipy.optimize.least_squares` – [documentation](https://docs.scipy.org/doc/scipy/reference/generated/scipy.optimize.least_squares.html)
