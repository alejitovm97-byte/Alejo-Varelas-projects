# Macro-Driven ETF & Fund Portfolio
### A Two-Stage Investment Strategy: Macro Cycle Detection + Black-Litterman Optimization

> **Academic Project** — Master in Finance & Banking, UPF Barcelona  
> **Analysis Date:** January 2024 | **Backtest Period:** Jan 2024 → Jan 2026

---

## Overview

This project implements a systematic, macro-driven asset allocation strategy for a mixed ETF and mutual fund universe. The core idea is simple: **where we are in the economic cycle determines what we should own**. The strategy combines a quantitative macro detector with a Black-Litterman portfolio optimizer, rebalanced semiannually based on real FRED data.

The full pipeline is structured in two blocks:

```
Block 1 — Macro Cycle Detector   →  Identifies the current economic phase
Block 2 — Fund Screening + BL    →  Selects the best instruments and optimizes weights
```

---

## Methodology

### Block 1 — Macro Cycle Detector

Inspired by the **Merrill Lynch Investment Clock**, the model classifies the economy into four phases using two composite vectors:

| Phase | Growth | Inflation | Recommended Asset Class |
|-------|--------|-----------|------------------------|
| I — Recovery | ↑ | ↓ | Equities |
| II — Overheating | ↑ | ↑ | Commodities, Inflation Hedges |
| III — Hard Landing | ↓ | ↑ | Cash, Short-Term Bonds |
| IV — Soft Landing | ↓ | ↓ | Long-Term Bonds |

**Macro variables sourced from FRED:**

| Variable | Series | Role |
|----------|--------|------|
| Industrial Production YoY | `INDPRO` | Growth proxy (replaces PMI — available without lag) |
| Yield Curve | `T10Y2Y` | Recession signal |
| CPI YoY | `CPIAUCSL` | Inflation level |
| PPI YoY | `PPIACO` | Inflation momentum |

> **Note on PMI:** In a production environment, ISM PMI or Markit PMI would be the preferred growth indicator (forward-looking, survey-based). INDPRO is used here as a FRED-accessible proxy — a documented methodological limitation.

**Scoring logic (level + trend):**
```
signal_level = +1 if level > threshold, -1 otherwise
signal_trend = +1 if 3M change > 0,     -1 otherwise
score = 0.40 × signal_level + 0.60 × signal_trend

growth_score    = 0.45 × INDPRO_level + 0.35 × INDPRO_momentum + 0.20 × YieldCurve
inflation_score = 0.50 × CPI_score    + 0.30 × PPI_score       + 0.20 × CPI_momentum
```

**Conviction** is measured as the Euclidean distance from the origin:
```
distance = sqrt(growth² + inflation²) / sqrt(2)

HIGH   ≥ 0.55  →  Core 80% / Satellite 20%
MEDIUM ≥ 0.30  →  Core 70% / Satellite 30%
LOW    < 0.30  →  Core 60% / Satellite 40%
```

**Result — January 2024:**
```
Growth: -1.00  |  Inflation: -0.24  →  Phase IV — Soft Landing  (HIGH conviction)
→ Overweight Long-Term Bonds
```

---

### Block 2 — Fund Screening & Portfolio Construction

#### Universe
- **761 instruments** sourced from Morningstar (327 ETFs + 434 Mutual Funds)
- **13 ETF categories** / **10 Fund categories**
- Cleaned to **684 valid instruments** after data quality filters

#### Scoring (within-group quintile ranking)
ETFs compete against ETFs in the same category. Funds compete against funds in the same category — apples to apples.

| Metric | Weight | Rationale |
|--------|--------|-----------|
| Sharpe 3Y | 30% | Risk-adjusted return |
| Return 3Y | 25% | Full-cycle performance |
| Alpha 3Y | 20% | Manager value-add |
| Return 1Y | 15% | Recent momentum |
| StdDev 3Y | 10% | Volatility penalty (inverted) |

Scores normalized **min-max within each category × type group** → quintile assigned (Q1 = top 20%).

