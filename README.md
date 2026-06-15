# AI-RISK-RADAR

**Country-level AI infrastructure risk analysis with scenario simulation**

[Turkish version / Türkçe versiyon](README_TR.md)

AI-RISK-RADAR is a data science project that analyzes how increasing artificial intelligence demand may create pressure on countries' energy capacity and digital infrastructure.

The project combines World Bank country-level indicators, feature engineering, scenario-based stress testing, machine learning, Power BI dashboarding, and a Streamlit prototype.

The goal of this project is not to predict a real infrastructure collapse probability. Instead, it creates a **relative AI infrastructure pressure indicator** that compares countries under different AI demand and energy capacity assumptions.

---

## Project Motivation

As artificial intelligence systems become more accessible and AI-based services grow, the pressure on energy systems, digital infrastructure, data centers, and national capacity may also increase.

The main question behind this project is:

> If AI demand increases, which countries may face higher infrastructure and energy capacity pressure compared to others?

Since there is no direct variable such as “AI infrastructure risk” in the original dataset, this project focuses on building proxy variables, engineering a risk score, testing stress scenarios, and visualizing the results.

The project aims to transform raw macro-level country indicators into a structured AI infrastructure risk analysis framework.

---

## Project Workflow

The project follows this workflow:

1. Collect country-level macro indicators from the World Bank API.
2. Clean and preprocess country-year level data.
3. Remove non-country aggregate records.
4. Create proxy variables for AI demand and energy capacity.
5. Generate a relative AI infrastructure risk score.
6. Expand the dataset using a 3x3 scenario stress-test structure.
7. Segment countries into Low / Medium / High Risk groups with KMeans.
8. Train and compare supervised regression models.
9. Select the final model with leakage control.
10. Visualize outputs with Power BI.
11. Build an interactive Streamlit scenario simulator.

---

## Data Source

The dataset was collected from the **World Bank API**.

Selected period:

```text
2015–2025
```

The data is country-year based. However, not every country and variable has complete data for every year. Therefore, the project uses the most usable available records within the selected time range.

---

## Raw Variables

The raw indicators were collected from World Bank. These variables do not directly represent AI infrastructure risk; they are used as macro-level proxies.

| Raw variable                | Meaning in the project                   |
| --------------------------- | ---------------------------------------- |
| `gdp_per_capita`            | Economic capacity / income level         |
| `population`                | Potential demand scale                   |
| `internet_users_pct`        | Digitalization and AI adoption potential |
| `electricity_kwh_pc`        | Energy capacity                          |
| `renewable_electricity_pct` | Energy sustainability                    |
| `energy_imports_net_pct`    | Energy dependency                        |

In the project context, some variables are interpreted analytically. For example, `population` literally means population, but in this project it is used as a proxy for **potential AI demand scale**.

---

## Dataset Size

The initial merged World Bank dataset contains:

```text
2,926 country-year rows
8 columns
```

After preprocessing and feature engineering, the model-ready dataset contains:

```text
1,475 country-year observations
14 columns
```

After applying the 3x3 scenario structure, the scenario dataset contains:

```text
13,275 country-year-scenario observations
21 columns
```

This expansion comes from:

```text
1,475 model-ready observations × 9 scenarios = 13,275 scenario observations
```

The country-level summary contains:

```text
155 countries
```

The country-scenario summary contains:

```text
1,395 rows
```

This is calculated as:

```text
155 countries × 9 scenarios = 1,395 country-scenario records
```

---

## Data Preprocessing

The World Bank data required several preprocessing steps before analysis and modeling.

### Wide to Long Format

The raw data originally contained year columns such as:

```text
YR2015, YR2016, YR2017, ...
```

These columns were transformed into a long country-year format.

### Indicator Merging

Each indicator was selected and then merged at the country-year level. This created one unified dataset where each row represents a country-year observation.

### Missing Value Handling

Missing values were handled differently depending on the analytical role of the variable.

#### Demand-side variables

The demand-side variables were critical for constructing the AI demand score:

* `gdp_per_capita`
* `internet_users_pct`
* `population`

Rows with missing values in these variables were removed because missing demand-side values could distort the AI demand score and the risk score.

#### Capacity-side variable

Energy capacity was mainly represented through:

* `electricity_kwh_pc`

For this variable, a controlled imputation strategy was used:

1. Missing `gdp_per_capita` values were filled with the median.
2. Countries were grouped into `low`, `mid`, and `high` GDP groups.
3. Missing `electricity_kwh_pc` values were filled using the average electricity consumption of the related GDP group.
4. Remaining missing values were filled using the overall median.

This approach made the electricity capacity imputation more meaningful by considering economic similarity between countries.

---

## Removing Non-Country Records

World Bank returns not only countries but also regional, income-level, and aggregate records.

Examples of removed non-country records include:

