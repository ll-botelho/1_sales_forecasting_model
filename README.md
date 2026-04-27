# Sales Forecasting with Ensemble Models

Multi-series monthly sales projection using Gradient Boosting, Random Forest, and Prophet — with Optuna-tuned hyperparameters and scipy-optimized ensemble weights.

---

## Overview

This project builds a complete forecasting pipeline for a portfolio of anonymized product-market combinations. The goal is to generate reliable 12-month forward projections at monthly granularity, combining the strengths of tree-based ML models with a time-series-native approach (Prophet) through a data-driven ensemble.

The notebook covers the full workflow: raw data ingestion, feature engineering, temporal cross-validation, hyperparameter optimization, ensemble construction, error analysis, and structured Excel export.

---

## Project Structure

```
├── market_sales_ensembles_projection.ipynb   # Main notebook (Google Colab)
└── README.md
```

> The notebook reads data from and writes outputs to Google Drive. Path references follow a Colab + Drive setup and should be adjusted for other environments.

---

## Dataset

- **Granularity:** Monthly
- **Period:** January 2021 – December 2024
- **Series:** ~12 anonymized product-market combinations (`Prod_anom` × `Merc_anom`)
- **Target variable:** `Unidades` (sales volume)
- All identifiers have been anonymized for this publication.

---

## Methodology

### 1. Feature Engineering

| Group | Features |
|---|---|
| Lagged values | `lag_1`, `lag_3`, `lag_6`, `lag_12` |
| Rolling averages | `rolling_mean_3`, `rolling_mean_6`, `rolling_mean_12` |
| Calendar | Month, year |
| Trend | Cumulative time index per series |
| Target encoding | Mean sales per product and per market (computed on training data only) |

### 2. Validation Strategy

- **Train set:** January 2021 – December 2023
- **Test set:** January – December 2024
- **Cross-validation:** `TimeSeriesSplit` with 4 folds, used during hyperparameter tuning

No random shuffling is applied at any stage to preserve temporal ordering.

### 3. Models

| Model | Role |
|---|---|
| Gradient Boosting (Optuna-tuned) | Primary ensemble component |
| Random Forest (Optuna-tuned) | Primary ensemble component |
| Prophet (per-series) | Primary ensemble component |
| XGBoost (default) | Benchmark |
| LightGBM (default) | Benchmark |
| Lag-1 Naive | Baseline |

**Why GB and RF over XGBoost/LightGBM?** With ~12 series, simpler models with fewer parameters generalize more reliably. Empirically, GB and RF outperformed the boosting alternatives on the hold-out test set.

**Why Prophet per series?** Prophet handles short time series with seasonal patterns without requiring manual feature engineering. Training one model per series lets it adapt individually to each product-market's dynamics.

### 4. Ensemble

The three primary models are combined with a weighted average. Weights are optimized via `scipy.optimize.minimize` (Nelder-Mead) to minimize SMAPE on the 2024 test set, starting from equal weights (1/3 each).

### 5. Projection Strategy

The 12-month forward forecast (Jan–Dec 2025) uses an **iterative multi-step** approach:

- In each future month, lag and rolling average features are recomputed using the combination of real historical data and previously generated forecasts.
- This replicates production behavior, where future actuals are never available at inference time.

---

## Evaluation Metric

**SMAPE** (Symmetric Mean Absolute Percentage Error) is the primary metric throughout:

$$\text{SMAPE} = \frac{1}{n} \sum_{t=1}^{n} \frac{|y_t - \hat{y}_t|}{(|y_t| + |\hat{y}_t|) / 2} \times 100$$

SMAPE is preferred over MAPE here because the dataset contains near-zero values that would cause MAPE to become unstable or undefined.

---

## Output

The final Excel workbook (`Projecoes_2025.xlsx`) contains three sheets:

| Sheet | Content |
|---|---|
| Projeção por Produto | Pivot table: products × months, with annual total |
| Detalhe Produto-Mercado | Granular view: one row per series per month |
| Métricas do Modelo | Full model comparison from the 2024 evaluation |

---

## Requirements

```
pandas
numpy
scikit-learn
xgboost
lightgbm
prophet
optuna
scipy
matplotlib
seaborn
openpyxl
```

Install all dependencies:

```bash
pip install pandas numpy scikit-learn xgboost lightgbm prophet optuna scipy matplotlib seaborn openpyxl
```

> The notebook includes `!pip install` cells for the packages not available by default in Google Colab.

---

## How to Run

1. Open the notebook in [Google Colab](https://colab.research.google.com/)
2. Mount your Google Drive when prompted
3. Update the `path` variable in Section 1 to point to your data file
4. Run all cells sequentially

The projection output will be saved automatically to the path defined in Section 15.

---

## Key Design Decisions

- **SMAPE over MAPE** — avoids division-by-zero instability on near-zero sales months
- **Optuna over grid search** — more efficient exploration of large hyperparameter spaces with fewer trials
- **Iterative multi-step projection** — correctly propagates uncertainty through the forecast horizon without leaking future values into lag features
- **Per-series Prophet** — each series gets its own trend and seasonality model rather than a pooled approximation
- **Scipy weight optimization** — ensemble weights are data-driven, not manually assigned
