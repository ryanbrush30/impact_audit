<b>Project: Multi-Year Workforce Performance and Pay Equity Modeling vs Business Outcomes</b><br><br>

<b>Goal</b><br>
This project builds an integrated, multi-year workforce analytics and business impact modeling pipeline that connects:

<ul> <li>Employee lifecycle data (hire, termination, tenure)</li> <li>Effective-dated compensation and performance history</li> <li>Business unit KPIs (revenue, productivity, vacancies, quality)</li> </ul>

<b>Predictive modeling for:</b>

<ul> <li>Attrition risk (via a stay probability model)</li> <li>Next-period revenue forecasting</li> <li>KPI “what-if” impact analysis</li> </ul>

The goal is not just prediction, but decision support: understanding which workforce and operational levers matter most and how changes in those levers are expected to impact business outcomes.
<br><br>

<b>Key Design Principles</b>

<ol> <li> <b>Time-aware panel modeling.</b><br> All models are trained on employee-month or business unit-month snapshots, strictly avoiding future data leakage. </li> <li> <b>Effective-dated joins.</b><br> Compensation and performance history are attached using robust groupwise as-of joins, not naive merges. </li> <li> <b>Stay modeling (not naive attrition classification).</b><br> The model predicts the probability of staying employed over a forward horizon. Attrition risk is derived as: <br> <code>attrition_risk = 1 - P(stay)</code><br> This avoids inverted signal issues common in HR panel data. </li> <li> <b>Business-level modeling.</b><br> Revenue forecasting is performed at the business unit–period level, not the employee level, preventing duplication bias. </li> <li> <b>Model-based KPI impact analysis.</b><br> Revenue models are reused to generate controlled “what-if” scenarios for KPI changes. </li> </ol> <br>

<b>Data Inputs</b><br>
Expected CSV files in the project root:
<br><br>

<b>employees.csv</b><br>
Required columns:

<ul> <li>employee_id</li> <li>hire_date</li> <li>term_date (nullable)</li> <li>business_unit_id</li> </ul> Optional: job family, location, role attributes <br><br>

<b>comp_history.csv</b><br>
Effective-dated compensation records:

<ul> <li>employee_id</li> <li>effective_date</li> </ul> Examples: <ul> <li>base_pay</li> <li>bonus_target</li> <li>compa_ratio</li> <li>merit_pct</li> <li>promo_flag</li> </ul> <br>

<b>perf_history.csv</b><br>
Performance review history:

<ul> <li>employee_id</li> <li>review_date</li> </ul> Examples: <ul> <li>rating</li> <li>calibrated_rating</li> <li>goal_attainment_pct</li> </ul> <br>

<b>business_kpi.csv</b><br>
Business unit KPIs by period:

<ul> <li>business_unit_id</li> <li>period_start</li> </ul> Examples: <ul> <li>revenue</li> <li>operating_margin_pct</li> <li>productivity_index</li> <li>avg_vacancy_days</li> <li>quality_incidents</li> <li>headcount_proxy</li> </ul> <br><br>

<b>Pipeline Steps</b>

<ol> <li> <b>Employee-Month Panel Construction</b><br> Builds a complete monthly panel from hire date through termination.<br> Drops termination month and later rows to ensure “employed at snapshot” integrity.<br> Computes tenure in months. </li> <li> <b>Effective-Dated Feature Attachment</b><br> Compensation and performance histories are attached using groupwise <code>merge_asof</code> logic.<br> Each snapshot reflects the most recent known state at that time. </li> <li> <b>Feature Engineering</b><br> <ul> <li>Lagged features (1, 3, 6 months)</li> <li>Rolling aggregates (mean, standard deviation)</li> <li>History depth filtering to ensure sufficient signal</li> </ul> </li> <li> <b>Stay / Attrition Modeling</b><br> Target: <code>stay_in_horizon</code> (default: 3 months)<br> Model: Logistic Regression with: <ul> <li>Balanced class weights</li> <li>Feature scaling</li> <li>Sparse one-hot encoding</li> </ul> Probability direction is automatically corrected if AUC indicates inversion. </li> <li> <b>Revenue Forecasting (Business Unit–Period)</b><br> Aggregates employee features to the business unit–month level.<br> Predicts next-period revenue using Ridge regression.<br> Prevents leakage by excluding contemporaneous revenue from features. </li> <li> <b>KPI Impact (“What-If”) Analysis</b><br> Applies controlled deltas to selected KPIs, for example: <ul> <li>Reduce vacancy days by 5</li> <li>Increase productivity index by 0.05</li> <li>Add 2 headcount</li> </ul> Measures the change in predicted next-period revenue. </li> </ol> <br>

