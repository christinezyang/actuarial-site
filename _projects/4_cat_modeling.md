---
layout: page
title: "Catastrophe Modeling & Portfolio Allocation"
subtitle: Risk Management in P&C Insurance, Columbia University
description: Modeled catastrophe risk and recommended significant portfolio growth for a fictitious P&C insurer. Selected as winning presenting team.
category: projects
importance: 1
tags: [Cat Modeling, VaR/TVaR, Capital Allocation, RAROC, Excel]
---

**Class:** Risk Management in P&C Insurance (2025)

Risk Management in Property & Casualty Insurance is a graduate elective that augments actuarial and insurance concepts with enterprise risk ones. The course was structured in three parts:

1. Foundations of P&C insurance (underwriting, claims, premiums, regulation)
2. Enterprise risk management (ERM) frameworks (qualitative governance structures, quantitative measurement tools)
3. Case study that applies theoretical knowledge

The capstone case study was presented to an industry guest who evaluated everyone's work as a professional deliverable. My team (of two) was selected as the winning group. This post is a walkthrough of what I built and how I approached it.

## The Problem

A catastrophe model is a large-scale stochastic simulation. It inputs data on hazard intensity, exposed assets, and vulnerability functions to produce a distribution of potential insured losses from events like hurricanes, earthquakes, and severe windstorms.

We were given 10,000 simulation years of catastrophe model output: losses from catastrophes across peril type (windstorm, earthquake, severe convection storm) and state (California, Texas, Louisiana, Florida, ...). We can call these state-perils (e.g. "Texas Windstorm"), and each count as an _exposure_.

|     | YearKey | Gross_Loss | TX Windstorm | TX Earthquake | ... |
| :-: | :-----: | :--------: | :----------: | :-----------: | :-: |
|  1  |    1    |    `$`     |      0       |       0       | ... |
|  2  |    1    |    `$`     |      0       |      `$`      | ... |
|  3  |    1    |    `$`     |      0       |       0       | ... |
| ... |   ...   |    `$`     |     ...      |      ...      | ... |
| 10  |    2    |    `$`     |      0       |      `$`      | ... |
| 11  |    2    |    `$`     |      0       |       0       | ... |
| ... |   ...   |    `$`     |     ...      |      ...      | ... |

<div class="caption">Recreated Sample of Cat Model Output</div>

The risk analyst's job is to extract business-relevant patterns to inform a decision.

The fictitious client is a P&C insurer who has underwritten catastrophe exposure in these state-perils. Our job was to:

- Analyze the portfolio's risk profile across the seven most significant state-peril exposures.
- Recommend how to deploy `$500 million` in excess capital to grow the portfolio while maximizing risk-adjusted return.

## Analytics Foundation

### Step 1: Data Quality

The raw data arrived as a flat CSV with 10,000 rows. We pivoted it into a matrix with each column representing a distinct state-peril combination and calculated a _Portfolio Total_ column as a sum across all exposures.

The project scope focused on seven specific state-peril combinations that made up the majority of potential portfolio loss:

- Florida Windstorm
- California Earthquake
- Texas Windstorm
- Louisiana Windstorm
- Washington Earthquake
- Hawaii Windstorm
- Hawaii Earthquake

| Year | FL WS | CA EQ | TX WS | ... |
| :--: | :---: | :---: | :---: | :-: |
|  1   |  `$`  |  `$`  |  `$`  | ... |
|  2   |  `$`  |  `$`  |  `$`  | ... |
| ...  |  `$`  |  `$`  |  `$`  | ... |

<div class="caption">Recreated Sample of Cleaned Data</div>

### Step 2: Probabilistic Risk Metrics

For each exposure (i.e. state-peril), we calculated a full suite of probabilistic metrics:

- Mean Loss

$$ \mu = \frac{1}{n} \Sigma_i \text{Loss}_i $$

- Standard Deviation

$$ \sigma = \sqrt{ \frac{1}{n} \Sigma_i (\text{Loss}_i - \mu)^2 } $$

- _Probable Maximum Loss_ (PML) at return periods of 10, 20, 50, 100, and 250 years. A 100-year PML is the same as _Value at Risk_ (VaR) at the 99th percentile, or the loss level exceeded 1% of the time, a "1-in-100-year loss." VaR at percentile $p$ is defined such that:

$$ \mathbb{P}(\text{Loss} \leq \text{VaR}_p) = p $$

- _Tail Value at Risk_ (TVaR) at the 99th percentile (average loss in the worst 1% of outcomes)

$$ \mathbb{E}[\text{Loss} \ | \ \text{Loss} > \text{VaR}_p] $$

- Coefficient of Variation

$$ \sigma / \mu $$

- Downside and Upside separation: _Downside_ (D) is the average amount by which losses exceed the mean. _P(D)_ is the share of simulation years landing below the mean. Similar concept for _Upside_ (U) and _P(U)_. The _D/U_ ratio combines the two: a ratio much greater than 1 means the bad years are far more extreme than the good years are mild.

Florida Windstorm dominated both Mean Loss and PML metrics, which is not surprising given its hurricane history and density of insured coastal assets. California Earthquake ranked second.

{% include figure.liquid path="assets/img/cat-modeling-proj_state-perils-pml.png" alt="Bar chart of 100-year and 250-year PML for the seven state-perils" caption="100-year and 250-year PML by state-peril" %}

### Step 3: Reinsurance Modeling

We modeled an Excess-of-Loss (XOL) reinsurance structure on the portfolio, which caps the insurer's net loss above a specified threshold — the insurer's retention (the attachment point at which reinsurance takes over). Reinsurance can reshape the tail by taking over the highest losses (subject to limits and other contractual agreements). XOL reinsurance specifically won't reduce everyday expected costs but does protect against large ones.

