ImpactAudit is an integrated, multi-year workforce analytics and business impact modeling framework that connects employee lifecycle data, compensation and performance history, and business unit KPIs to support evidence-based workforce and operational decision-making.
The project is designed not merely to predict outcomes, but to explain structural workforce risk, prioritize proactive intervention, and quantify how changes in workforce and operational levers are expected to impact business results.

ImpactAudit is intentionally built for low-attrition, regulated, and mission-driven organizations, where exits are rare, structural, and not driven by short-term shocks.

**Project Goals**

This project builds an end-to-end analytics pipeline that connects:
- Employee lifecycle data (hire, termination, tenure)
- Effective-dated compensation and performance history
- Business unit KPIs (revenue, productivity, vacancies, quality, headcount)

Predictive modeling for:
- Attrition risk (via forward-looking stay/exit modeling)
- Next-period revenue forecasting
- KPI-based “what-if” impact analysis

The goal is decision support, not black-box prediction:
Understanding which workforce and operational levers matter most, how risk is distributed across the organization, and how targeted improvements are expected to impact business outcomes.

**Core Design Principles**
Time-Aware Panel Modeling
All models are trained on employee-month or business unit–month snapshots using strict time-based splits.
No future information is used in training or feature construction.

Effective-Dated Joins
Compensation and performance history are attached using robust groupwise merge_asof logic, ensuring each snapshot reflects only information known at that point in time.

Structural Risk Modeling (Not Naive Churn Classification)
Rather than relying on brittle binary churn labels, ImpactAudit models forward-looking attrition risk using time-bucketed horizons. Attrition risk is derived from predicted exit probabilities across future windows, enabling stable ranking in highly imbalanced datasets.

Business-Level Outcome Modeling
Revenue forecasting is performed at the business unit–period level, not the employee level, preventing duplication bias and ensuring business realism.

Model-Based KPI Impact Analysis
Revenue models are reused to generate controlled “what-if” scenarios for KPI changes, allowing leadership to explore how operational and workforce improvements are expected to affect outcomes.

**Data Inputs**

Expected CSV files in the project root:
employees.csv

Required columns:
employee_id
hire_date
term_date (nullable)
business_unit_id

Optional:
job_family
level
location
manager_id

other role attributes
comp_history.csv

Effective-dated compensation records:
employee_id
effective_date
Examples:
base_pay
bonus_target
compa_ratio
merit_pct
promo_flag
perf_history.csv

Performance review history:
employee_id
review_date
Examples:
rating
calibrated_rating
goal_attainment_pct

business_kpi.csv
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

Pipeline Steps
1. Employee–Month Panel Construction
- Builds a complete monthly panel from hire date through termination
- Drops termination month and later rows to ensure “employed at snapshot” integrity
- Computes tenure in months

2. Effective-Dated Feature Attachment
- Compensation and performance histories are attached via groupwise as-of joins
- Each snapshot reflects the most recent known state at that time

3. Feature Engineering
- Lagged features (1, 3, 6 months)
- Rolling aggregates (means, standard deviations)
- Peer-relative z-scores
- Manager and organizational context features
- History-depth filtering to ensure sufficient signal
- All engineered features are strictly shifted to avoid contemporaneous leakage.

4. Attrition Risk Modeling
Target: forward attrition within a configurable horizon (default: 3 months)
Model: regularized Logistic Regression with:
- Feature scaling
- Sparse one-hot encoding
- Class imbalance handling
- Performance is evaluated using:
- ROC-AUC
- PR-AUC (interpreted relative to base rate)
- Lift@Top-K (primary decision metric)
- Extensive leakage diagnostics are included:
- Feature ablation (all vs no-change vs static)
- Single-feature scans
- Shuffled-label sanity checks

5. Revenue Forecasting (Business Unit–Period)
- Aggregates employee features to the business unit–month level
- Predicts next-period revenue using Ridge regressio
- Excludes contemporaneous revenue to prevent leakage

6. KPI Impact (“What-If”) Analysis
- Applies controlled deltas to selected KPIs, for example:
- Reduce vacancy days by 5
- Increase productivity index by 0.05
- Add 2 headcount
- Measures the change in predicted next-period revenue
- Outputs are explicitly model-based associations, not causal claims

Model Performance (Leakage-Validated)
Attrition Risk Model (3-Month Horizon)

Typical observed performance in low-attrition environments:
- ROC-AUC: ~0.63–0.67
- PR-AUC: ~0.015 (vs ~0.006 base rate)
- Lift @ Top 10% risk: ~2.0–2.5x
These results indicate moderate but reliable ranking power, appropriate for population-level prioritization and planning in highly imbalanced settings.

High ROC-AUC values observed in earlier iterations were identified as leakage and intentionally rejected.
- Revenue Forecast Model (Business Unit–Period)
- Mean Absolute Error: ~300k–325k
- R²: ~0.5–0.8 depending on data density and volatility
- Revenue forecasts are trained strictly on prior-period signals and reused downstream for KPI impact analysis.

Outputs
Employee-Level

panel_scored.csv
Full employee-month dataset with features and predictions

panel_scores_min.csv
Minimal scoring output:
employee_id
period_start
business_unit_id
attrition_risk
exit_bucket

Business Unit–Level
bu_period_features.csv
Aggregated BU-period feature set
bu_period_scored.csv
BU-period revenue predictions

KPI Impact
kpi_impact_output.csv
BU-period predicted revenue deltas per KPI change
kpi_impact_summary.csv
Summary statistics (mean, p10, p50, p90 impact per KPI)

Interpreting Results
High lift matters more than raw accuracy in rare-event attrition
PR-AUC must always be evaluated relative to base rate
Attrition risk scores are ranking tools, not certainty estimates
KPI impact outputs are scenario-based, not causal conclusions

Results should be interpreted as:

“Holding other modeled factors constant, the model estimates that improving KPI X by Y is associated with a change of Z in next-period revenue.”

Ethical & Responsible Use

ImpactAudit is intended for:
- Workforce planning
- Scenario modeling
- HR, Finance, and Operations alignment
- Executive decision support

It is not intended for:
- Automated employment decisions
- Disciplinary action
- Compensation decisions in isolation
- Human judgment is always required.

Configuration
Key parameters are centralized in the Config dataclass:
- Prediction horizon
- Train/test split window
- Minimum history length
- Top-K risk flagging rate
- KPI impact step sizes

Project Status
✔ Architecture complete
✔ Leakage validated
✔ Governance-ready
✔ Production-appropriate for intended use

Further performance gains would require new data, not additional tuning.

Author
Developed by Ryan Brush
Enterprise Data, People Analytics, and AI Modeling