<b>Model Performance (Observed Run Output)</b><br><br>

<b>Attrition Risk Model (3-month horizon)</b><br>

<ul> <li>Test split start date: 2025-05-01</li> <li>Positive rate (attrition in horizon): 2.04%</li> <li>ROC-AUC: 0.941</li> <li>PR-AUC: 0.244</li> <li>Lift @ Top 10% risk: 9.17x</li> </ul>

At a Top-10% risk threshold (stay probability <= 0.60), the model captures approximately 92% of attrition events while flagging 10% of the population, providing strong prioritization power for targeted intervention in a highly imbalanced setting.
<br><br>

<b>Revenue Forecast Model (Business Unit–Period)</b><br>

<ul> <li>Test split start date: 2025-04-01</li> <li>Mean Absolute Error: 324,789</li> <li>R²: 0.511</li> </ul>

Revenue forecasts are trained strictly on prior-period workforce and KPI signals, and reused downstream to estimate controlled KPI-driven revenue deltas.
<br><br>

<b>Outputs</b><br><br>

<b>Employee-Level</b><br>
<b>panel_scored.csv</b><br>
Full employee-month dataset with predictions and features.<br><br>

<b>panel_scores_min.csv</b><br>
Minimal scoring output:

<ul> <li>employee_id</li> <li>period_start</li> <li>business_unit_id</li> <li>stay_proba</li> <li>attrition_risk</li> </ul>

<b>Business Unit–Level</b><br>
<b>bu_period_features.csv</b><br>
Aggregated BU-period feature set.<br><br>

<b>bu_period_scored.csv</b><br>
BU-period revenue predictions.<br><br>

<b>KPI Impact</b><br>
<b>kpi_impact_output.csv</b><br>
BU-period predicted revenue deltas per KPI change.<br><br>

<b>kpi_impact_summary.csv</b><br>
Summary statistics (mean, p10, p50, p90 impact per KPI).
<br><br>

<b>Interpreting Results</b>

<ul> <li><b>High ROC-AUC with low base rate is expected</b><br> Attrition is rare; ranking power and lift are more decision-relevant than raw accuracy.</li> <li><b>PR-AUC must be evaluated against the 2% base rate</b><br> A PR-AUC of 0.244 represents strong enrichment over random selection.</li> <li><b>KPI impact outputs are model-based, not causal</b><br> Results should be interpreted as:<br> “Holding other factors constant, the model predicts that improving KPI X by Y is associated with a change of Z in next-period revenue.”</li> </ul> <br>

<b>Configuration</b><br>
Key parameters live in the <code>Config</code> dataclass:

<ul> <li>Prediction horizon</li> <li>Train/test split window</li> <li>Minimum history length</li> <li>Top-K risk flagging rate</li> <li>KPI impact step sizes</li> </ul> <br>

<b>Intended Use</b><br>
This project is designed for:

<ul> <li>Workforce planning and scenario modeling</li> <li>HR, Finance, and Operations alignment</li> <li>Executive decision support</li> <li>Demonstrating advanced people analytics capability</li> </ul>

It is not intended as a black-box churn model, but as a transparent, extensible analytics framework.
