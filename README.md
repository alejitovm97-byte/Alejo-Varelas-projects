# Alejo-Varelas-projects
Finance projects
# HRP + Quality Boost Portfolio

> **Hierarchical Risk Parity with a multiplicative quality tilt — backtested vs MSCI World (2020–2024)**

## Overview

This project implements an equity portfolio strategy that combines two complementary ideas:

1. **Hierarchical Risk Parity (HRP)** — a graph-theory based allocation method introduced by Marcos López de Prado (2016) that uses hierarchical clustering to allocate risk without inverting the covariance matrix, making it more robust than classical mean-variance optimisation.

2. **Quality Boost** — a multiplicative overlay that tilts HRP weights towards higher-quality compounders based on fundamental scores, without breaking the hierarchical risk structure.

The strategy is rebalanced across four periods (Mar 2020 – Sep 2024) and evaluated against the MSCI World Index (ETF: URTH).

---

## Methodology

### 1. Asset Universe

A concentrated universe of 25 global quality compounders, rotated semi-annually. Assets are selected based on qualitative fundamental analysis: strong ROIC, pricing power, low leverage, and durable competitive advantages. Examples include MSFT, MA, NVO, ASML, SAP, and LVMUY.

### 2. HRP Optimisation

Returns are downloaded from Yahoo Finance (adjusted close prices). The Spearman rank correlation matrix is used as the codependence measure, making the model robust to non-normal return distributions. Ward linkage is used for hierarchical clustering.

```
Codependence : Spearman rank correlation
Linkage      : Ward
Risk measure : Variance (MV)
Max clusters : 10
```

### 3. Quality Boost (Multiplicative)

Raw quality scores (proprietary, based on fundamental analysis) are converted to **percentile ranks** to avoid sensitivity to outlier scores. A boost factor is then computed:

```
boost_factor = 1 + α × (percentile_rank − 0.5)
```

This factor is multiplied by each asset's HRP weight, and the result is re-normalised to sum to 1. The intensity parameter `α = 0.30` produces a moderate tilt (±15% maximum adjustment).

**Key property:** the multiplicative approach preserves the relative risk structure produced by HRP. Assets receive a higher share only if they already have a non-trivial HRP allocation, preventing the quality overlay from overriding diversification.

### 4. Smart Rebalancing Buffer

At each rebalancing date, positions with `|Δw| < 0.5%` are frozen. The freed capital is redistributed proportionally among positions that do require adjustment. This reduces unnecessary turnover while keeping the portfolio close to its target.

```
threshold = 0.5%
frozen capital → unchanged
available capital → redistributed to positions with |Δw| ≥ threshold
```

### 5. Backtesting

The portfolio is backtested across four non-overlapping holding periods:

| Period | Training Window       | Holding Window        |
|--------|-----------------------|-----------------------|
| T0     | Mar 2020 – Mar 2023   | Mar 2020 – Sep 2021   |
| T1     | Sep 2021 – Sep 2023   | Sep 2021 – Mar 2022   |
| T2     | Mar 2021 – Mar 2024   | Mar 2022 – Sep 2023   |
| T3     | Sep 2021 – Sep 2024   | Sep 2023 – Sep 2024   |

Daily returns are stitched across periods. No look-ahead bias: weights are always trained on data available at the rebalancing date.

---

## Project Structure

```
hrp-quality-portfolio/
│
├── src/
│   └── hrp_quality.py      # Main module — full pipeline
│
├── notebooks/
│   └── hrp_v3.ipynb        # Original Colab notebook (exploratory)
│
├── results/                # Output charts and metrics (generated on run)
│
├── requirements.txt
└── README.md
```

---

## Quickstart

```bash
# Clone the repository
git clone https://github.com/<your-username>/hrp-quality-portfolio.git
cd hrp-quality-portfolio

# Install dependencies
pip install -r requirements.txt

# Run the full pipeline
python src/hrp_quality.py
```

The script will:
- Download price data automatically via `yfinance`
- Run HRP optimisation for each period
- Apply the quality boost and rebalancing buffer
- Print a performance summary table
- Display charts: correlation heatmap, dendrogram, cumulative returns, drawdown, rolling returns

---

## Key Design Decisions

**Why Spearman correlation?**
Equity returns are heavy-tailed and exhibit non-linear dependencies, particularly during market stress. Spearman's rank correlation captures monotonic (not just linear) relationships and is more robust to outliers than Pearson.

**Why multiplicative (not additive) boost?**
An additive overlay can assign meaningful weights to assets that HRP allocates near zero — effectively overriding the risk-parity structure. The multiplicative approach ensures the quality tilt is proportional to the existing HRP allocation, maintaining diversification.

**Why a buffer at rebalancing?**
Frequent small trades generate transaction costs without meaningfully improving portfolio characteristics. The 0.5% threshold freezes stable positions and concentrates rebalancing activity where it matters.

---

## Dependencies

| Library        | Purpose                          |
|----------------|----------------------------------|
| `riskfolio-lib`| HRP optimisation and clustering  |
| `yfinance`     | Price data download              |
| `pandas`       | Data manipulation                |
| `numpy`        | Numerical computation            |
| `matplotlib`   | Visualisation                    |
| `seaborn`      | Correlation heatmap              |

---

## Academic Context

Developed as part of the **Master in Finance & Banking (Markets Specialization)** at Universitat Pompeu Fabra, Barcelona. The model was designed to explore whether a fundamental quality overlay can improve the risk-adjusted performance of an HRP portfolio without sacrificing its diversification properties.

**Reference:**
López de Prado, M. (2016). *Building diversified portfolios that outperform out of sample*. Journal of Portfolio Management, 42(4), 59–69.

## Technologies & Libraries
**Riskfolio-Lib**: Portfolio optimization engine based on the work by [Dany Cajas (dcajasn)](https://github.com/dcajasn/Riskfolio-Lib).
---

## Author

**Alejo Varela**
Master in Finance & Banking — Universitat Pompeu Fabra (2026)
[Email](mailto:alejo.varela.m@hotmail.com)
