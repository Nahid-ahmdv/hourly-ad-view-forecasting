# Hourly Ad-View Forecasting

I built this project around a specific question: **when forecasting the next
48 hours of ad views, how much additional modeling complexity was worth it?**

Using hourly advertising data from one Zeptolab mobile game, I compared
32 forecasting procedures against a benchmark that repeats the last observed
day. I wanted to understand which choices improved the forecasts, why plausible
alternatives sometimes failed to help, and what the accuracy ranking left open
about uncertainty.

Every procedure had to issue all 48 predictions together, using only information
available at the forecast origin. That requirement shaped the features, tuning,
evaluation, and final forecast.

**Full analysis and code:** [Executed forecasting notebook](hourly_ads_forecasting_final.ipynb)

**Medium article:** Link coming soon.

The saved analysis includes the explanations, result tables, and 13 figures.
You can read the findings without running the models.

## Main findings

I selected **SARIMA(1,0,0)(0,1,1)[24] with weekly Fourier regressors of period
168** because it had the lowest mean development MASE. At each origin, the
procedure chooses the harmonic count **K from {2, 3, 4} by AIC**, using a common
likelihood sample within each search. The label `SARIMA + weekly Fourier`
refers to this complete refit-and-select rule.

| Procedure | Mean development MASE | Mean historical validation MASE |
|---|---:|---:|
| **SARIMA + weekly Fourier** | **0.954947** | **1.016** |
| Daily/weekly harmonic regression with ARIMA errors | 1.047932 | 1.230 |
| Equal-weight combination | 1.115981 | 1.106 |
| Seasonal naive, 24 hours | 1.393330 | 1.319 |

MASE is Mean Absolute Scaled Error; lower is better. Validation values retain
the saved table's three-decimal precision. The harmonic-regression alternative
replaces seasonal differencing with Fourier structure.

The selected procedure beat seasonal naive on **25 of 29 development paths**
and **four of six historical validation paths**. I had previously examined the
validation outcomes, so these are exploratory comparisons, rather than a fresh
confirmatory holdout test. They do not establish population superiority.

Weekly structure helped the leading specification. Transformations, more flexible
learners, and combinations did not consistently improve on it. However, the
uncertainty results changed how I interpreted that favorable ranking: the
selected procedure's nominal **80% pointwise interval covered only 70.83%
(204/288)** of validation hours, with several actual peaks above the upper limit.

Separate simultaneous bands contained all six paths, but whole-path coverage
is a different target and does not repair pointwise undercoverage. I also
reported interval scores and CRPS for a separate empirical signed-residual
predictive distribution.

## Dataset

The data contain **2,085 complete, unique hourly observations** from
**August 3, 2017, at 00:00 through October 28, 2017, at 20:00**. Each observation
records a positive count of ads watched. No timezone is supplied.

[data/ads_hour.csv](data/ads_hour.csv) contains two columns:

- `Date`: hourly timestamp, parsed with `%m/%d/%y %H:%M`.
- `ads`: number of ads watched.

The dataset and initial lag-feature/PCA exercise came through mlcourse.ai.
Source credits and the existing redistribution qualification appear below.

## Approach and evaluation

I first inspected changes in level, daily and weekly patterns, and unusual
observations. A retrospective interpolation exercise explored sensitivity to
sharp drops, but **all primary forecasting comparisons used raw observations**.

I compared classical smoothing, SARIMA, Fourier representations, and Box-Cox
transformations with feature-based alternatives: linear regression, Ridge,
Lasso, PCA, partial least squares, and XGBoost. Prophet and forecast combinations
provided further comparisons.

The feature design used lags 48 through 119 and five calendar features.
Every lag was available throughout the issued horizon. For tuned feature-based
procedures, chronological inner validation used 48-hour blocks, with learned
preprocessing inside the tuning pipeline.

After **336 initial training hours**, I evaluated forecasts at origins 48 hours
apart:

| Stage | Forecast paths | Target period | Target hours |
|---|---:|---|---:|
| Development | 29 | August 17 to October 13, 2017 | 1,392 |
| Historical validation | 6 | October 14 to October 25, 2017 | 288 |

