aInsider Threat Detection & User Behavior Analytics (UBA)
Project 8 — Insider Threat and User Behavior Analytics from Logs

A machine learning project that analyzes employee activity logs to identify potentially suspicious behavior and prioritize user-day records for security analyst investigation.
Project status: Phase 1 (data preparation, exploratory analysis, labeling, and Logistic Regression baseline) is implemented. Phase 2 model comparisons, explainability, and robustness experiments are planned and must not be interpreted as completed results.

1. Problem and Market Case
Employees and contractors often need legitimate access to organizational systems and information. This makes insider threats difficult to detect: malicious activity can take place through valid accounts and may resemble ordinary work.

Examples include:
- Copying organizational files to removable storage.
- Accessing systems outside normal working hours.
- Sending unusual volumes of email or communicating with external recipients.
- Using another employee's account or workstation.
- Combining several individually ordinary actions into a suspicious sequence.
Traditional security rules are useful for known events, but fixed thresholds can miss changes that are unusual for a particular employee. A User Behavior Analytics (UBA) approach can establish behavioral baselines and rank deviations for further review.
Target customer and use case
Target customer: A hypothetical regional commercial bank's Security Operations Center (SOC).

The SOC is responsible for monitoring employee and contractor access to sensitive systems and information. Its analysts cannot manually investigate every daily activity record. A behavioral risk-ranking system could help prioritize a limited investigation queue.

This is an academic customer scenario, not a claim that the project has been deployed for or commissioned by a real bank.
Threat impact and cost considerations
An insider incident can create several categories of cost:
- Direct financial loss: theft, fraud, or misuse of proprietary information.
- Incident response: analyst time, digital forensics, containment, and recovery.
- Regulatory and compliance exposure: investigation, reporting, remediation, and possible penalties where applicable.
- Operational disruption: service interruption, account resets, system restoration, and business-process delays.
- Reputational impact: loss of customer and partner confidence.
This project does not assign a monetary value to these impacts. A defensible financial estimate would require a specified geography, incident type, organization size, and cited cost study. The project therefore measures analyst workload and detection performance rather than claiming a particular financial saving.
Why machine learning?
Machine learning can help identify combinations of activity and deviations from an employee's historical behavior that may not be captured by a single static rule. The model is intended to support analyst triage—not to determine intent or automatically accuse an employee.
Business success criteria
The project uses an investigation-capacity constraint: review the highest-risk 1% of user-days.

The primary measures are:
Measure	Why it matters
Precision-Recall AUC (PR-AUC)	Evaluates ranking quality under severe class imbalance.
Recall@1% review budget	Measures the share of labeled positive user-days found within the analyst review capacity.
Precision@1% review budget	Measures the proportion of reviewed user-days that are labeled positive.
False positives	Indicates investigation burden and potential unnecessary scrutiny of employees.
Analyst review workload	Translates the review queue into an estimated operational effort.


These are project evaluation criteria, not claimed production service-level targets. The 1% budget is an explicit modeling assumption and should be confirmed with a real customer before deployment.
2. Dataset and Sources
Dataset: CMU CERT Insider Threat Dataset, release r4.2.

The current implementation uses four activity logs:
File	Activity represented
logon.csv	Login and logout events
device.csv	Removable-device connection and disconnection events
file.csv	File-copy events
email.csv	Email metadata and communication activity


The release also includes http.csv, psychometric.csv, and monthly LDAP files. HTTP and psychometric data are not currently used as model predictors in this implementation.
Current data volume
Activity log	Records
Logon	854,859
Device	405,380
File	445,581
Email	2,629,979


The four logs are aggregated into an initial table of 330,452 user-day records, covering 1,000 users and 501 days.
License and use
The downloaded dataset directory contains a license.txt file. The exact license terms have not yet been transcribed or reviewed in this project documentation. Before redistribution or publication, review that file and cite the official CERT dataset source and applicable terms.

The dataset itself is not included in this GitHub repository. Obtain it from its authorized source and follow its license and usage conditions.
Second dataset
The project brief mentions the LANL authentication dataset as a possible second source. The current implementation uses CERT r4.2 only. LANL has not been incorporated, and no results from LANL are claimed.
3. Repository Structure
insider-threat-uba/
├── docs/
│   └── .gitkeep
├── notebooks/
│   ├── .gitkeep
│   └── 01_eda_baseline.ipynb
├── .gitignore
└── README.md
Raw datasets, incident evidence files, and intermediate processed data are stored separately in Google Drive. They should not be committed to this repository.
4. Data Preparation and Feature Engineering
The raw event logs are converted into a user-day feature table. Each row represents one user's activity on one calendar day.
Logon features
- logons: Number of login events.
- ah_logons: Number of after-hours login events.
- wk_logons: Number of weekend login events.
- pcs: Number of distinct PCs associated with the user's logon records.
Device features
- usb_connects: Number of removable-device connection events.
- ah_usb: Number of after-hours device connections.
File features
- file_copies: Number of file-copy events.
- ah_file: Number of after-hours file-copy events.
Email features
- emails: Number of emails.
- email_att: Number of emails containing attachments.
- att_total: Total number of attachments.
- email_size_log: Log-transformed total email size.
- rcpts: Total number of recipients.
- ext_rcpts: Number of external recipients.
- ext_emails: Number of emails with external recipients.
- ah_emails: Number of after-hours emails.
Calendar feature
- is_weekend: Indicates whether the user-day falls on a weekend.
The current baseline uses 17 features.
Cleaning and validation
The implemented checks include:
- Parsing event timestamps using the dataset's date format.
- Checking unparsed dates, missing values, duplicate IDs, and exact duplicate rows.
- Aggregating events by user and day.
- Filling missing activity counts with zero after joining the daily aggregates.
- Checking for negative values, constant features, and highly correlated features.
- Checking that all evidence-based positive user-days match the activity data.
The current EDA found no missing values or duplicate IDs in the inspected activity logs, and no feature pairs with absolute correlation above 0.95 in the training data.
User-specific behavioral baseline
A custom Scikit-learn transformer, UserBaseline, creates deviations from each user's historical behavior.