#### Phase IV Asset Allocation
```
CORE (80%)        →  USD Bond Long Term  +  EUR Bond
TRANSITION (20%)  →  US Large-Cap Equity + Global Large-Cap Equity
```

#### Final Portfolio — 8 Selected Instruments

| Role | Type | Instrument | ISIN | Sharpe 3Y | Return 3Y |
|------|------|-----------|------|-----------|-----------|
| CORE | ETF | iShares US Mortgage Backed Securities UCITS ETF | IE00BZ6V7883 | 0.06 | +4.68% |
| CORE | Fund | BNP Paribas Flexi I US Mortgage I Cap | LU1080341909 | 0.23 | +6.10% |
| CORE | ETF | iShares France Govt Bond UCITS ETF | IE00B7LGZ558 | -0.04 | +2.70% |
| CORE | Fund | CT European Bond Fund Inst. Gross Acc | GB00B3WLPN99 | 0.17 | +2.80% |
| TRANSITION | ETF | Invesco S&P 500 Scored & Screened ETF | IE00BKS7L097 | 1.40 | +21.20% |
| TRANSITION | Fund | Capital Group Investment Company of America (LUX) Z | LU1378997875 | 1.45 | +20.70% |
| TRANSITION | ETF | HSBC Multi Factor Worldwide Equity UCITS ETF | IE0000378O66 | 1.50 | +18.70% |
| TRANSITION | Fund | Allianz Best Styles Global AC Equity Fund C | GB00BYQ91X80 | 1.87 | +19.30% |

> **Note:** Negative Sharpe ratios on bond instruments reflect the 2022 rate-hiking cycle — not a selection error. Alpha values are net of fees (Morningstar methodology).

---

### Black-Litterman Optimization

Views are defined **per phase**, translating the macro signal into quantitative return expectations:

**Phase IV Views (January 2024):**
```
View 1: USD Bonds  > US Equity      by +5% annual  (medium confidence)
View 2: EUR Bonds  > Global Equity  by +3% annual  (low confidence)
View 3: All Bonds  > All Equity     by +4% annual  (medium confidence)
```

**Implementation details:**
- BL posterior: `riskfolio-lib v7.2.1`
- Optimization: `scipy.optimize.minimize` — SLSQP (Maximum Sharpe Ratio)
- Risk-free rate: **€STR 3.9%** (European investor base, January 2024)
- Risk aversion δ: 2.5
- Weight constraints: min 5% / max 25% per instrument; Bonds ≥ 40%; Equity ≤ 60%

**Optimized Weights — Phase IV:**
```
iShares US MBS        25%  ████████████████████████
BNP US Mortgage        5%  █████
iShares France Govt   25%  ████████████████████████
CT European Bond      25%  ████████████████████████
Invesco S&P 500        5%  █████
Capital Group US       5%  █████
HSBC Global            5%  █████
Allianz Global         5%  █████
──────────────────────────────────────────────────
CORE (Bonds)          80%
TRANSITION (Equity)   20%

Expected Return BL : 4.64%  |  Volatility: 2.11%  |  Sharpe: 0.350
```

---

## Backtest Results

**Period:** January 2024 → January 2026  
**Rebalancing:** Semiannual  
**Macro re-evaluation:** Fully automated via FRED at each rebalancing date  
**Initial capital:** 100 (base index)

### Phase Timeline & Returns

| Period | Phase Detected | Conviction | Portfolio Tilt | Return |
|--------|---------------|------------|----------------|--------|
| Jan → Jul 2024 | IV — Soft Landing | HIGH | 80% Bonds / 20% Equity | +3.51% |
| Jul 2024 → Jan 2025 | I — Recovery | MEDIUM | 20% Bonds / 80% Equity | +8.42% |
| Jan → Jul 2025 | II — Overheating | HIGH | 20% Bonds / 80% Equity | +1.69% |
| Jul 2025 → Jan 2026 | II — Overheating | HIGH | 20% Bonds / 80% Equity | +10.27% |

### Performance Summary