Selection used the **mean per-origin MASE**. Each origin's 48-hour MAE was scaled
by the average absolute 24-hour difference in its training history. That
historical denominator means the evaluated seasonal-naive benchmark need not
score exactly one. MAE in original count units and RMSE supplied additional
views of error.

Models refitted as observations became available. Within the implemented run,
selection and combination weights remained development-derived; earlier
validation observations could enter later refits.

I finished by refitting the selected rule on all observed history, including
the **69-hour tail excluded from evaluation targets**. The final fit selected
**K = 4**, corresponding to eight weekly Fourier regressors. The final forecast covers
**October 28, 2017, at 21:00 through October 30 at 20:00**. Future actuals are
unavailable for that path, so it is an application of the selected procedure
rather than another accuracy result.

## Repository structure

```text
hourly-ad-view-forecasting/
├── README.md
├── hourly_ads_forecasting_final.ipynb
├── requirements.txt
├── .gitignore
└── data/
    └── ads_hour.csv
```

The analysis reads the CSV relative to this project root. It has no dependencies
on another project directory, an external source builder, or historical
execution folders. Article drafts and editorial revision records are kept
locally and excluded from Git.

## Running the project

The recorded analysis environment used **Python 3.10.18 on macOS arm64**.
From the project root, with that Python version available:

```sh
python3.10 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m ipykernel install --user --name hourly-ad-view-forecasting --display-name "Hourly ad-view forecasting"
```

Open the analysis in an existing Jupyter frontend, select the registered kernel,
and keep its working directory at this project root. For a fresh execution,
use a separate copy, restart the kernel, and run all cells in order. Full
execution fits many models and can take an hour or longer.

The package versions used for the analysis are listed in
[requirements.txt](requirements.txt). I have not tested a clean installation.
On a fresh run, check the outputs before reusing the written summaries, as
optimization and simulation can change some results.

## Limitations and next steps

- **Scope and research history:** one short series, adaptive development, and
  prior validation exposure limit what these comparisons establish.
- **Dependence and inference:** nonoverlapping target paths are not independent
  replications. HAC comparisons, block bootstrap, multiplicity adjustments, and
  the Model Confidence Set remain exploratory.
- **Numerical sensitivity:** a small matched Box-Cox order-search contrast
  changed sign across executions. Its cause is unresolved, and complete fit-level
  state was not exported.
- **Uncertainty transfer:** calibration and validation paths began at midnight,
  whereas the final forecast begins at 21:00. Transfer of horizon-specific
  calibration to that schedule was not evaluated. No operational coverage
  guarantee is established.

My most useful next step would be to evaluate a fixed procedure on a previously
unexamined period at the intended forecast schedule. I would examine directional
misses and uncertainty calibration alongside point accuracy before deciding
whether another model family adds enough value.

## Sources and attribution

The Zeptolab data and initial lag-feature/PCA setup came through
[Yury Kashnitsky's mlcourse.ai Bonus Assignment 9](https://mlcourse.ai/book/topic09/bonus_assignment09.html).
Related educational starting points include Dmitriy Sergeyev's
[Time series analysis in Python](https://mlcourse.ai/book/topic09/topic9_part1_time_series_python.html)
and Egor Polusmak's
[Forecasting with Prophet](https://mlcourse.ai/book/topic09/topic9_part2_facebook_prophet.html).
I extended those starting points with the comparative evaluation, diagnostics,
and uncertainty assessment documented in this project. The complete analysis
includes the methodological references.

The two public course articles identify their license as
[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/).
That license does not automatically establish permission to redistribute this
specific CSV or the bonus-assignment pack, which has separate restrictions on
public sharing. **Public redistribution permission for this CSV has not been
independently established.** Confirm the applicable permission before publishing
the CSV publicly. This project grants no new rights over third-party data or
educational material.

## Author

Nahid Ahmadvand

[LinkedIn](https://www.linkedin.com/in/nahid-ahmadvand-00382b365/) |
[Website](https://www.nahidahmadvand.website/)

Questions, corrections, and ideas for extending the analysis are welcome.