It:
1. Learns per-user statistics from the data passed to fit.
2. Uses training rows labeled as non-positive when fitting the baseline.
3. Calculates standardized deviations from each user's historical mean.
4. Falls back to global training statistics when fewer than five baseline days are available.
5. Clips deviations to the range [-10, 10].
The transformer is part of the model pipeline so that its statistics are learned from training data rather than the test period.
5. Labeling Strategy
Labels are derived from the CERT r4.2 incident evidence files.

The current rules distinguish positive evidence from ambiguous or contaminated user-days:
Label action	Rule	User-days
Positive	Insider user-day with at least one malicious non-HTTP event	521
Excluded	HTTP-only malicious user-day	445
Excluded	Victim account used by an insider	20


All 521 positive user-days were matched to the activity logs. The 465 excluded user-days are removed rather than treated as negative examples.

The resulting modeling table contains:
- 329,987 user-days.
- 521 positive user-days.
- 329,466 user-days without supplied positive evidence.
Label limitation: A user-day without supplied malicious evidence is treated as negative in this experiment, but it is not necessarily verified benign. The labels should therefore be interpreted as evidence-based and potentially incomplete.
6. Experimental Design
A chronological split is used to approximate the task of detecting future activity using past observations.
Split	Period	User-days	Positive user-days
Training	January–December 2010	246,074	391
Testing	January–May 2011	83,913	130


The training cutoff is December 31, 2010.

A separate temporal validation split is created inside the training period:
- Model fitting: January–September 2010.
- Validation: October–December 2010.
The 2011 test set is not used for hyperparameter selection.

Random seed: 42
7. Phase 1 Baseline Model
The baseline classifier is Logistic Regression.
Pipeline
User-day behavioral features
          |
          v
     UserBaseline
          |
          v
    StandardScaler
          |
          v
 Logistic Regression
          |
          v
  Insider risk score
Configuration
- Classifier: Logistic Regression.
- Class weighting: balanced.
- Maximum iterations: 2,000.
- Random state: 42.
- Tuned parameter: regularization strength C.
The selected value is C = 0.01, chosen using validation PR-AUC.
Temporal validation results
C	Validation PR-AUC	Recall@1%
0.01	0.1309	0.467
0.1	0.1198	0.473
1.0	0.1183	0.467
10.0	0.1180	0.467


8. Baseline Test Results
The selected pipeline was refitted on all training data and evaluated on the untouched 2011 test period.
Overall ranking performance
Measure	Result
Test PR-AUC	0.0493
Test positive prevalence	Approximately 0.00155
Test user-days	83,913
Test positive user-days	130
Maximum PR-AUC across 8 shuffled-label runs	0.027


The real model's PR-AUC exceeded the scores from the eight shuffled-label experiments. This is a useful sanity check, but eight permutations are not a formal statistical significance test.
Operating point: threshold 0.5
Measure	Result
True positives	105
False positives	4,046
True negatives	79,737
False negatives	25
Precision	0.025
Recall	0.808
F1-score	0.049


Operating point: top 1% review budget
Measure	Result
User-days selected for review	840
True positives	47
False positives	793
True negatives	82,990
False negatives	83
Precision	0.056
Recall	0.362
F1-score	0.097
Insiders detected at least once	13 of 18


