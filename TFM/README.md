# Contexto Notebook 02 (v3) — HRP Portfolio Backtest
## Documento de referencia para redacción del TFM

**Título del TFM:** Quality + Value + Momentum Portfolio con Hierarchical Risk Parity (HRP)

**Propósito de este documento:** Servir como base de conocimiento para redactar el capítulo del TFM correspondiente al Notebook 02 (construcción de cartera, backtesting y resultados). El NB01 (factor investing / stock selection) se documenta por separado.

**Nota de versión:** Los resultados de este documento corresponden a la última corrida del notebook (ejecución 2026-07-10), después de las correcciones aplicadas al NB01. Como los dos notebooks están encadenados (el NB01 genera los universos que consume el NB02), los cambios en el factor model afectaron los resultados del backtest, especialmente la estrategia S2_QM.

---

## 1. OBJETIVO Y CONTRIBUCIÓN

### Objetivo general
Construir una estrategia de inversión que maximice el Sharpe ratio y minimice drawdowns, combinando factor investing (selección de acciones) con Hierarchical Risk Parity (HRP) para la asignación de pesos.

### Separación conceptual (clave del trabajo)
El diseño separa deliberadamente dos decisiones:
- **Alpha (stock selection):** proviene del NB01 — scores de Quality (QMJ), Quality+Momentum (QM), y Quality+Value+Momentum (QVM) por año fiscal.
- **Gestión de riesgo (weight allocation):** proviene del NB02 — HRP sobre covarianza estimada con Ledoit-Wolf.

Esta separación permite **medir la contribución de cada componente** por separado, que es el hallazgo central del trabajo.

### Contribución académica
El trabajo conecta tres líneas de literatura que raramente se combinan: factor investing (Asness), Hierarchical Risk Parity (López de Prado), y shrinkage de covarianza (Ledoit-Wolf). La mayoría de los estudios de HRP lo testean con activos genéricos (ETFs, índices), no con un universo stock-level pre-filtrado por factores fundamentales.

---

## 2. PAPERS BASE

| Paper | Contribución al NB02 |
|-------|---------------------|
| López de Prado (2016) — "Building Diversified Portfolios that Outperform Out-of-Sample" | Implementación HRP: correlDist, getQuasiDiag, getRecBipart, getClusterVar, getIVP |
| Ledoit & Wolf (2004) — "A well-conditioned estimator for large-dimensional covariance matrices" | Justificación del shrinkage para estabilizar la matriz de covarianza |
| Asness, Frazzini & Pedersen (2017) — "Quality Minus Junk" | Score de Quality (usado como input, del NB01) |
| Asness, Moskowitz & Pedersen (2013) — "Value and Momentum Everywhere" | Momentum 12-2 y Value B/M (input, del NB01) |
| Jobson & Korkie (1981), corregido por Memmel (2003) | Test estadístico de diferencia de Sharpe ratios |
| Leland (1999) | No-trade bands (usado en v2, eliminado en v3) |

---

## 3. DATOS

### Fuente de precios
- **Refinitiv (vía API RDP):** precios diarios de cierre (campo TRDPRC_1, ajustados por splits) de 590 RICs, período enero 2008 – diciembre 2025 (4,529 trading days). Descarga directa vía `get_historical_price_summaries`, guardada en un único archivo `daily_prices_refinitiv_all.csv` para no depender de la conexión en vivo.
- **yfinance:** únicamente para ^GSPC (benchmark S&P 500) y ^IRX (T-Bill 3 meses, tasa libre de riesgo). No se usa para precios individuales, evitando problemas de empalme entre fuentes.

### Universo
- 591 RICs únicos provenientes de los 3 CSVs de universo del NB01 (top 60% por score en cada variante, FY2010-2025).
- 590 descargados exitosamente; 1 (HOLX.OQ) no disponible, afecta solo a S1 en 2 de 15 años fiscales y se reemplaza con el siguiente del ranking.
- Tras la corrección del NB01, los CSVs de universo tienen: S1_Quality 4,729 filas, S2_QM 4,715 filas, S3_QVM 4,520 filas.
- Los RICs tienen formato Refinitiv (ej: AAPL.OQ, KO.N).

