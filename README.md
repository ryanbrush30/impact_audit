<b> Project: Multi-Year Workforce Performance and Pay Equity Modeling vs Business Outcomes Goal</b>

This project builds an integrated, multi-year workforce analytics and business impact modeling pipeline that connects:
Employee lifecycle data (hire, termination, tenure)
Effective-dated compensation and performance history
Business unit KPIs (revenue, productivity, vacancies, quality)

<b>Predictive modeling for:</b>
Attrition risk (via a stay probability model)
Next-period revenue forecasting
KPI “what-if” impact analysis

The goal is not just prediction, but decision support:
understanding which workforce and operational levers matter most and how changes in those levers are expected to impact business outcomes.

<b>Key Design Principles</b>
1. Time-aware panel modeling.  All models are trained on employee-month or BU-month snapshots, strictly avoiding future leakage.

2. Effective-dated joins.  Compensation and performance history are attached using robust groupwise as-of joins, not naive merges.

3. Stay modeling (not naive attrition classification) The model predicts probability of staying employed over a forward horizon. Attrition risk is derived as 1 − P(stay) to avoid inverted signal issues common in HR panel data.

4. Business-level modeling.  Revenue forecasting is performed at the business unit–period level, not the employee level, preventing duplication bias.

5. Model-based KPI impact analysi.  Revenue models are reused to generate controlled “what-if” scenarios for KPI changes.

<b>Data Inputs</b>
Expected CSV files in the project root:

<b>employees.csv</b>
Required columns:
- employee_id
- hire_date
- term_date (nullable)
- business_unit_id
Optional: job family, location, role attributes

<b>comp_history.csv</b>
Effective-dated compensation records:
employee_id
effective_date
Examples:
- base_pay
- bonus_target
- compa_ratio
- merit_pct
- promo_flag

<b>perf_history.csv</b>
Performance reviews:
- employee_id
- review_date
Examples:
rating
calibrated_rating
goal_attainment_pct

<b>business_kpi.csv</b>
Business unit KPIs by period:
business_unit_id
period_start

Examples:
revenue
operating_margin_pct
productivity_index
avg_vacancy_days
quality_incidents
headcount_proxy

<b>Pipeline Steps</b>
1. Employee–Month Panel Construction
Builds a complete monthly panel from hire date through termination.
Drops termination month and later rows to ensure “employed at snapshot” integrity.
Computes tenure in months.

2. Effective-Dated Feature Attachment
Compensation and performance histories are attached using groupwise merge_asof logic.
Ensures each snapshot reflects the most recent known state at that time.

3. Feature Engineering
Lagged features (1, 3, 6 months)
Rolling aggregates (mean, std)
History depth filtering to ensure sufficient signal

4. Stay / Attrition Modeling
Target: stay_in_horizon (default: 3 months)
Model: Logistic Regression with:
Balanced class weights
Scaling
Sparse one-hot encoding

Probability direction is automatically corrected if AUC indicates inversion.
Attrition risk defined as:
attrition_risk = 1 − stay_probability

5. Revenue Forecasting (BU-Period)
Aggregates employee features to the business unit–month level.
Predicts next-period revenue using a Ridge regression model.
Prevents leakage by excluding contemporaneous revenue from features.

6. KPI Impact (“What-If”) Analysis
Applies controlled deltas to selected KPIs (for example):
Reduce vacancy days by 5
Increase productivity index by 0.05
Add 2 headcount
Measures change in predicted next-period revenue.
Outputs both detailed and summarized impact tables.

<b>Outputs</b>
Employee-Level
panel_scored.csv
Full employee-month dataset with predictions and features.
panel_scores_min.csv
Minimal scoring output:
employee_id
period_start
business_unit_id
stay_proba
attrition_risk
Business Unit–Level
bu_period_features.csv

<b>Aggregated BU-period feature set.</b>
bu_period_scored.csv
BU-period revenue predictions.

<b>KPI Impact</b>
kpi_impact_output.csv
BU-period predicted revenue deltas per KPI change.
kpi_impact_summary.csv
Summary statistics (mean, p10, p50, p90 impact per KPI).

<b>Interpreting Results</b>
Attrition AUC ~0.60
This is realistic for workforce data and represents meaningful ranking signal.
PR-AUC should be compared to base rate
Small absolute values are expected with rare attrition events.
KPI impact is model-based, not causal
Results should be interpreted as:
“Holding other factors constant, the model predicts that improving KPI X by Y is associated with a change of Z in next-period revenue.”

<b>Configuration</b>
Key parameters live in the Config dataclass:
Prediction horizon
Train/test split window
Minimum history length
Top-K risk flagging rate
KPI impact step sizes

<b>Intended Use</b>
This project is designed for:
Workforce planning and scenario modeling
HR–Finance–Operations alignment
Executive decision support
Demonstrating advanced people analytics capability

It is not intended as a black-box churn model, but as a transparent, extensible analytics framework.
