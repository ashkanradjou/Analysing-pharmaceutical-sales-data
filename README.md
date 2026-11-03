# Pharmaceutical Sales Analysis & Forecasting

Time-series analysis and sales forecasting for pharmaceutical products sold at a single-pharmacy Point-of-Sale (POS) system between 2014 and 2019. The project covers exploratory data analysis, feature engineering for seasonality, comparison of two forecasting models (Linear Regression vs. Holt-Winters), multi-fold backtesting for robust model validation, and a 2020 sales forecast.

## Table of Contents

- [Dataset](#dataset)
- [Project Structure](#project-structure)
- [Exploratory Data Analysis](#exploratory-data-analysis)
- [Forecasting Methodology](#forecasting-methodology)
- [Model Comparison](#model-comparison)
- [Backtesting (Multi-Fold Validation)](#backtesting-multi-fold-validation)
- [2020 Sales Forecast](#2020-sales-forecast)
- [Key Takeaways](#key-takeaways)
- [Limitations](#limitations)
- [Tech Stack](#tech-stack)
- [How to Run](#how-to-run)

## Dataset

The dataset originates from the POS system of a single pharmacy and contains roughly 600,000 transactional records spanning six years (2014–2019). It includes transaction timestamps, drug brand names, and quantities sold. Sales have been aggregated into eight drug categories based on the Anatomical Therapeutic Chemical (ATC) Classification System:

| ATC Code | Category |
|---|---|
| M01AB | Anti-inflammatory/antirheumatic, non-steroid (acetic acid derivatives) |
| M01AE | Anti-inflammatory/antirheumatic, non-steroid (propionic acid derivatives) |
| N02BA | Analgesics/antipyretics - salicylic acid derivatives |
| N02BE | Analgesics/antipyretics - pyrazolones and anilides |
| N05B | Psycholeptics - anxiolytics |
| N05C | Psycholeptics - hypnotics and sedatives |
| R03 | Drugs for obstructive airway diseases |
| R06 | Antihistamines for systemic use |

Data was pre-cleaned (outlier treatment, missing-value imputation) and resampled at hourly, daily, weekly, and monthly frequencies. This project works primarily with the monthly aggregation.

## Project Structure

1. Exploratory Data Analysis - sales distribution, seasonal patterns, and correlations
2. Feature engineering - cyclical encoding of seasonality for regression
3. Forecasting - Linear Regression vs. Holt-Winters Exponential Smoothing on 2019
4. Model comparison - single train/test split
5. Backtesting - 3-fold rolling validation (2017 / 2018 / 2019) for both models
6. Final forecast - 2020 sales projection using Linear Regression

## Exploratory Data Analysis

**Overall sales volume (2014–2019).** N02BE (63,005 units) is by far the highest-volume category - roughly 50x higher than the lowest, N05C (1,250 units). N05B (18,646) and R03 (11,609) follow at a distant second and third, indicating N02BE dominates overall pharmacy sales.

**Top sellers by period.** N02BE and N05B consistently rank first and second across three sampled months (Jan 2015, Jul 2016, Sep 2017), showing stable demand independent of season. The third-place spot varies seasonally: R03 (a respiratory drug) appears in colder months (Jan, Sep) but is displaced by M01AB in the summer month (Jul), consistent with lower respiratory-illness incidence in warm weather.

**Weekday pattern.** N02BE is also the top seller on Mondays in 2017, but the ranking order matches the overall 2014–2019 ranking almost exactly - suggesting weekday has little independent effect on drug-level demand; N02BE's dominance is a stable, category-level trait rather than a day-of-week effect.

**Seasonality (heatmap).** R03 sales decrease in Q3 (summer) and rise in Q4 (winter), consistent with respiratory illness seasonality. N02BE also rises in Q4 and partially in Q1, likely tracking cold/flu-related pain and fever. This recurring yearly pattern directly motivates the cyclical (`sin`/`cos`) seasonal features used in the regression model.

**Correlation between drug categories.** The strongest pairwise correlation is N02BE–R03 (0.62), followed by M01AE–N02BE (0.62) - both moderate, likely reflecting shared winter-season demand or co-prescription for pain/inflammation. All other pairs show weaker correlation, indicating most drug categories behave largely independently of one another.

## Forecasting Methodology

Monthly sales are modeled per drug category using:

- **`time_index`** - a linear month counter capturing long-term trend
- **`month_sin`, `month_cos`** - cyclical encoding of month-of-year, capturing seasonality without artificially treating December and January as "far apart"

Two models were trained per drug and compared:

1. **Linear Regression** on the engineered features above
2. **Holt-Winters Exponential Smoothing** (additive trend + additive seasonality, 12-month period)

## Model Comparison

Trained on 2014–2018, tested on 2019, across all eight drugs:

| Metric | Result |
|---|---|
| RMSE | Linear Regression outperformed Holt-Winters on **all 8 drugs** |
| MAPE | Holt-Winters was better on 2 of 8 drugs (M01AB, N05B) |

RMSE penalizes large errors more heavily, while MAPE measures relative error — the split result shows model ranking can depend on the evaluation metric chosen, which is why a single-split comparison alone is not sufficient evidence.

## Backtesting (Multi-Fold Validation)

To avoid drawing conclusions from a single train/test split, both models were re-evaluated across three independent folds:

| Fold | Train | Test |
|---|---|---|
| 1 | 2014–2016 | 2017 |
| 2 | 2014–2017 | 2018 |
| 3 | 2014–2018 | 2019 |

**Result: Linear Regression had a lower mean RMSE than Holt-Winters on all 8 drugs, across all 3 folds** - a considerably stronger result than the single-split comparison, since it rules out the possibility that 2019 was simply a favorable year for one model.

| Medicine_ATC | Mean RMSE (LR) | Mean RMSE (HW) | Std RMSE (LR) | Std RMSE (HW) |
|---|---|---|---|---|
| N05C | 9.36 | 11.18 | 1.99 | 3.25 |
| N02BA | 27.28 | 31.71 | 9.81 | 14.68 |
| M01AE | 29.95 | 38.43 | 10.92 | 18.97 |
| R06 | 30.96 | 32.39 | 2.14 | 9.22 |
| M01AB | 44.50 | 45.13 | 10.36 | 26.41 |
| R03 | 74.35 | 104.54 | 17.29 | 69.63 |
| N05B | 91.89 | 119.80 | 30.36 | 18.04 |
| N02BE | 321.07 | 452.94 | 234.60 | 193.93 |

Two additional findings from backtesting:

- **More training data did not guarantee better accuracy.** For several drugs (M01AE, M01AB, N02BA, N02BE, R03), RMSE did not improve monotonically across folds — 2019 was a relatively harder year to predict for most drugs regardless of training set size, suggesting year-specific volatility matters more than data volume alone.
- **Linear Regression was also more stable**, not just more accurate: it had lower RMSE standard deviation than Holt-Winters on 5 of 8 drugs, meaning its performance varied less across different test years.
- **N02BE remained the hardest drug to forecast for both models** (highest Mean RMSE and, for Linear Regression, the highest standard deviation), consistent with it being the largest-scale, highest-variance category in the dataset.

## 2020 Sales Forecast

Since the dataset does not include 2020 data, a pure Linear Regression was refit on the full 2014–2019 range and extrapolated forward. Extracting the `time_index` coefficient per drug reveals the projected trend direction:

| Medicine_ATC | Trend Coefficient | Direction |
|---|---|---|
| R03 | +1.763 | Increasing |
| R06 | +0.510 | Increasing |
| M01AB | +0.268 | Increasing |
| N05C | +0.001 | Flat |
| M01AE | -0.046 | Flat/slightly decreasing |
| N02BE | -0.232 | Decreasing |
| N02BA | -0.928 | Decreasing |
| N05B | -1.376 | Decreasing |

Notably, N02BE - the highest-volume drug overall is forecast to slightly decline, while N05B, the second-highest-volume drug, shows the steepest projected decline of all eight categories.

**Important caveat:** this forecast is a linear extrapolation of 2014–2019 trends only. It does not and cannot account for the COVID-19 pandemic, which began in early 2020 and materially disrupted demand for respiratory drugs (R03) and antihistamines (R06) in ways no model trained purely on pre-2020 data could anticipate.

## Key Takeaways

- N02BE is the dominant drug category by volume, but also the hardest to forecast accurately for both models tested.
- Seasonality is real and predictable for respiratory-related drugs (R03) and, to a lesser extent, general analgesics (N02BE) both peak in winter.
- A simple Linear Regression with hand-engineered cyclical features outperformed Holt-Winters consistently across drugs and time periods, likely because Holt-Winters needs more data than was available (max. 6 years / 72 months) to reliably estimate trend and seasonal components.
- Single train/test evaluations can be misleading; backtesting across multiple folds materially strengthened (and in this case, confirmed) the model selection conclusion.

## Limitations

- The dataset does not contain 2020 data, so the final forecast could not be validated against ground truth.
- Neither model accounts for exogenous shocks (e.g., COVID-19, supply shortages, regulatory changes).
- Both models assume each drug's sales are independent of the others, despite the moderate correlation observed between N02BE and R03.
- Linear Regression assumes a constant linear trend, which may not hold over longer forecast horizons.
- Results reflect a single pharmacy's POS data and may not generalize to other pharmacies or regions.

## Tech Stack

- Python, pandas, NumPy
- scikit-learn (Linear Regression)
- statsmodels (Holt-Winters Exponential Smoothing)
- matplotlib, seaborn (visualization)

## How to Run

```bash
pip install pandas numpy scikit-learn statsmodels matplotlib seaborn
jupyter notebook pharma_sales_analysis.ipynb
```

Place `salesdaily.csv` in the project root before running.