### Tres estrategias evaluadas
- **S1_Quality:** Top 50 por score Quality (QMJ). Estrategia más estable y mejor ajustada por riesgo.
- **S2_QM:** Top 50 por score Quality+Momentum. Estrategia principal por diseño (mejor IC en NB01).
- **S3_QVM:** Top 50 por score Quality+Value+Momentum. Value diluye la señal.

---

## 4. METODOLOGÍA DEL BACKTEST

### Parámetros globales
- TOP_N = 50 acciones por portfolio
- WINDOW = 504 trading days (~2 años) para la covarianza
- LINKAGE = 'single' (caso base, trazable al paper); 'ward' como robustness
- Sin cap ni threshold (HRP puro en v3)
- COST_BP = 0.0010 (10 basis points sobre turnover neto)
- Período efectivo: abril 2011 – diciembre 2025 (~14.75 años, 59 rebalanceos)
- Rebalanceo: trimestral de pesos (enero, abril, julio, octubre); reconstitución anual del universo (abril)

### Decisiones metodológicas clave
1. **Log-retornos para covarianza, retornos simples para NAV.** Log-retornos tienen mejores propiedades estadísticas para estimar covarianza; la suma ponderada de retornos simples es el retorno exacto del portfolio.
2. **Ventana rolling de 504 días** como balance entre historia y adaptabilidad.
3. **Single linkage** como caso base por trazabilidad al paper original de López de Prado.
4. **HRP puro sin constraints** para preservar la señal del modelo. UCITS 5/10/40 como robustness check.
5. **Reconstitución anual del universo, rebalanceo trimestral de pesos.** Separa la decisión de "qué comprar" (anual) de "cuánto" (trimestral).
6. **Pesos drifted** calculados por market returns antes de cada rebalanceo, para medir el turnover real.

### Control de look-ahead bias (punto crítico)
- Fundamentales de FY(t) cierran en diciembre del año t, se publican ~marzo del año t+1, el portfolio entra el primer día hábil de abril del año t+1.
- **Reporting lag verificado: mínimo 91 días.**
- Precios para covarianza son estrictamente anteriores a la fecha de rebalanceo (máscara `index < rebal_date`).
- El ranking de acciones usa scores del NB01 sobre datos del año fiscal anterior, nunca del corriente.

### Ejemplo del primer rebalanceo
- Fundamentales usados: FY2010 (cierre dic 2010, disponibles ~abril 2011)
- Ventana covarianza: abril 2009 – marzo 2011 (504 días hábiles hacia atrás)
- Entrada al portfolio: 1 abril 2011

### Pipeline por rebalanceo
1. **Selección de universo** (solo abril): top 50 RICs por score del FY anterior. Validar ≥80% de datos en la ventana; si no, reemplazar con el siguiente del ranking extendido (top 75).
2. **Covarianza:** log-retornos 504 días → Ledoit-Wolf shrinkage.
3. **Pesos HRP:** correlación → distancia d(i,j)=√(0.5(1-ρ)) → clustering jerárquico (single) → quasi-diagonalización → bisección recursiva. Sin cap.
4. **Ejecución:** turnover vs pesos drifted → costos 10bp → descontar del NAV el día del rebalanceo. NAV diario con retornos simples ponderados.

### Detección de delisting
Implementada detección de acciones que dejan de cotizar entre rebalanceos (gap >5 días consecutivos sin precio), con reemplazo inmediato para mantener 50 posiciones. En la práctica no se detectaron delistings en ninguna variante (coherente con universo de alta calidad del S&P 500). Sí hubo 8 reemplazos por datos insuficientes en el momento del rebalanceo (RICs sin 2 años de historia de precios).

### Funciones HRP (López de Prado 2016)
- `correlDist(corr)`: distancia d(i,j) = √(0.5·(1-ρ))
- `getQuasiDiag(link)`: reordena las hojas del dendrograma para quasi-diagonalizar
- `getIVP(cov)`: inverse-variance portfolio, w = (1/σ²)/Σ(1/σ²)
- `getClusterVar(cov, cItems)`: varianza del cluster con pesos IVP
- `getRecBipart(cov, sortIx)`: bisección recursiva top-down, reparte peso inversamente proporcional a la varianza de cada sub-cluster