* World
* High income
* Low income
* Middle income
* Europe & Central Asia
* Sub-Saharan Africa
* Latin America & Caribbean
* Fragile states
* Small states

These records were removed because the model and maps are designed to analyze real countries, not regional or aggregate groups.

---

## Feature Engineering

The original World Bank indicators do not directly measure AI infrastructure risk. Therefore, feature engineering is one of the core parts of this project.

The goal was to transform raw macro indicators into variables that better represent the business problem.

---

## AI Demand Score

AI demand is not directly available in the dataset, so a proxy AI demand score was created.

The logic was:

* Higher GDP per capita may indicate stronger economic and investment capacity.
* Higher internet usage may indicate stronger digitalization and AI adoption potential.
* Higher population may indicate larger potential user and demand scale.

Before weighting, variables were transformed using `log1p` and then standardized.

The weighted AI demand score was calculated as:

| Variable             | Weight |
| -------------------- | -----: |
| `gdp_per_capita`     |    40% |
| `internet_users_pct` |    35% |
| `population`         |    25% |

This created the variable:

```text
ai_demand_score_weighted
```

---

## Energy Capacity Score

Energy capacity was mainly represented through:

```text
electricity_kwh_pc
```

Since AI infrastructure, data centers, and computational systems are closely related to energy availability, electricity consumption per capita was used as a proxy for energy capacity.

The energy capacity score was standardized and then converted into a positive scale.

Important engineered variables:

```text
energy_capacity_score
energy_capacity_pos
```

`energy_capacity_pos` was created because standardized values can be negative, and the risk formula uses energy capacity in the denominator. A positive version was required to avoid negative or unstable risk values.

---

## AI Risk Score

The base AI infrastructure risk score follows this logic:

```text
AI Demand ↑
Energy Capacity ↓
= AI Infrastructure Risk ↑
```

Simplified formula:

```text
AI Risk Score = AI Demand Score / Energy Capacity
```

This created the first non-scenario risk variable:

```text
ai_risk_score
```

This score is not a real probability of failure. It is a **relative country-level AI infrastructure pressure score**.

---

## Scenario Analysis

The project uses a 3x3 scenario stress-test structure.

### AI Demand Scenarios

| Scenario          | Multiplier | Meaning                     |
| ----------------- | ---------: | --------------------------- |
| `baseline_demand` |       1.00 | Current demand              |
| `high_demand`     |       1.50 | AI demand increases by 50%  |
| `extreme_demand`  |       2.00 | AI demand increases by 100% |

### Energy Capacity Scenarios

| Scenario            | Multiplier | Meaning                             |
| ------------------- | ---------: | ----------------------------------- |
| `baseline_energy`   |       1.00 | Current energy capacity             |
| `low_energy`        |       0.50 | Energy capacity is pressured by 50% |
| `severe_low_energy` |       0.25 | Energy capacity is pressured by 75% |

The two dimensions were crossed:

```text
3 demand scenarios × 3 energy scenarios = 9 scenarios
```

This expanded the dataset from:

```text
1,475 country-year observations
```

to:

```text
13,275 country-year-scenario observations
```

The main scenario-based risk variable is:

```text
scenario_risk_score
```

---

## Risk Segmentation

Countries were segmented into three risk groups using KMeans clustering:

* Low Risk
* Medium Risk
* High Risk

The segmentation was based on country-level average scenario risk scores.

This made it easier to interpret the results in Power BI maps and dashboards.

---

## Supervised Modeling

The project also includes supervised regression modeling to predict the scenario risk score.

Several model families were compared:

* Linear Regression
* Ridge Regression
* Lasso Regression
* KNN
* Decision Tree
* Random Forest
* Extra Trees
* Gradient Boosting
* XGBoost
* LightGBM
* CatBoost
* Voting Regressor
* Stacking Regressor

Evaluation metrics:

* RMSE
* MAE
* R²

The final selected model was:

```text
Tuned LightGBM Regressor
```

Final model performance:

| Model          |   RMSE |    MAE |     R² |
| -------------- | -----: | -----: | -----: |
| LightGBM Tuned | 0.0665 | 0.0428 | 0.9991 |

---

## Leakage Control

The project paid special attention to target leakage.

Some engineered variables directly contribute to the target risk score. Therefore, they were not used as final model inputs.

Excluded leakage-prone variables included:

```text
ai_demand_score_weighted
energy_capacity_score
energy_capacity_pos
ai_risk_score
scenario_demand
scenario_energy_capacity
scenario_risk_score
risk_cluster
risk_level
```

The final model used 6 explainable input variables:

| Final model feature  | Meaning                  |
| -------------------- | ------------------------ |
| `gdp_per_capita`     | Economic capacity        |
| `internet_users_pct` | Digitalization           |
| `population`         | Demand scale             |
| `electricity_kwh_pc` | Energy capacity          |
| `demand_multiplier`  | AI demand scenario       |
| `energy_multiplier`  | Energy capacity scenario |