| Metric | Strategy | S&P 500 (same period, approx.) |
|--------|----------|--------------------------------|
| Total Return | **+25.84%** | ~+40% |
| Annual Return | **+12.18%** | ~+20% |
| Volatility | **6.97%** | ~17% |
| Sharpe Ratio | **1.19** | ~1.10 |
| Max Drawdown | **-6.88%** | ~-15% |

The strategy captures approximately **60% of S&P 500 returns with less than half the volatility and drawdown** — a favorable risk-adjusted profile for a diversified multi-asset mandate.

**Key insight:** The Phase IV → Phase I rotation in July 2024 was the pivotal moment. The model correctly detected the macro shift and rotated into equity ahead of the H2 2024 rally, generating the most efficient period of the backtest (Sharpe 0.948). This validates the core thesis: systematic macro rotation adds value over static allocation.

---

## Limitations & Future Work

### Current Limitations

| Limitation | Impact | Mitigation |
|------------|--------|------------|
| Small universe (8 instruments) | Phases II & III poorly covered | Expand to 20-30 instruments |
| INDPRO instead of PMI | Lagging signal, may miss early cycle turns | ISM/Markit PMI via paid API |
| Short backtest (2 years) | Equity-favorable period, no stress scenario | Extend to 2010–2026 |
| No transaction costs | Overstates net performance | Model bid-ask + fund fees |
| Morningstar data gaps | ~43% missing 10Y data; fees only for funds | Full Bloomberg/Refinitiv dataset |

### Version 2 Roadmap

- [ ] **Expand universe** to 20-30 instruments covering all four phases (add gold, commodity ETFs, T-Bills, TIPS, REITs)
- [ ] **Momentum overlay** — only allocate to instruments with positive 3M price momentum. Well-documented in Fama-French (2012); improves Sharpe without increasing drawdown
- [ ] **Real PMI data** via ISM or Markit API for earlier cycle detection
- [ ] **Equal Risk Contribution (ERC)** as alternative to Max Sharpe — more robust out-of-sample
- [ ] **Extended backtest** (2010–2026) including 2022 stress scenario (simultaneous bond + equity drawdown)
- [ ] **Transaction cost modeling** with realistic spreads and rebalancing costs

---

## Project Structure

```
macro-etf-portfolio/
│
├── macro_cycle_detector.py     # Block 1: FRED data pipeline + phase detection
├── fund_screening.py           # Block 2: Morningstar data + quintile scoring
├── black_litterman.py          # BL views + scipy optimization engine
├── backtest.py                 # Semiannual rebalancing + performance attribution
│
├── data/
│   ├── etfs/                   # 13 categories × 2 files (performance + risk)
│   └── funds/                  # 10 categories × 3 files (performance + risk + fee)
│
├── outputs/
│   ├── portfolio_final.png     # BL optimized weights visualization
│   └── backtest_results.png    # Full backtest dashboard (NAV, phases, metrics)
│
├── requirements.txt
└── README.md
```

---

## Requirements

```
pandas>=2.0
numpy>=1.24
matplotlib>=3.7
scipy>=1.11
yfinance>=0.2
fredapi>=0.5
riskfolio-lib>=7.2.1
openpyxl>=3.1
```

---

## Methodological References

- **Merrill Lynch Investment Clock** — Trevor Greetham (2004). Asset allocation across the business cycle.
- **Black, F. & Litterman, R.** (1992). Global Portfolio Optimization. *Financial Analysts Journal, 48(5).*
- **Idzorek, T.** (2005). A Step-by-Step Guide to the Black-Litterman Model. *Zephyr Associates.*
- **Fama, E. & French, K.** (2012). Size, Value, and Momentum in International Stock Returns. *Journal of Financial Economics, 105(3).*

---

*Academic portfolio project demonstrating quantitative macro analysis, systematic asset allocation, and Python-based financial modeling.*  
*Data sources: Morningstar (fund universe), FRED — Federal Reserve Economic Data (macro indicators), Yahoo Finance (price history).*