### Ejemplo de covarianza (primer rebalanceo, S2_QM, abril 2011)
- Matriz 50×50, simétrica, PSD (min eigenvalue = 8.19e-05)
- Shrinkage Ledoit-Wolf = 0.0204 (bajo; la matriz sample ya era razonablemente estable con 504 obs)
- Condition number = 143
- Volatilidad anualizada: 19.2% – 72.7%
- Correlación promedio off-diagonal = 0.310 (rango: -0.02 a 0.826)

---

## 5. BENCHMARKS

- **B1: S&P 500 cap-weighted** (^GSPC normalizado a base 100). NAV final 513.8.
- **B2: Equal-Weight con rebalanceo trimestral** — misma frecuencia que HRP, mismo universo, mismas fechas, mismos costos (incluye costos de rotación anual del universo + rebalanceo a 1/N). Aísla la contribución de la metodología de pesos. Turnover EW menor que HRP (11.8%-22.5% vs 16.3%-27.2%).

---

## 6. MÉTRICAS

- **CAGR** (geométrico): (NAV_T/NAV_0)^(252/T) - 1
- **Volatilidad anualizada:** std(retornos diarios) × √252
- **Sharpe:** (CAGR - rf) / vol, con rf = T-Bill 3m promedio (1.48%)
- **Sortino:** (CAGR - rf) / downside deviation
- **Max Drawdown:** máxima caída desde un pico del NAV
- **Calmar:** CAGR / |MaxDD|
- **Test de Jobson-Korkie (1981) corregido por Memmel (2003):** diferencia de Sharpe entre estrategias, considerando la correlación entre ellas.
- **Descomposición de turnover:** componente de universo (cambio anual de acciones) vs componente de pesos (rebalanceo trimestral HRP).

---

## 7. RESULTADOS PRINCIPALES (última corrida)

### Tabla de métricas (período abr 2011 – dic 2025, Rf=1.48%, costos 10bp)

| Estrategia | CAGR | Vol | Sharpe | Sortino | Max DD | Calmar | NAV | Costo total |
|-----------|------|-----|--------|---------|--------|--------|-----|-------------|
| S1 Quality HRP | 14.71% | 17.11% | 0.77 | 0.99 | -30.33% | 0.48 | 753.1 | 0.96% |
| S1 Quality EW | 16.03% | 19.46% | 0.75 | 0.98 | -30.83% | 0.52 | 890.7 | 0.69% |
| S2 QM HRP | 13.67% | 18.57% | 0.66 | 0.86 | -31.26% | 0.44 | 658.6 | 1.60% |
| S2 QM EW | 15.68% | 21.15% | 0.67 | 0.88 | -31.48% | 0.50 | 852.1 | 1.30% |
| S3 QVM HRP | 12.28% | 17.62% | 0.61 | 0.75 | -37.74% | 0.33 | 549.3 | 1.59% |
| S3 QVM EW | 13.76% | 20.20% | 0.61 | 0.77 | -39.47% | 0.35 | 666.2 | 1.33% |
| B1 S&P 500 | 11.76% | 17.35% | 0.59 | 0.73 | -33.92% | 0.35 | 513.8 | 0.00% |

**Cambio importante vs la versión anterior del documento:** tras la corrección del NB01, S2_QM bajó su Sharpe de ~0.71 a 0.66 y su NAV de ~731 a 658.6. S1_Quality se mantuvo estable (0.77) e incluso mejoró levemente. Esto refuerza que S1 Quality es ahora, con más claridad, la estrategia dominante ajustada por riesgo.

### Contribución de HRP (vs Equal-Weight trimestral)

| Variante | Sharpe HRP | Sharpe EW | Δ Sharpe | Vol HRP | Vol EW | Δ Vol | Cost HRP | Cost EW |
|----------|-----------|-----------|----------|---------|--------|-------|----------|---------|
| S1 Quality | 0.77 | 0.75 | +0.03 | 17.11% | 19.46% | -2.35% | 0.96% | 0.69% |
| S2 QM | 0.66 | 0.67 | -0.01 | 18.57% | 21.15% | -2.57% | 1.60% | 1.30% |
| S3 QVM | 0.61 | 0.61 | +0.00 | 17.62% | 20.20% | -2.58% | 1.59% | 1.33% |