In this case, reinsurance reduced portfolio mean loss by 18.9% (`$278.7M` → `$226.0M`) and 99% TVaR by 30% (`$5.27B` → `$3.69B`).

This reinsurance exercise was its own separate question, so the following capital allocation analysis works from the portfolio's gross loss distribution.

## Capital Allocation

### Step 4: Three Methods of Capital Allocation

We compared three methods to fairly allocate risk capital across business units.

#### Proportional Allocation

Each exposure gets capital proportional to its share of total VaR. For example, if Florida Windstorm represents X% of total portfolio VaR, it gets X% of total capital too. This method is additive (pieces sum to the whole) and easy to explain. However, it ignores diversification since it charges each exposure based on its standalone size, not on how much that exposure actually contributes to the portfolio's tail risk. The effect is that it may ignore that two individually dangerous risks may offset each other at a portfolio level.

#### Incremental Allocation

Calculate, for each exposure, its _increment_, which is how much the total portfolio risk (VaR) changes if you remove that exposure. This increment becomes the exposure's capital allocation (in % of the total). This approach captures some diversification because each increment reflects what the exposure adds to the portfolio. However, the results aren't additive: the calculated increments don't sum to the portfolio total. This result makes communication difficult and feels unintuitive.

#### Co-TVaR Allocation (Industry Standard)

Consider the portfolio only in a tail event, such as the worst 1% outcomes of all 10,000 simulation years. How much does each exposure contribute in this case? To calculate, use the expected loss conditioned on the portfolio being in its worst 1% of outcomes.

For those whose core concern is solvency under stress, this is the preferred capital allocation method despite its higher computational demand. Co-TVaR is also additive and captures diversification.

|    Method    | Complexity | Diversification Recognition | Additivity |
| :----------: | :--------: | :-------------------------: | :--------: |
| Proportional |    Low     |            None             |    Yes     |
| Incremental  |   Medium   |           Partial           |     No     |
|   Co-TVaR    |    High    |            Full             |    Yes     |

<div class="caption">Comparison Table of Capital Allocation Methods</div>

### Step 5: Portfolio Growth Recommendation

We were given `$500M` in excess capital to deploy, with a target total capital of `$4,606.5M`. The only constraints were that each exposure could grow up to 100% and could not be reduced.

The objective was to maximize portfolio _Return on Risk-Adjusted Capital_ (RAROC), a standard measure of how much profit the portfolio generates per unit of risk capital consumed.

$$ RAROC = \text{Profit Load} / \text{Allocated Risk Capital} $$

At the portfolio level, pre-growth: `$388.1M` / `$4,106.5M` = 9.45%.

I used the Co-TVaR capital allocation method for the basis of my growth recommendation. Using this method, I calculated individual RAROCs for each state-peril. Exposures with high RAROC and (relatively) low Co-TVaR capital consumption are the most efficient uses of incremental capital. I scaled those aggressively. Exposures with low RAROC, especially if they already consume a large share of capital, should stay more flat. The specific numbers for growth factors were decided by optimizing portfolio RAROC.

|  State/Peril  | RAROC (99% Co-TVaR) | Capital Consumed |
| :-----------: | :-----------------: | :--------------: |
|   Hawaii EQ   |        53.0%        |     `$1.4M`      |
|   Hawaii WS   |        48.6%        |     `$11.5M`     |
| Washington EQ |        33.9%        |     `$26.8M`     |
| Louisiana WS  |        42.1%        |     `$43.2M`     |
|   Texas WS    |        62.5%        |     `$56.3M`     |
| California EQ |        23.3%        |    `$244.6M`     |
|  Florida WS   |        7.0%         |   `$3,722.6M`    |

{% include figure.liquid path="assets/img/cat-modeling-proj_raroc-tvar.png" alt="Bar chart of RAROC by state-peril at four Co-TVaR thresholds: 90%, 96%, 98%, and 99%" caption="RAROC by state-peril across Co-TVaR thresholds (90%, 96%, 98%, 99%)" %}

Notice that Florida Windstorm was held flat. It already dominated capital consumption, and its return doesn't justify further portfolio concentration. By contrast, Texas Windstorm offered strong return with less capital tied up, so I targeted this exposure for aggressive scaling.

The recommended growth factors:

|  State/Peril  | Growth Factor |
| :-----------: | :-----------: |
|   Hawaii EQ   |     2.00×     |
|   Hawaii WS   |     2.00×     |
| Washington EQ |     2.00×     |
| Louisiana WS  |     1.77×     |
|   Texas WS    |     2.00×     |
| California EQ |     1.25×     |
|  Florida WS   |     1.00×     |

#### Result

- Portfolio profit increased from `$388.1M` to `$467.0M` (+20.3%).
- Portfolio RAROC improved from 9.45% to 10.15% (+70bps).
- Total risk capital was `$4,602.1M` (`$4.4M` under the client's target constraint).

### Step 6: Presentation

Finally, I compiled all processes and results and created a final presentation deck that aligned with course expectations. All groups presented in front of the professor (formerly Chief Risk Officer @ RenaissanceRe) and an invited guest, an industry professional (Actuarial Consultant @ Ernst & Young), who were acting as clients. My group was chosen as the winning presentation.

## Skills & Tools

**Tools:** Excel, PowerPoint.

**Concepts:** Cat Modeling, VaR, TVaR, Capital Allocation, XOL Reinsurance Structuring, RAROC, ERM Frameworks.
