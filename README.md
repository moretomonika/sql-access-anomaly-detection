\# SQL-Driven User Access Anomaly Detection



A system that flags suspicious employee login and file-access behavior using SQL-engineered behavioral features and both unsupervised and supervised anomaly detection.



\## Overview



This project simulates an enterprise access-log environment — employees, login events, and file-access events — then deliberately injects a small set of suspicious sessions into that data. Behavioral features (off-hours access, session frequency, file-access volume, unfamiliar IPs) are engineered \*\*entirely in SQL\*\* using CTEs and window functions, and two models are trained and compared on their ability to catch the hidden anomalies:



\- \*\*Isolation Forest\*\* (unsupervised — never sees the true labels during training)

\- \*\*Logistic Regression\*\* (supervised — trained directly on the labels)



Because the data and anomalies are synthetic, the ground truth is fully known, which makes it possible to objectively evaluate both approaches rather than just eyeball the results.



\## Why synthetic data



Real, labeled insider-threat data is essentially never public — companies don't publish confirmed breach logs. Generating the data also means the ground truth (which sessions are actually anomalous) is known exactly, enabling real precision/recall/ROC-AUC evaluation rather than guesswork.



\## Tech Stack



Python · SQLite · Pandas · scikit-learn · Matplotlib / Seaborn



\## Project Structure



\- \*\*Data generation:\*\* 50 synthetic employees, 90 days of activity, \~2,900 login sessions, \~12,000 file-access events, weekdays only, login times drawn from a normal distribution around each employee's typical hours

\- \*\*Anomaly injection:\*\* \~3% of sessions corrupted across 4 categories — off-hours logins, file-access volume spikes, sensitivity spikes, and unfamiliar IPs — with a ground-truth label table tracking exactly which sessions were altered and how

\- \*\*Feature engineering (SQL):\*\* one query chaining 6 CTEs, using JOINs, CASE WHEN, a correlated subquery (is this IP new for this employee?), a window function (`LAG()` for days since last login), and conditional aggregation (`SUM(CASE WHEN...)`) for daily file-activity stats — no heavy pandas preprocessing needed

\- \*\*Modeling:\*\* Isolation Forest (unsupervised) vs. Logistic Regression with `class\_weight='balanced'` (supervised, to handle the \~3% class imbalance)



\## Key Results



| Model | Type | ROC-AUC |

|---|---|---|

| Isolation Forest | Unsupervised | 0.848 |

| Logistic Regression | Supervised | 0.953 |



\## Key Insight



Volume- and sensitivity-based signals (`file\_access\_count`, `high\_sensitivity\_count`) were the strongest predictors of anomalous sessions — both models picked up on these clearly, since they created large, obvious numeric deviations from normal behavior. The unfamiliar-IP signal was comparatively weak, which traced back to a real bug in data generation: employees weren't originally assigned a consistent set of "usual" IPs, so almost every login looked like a "new IP" regardless of whether it was actually suspicious. Fixing this (giving each employee a small fixed pool of usual IPs) made the feature meaningful. This is a good example of how a feature-engineering problem can actually be a data-generation problem in disguise.



\## Known Limitations



\- File-access events aren't linked to a specific login session via an explicit foreign key — only implicitly related by employee and approximate timing

\- The anomaly injection rate (3%) is far higher than a realistic real-world rate, chosen so the small synthetic dataset would have enough positive examples to learn from and evaluate against

\- Sensitivity access is treated as anomalous relative to a flat global threshold, not relative to each employee's department/role — in reality, a Director accessing high-sensitivity files is normal, while the same access from an unrelated department is suspicious

\- Volume-spike anomalies are generated as a timing burst (compressed into a short window), but the current features only aggregate at the daily level, so that timing detail isn't actually used as a detection signal yet

\- No holidays, no genuinely high-workload-but-legitimate days, and no failed-login patterns are simulated



\## Future Improvements



\- Add an explicit `session\_id` linking file access events to their originating login

\- Model department/role-based sensitivity norms so anomalies are relative to an employee's own baseline, not a global threshold

\- Add legitimate high-activity days (e.g. month-end for Finance) to test whether the model overfits to "unusual = bad"

\- Add a failed-login anomaly pattern (e.g. several failed attempts followed by a success)

\- Add a timing-concentration feature (e.g. time span between first and last file access per day) so burst-style anomalies are actually detectable by the feature set

\- Compare against a tree-based model (XGBoost/Random Forest) and SHAP for feature importance

\- Evaluate using Precision-Recall AUC alongside ROC-AUC, since ROC-AUC can look optimistic on imbalanced data



\## How to Run



1\. Clone the repo

2\. `pip install -r requirements.txt`

3\. Open the notebook in Jupyter and run all cells top to bottom — the database is created fresh each run



\## Author's Note



Built to practice SQL feature engineering (CTEs, window functions, conditional aggregation) applied directly to an ML problem, and to compare how much value labeled data adds over unsupervised detection when both are evaluated fairly on the same ground truth.