HRP reduce la volatilidad ~2.4-2.6pp en las 3 variantes. Solo en S1 Quality el Δ Sharpe es positivo (+0.03); en S2 y S3 es neutro o marginalmente negativo.

### Test estadístico de Sharpe (vs S&P 500)
- S1 Quality HRP: z=1.69, p=0.091 (*significativo al 10%*)
- S2 QM HRP: z=0.64, p=0.523 (no significativo)
- S3 QVM HRP: z=0.21, p=0.834 (no significativo)
- S1 Quality EW: z=1.67, p=0.094 (*significativo al 10%*)
- HRP vs EW: ninguna diferencia significativa (p>0.68 en todas las variantes)

### Distribución de pesos ejemplo (S2_QM, abril 2011)
- Peso máximo 9.1% (SJM.N, Consumer Staples, vol 19.2%), mínimo 0.2% (LVS.N, vol 72.7%)
- Top 5 concentran 30.2%
- Ilustra el mecanismo de HRP: sobrepondera baja volatilidad (SJM, defensivo), subpondera alta volatilidad (LVS, WYNN, casinos)

### Descomposición de turnover (promedio anual)
- S1 Quality: total 64.2% (universo 19.4% + pesos 44.8%)
- S2 QM: total 106.8% (universo 40.9% + pesos 65.9%)
- S3 QVM: total 105.9% (universo 40.9% + pesos 65.0%)

Quality rota mucho menos el universo que Momentum (19.4% vs 40.9%), lo que explica sus menores costos (0.96% vs 1.60% acumulado).

---

## 8. ANÁLISIS POR SUBPERÍODOS

| Métrica | Período | S1 HRP | S1 EW | S2 HRP | S2 EW | S&P 500 |
|---------|---------|--------|-------|--------|-------|---------|
| Sharpe | 2011-2014 | 1.20 | 1.01 | 0.85 | 0.76 | 0.70 |
| Sharpe | 2015-2019 | 1.01 | 0.91 | 0.90 | 0.80 | 0.59 |
| Sharpe | 2020-2025 | 0.46 | 0.56 | 0.45 | 0.58 | 0.56 |
| Max DD | 2011-2014 | -16.06% | -18.79% | -19.87% | -23.85% | -19.39% |
| Max DD | 2015-2019 | -18.15% | -22.05% | -20.00% | -23.77% | -19.78% |
| Max DD | 2020-2025 | -30.33% | -30.83% | -31.26% | -31.48% | -33.92% |

HRP brilla en 2011-2019 (mercados "normales", Sharpe S1 de 1.20 y 1.01) y pierde ventaja en 2020-2025 (rally tech concentrado, donde HRP subpondera los activos de alta volatilidad que más subieron). HRP reduce el max drawdown en todos los subperíodos.

---

## 9. ROBUSTNESS CHECKS

Los resultados son estables en todas las dimensiones (Sharpe varía poco):
- **UCITS 5/10/40:** idéntico al caso base (S1: 0.77, S2: 0.66). HRP con 50 activos cumple UCITS naturalmente; el pico ocasional de S2 (10.5%) se recorta sin impacto.
- **Linkage ward vs single:** S1 baja de 0.77 a 0.76, S2 baja de 0.66 a 0.65. Diferencia mínima. Ward produce clusters más balanceados; single sufre "chaining effect" (un mega-cluster de ~45 activos + singletons).
- **N = 30/50/75 (S2_QM):** N=30 mejor CAGR (15.55%, concentra alpha), N=75 mejor Sharpe (0.69) y más diversificado, N=50 caso base (0.66).
- **Ventana 252 vs 504 días (S2_QM):** 252 gana marginalmente en Sharpe (0.68 vs 0.66).

---

## 10. ANÁLISIS DE CONCENTRACIÓN DE PESOS

