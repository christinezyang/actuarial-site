---
layout: page
title: "Wildfire Expected Loss Modeling"
subtitle: Data Science in Finance & Insurance, Columbia University
description: Built a three-part parcel-level wildfire risk model — occurrence probability, damage severity, and expected dollar loss — across roughly 21,400 LA County properties.
category: projects
importance: 3
tags: [Multinomial Naive Bayes, Random Forest, Gradient Boosting, Data Imputation, Python]
---

**Class:** Data Science in Finance & Insurance (2025)

This course is a graduate core actuarial course covering machine learning for regression, classification, and unsupervised learning, along with the surrounding toolkit of cross-validation, regularization, dimension reduction, and ensemble methods. Coursework leans on implementing concepts in Python and ties into SOA/CAS statistical learning curriculum.

The final group project was assigned as an open-ended data science exploration (as long as it related to insurance or finance) that may be useful to a potential client. I was in a team of seven.

Through the beginning and even middle of the project, we had an ambitious project idea. I played an important role in pushing our group toward setting what we'd actually predict, concretely defining the three-part project scope, and generally running team meetings. I also wrote the final submitted project report and was one of the team members who presented results to the class and professor.

Below is a summary of the work done in this group project.

## Project Background

The Los Angeles-area wildfires have made clear how costly these catastrophe events are for residents, communities, and insurers. In response, some insurers scaled back coverage or raised rates in catastrophe-prone parts of California, which makes accurate, building-level wildfire risk assessment increasingly relevant.

For a dataset of about 21,400 parcels (generally corresponds to each building), our project set out to:

- **Part A:** Estimate the probability that a fire will occur at each parcel, or fire occurrence risk.
- **Part B:** Estimate the percent of property damage at each parcel, given a fire occurs there.
- **Part C:** Combine Parts A and B to estimate the expected dollar loss at each parcel.

Across all three parts, we sourced and synthesized across six datasets, trained five models, and delivered insights in an in-person presentation and write-up report.

## Part A: Fire Occurrence Risk

### Step 1: Map Historical Fire Exposure

We used parcel geometries for historically affected properties in LA County together with California's historical firezone burn perimeters from the [California State Geoportal](https://gis.data.ca.gov/), processed in QGIS.

{% include figure.liquid path="assets/img/wildfire-proj_historical-fire-zones.png" alt="Map of historical wildfire zones across LA County with 5,000 ft and 10,000 ft buffer zones, parcel locations overlaid" caption="Historical Fire Zones" %}

Each firezone perimeter was buffered by 5,000 feet and 10,000 feet. For every parcel, we then counted how many historical firezones and buffer zones it overlapped with.

{% include figure.liquid path="assets/img/wildfire-proj_overlapping-zones.png" alt="Zoomed map showing parcels overlapping historical fire zones and buffer zones near the LA urban boundary" caption="Overlapping Fire Zones and Buffer Zones for Parcels" %}

### Step 2: Score Relative Risk

We combined those three overlap counts into a single relative risk score $R$, weighting direct firezone overlaps more heavily than buffer overlaps:

$$ R = 5 \times \text{firezone overlaps} + 2 \times \text{5,000 ft buffer overlaps} + 1 \times \text{10,000 ft buffer overlaps} $$

We didn't find existing literature specifying a standard weighting scheme, so we chose one a potential client could reasonably adjust.

Then, we normalized $R$ so that the parcel average came out to 1:

$$ \bar{R} = \frac{1}{N} \sum_{i=1}^N R_i, \quad R_{norm} = \frac{R}{\bar{R}} $$

A normalized score of 2 means a parcel carries twice the county-average relative fire risk, based on its geographic proximity to past fire activity.

### Step 3: Convert to Occurrence Probability