The top-1% operating point is more closely aligned with the intended SOC workflow because it limits the number of records sent to analysts. It has lower recall than the 0.5 threshold, but it reduces the number of investigations substantially.
Illustrative analyst workload
Using an assumption of 15 minutes per review:
- Reviewing all 83,913 test user-days would take approximately 20,978 analyst hours.
- Reviewing the top 840 user-days would take approximately 210 analyst hours.
- The difference is approximately 20,768 hours.
These are illustrative calculations based on a fixed review-time assumption, not measured savings or a validated operational forecast. The model found 47 of the 130 positive user-days within the selected watchlist.
9. Leakage and Reliability Checks
The notebook includes checks confirming that:
- Training ends before testing begins.
- Validation is inside the training period and precedes testing.
- No (user, day) record appears in both train and test.
- Labels and evidence-rule columns are not model features.
- User ID and date are not passed directly to the classifier.
- User baseline statistics are learned only from training data.
- Positive training days are excluded from user-baseline estimation.
- The scaler is fitted only on training rows.
- The real model outperformed all eight shuffled-label runs in the current experiment.
These checks passed in the current baseline run.
10. Limitations, Privacy, and Ethics
Technical limitations
1. HTTP not included: Web browsing activity is part of CERT r4.2 but is not used in the current feature table.
2. One dataset: LANL has not been added; the current results are based on CERT r4.2 only.
3. Severe class imbalance: Positive user-days represent approximately 0.158% of the labeled modeling table.
4. Incomplete labels: Absence of incident evidence does not establish that an activity day is benign.
5. Limited overlap of incidents across time: Only four insiders have positive user-days on both sides of the current temporal cutoff. This makes generalization to previously unseen insider incidents an important limitation.
6. Synthetic data: CERT is a research dataset, and results may not transfer directly to real organizations.
7. Daily aggregation: Daily counts may hide event order and short sequences of suspicious actions.
Privacy and ethical considerations
Employee behavior analytics can affect privacy, workplace trust, and reputation. A model score is an indicator for review, not proof of malicious intent.

A responsible deployment should include:
- Human investigation before taking action.
- Data minimization and purpose limitation.
- Role-based access to employee activity data.
- Secure storage, retention limits, and audit logging.
- Documented policies and appropriate legal/compliance review.
- Monitoring of false positives and disparate operational impact.
- A process for correcting inaccurate data and reviewing decisions.
This project is an academic prototype and is not intended to make automated disciplinary or employment decisions.
11. Phase 2 Development Plan
Phase 2 will extend the baseline and address the project brief's comparison of supervised and unsupervised approaches.

Planned work:
1. Isolation Forest: Generate unsupervised anomaly scores for user-day records.
2. Random Forest: Train a supervised classifier using the evidence-based labels.
3. Local Outlier Factor: Add an anomaly-detection comparison where the training and scoring setup is appropriate.
4. Model comparison: Compare supervised and unsupervised approaches using a consistent temporal evaluation and investigation budget.
5. Ranked watchlist: Produce a prioritized list of user-days for analyst review.
6. Explainability: Use SHAP or an appropriate explanation method to identify reasons behind selected user-level alerts and false positives.
7. False-positive impact: Quantify alert burden and discuss the potential effect of unnecessary employee scrutiny.
8. Robustness and drift: Test sensitivity to changes in activity patterns and time periods.
Stretch objective
Develop peer-group baselines so that employee behavior can be compared with others in similar organizational roles, where reliable role information is available.

No Phase 2 model results are claimed until these experiments have been implemented and evaluated.
12. Technologies
- Python
- Google Colab
- Google Drive
- Jupyter Notebook
- Pandas
- NumPy
- Matplotlib
- Scikit-learn
- GitHub
13. How to Run
Requirements
- A Google account with access to Google Colab.
- Access to the CMU CERT r4.2 dataset.
- The project notebook from this repository.
Step 1 — Open the notebook
Open notebooks/01_eda_baseline.ipynb in GitHub and choose Open in Colab.
Step 2 — Prepare Google Drive
Place the dataset in Google Drive using the expected structure:
MyDrive/
└── cert_r42/
    ├── r4.2/
    │   ├── logon.csv
    │   ├── device.csv
    │   ├── file.csv
    │   ├── email.csv
    │   ├── license.txt
    │   └── LDAP/
    ├── answers/
    │   ├── insiders.csv
    │   └── r4.2-*/
    └── outputs/
The notebook expects the root directory to be:
DATA_DIR = "/content/drive/MyDrive/cert_r42"
Step 3 — Run the notebook
Run the notebook cells in order. The notebook mounts Google Drive, loads the logs, constructs the user-day feature table, applies evidence-based labels, creates temporal splits, trains the Logistic Regression baseline, and evaluates it.
14. Project Status
Component	Status
CERT r4.2 data preparation	Completed
Exploratory data analysis	Completed
User-day feature engineering	Completed
Evidence-based labeling	Completed
Temporal train/test split	Completed
Logistic Regression baseline	Completed
Baseline evaluation and confusion matrices	Completed
Leakage and shuffled-label checks	Completed
Named customer and market-case framing	Documented as a hypothetical use case
Dataset license review	Pending verification of license.txt
Isolation Forest	Planned
Random Forest	Planned
Local Outlier Factor	Planned
SHAP/LIME explanations	Planned
False-positive impact analysis	Planned
Robustness and drift testing	Planned
Peer-group baselines	Stretch objective


Project: Project 8 — Insider Threat and User Behavior Analytics from Logs
Dataset: CMU CERT Insider Threat Dataset r4.2
Environment: Google Colab
Repository: GitHub
