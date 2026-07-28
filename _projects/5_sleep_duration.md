---
layout: page
title: "The Data Ceiling in Predicting Sleep Duration"
subtitle: Predictive Modeling in Finance & Insurance, Columbia University
description: Explored six different modeling approaches, but they all pointed to the same finding: demographic and economic survey data explain only about 10% of sleep duration.
category: projects
importance: 2
tags: [OLS, Ridge/Lasso/Elastic Net, PCA, Random Forest, Cross-Validation, R]
---

**Class:** Predictive Modeling (2026)

Predictive Modeling is a graduate statistics course covering the full modern regression toolkit: OLS and model selection, regularization (Ridge, Lasso, Elastic Net), GLMs, dimension reduction (PCA), tree-based methods, cross-validation. The final project was an open-ended group product (four members) applying that toolkit on a real dataset.

Our team investigated the demographic, economic, and behavioral determinants of sleep duration using a cross-sectional survey of 706 American workers found on Kaggle called [Sleep Patterns](https://www.kaggle.com/datasets/kapturovalexander/sleep-patterns/data). The outcome variable is `sleep`: minutes of nighttime sleep per week.

The research question had two parts, one inferential and one predictive:

1. **Inference:** What factors significantly influence sleep duration?
2. **Prediction:** Which statistical model provides the most reliable prediction of sleep?

**Result Summary:** Every model we tried converged on roughly the same low explanatory power (R² ≈ 0.10). Thus, we concluded that there is a limitation to predicting sleep only from this demographic data, and we recommended trying other explanators in a longitudinal study for better results.

## Step 1: Data Quality & Exploration

Average sleep in the sample was 3,266 minutes per week (about 7.8 hours per night), with average work time of 2,123 minutes per week.

Before modeling, we made several variable exclusion decisions:

- **Outcome leakage variables dropped:** Variables like `slpnaps` (sleep including naps) and `leis1`/`leis2`/`leis3` (leisure measures) are mathematical functions of `sleep` itself. Including them would artificially inflate R².
- **Redundant work variables dropped:** `worknrm` and `workscnd` are components of `totwrk` (total minutes worked per week), so we dropped those to prevent multicollinearity.
- **Derived experience variable dropped:** `exper = age - educ - 6` exactly, so we kept `age` and `educ` (plus `agesq` for a nonlinear age effect) and dropped `exper`.
- **Missing-wage rows dropped:** 174 rows had missing hourly wage, and every one of them not in the labor force (`inlf = 0`). We dropped these observations and scoped the analysis to only labor force participants, leaving 532 observations.
- **Log wage over raw wage:** `hrwage` was heavily right-skewed, so we used `lhrwage` (log hourly wage) for better behavior.

Other variables included continuous ones, such as `totwrk` (minutes worked per week) and `educ` (years of schooling), and binary ones, such as `male`, `marr` (married), `selfe` (self-employed), and `smsa` (lives in metro area).

Residual diagnostics confirmed OLS was a reasonable starting framework: all Variance Inflation Factors (VIF) ranged between 1.04 and 1.48 (no multicollinearity concern), and the QQ Plot of residuals adhered closely to the diagonal (normality holds).

## Step 2: Inference with OLS

The full Ordinary Least Squares (OLS) model regressing `sleep` on all predictors produced R² ≈ 0.13 (adjusted R² ≈ 0.11). The overall F-test was significant (so the model explains meaningfully more variance than an intercept-only baseline one), but at the individual level only one predictor cleared the 5% threshold: `totwrk`, with a coefficient of about -0.155 (p < 0.001), meaning each additional minute worked per week is associated with roughly 0.155 fewer minutes of sleep, or ~9 minutes less sleep every hour worked.

From there, we tested four reduced models and four interaction models answering if the work-sleep tradeoff differs by gender, marriage, young children, or region.

All model candidates were compared on R², adjusted R², AIC, BIC, and Mallows' Cp. **Interaction Model 1** (`sleep ~ totwrk × male + south + educ`) won on AIC and Cp while staying simple and interpretable; the extreme parsimonious case (`totwrk` alone) won on BIC.

We also tried polynomial extensions (`totwrk²`, `age²`), but they only improved adjusted R² modestly, suggesting the work-sleep relationship is essentially linear.

## Step 3: Prediction Comparison

With the inferential winner chosen, we turned to a different question: which model actually predicts best on data it hasn't seen? We compared eight models via 10-fold cross-validation on the full 532-observation dataset: OLS Interaction 1, Ridge, Lasso, and Elastic Net (each with and without the interaction term), and Principal Component Analysis (PCA) Regression (3 components capturing 88.4% of continuous-predictor variance, combined with the binary variables). The table below is a snippet of the results:

|         Model         |    CV MSE     |  CV RMSE   |   CV R²    |
| :-------------------: | :-----------: | :--------: | :--------: |
| **OLS Interaction 1** | **165,999.9** | **407.43** | **0.1032** |
|         Ridge         |   167,631.9   |   409.43   |   0.0946   |
|  Ridge + Interaction  |   167,839.3   |   409.68   |   0.0935   |
|         Lasso         |   168,294.7   |   410.24   |   0.0910   |
|      Elastic Net      |   172,670.9   |   415.54   |   0.0675   |
|          PCA          |   247,905.4   |   497.90   |  -0.3393   |

Note that RMSE reports in the original units of sleep minutes per week.

The simple 5-parameter OLS beat everything. Three observations:

- **Regularization couldn't do much:** Ridge and Lasso are most valuable with many predictors of uncertain relevance and high overfitting risk. Here, OLS variable selection had already found the only real signal, so the added work didn't achieve much. Lasso zeroed the interaction term immediately, confirming it was noise for prediction purposes.
- **PCA was worse:** The dataset has one dominant predictor (`totwrk`) and many weak, mostly uncorrelated ones. Undergoing PCA actually diluted `totwrk` across components, and its negative CV R² means it predicted _worse than just guessing the mean_ on held-out folds.
- **Every reasonable model landed near R² ≈ 0.10:** This was a strong hint that the limitation was in the data itself.

## Step 4: Verification with Random Forest

Could the low R² be an artifact of linear assumptions? To test that, we fit a Random Forest (RF) challenger (500 trees, `mtry = 4` tuned by out-of-bag error) evaluated on the same 10-fold CV structure.

|       Model       |  CV MSE   | CV R²  |
| :---------------: | :-------: | :----: |
| OLS Interaction 1 | 165,999.9 | 0.1032 |
|   Random Forest   | 179,132.1 | 0.0320 |

RF also did worse. Its flexibility added unnecessary variance, which aligns with expectations when the data has one dominant, essentially linear predictor.

Two cross-checks reinforced the picture: both RF's importance ranking (% increase in MSE) and OLS's absolute t-statistics independently rank `totwrk` first by a wide margin, and RF's marginal-effect curve for work hours wiggles around the OLS line without any systematic nonlinear shape.

{% comment %}
TODO: insert marginal-effect comparison plot (OLS vs. Random Forest, effect of work hours on sleep) here.
Upload the image to assets/img/ then use:
{% include figure.liquid path="assets/img/FILENAME.png" caption="Marginal effect of work hours on sleep: OLS vs. Random Forest" %}
{% endcomment %}

## Conclusion: A Data Ceiling

Across six methodologically distinct approaches (OLS, Ridge, Lasso, Elastic Net, PCA, RF), the answers to our research problems were consistent: total work time is the dominant and only unambiguous predictor of sleep duration, and the best predictive model by 10-fold CV is the simplest one (OLS).

The demographic and economic variables in this survey explain roughly 10% of the variance in how long people sleep. It seems that these variables simply do not capture differences in sleep hours, reinforced by all models, linear and nonlinear, simple and complex, hitting a similar ceiling. Thus, we verified that the limitation lies in the data rather than modeling choice.

In my presentation conclusion, I recommended that further study may be done to tackle the other ~90% through variables the dataset didn't address, such as health history, stress, commute, and personal habits over a longitudinal period.

Further, I pointed out other data limitations:

- The data is cross-sectional, so it's hard to establish causality in findings without tracking individuals over time.
- The data is from the 1970s, and work norms and wages have changed substantially since then.
- Since we restricted the (very small) sample to labor force participants, the coefficients may not generalize to non-labor force participants.

I wanted to further confirm our conclusions, so I researched papers that address this research topic, some of which are included here for further reading on this topic as well as research methodology:

- Biddle & Hamermesh (1990), "Sleep and the Allocation of Time," _Journal of Political Economy_, Vol. 98, No. 5. Found in [The University of Chicago Press Journals](https://www.journals.uchicago.edu/doi/abs/10.1086/261713). This is actually where that Kaggle dataset came from, and there are similar results.
- Grandner et al. (2015), "Social and Behavioral Determinants of Perceived Insufficient Sleep," _Frontiers in Human Neuroscience_. From [PMC PubMed Central](https://pmc.ncbi.nlm.nih.gov/articles/PMC4456880/). Although they had much richer survey results in sample size and predictor set, explained variance for sleep duration remains modest.

At the conclusion of the project, we compiled all our findings into a presentation, and I also made the project report.

## Skills & Tools

**Tools:** R, RMarkdown.

**Concepts:** OLS, Model Selection (AIC/BIC/Mallows' Cp), Residual Diagnostics (VIF, QQ), Ridge/Lasso/Elastic Net, PCA, Random Forest, K-fold Cross Validation.
