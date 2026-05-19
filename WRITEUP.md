# 30-day readmission modeling exercise and wanted to summarize my approach, results, and recommendations.

## 1. Approach
### How did you approach the problem, and why?

I treated this as a supervised binary classification problem: for each in-scope admitted patient, predict whether they will be readmitted within 30 days. I aligned the feature generation to the intended workflow: Case Management meets at 7AM, so the model is designed to run at 6AM using only information available before that prediction time.

The raw data included 5,583 visits and 743,097 vital sign records. After restricting to visits that were eligible for a first 6AM prediction and excluding patients who died during the index admission, the final modeling cohort contained 5,297 encounters with an overall 30-day readmission rate of 11.0%.

To reduce leakage risk, I only used pre-prediction information. I removed direct identifiers, raw timestamps, length of stay, post-prediction information, duplicate target fields, and other outcome-definition artifacts. I also used a patient-level train/test split so the same patient could not appear in both train and test; the final split had zero patient overlap.

Feature engineering focused on clinically plausible signals available before 6AM, including:

* Demographics and admission timing features.
* Prior utilization and prior readmission history.
* Vital sign summaries for heart rate, blood pressure, temperature, MAP, pulse pressure, and shock index.
* Vital sign variability features such as standard deviation, range, IQR, coefficient of variation, and quantiles.
* Short-term trajectory features such as slopes and first-half vs. second-half changes in the pre-prediction window.

The exploratory analysis suggested that physiologic variability was more predictive than single median vital sign values. For example, heart rate variability, temperature variability, shock index variability, and diastolic blood pressure features showed stronger separation between readmitted and non-readmitted encounters.

Because readmissions were the minority class, I compared models with imbalance-aware approaches. Logistic regression used balanced class weights, Random Forest used balanced subsampling, and XGBoost used a `scale_pos_weight` based on the training class ratio. I prioritized AUPRC and F1 Score in tuning because precision-recall evaluation is especially useful when the positive class is relatively uncommon. 

## 2. Model performance
### What was the model performance?

The final selected model was a reduced 25-feature Random Forest at a 0.50 threshold. I reduced the initial 140+ feature set to a set of 25 features for ease of explainability and post-deployment maintenance. I selected this model because it preserved the performance of the larger models while reducing complexity and producing a practical alert volume.

On the held-out test set of 1,071 encounters, the final 25-feature Random Forest achieved:

| Metric               |                Result |
| -------------------- | --------------------: |
| AUROC                |                 0.993 |
| AUPRC                |                 0.957 |
| Sensitivity / Recall |                 90.8% |
| Specificity          |                 97.1% |
| PPV / Precision      |                 81.4% |
| NPV                  |                 98.7% |
| F1-score             |                 0.858 |
| True positives       |                   118 |
| False positives      |                    27 |
| True negatives       |                   914 |
| False negatives      |                    12 |
| Patients flagged     | 145 / 1,071, or 13.5% |

I also compared this with tuned XGBoost and the full Random Forest. Tuned XGBoost had slightly higher AUPRC and recall, but it generated many more false positives. At the 0.50 threshold, XGBoost captured 11 additional readmissions compared with the full Random Forest, but required 53 additional unnecessary interventions. This creates a business tradeoff: XGBoost is preferable only if the cost of missing one readmission is more than about 4.8 times the cost of one unnecessary intervention.

For this use case, I selected the Random Forest because it offered a stronger operational balance: high sensitivity, high PPV, high specificity, and a more manageable alert stream.

## 3. How I would present this to Case Management
### The next time you meet with Case Management, how would you present these results to a business audience? What would be your recommendations?
I would avoid leading with AUROC and instead frame the model in operational terms:

“Among 1,071 patients in the test set, the model would have flagged 145 patients for additional review. Of those flagged patients, 118 were true 30-day readmissions and 27 were not. The model missed 12 readmissions. In other words, at the selected threshold, about 81% of flagged patients were true readmissions, and the model captured about 91% of all readmissions.”

I would also explain that the model is not intended to replace clinical judgment. It should be used as a prioritization tool to help Case Management focus attention on patients who may benefit from additional discharge planning, follow-up scheduling, medication reconciliation, social support, or post-discharge outreach.

My recommendation would be to pilot the model in a silent-run or shadow mode before live deployment. During this period, Case Management could review the daily 6AM risk list, compare it with their current prioritization process, and give feedback on whether the flagged patients make clinical and operational sense. Since readmission reduction is tied to care coordination and hospital quality initiatives, this type of workflow-aligned intervention is consistent with broader hospital readmission reduction goals.

I would recommend starting with the 0.50 threshold because it produced a practical balance: it flagged 13.5% of patients, achieved 81.4% PPV, and captured 90.8% of readmissions. If Case Management has more capacity, we could lower the threshold to capture more readmissions. If alert burden is too high, we could raise the threshold to increase precision.

Note: In practice, I would have met with this team beforehand to understand business requirements around cost-benefit associated with intervention, readmission fines, etc.

## 4. Next steps to improve the model

The next development steps I would prioritize are:

1. Validate the model prospectively or temporally. The current results are very strong, so I would want to confirm that performance holds on a more realistic future time split.

2. Add richer clinical features. The current model is driven heavily by vitals. I would add diagnoses, medications, procedures, labs, comorbidities, discharge disposition, prior ED utilization, and markers of care complexity.

3. Add social risk and access-to-care variables if available. Race, ethnicity, insurance type, language, geography, social needs, and other SDoH variables could improve both performance and fairness monitoring.

4. Calibrate predicted probabilities. Before production, I would evaluate whether predicted probabilities correspond to observed readmission rates and apply calibration if needed.

