# Spain_nitrate_trend_analysis

# Nitrate Pollution in Spain: Structural Break & Time-Series Analysis

Statistical analysis of 30+ years of Spanish water-quality monitoring data (EEA Waterbase, 1992–2023), focused on one question: **did nitrate pollution in Spain change, when, and by how much?**

**Key finding:** Spain's national mean nitrate level shifted from about **5.1 mg/L** (1992–2016) to about **7.0 mg/L** (2017–2023). A Chow test confirms a significant structural break at **2018** (F = 5.95, p = 0.007), and an interrupted time-series model explains **52%** of year-to-year variation.

---

## Research Questions

1. Which water-quality variables are most associated with nitrate levels?
2. Is there a structural break in the long-term nitrate trend?
3. How large is the break, and is the post-break level a real change or noise?

---

## Data

| | |
|---|---|
| **Source** | [EEA Waterbase](https://www.eea.europa.eu/en/datahub) – water quality monitoring data |
| **Scope** | Spain (`countryCode = ES`), 1992–2023 |
| **Raw size** | 2.69M records → ~349k after cleaning |
| **Target** | Annual mean nitrate (mg NO₃/L) |
| **Other variables** | Ammonium, phosphate, pH, dissolved oxygen, conductivity, BOD5, temperature, chloride |

**Processing pipeline**
1. Chunked loading (200k rows at a time) to handle the full EEA file in memory
2. Physical validity filter (removed negative and implausible values)
3. Selected 9 chemistry parameters
4. Built a site × water body × year panel (56,265 rows)
5. Aggregated to a national annual series (32 years)

---

## Analysis

### 1. Exploratory analysis
- Nitrate is stable around 5 mg/L for 25 years, then rises sharply in 2018 (≈ 8.5 mg/L).
- Raw values are right-skewed → modelled on `log(1 + nitrate)`.
- Spearman correlation heatmap to identify variables moving with nitrate (rank-based, robust to skew and outliers).

### 2. Stationarity diagnostics
| Test | Null hypothesis | p-value | Reading |
|---|---|---|---|
| ADF | Non-stationary | 0.02 | Stationary |
| KPSS | Stationary | 0.049 | Non-stationary |

The tests disagree, a pattern typical of a series with a **level shift**. This was the first statistical signal of a structural break. First differencing resolves it (ADF p ≈ 0).

### 3. Structural break detection
- **Chow test at 2018:** F = 5.95, **p = 0.007** → significant break.
- **CUSUM of residuals:** no boundary breach (6.36 vs 7.69 limit). CUSUM detects gradual drift better than one-off jumps, so this is consistent with a sudden shift.

### 4. Interrupted Time Series (ITS-SARIMAX)
A regression with AR(1) errors that separates the long-term trend from the break:

| Term | Meaning | Estimate |
|---|---|---|
| `time_trend` | Underlying long-term drift | +0.34% / year (p = 0.55) |
| `step_2018` | Immediate level shift at 2018 | ≈ +70% (p = 0.14) |
| `slope_post` | Change in trend after 2018 | −5.9% / year (p = 0.49) |
| `log_DO` | Dissolved oxygen association | +0.75 |

- **In-sample R² = 0.52**
- Residuals pass Ljung-Box (lag 5 & 10), Jarque-Bera and show zero mean → well-specified model.
- **Counterfactual analysis:** break terms set to zero to estimate what nitrate would have been with no 2018 shift.

### 5. Why forecasting models fail here (supporting evidence)
ARIMA, SARIMA and an LSTM were fitted on 1992–2016 and tested on 2017–2023. All produce **negative out-of-sample R²**, because they learn the pre-2018 level and cannot anticipate a regime change. ARIMA(0,0,1) was the most accurate (RMSE 1.96 mg/L), only slightly better than a naive baseline (2.07). This supports the break finding: the post-2018 period behaves differently from the past.

---

## Key Takeaways

- Spain's nitrate levels show a **statistically significant regime change around 2018**, about 40% higher on average afterwards.
- **Dissolved oxygen** has the strongest association with nitrate in the explanatory model. This is an association, not a causal claim.
- Standard forecasting models are unreliable across structural breaks; **break-aware models** are needed to interpret such data.

---

## Limitations

- **Small series:** 32 annual national values limit statistical power. The individual ITS break terms are not significant on their own, even though the Chow test is.
- **Break date chosen from the plot:** a data-driven test (Bai-Perron / Quandt-Andrews) would give a stricter result.
- **Changing monitoring network:** the set of reporting sites varies by year, so part of the 2018 shift may reflect *which* sites were measured.
- **National aggregation** hides regional differences (e.g. agricultural vs coastal basins).

## Next Steps

- Repeat the analysis on a fixed panel of sites that report every year.
- Move to monthly, basin-level data to capture seasonality and regional effects.
- Add fertiliser use, rainfall and land-use data to explain *why* nitrate rose.

---

## Tech Stack

Python · pandas · NumPy · statsmodels · pmdarima · scipy · scikit-learn · TensorFlow/Keras · matplotlib · seaborn

## Repository

```
├── Spain_Water_ARIMA_NN.ipynb   # Full analysis notebook
├── presentation/                # Slide deck summarising results
└── README.md
```

## How to Run

1. Download the EEA Waterbase disaggregated data CSV and set `DATA_PATH` in the notebook.
2. Install dependencies:
   ```bash
   pip install pandas numpy statsmodels pmdarima scipy scikit-learn tensorflow matplotlib seaborn
   ```
3. Run `Spain_Water_ARIMA_NN.ipynb` top to bottom.