To turn the relative score to actual probability, we needed a baseline. Sampled uniformly, the average probability that a given building experiences a fire in a year is the number of fire events in the county divided by the number of parcels. Using 2024 [ReGrid](https://app.regrid.com/store/us/ca/los-angeles) and [LA County Economic Development Corporation](https://laedc.org/wp-content/uploads/2025/02/LAEDC_2025-LA-Wildfires-Study_090525-UPDATE.pdf) figures:

$$ p_{mean} = \frac{\text{\# of fire events}}{\text{\# of parcels}} = \frac{20{,}218}{2{,}429{,}237} \approx 0.008322 $$

Multiplying this baseline by each parcel's normalized relative risk gives the final fire occurrence risk for each parcel:

$$ \text{fire occurrence risk} = R_{norm} \times p_{mean} $$

Thus, a parcel sitting inside multiple historical firezones ends up meaningfully above the countywide baseline of about 0.83%, while a parcel outside all buffer zones sits near 0.

## Part B: Damage Prediction

### Step 4: Data Preprocessing

For damage severity, we used fire damage and building details data from [State of California Open Data](https://lab.data.ca.gov/dataset/cal-fire-damage-inspection-dins-data/fd2ee9be-8555-43e2-92d7-13a454d0a89c).

|      Damage      |     Structure Type      | Roof Construction |   Exterior Siding   | ... |
| :--------------: | :---------------------: | :---------------: | :-----------------: | :-: |
| Destroyed (>50%) | Single Family Residence |      Unknown      | Stucco Brick Cement | ... |
| Destroyed (>50%) | Utility Misc Structure  |       Metal       |       Unknown       | ... |
| Affected (1-9%)  | Utility Misc Structure  |       Wood        |        Wood         | ... |
|       ...        |           ...           |        ...        |         ...         | ... |

<div class="caption">Recreated Sample of Building Damage and Details Dataset</div>

The raw data was full of blank fields or ones filled as "Unknown". Two features (distance from propane tank to structure, distance from residence to utility structures) were missing more than 89% of values, so we dropped them outright.

For other features, we tested five approaches to create five differently cleaned versions of the dataset, all of which were tested in the models later:

- **Drop rows with NaN and "Unknown":** simplest approach but removes a lot of data.
- **Drop rows with NaN but keep "Unknown" as its own category:** retains more data and could capture ambiguity in building features.
- **Mode imputation on NaN and "Unknown":** fill in missing values with the most common observed category.
- **KNN imputation on NaN and "Unknown":** estimate missing values using the most "similar" buildings.
- **Iterative imputation on NaN and "Unknown":** more complex method that models each missing feature as a function of the remaining variables.

The final feature set used to predict percent damage combined 12 building characteristics (eaves, vent screen, roof construction, year built, and similar) with the fire zone overlap counts from Part A.

### Step 5: Model Comparison

We ran five models with all five cleaned datasets:

1. **Multinomial Naive Bayes:** A probabilistic approach to classify damage based on categorically-encoded building features. We applied one-hot encoding to the features and mapped the target prediction (% damage) into five categories:
   - Class 0: no damage
   - Class 1: affected (1-9%)
   - Class 2: minor (10-25%)
   - Class 3: major (26-50%)
   - Class 4: destroyed (>50%)
2. **Gradient Boosting:** A sequential ensemble of decision trees that can model complex relationships by minimizing prediction error. We mapped damage categories to numeric severity values (0.000-0.750) and applied one-hot encoding for categorical variables to enable regression.
3. **Light Gradient Boosting (LightGBM):** A boosted ensemble of decision trees designed for efficient training.
4. **Random Forest Regression:** An ensemble of randomized decision trees that tries to capture nonlinear relationships.
5. **eXtreme Gradient Boosting:** A boosted ensemble of decision trees that incorporates regularization to try to reduce overfitting.

We found that data preprocessing approach #2 (dropped rows with NaN and consider "Unknown" as its own category) allowed for highest accuracy across all models.

|          Model           | R² (Option 2) |
| :----------------------: | :-----------: |
| Multinomial Naive Bayes  |   **0.809**   |
|    Gradient Boosting     |     0.706     |
|         LightGBM         |   **0.746**   |
| Random Forest Regression |     0.737     |
|         XGBoost          |     0.739     |

<div class="caption">$R^2$ across Five Models Using Data Preprocessing Approach #2</div>

We saw that Multinomial Naive Bayes and LightGBM performed the best here. However, Naive Bayes had a significant issue of misclassifying low-damage buildings as _no damage_, which leads to overall underestimation of loss severity and reduces the model's suitability for risk assessment.

The LightGBM model has fewer severe misclassifications between the lowest and highest extremes, but its overall accuracy is lower.

Thus, we recommended the LightGBM model for this project. While Naive Bayes achieved a higher raw accuracy, its tendency for misclassification makes LightGBM a more reliable option for real-world application.

**Feature Importance:** Using the models, we got the feature importance of various building details. From Naive Bayes, the top two most indicative features for the _Destroyed (>50%)_ class are structural characteristics like Exterior Siding (stucco, brick, cement) and Structure Category (single residence). This result makes sense for a probabilistic model because it assigns higher importance to features that are most strongly and consistently linked to severe damage outcomes.

From LightGBM, the distribution of importance is much more spread out compared to Naive Bayes, indicating that it may be using more complex interactions among features to generate its predictions.

## Part C: Expected Loss Calculation

### Step 6: Combine Occurrence, Damage, and Property Value

With fire occurrence risk (Part A) and percent damage (Part B), we folded in property improvement values from [County of Los Angeles Open Data](https://data.lacounty.gov/datasets/70d93266f45a4080a97b285a471493cd_0/explore) and calculated expected loss per parcel:

$$ \text{Expected Loss} = \mathbb{P}(\text{Fire Occurrence}) \times \text{\% Damage} \times \text{Property Improvement Value \$} $$

|         | Multinomial Naive Bayes |   LightGBM    |
| :-----: | :---------------------: | :-----------: |
|  Mean   |        \\$875.39        |  \\$1,090.45  |
|   Max   |      \\$57,077.72       | \\$205,408.44 |
| Std Dev |       \\$2,044.43       |  \\$2,477.56  |

<div class="caption">Loss Metrics Based on Model</div>

LightGBM's higher mean, max, and spread seem more conservative and may be a more realistic estimate. It may be able to provide a better safety margin for wildfire risk estimation compared to Naive Bayes.

### Takeaways

Splitting wildfire risk into an occurrence component and severity component mirrors how catastrophe and actuarial models are commonly built. Our project maintains both transparency and interpretability by using publicly available data and well-documented modeling techniques.

Our project illustrated a data-driven, parcel-level approach to risk modeling to address an increasingly volatile insurance landscape.

As climate-driven catastrophe risk keeps evolving, granular, parcel-level risk assessment may come to matter more for pricing, underwriting, and risk mitigation. Our project provided a scalable foundation for future work. With additional data, our project can be refined into a more comprehensive ratemaking tool.

## Skills & Tools

**Tools:** Python

**Concepts:** Multinomial Naive Bayes, Random Forest, Gradient Boosting, Data Imputation