5. Continue fairness evaluation. The current notebook evaluates performance by sex and found generally similar AUROC, AUPRC, sensitivity, specificity, and PPV across male and female patients. However, I would extend this to race, ethnicity, payer, and other protected or operationally important groups if available.

6. Optimize the threshold with Case Management capacity in mind. The “best” threshold depends on staffing, intervention cost, and the relative harm of missed readmissions vs. unnecessary intervention.

7. Monitor post-deployment drift. I would track data quality, missingness, feature distributions, flagged volume, PPV, sensitivity, calibration, and fairness metrics over time.

## 5. Case Management Explainability Plan

To make the model explainable for Case Management, I would use SHAP values to generate patient-level explanations. SHAP assigns feature contribution values for a specific prediction, which makes it useful for explaining why an individual patient was flagged.

In the notebook, SHAP analysis showed that the strongest model drivers were clinically plausible pre-prediction vital sign features, especially heart rate variability, heart rate extremes, temperature variability, minimum diastolic blood pressure, and shock index variability. This is encouraging because the model appears to rely on physiologic instability before the 6AM prediction time rather than leakage fields or identifiers.

For implementation, each flagged patient could have a short explanation displayed alongside the risk score, such as:

* “Flagged due to elevated temperature variability, abnormal heart rate range, and low minimum diastolic blood pressure.”
* “Risk increased by high shock index variability and recent prior utilization.”
* “Risk decreased by stable vitals and no recent prior readmission history.”

I would present explanations in plain language, grouped into clinical categories rather than raw model feature names. For example, instead of showing `temperature_cv` or `heart_rate_q90`, the UI could show “temperature variability” or “heart rate instability.” I would also include the most recent underlying vital values and trends so the care team can verify the explanation against the chart.

Overall, the model appears promising as a Case Management prioritization tool. I would recommend moving forward with a shadow pilot, workflow review, probability calibration, expanded fairness checks, and additional clinical feature integration before full production deployment.

### 5.1 Post-Deployment Explainability and Monitoring

For production deployment, explainability would be incorporated directly into the model serving and monitoring workflow using MLFlow (or similar.) In addition to versioning the trained models, MLflow would allow us to track model metadata, hyperparameters, evaluation metrics, feature importance outputs, and post-deployment monitoring artifacts in a centralized and reproducible manner.

To support clinical interpretability, patient-level SHAP explanations would be generated alongside each prediction. Rather than presenting only a raw risk score, the system could surface the top contributing factors that increased or decreased a patient’s predicted readmission risk, as mentioned in the Care Management Explainability Plan.

MLflow could also be used to operationalize ongoing model governance after deployment. This would include:

* Tracking model versions and promotion history.
* Logging production performance metrics such as PPV, sensitivity, specificity, and alert volume.
* Monitoring for feature drift or changes in patient population characteristics over time.
* Comparing model behavior across demographic groups to support fairness monitoring.
* Storing explainability artifacts and feature attribution summaries for auditing and retrospective review.

This type of explainability framework is especially important in healthcare settings because it increases clinician trust, supports model transparency, and enables continuous validation that the model is behaving as expected after deployment.

## 6. Deployment
Since HCA uses GCP, the model could be deployed as a daily batch inference workflow. I would collaborate with the MLOps team to get the trained model registered in Vertex AI Model Registry, with feature generation and scoring packaged into a Vertex AI Pipeline. A Cloud Scheduler job would trigger the pipeline every morning at 6AM, using the latest admitted-patient data from BigQuery or GCS. Vertex AI Batch Prediction can then write daily risk scores, flags, and explanation outputs back to BigQuery for Case Management review before their 7AM meeting. Vertex AI supports scheduled recurring pipeline runs, and batch prediction jobs can read from and write back to BigQuery or Cloud Storage.

## Now, Next, Later Vision for this Modeling Project
### Now (Pilot / Validation Phase)
The immediate next step would be a silent or shadow deployment. The model would run daily at 6AM and generate predictions for Case Management without yet driving clinical interventions. During this phase, the goals would be:

- Validate real-world performance prospectively.
- Measure operational alert volume.
- Gather clinician feedback on whether flagged patients are clinically appropriate.
- Monitor fairness, calibration, and data drift.
- Refine the intervention threshold based on staffing capacity and workflow needs.

This phase is primarily about proving the model is both clinically useful and operationally sustainable.

### Next (Operational Integration Phase)
Once validated prospectively, the model could move into an active workflow integration phase:

- Integrate predictions directly into the EHR or Case Management dashboard.
- Surface patient-level explanations and top contributing factors.
- Trigger targeted interventions for high-risk patients (follow-up scheduling, medication reconciliation, discharge planning, social work referral, etc.).
- Implement automated MLflow and Vertex AI monitoring for performance, drift, and retraining triggers.
- Expand features to include labs, diagnoses, medications, prior utilization, and SDoH variables.

At this stage, the focus shifts from “Can the model predict?” to “Can the model improve outcomes and reduce preventable readmissions?”

### Later (Enterprise AI / Optimization Phase)
Longer term, the model could evolve into a broader longitudinal patient-risk platform:

- Transition from daily batch inference to near real-time scoring.
- Move from binary readmission prediction to personalized risk trajectories over time.
- Develop intervention recommendation models (“which intervention is most likely to help this patient?”).
- Incorporate temporal deep learning architectures for sequential EHR modeling.
- Expand across additional hospital quality initiatives such as mortality, sepsis deterioration, ICU transfer risk, ED bounceback, or discharge readiness.
- Continuously retrain using new patient data and outcomes to maintain calibration and generalizability.

The long-term vision would be an enterprise clinical decision support ecosystem where predictive models proactively identify patients at risk and help care teams intervene earlier and more effectively.
