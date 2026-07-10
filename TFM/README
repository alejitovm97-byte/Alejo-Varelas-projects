# Construcción de factores y gestión de riesgo con Hierarchical Risk Parity en el S&P 500

> Trabajo Final de Máster — Máster en Finanzas y Banca, UPF Barcelona School of Management (2025-26)

Un proceso de inversión reproducible en dos bloques: **(1)** un modelo multifactorial que rankea las acciones del S&P 500 por Quality, Value y Momentum, y **(2)** la construcción de carteras sobre ese universo con **Hierarchical Risk Parity (HRP)**, evaluada con un backtest dinámico 2011–2025.

El diseño separa deliberadamente *qué comprar* (selección de factores) de *cuánto comprar* (asignación por riesgo), lo que permite medir la contribución de cada componente por separado. Combina tres líneas de la literatura que rara vez se juntan: factor investing (Asness et al.), Hierarchical Risk Parity (López de Prado) y shrinkage de covarianza (Ledoit-Wolf).

## Resultados principales

Backtest abr 2011 – dic 2025 · costes 10 pb · Rf = 1,48%

| Estrategia | CAGR | Vol. | Sharpe | Max DD | €100k → |
|---|---|---|---|---|---|
| **S1 · Quality (HRP)** | 14,17% | 16,79% | **0,76** | −30,82% | **€702k** |
| S2 · Quality + Momentum (HRP) | 14,48% | 18,36% | 0,71 | −31,84% | €731k |
| S3 · Quality + Value + Momentum (HRP) | 12,62% | 17,49% | 0,64 | −38,01% | €574k |
| Benchmark · S&P 500 | 11,76% | 17,35% | 0,59 | −33,92% | €514k |

**Hallazgos:**

1. **Los factores predicen el retorno futuro, medido correctamente.** El alpha ajustado por Fama-French (spread quintil alto − bajo) es positivo y significativo para Quality (3,45% anual, t = 2,19) y Quality + Momentum (4,82%, t = 2,23). El Information Coefficient simple no cruza el 5%: la señal es de baja frecuencia y se vuelve detectable al controlar por mercado, tamaño y value.
2. **Lo simple gana.** Añadir Value diluye la señal —QVM es la peor variante en rentabilidad y drawdown—, coherente con un value premium débil en el período.
3. **El alpha viene de la selección, no de HRP.** HRP reduce el riesgo de forma consistente (volatilidad −2,3 a −2,7 pp frente a la equiponderada) pero no mejora el Sharpe de forma estadísticamente significativa. Elegir bien las acciones importa más que optimizar los pesos.
4. **Quality es la mejor estrategia ajustada por riesgo:** mejor Sharpe (0,76), la única marginalmente significativa frente al S&P 500 (al 10%), menor rotación y compatible de forma natural con los límites UCITS.
5. **Validación externa:** el 78% de las posiciones *long* del fondo de AQR presentes en el universo caen en el top 60% del modelo, lo que confirma que identifica un universo de calidad creíble.

> **Nota metodológica:** la mejora frente al benchmark es económicamente relevante pero no estadísticamente significativa en Sharpe. El aporte del trabajo es el marco reproducible que separa alpha (selección) de gestión de riesgo (HRP) y mide cada uno por separado.

## Bloque 1 · Selección de factores

Ranking anual de las acciones del S&P 500 replicando *Quality Minus Junk* (Asness, Frazzini & Pedersen) y *Value and Momentum Everywhere* (Asness, Moskowitz & Pedersen).

- **Universo:** constituyentes actuales e históricos del S&P 500 (~608 empresas válidas), incluyendo empresas que salieron del índice para **mitigar el survivorship bias**.
- **Factores** (rank z-score cross-seccional por año):
  - **Quality** = Profitability + Growth + Safety (GPOA, ROE, ROA, CFOA, margen, accruals; crecimiento a 5 años; beta, leverage, Altman-Z, Ohlson-O, volatilidad del ROE).
  - **Value** = Book-to-Market.
  - **Momentum** = retorno de los meses −12 a −2.
- **Tres variantes de score:** Quality · Quality + Momentum · Quality + Value + Momentum.
- **Validación:** Information Coefficient (anual y mensual con errores Newey-West), análisis de quintiles, alpha CAPM y Fama-French 3 factores, estabilidad del ranking y overlap con posiciones reales de AQR.
- **Output:** top 60% por año → universo invertible que alimenta el Bloque 2.

## Bloque 2 · Construcción de cartera con HRP

Asignación de pesos sobre el universo seleccionado con **Hierarchical Risk Parity** (López de Prado, 2016), que nunca invierte la matriz de covarianza y evita así los pesos extremos e inestables de Markowitz.

- **Método:** correlación → distancia → clustering jerárquico → cuasi-diagonalización → bisección recursiva.
- **Covarianza:** log-retornos diarios, ventana rolling de 504 días, con shrinkage de Ledoit-Wolf.
- **Backtest dinámico:** reconstitución anual del universo + rebalanceo trimestral de pesos, con control estricto de look-ahead bias (reporting lag verificado ≥ 91 días) y costes de transacción de 10 pb.
- **Benchmarks:** S&P 500 (cap-weighted) y equiponderada 1/N con la misma frecuencia y costes, para aislar la contribución de la metodología de pesos.
- **Robustez:** linkage (single/ward), tamaño del universo (30/50/75), ventana de covarianza y límites UCITS 5/10/40.

## Estructura del repositorio

```
├── notebooks/
│   ├── 01_seleccion_factores.ipynb        # Bloque 1
│   └── 02_construccion_cartera_hrp.ipynb  # Bloque 2
├── outputs/                               # pesos, NAV, métricas y figuras
├── docs/                                  # documentación metodológica
├── data/                                  # no incluido (ver Datos)
└── README.md
```

## Datos

Los datos fundamentales y de precios provienen de **Refinitiv** (Eikon / RDP) y se usan con fines **académicos**. Por licencia, **los datos crudos no se incluyen** en el repositorio. El benchmark (S&P 500) y la tasa libre de riesgo se obtienen vía `yfinance` (^GSPC, ^IRX), y los factores de Fama-French de la Kenneth French Data Library.

## Ejecución

Requiere Python 3 con `pandas`, `numpy`, `scipy`, `scikit-learn`, `statsmodels`, `matplotlib` y `yfinance`. Los notebooks se ejecutan en orden: el Bloque 2 consume los universos seleccionados que genera el Bloque 1.

## Referencias

- Asness, C. S., Frazzini, A., & Pedersen, L. H. (2017). *Quality Minus Junk*.
- Asness, C. S., Moskowitz, T. J., & Pedersen, L. H. (2013). *Value and Momentum Everywhere*. Journal of Finance.
- López de Prado, M. (2016). *Building Diversified Portfolios that Outperform Out-of-Sample*. Journal of Portfolio Management.
- Ledoit, O., & Wolf, M. (2004). *A well-conditioned estimator for large-dimensional covariance matrices*.

## Autor

**Alejo Varela** — Trabajo Final de Máster, Máster en Finanzas y Banca, UPF Barcelona School of Management (2025-26).