- HRP siempre mantiene 50 posiciones.
- Concentración por acción moderada: peso máximo promedio ~6.4-6.5%, pico máximo 9.3-10.5% según variante. Top 5 acciones promedian ~24% del portfolio.
- La concentración correlaciona con la **dispersión de volatilidad del mercado**, no con el paso de los trimestres. Años de alta dispersión tienen mayor concentración; años de baja dispersión, menor.
- No hay concentración acumulativa intra-anual (HRP recalcula todo desde cero cada trimestre).
- Distribución sectorial razonable: en S2, Information Technology (20.6%) y Health Care (18.5%) empatan arriba, ningún sector domina en exceso. En S1, Health Care es más pesado (26.7%). HRP diversifica por estructura de correlaciones, no por clasificación sectorial (dos empresas de sectores distintos con beta/momentum similar quedan en el mismo cluster).

---

## 11. HALLAZGOS CENTRALES (para conclusiones del TFM)

1. **El alpha viene de la selección de acciones, no de HRP.** Las 3 variantes superan al S&P 500, pero equal-weight supera a HRP en retorno absoluto. Elegir *qué* comprar importa más que decidir *cuánto*.

2. **HRP reduce el riesgo de forma consistente pero modesta.** Volatilidad -2.4 a -2.6pp y max drawdown -0.2 a -0.9pp vs equal-weight, en las 3 variantes. Pero esta reducción no se traduce en mejora significativa de Sharpe (test de Jobson-Korkie no significativo).

3. **S1 Quality es la mejor estrategia ajustada por riesgo.** Mejor Sharpe (0.77), la única significativa vs S&P 500 al 10% (p=0.091), único Δ Sharpe positivo vs EW (+0.03), menor turnover (64% vs 107%), menor costo (0.96% vs 1.60%). Quality es más estable que Momentum (turnover de universo 19.4% vs 40.9%). Tras la corrección del NB01, esta ventaja de S1 se amplió: S2_QM bajó su Sharpe a 0.66.

4. **Value diluye la señal.** S3 QVM tiene el peor Sharpe (0.61) y el peor drawdown (-38%), consistente con el hallazgo del NB01.

5. **HRP funciona mejor en mercados "normales" y en períodos de estrés, peor en rallies concentrados.** Su mecanismo de subponderar alta volatilidad protege en crisis pero cuesta retorno en bull markets liderados por growth/tech (2020-2025).

6. **HRP es UCITS-compatible naturalmente** con 50 activos, sin necesidad de constraints explícitos.

7. **Los resultados son robustos** a N, ventana, linkage, y a correcciones en el factor model del NB01. El hecho de que la narrativa se mantenga tras corregir un bug del NB01 refuerza las conclusiones.

---

## 12. LIMITACIONES Y EXTENSIONES FUTURAS

### Limitaciones
- Un solo índice (S&P 500), un solo país (US), un solo período (2011-2025, mayormente bull market).
- Costos de transacción de 10bp son optimistas para portfolios grandes (no capturan impacto de mercado ni slippage).
- Supuesto conservador: si una acción deja de cotizar y su retorno es 0, el peso no se redistribuye hasta el próximo rebalanceo (equivale a mantener cash).
- Significancia estadística limitada: solo S1 es marginalmente significativa vs S&P 500; detectar diferencias de Sharpe de 0.10-0.15 requiere más años de datos.

### Extensiones para convertir en paper publicable
1. **Validación en otros universos:** STOXX 600 (Europa), Nikkei (Japón), Emerging Markets. Si HRP mejora el Sharpe en mercados más heterogéneos, reforzaría la hipótesis de que su valor depende de la diversidad del universo.
2. **Universo sin pre-filtrar:** HRP sobre las 500 acciones del S&P 500 completo (donde el shrinkage y la ventaja de HRP frente a EW serían mayores por covarianza ill-conditioned).
3. **Análisis formal por régimen de volatilidad:** partir la muestra en VIX bajo vs alto y testear si HRP aporta más en crisis.

---

## 13. NOTAS SOBRE INTERPRETACIÓN (evitar overselling)

- **No presentar como "estrategia que bate al mercado significativamente".** La narrativa correcta: factor investing con Quality genera alpha económicamente relevante; HRP aporta reducción de riesgo consistente pero modesta; la combinación produce mejor perfil riesgo-retorno que el benchmark, aunque la significancia estadística requeriría más años.
- **La contribución metodológica es independiente de la significancia:** separar alpha (selección) de risk management (HRP) y medir la contribución de cada uno es un aporte en sí mismo.
- **Económicamente los resultados sí son relevantes:** €100,000 → €753,000 (S1 HRP) vs €514,000 (S&P 500) en ~15 años.

