# Enterprise Network Intrusion Detection Using Cost-Sensitive Machine Learning

**Team:** codeofduty  
**Model:** LightGBM Binary Classifier  
**Domain:** Cybersecurity · Machine Learning · Anomaly/Intrusion Detection

## Overview

Built a machine-learning-based network intrusion detection system (IDS) to classify network traffic as benign or malicious. The project focuses on a practical security challenge: **missing an attack is 40 times more costly than generating a false alarm**.

Rather than optimizing accuracy alone, I compared model configurations, evaluated precision–recall trade-offs, and selected a decision threshold using an explicit cost function.

## Key Results

Evaluated on a held-out validation set containing 381,007 records.

| Metric | F1-oriented threshold | Cost-sensitive threshold |
|---|---:|---:|
| Decision threshold | 0.50 | 0.02 |
| F1-score | **98.10%** | 96.36% |
| PR-AUC | **99.90%** | 99.90% |
| Precision | 98.42% | 92.99% |
| Recall | 97.78% | **99.984%** |
| False positives | 673 | 3,320 |
| False negatives | 1,081 | **7** |
| Cost | 43,913 | **3,600** |

Cost is calculated as:

`Cost = 40 × False Negatives + False Positives`

The cost-sensitive threshold reduced the validation cost substantially by prioritizing detection recall, at the expense of more false alarms.

## Architecture

```text
Network Traffic Records
          |
          v
Feature Validation
(38 network-flow features)
          |
          v
Preprocessing
- Categorical feature handling
- Missing-value cleanup
          |
          v
Tuned LightGBM Classifier
          |
          v
Attack Probability Scores
          |
          v
Cost-Sensitive Threshold (0.02)
          |
          v
Binary Predictions + CSV Output
```

## Technical Implementation

- Trained a **LightGBM binary classifier** on approximately 1.52 million training records using 38 network-flow features.
- Tuned tree complexity, learning rate, estimator count, sampling, and regularization parameters.
- Handled categorical features including `proto`, `state`, and `service`.
- Cleaned the `ct_ftp_cmd` feature by converting blank strings to missing values and coercing values to numeric.
- Serialized the trained model using **Joblib** and created a command-line inference script for reproducible predictions.

## Model Development and Trade-offs

### 1. Baseline model

Established an initial LightGBM baseline and evaluated it using precision, recall, F1-score, ROC-AUC, and PR-AUC.

### 2. Feature engineering experiment

Added derived ratios and logarithmic transformations for selected traffic statistics, including byte, packet, load, and loss features.

**Why it was not retained:** The engineered variant achieved approximately 98.02% F1, but its PR-AUC decreased to approximately 99.72%. Since it did not improve the overall evaluation results, the original feature set was retained.

### 3. Hyperparameter tuning

Selected a tuned LightGBM configuration with 600 estimators, a 0.03 learning rate, 63 leaves, and regularization. It achieved **98.10% validation F1 and 99.90% PR-AUC**.

### 4. Feature ablation

Evaluated a reduced model using the five most important features. Its F1-score and PR-AUC were lower than those of the full model, so all 38 features were retained.

## Cost-Sensitive Decision Making

The central design decision was separating **probability estimation** from the **classification threshold**.

A threshold of 0.50 gave the best F1-score among the tested thresholds. However, it produced 1,081 false negatives on validation data.

Because missed attacks carry a 40× penalty, threshold 0.02 was selected for cost-sensitive operation. This reduced false negatives to **7**, increasing recall to **99.984%**, while increasing false positives to 3,320.

This illustrates a key production ML trade-off: the best threshold depends on the real-world cost of different types of mistakes, not just a single metric.

## Distribution Shift and Failure Analysis

The unlabeled challenge dataset showed a different protocol distribution from the training data, including a broader range of protocol categories.

The model predicted attacks for **15.46% of challenge records** at threshold 0.02. This is a prediction rate, not a measured detection rate.

### Limitations identified

- **Unlabeled challenge data:** True challenge F1, recall, and protocol-specific detection performance could not be calculated.
- **Distribution shift:** Changes in protocol, state, and service distributions may reduce reliability on unfamiliar traffic.
- **Unseen categories:** Novel categorical values may be treated as missing during inference.
- **Threshold dependence:** The selected threshold is optimized for the validation data and specified cost function; production traffic may require a different threshold.
- **Operational false alarms:** High recall increases alert volume, which can create additional analyst workload.
- **Unverified zero-day detection:** The project does not establish that previously unseen attacks are detected reliably without labeled evaluation data.

### Future improvements

- Evaluate on a labeled, time-separated test set.
- Measure precision, recall, and cost across individual attack families and protocols.
- Monitor feature drift, unseen categories, and alert volume.
- Compare threshold tuning with class weighting and cost-sensitive training objectives.
- Calibrate probabilities and validate the decision threshold on representative deployment traffic.

## Reproducible Inference

### Project files

- `model.joblib` — serialized trained model.
- `predict.py` — command-line inference script.
- `requirements.txt` — dependencies.
- `metadata.json` — model configuration, feature order, and threshold.

### Run locally

The input CSV must be headerless and contain exactly 38 feature columns in the expected competition order.

```bash
pip install -r requirements.txt
python predict.py --input test.csv --output pred.csv
```

Output format:

```csv
row_id,prediction,probability
1,0,0.001234
2,1,0.084321
```

The inference script uses the 0.02 cost-sensitive threshold. The package was smoke-tested on synthetic input to verify that inference runs and produces the expected output structure. This does not replace the organizers' hidden-data reproducibility test.

## Technology Stack

- **Language:** Python
- **ML framework:** LightGBM
- **Data processing:** Pandas, NumPy
- **Model serialization:** Joblib
- **Environment:** Google Colab

## Team

**codeofduty**

## Disclaimer

This project is a hackathon prototype. Further validation, monitoring, and testing on representative labeled traffic are necessary before using it as a production security control.