This helped reduce leakage risk and made the model more interpretable and product-ready.

---

## Feature Importance

Feature importance was analyzed to understand which variables the final model relied on most.

According to permutation feature importance, the most important variables were:

1. `internet_users_pct`
2. `energy_multiplier`
3. `gdp_per_capita`
4. `demand_multiplier`
5. `population`
6. `electricity_kwh_pc`

The high importance of `internet_users_pct` shows that digitalization level plays a major role in the model’s risk estimation.

The importance of `energy_multiplier` and `demand_multiplier` shows that the model responds strongly to stress scenario assumptions.

The importance of `population` does not mean that crowded countries are automatically risky. In this project, population acts as a proxy for potential AI demand scale.

---

## Power BI Dashboard

Power BI was used to create visual summaries of the project outputs.

The dashboard includes:

* Risk level world map
* Scenario-based AI infrastructure risk map
* Top 10 risky countries
* Scenario-based Top 10 risky countries
* Low / Medium / High risk segmentation

Two different perspectives were visualized:

1. **Baseline / current risk view**
   Shows the current or non-stressed risk profile.

2. **Scenario-based risk view**
   Shows how country rankings change when AI demand and energy capacity stress scenarios are included.

This helps show that risk rankings may change under different stress conditions.

---

## Streamlit Prototype

A Streamlit prototype was built to make the project interactive.

The Streamlit app allows users to:

* Select a country
* Adjust AI demand change
* Adjust energy capacity change
* Generate a predicted risk score
* View a normalized risk index
* Interpret country-level risk under different scenarios

The Streamlit app turns the project from a static analysis into a scenario simulation prototype.

---

## Output Files

The repository contains the following output files:

| File                                    | Description                                     |
| --------------------------------------- | ----------------------------------------------- |
| `01_merged_worldbank_data.csv`          | Merged World Bank country-year data             |
| `02_model_ready_data.csv`               | Cleaned model-ready dataset                     |
| `03_scenario_results.csv`               | Scenario-expanded country-year-scenario data    |
| `04_country_risk_results.csv`           | Country-level average risk results and segments |
| `05_top10_risk_countries.csv`           | Top 10 risky countries by risk score            |
| `06_high_risk_top10.csv`                | Top countries within the High Risk segment      |
| `07_medium_risk_top10.csv`              | Top countries within the Medium Risk segment    |
| `08_low_risk_top10.csv`                 | Top countries within the Low Risk segment       |
| `09_country_scenario_risk_results.csv`  | Country-scenario-level risk results             |
| `10_initial_model_comparison.csv`       | Initial model comparison results                |
| `11_all_model_comparison.csv`           | All model comparison results                    |
| `12_final_model_comparison.csv`         | Final model comparison including tuned model    |
| `13_feature_importance_native.csv`      | Native feature importance results               |
| `14_feature_importance_permutation.csv` | Permutation feature importance results          |

---

## Technologies Used

* Python
* Pandas
* NumPy
* Scikit-learn
* LightGBM
* XGBoost
* CatBoost
* KMeans
* World Bank API
* Streamlit
* Power BI
* GitHub

---

## How to Interpret the Risk Score

The risk score should not be interpreted as:

```text
"This country has X% probability of infrastructure failure."
```

Instead, it should be interpreted as:

```text
"This country has a higher or lower relative AI infrastructure pressure compared to other countries under the selected assumptions."
```

The score is useful for:

* comparing countries,
* identifying relative infrastructure pressure,
* testing scenarios,
* observing risk ranking changes,
* supporting early-stage infrastructure risk analysis.

---

## Limitations

This project is a prototype and has several limitations.

Important missing dimensions include:

* real data center locations,
* water stress,
* cooling requirements,
* electricity grid reliability,
* energy prices,
* carbon intensity,
* real-time AI investment data,
* regional differences within countries.

The current version works at the country level and uses macro indicators. Therefore, it should be treated as an early-stage risk scoring and scenario simulation study, not a final commercial decision system.

---

## Future Work

Future versions of this project can be improved by adding:

* water stress and cooling demand indicators,
* real data center location data,
* electricity grid reliability metrics,
* energy price and carbon intensity data,
* renewable energy capacity indicators,
* positive infrastructure improvement scenarios,
* region-level or city-level analysis,
* automated data updates,
* more advanced scenario simulation.

The long-term direction of AI-RISK-RADAR could be an **AI infrastructure early warning and scenario simulation platform**.

---

## Project Positioning

This project is not a direct investment recommendation system.

It is better understood as:

> A prototype risk scoring and scenario simulation system that makes country-level AI infrastructure pressure visible through data.

Its main contribution is the process of transforming raw macroeconomic and infrastructure data into a structured AI infrastructure risk analysis framework.