---

## 14. ANEXO — CICLO DE CONSTRUCCIÓN DE CARTERA (material visual para presentación/TFM)

Ejemplo desarrollado: S2_QM, FY2017 → entrada abril 2018 (período estable, pre-COVID, datos completos).

Secuencia visual del ciclo:
1. **Universo top 50** (tabla con RIC, sector, score, ranking). En FY2017 el top está dominado por Health Care y Tech de alto momentum (ALGN, ADBE, ISRG, VRTX, NVDA como top 5 por score).
2. **Matriz de correlación:** original (alfabética) vs quasi-diagonalizada. Correlación promedio 0.310, máximo 0.826. La quasi-diagonalización agrupa activos correlacionados en bloques.
3. **Dendrograma** con clusters coloreados por sector. Con single linkage aparece el chaining effect (un cluster de 45 activos + 5 singletons como EW, RMD, MNST). Se recomienda mostrar ward para legibilidad en la presentación.
4. **Pesos HRP resultantes.** En el ejemplo: SJM 9.1% (top), sobrepondera Health Care (+10.7pp vs EW) y Consumer Staples (+7pp), subpondera Information Technology (-7.9pp) y Consumer Discretionary (-7.6pp). 17 acciones por encima de EW (suman 61.3%), 33 por debajo.
5. **Comparación trimestral T vs T+1** (abril vs julio 2018): mismo universo, nueva covarianza. Turnover de 86.3% entre trimestres; mayor reducción SJM (-9.1%), mayor aumento CBOE (+5.2%). El shrinkage cambia de 0.0363 a 0.0345.
6. **Reconstitución anual** (abril 2018 vs abril 2019): 37 de 50 acciones rotan (74% de rotación), reflejo de que Momentum rota rápido. Costo de rotación de ese rebalanceo: 0.08% del NAV, dominado por las compras (las ventas tienen peso ~0 porque las acciones salientes ya habían sido subponderadas por HRP).

Hallazgo visual relevante: en S2_QM la rotación anual del universo es muy alta (~37 de 50 acciones cambian entre FY2017 y FY2018) porque Momentum rota rápido. En S1_Quality la rotación es mucho menor (turnover de universo 19.4% vs 40.9%).

---

## 15. ARCHIVOS GENERADOS (outputs en Drive)

- `daily_prices_refinitiv_all.csv` — precios diarios (590 RICs × 4595 días)
- `nb02_v3_weights_S1/S2/S3.csv` — pesos históricos por rebalanceo (59 rebalanceos × N acciones únicas: S1 154, S2 265, S3 280)
- `nb02_v3_nav_daily.csv` — NAV diario de todas las estrategias (3,713 días × 7 series)
- `nb02_v3_turnover_S1/S2/S3.csv` — descomposición de turnover
- `nb02_v3_table.tex` — tabla de métricas en LaTeX
- `nb02_v3_performance.png`, dendrogramas, heatmaps, gráficos del anexo (300 dpi)

---

## 16. RESUMEN DE CAMBIOS RESPECTO A LA VERSIÓN ANTERIOR DEL DOCUMENTO

Tras la corrección del NB01 (que arregló entre otras cosas el look-ahead bias en el cálculo de EVOL dentro del factor Safety), los universos cambiaron y con ellos los resultados del backtest:

- **S2_QM se debilitó:** Sharpe de ~0.71 a 0.66, NAV de ~731 a 658.6. Sigue superando al S&P 500 pero con menos holgura.
- **S1_Quality se mantuvo/mejoró:** Sharpe 0.77, NAV 753.1. Se consolida como la estrategia dominante.
- **S3_QVM estable:** Sharpe 0.61, NAV 549.3.
- **Significancia estadística:** S1 sigue siendo la única marginalmente significativa vs S&P 500 (p=0.091). S2 perdió fuerza (p=0.523, antes ~0.26).
- **La narrativa central no cambió:** el alpha viene de la selección, HRP reduce riesgo de forma modesta, S1 Quality es la mejor estrategia ajustada por riesgo. Si acaso, la ventaja de S1 sobre S2 se hizo más clara.
