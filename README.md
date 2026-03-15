# IoT Botnet Detection using Network Traffic Classification

> A multi-class network intrusion detection system that classifies IoT device traffic as benign, Mirai malware, or Gafgyt (Bashlite) malware — trained on 7.5 million real network observations across 9 IoT device types, with recall as the primary evaluation criterion.

---

## Overview

IoT devices — thermostats, doorbells, baby monitors, security cameras — are increasingly weaponised by botnets to launch distributed denial-of-service attacks, exfiltrate data, or serve as entry points into broader network infrastructure. The challenge is not just detection accuracy, but detection speed and sensitivity: in a cybersecurity context, a missed attack (false negative) is categorically more costly than a false alarm.

This project builds a per-device botnet detection pipeline trained on the N-BaIoT dataset from the UCI Machine Learning Repository, covering real network traffic from 9 IoT devices under benign conditions and under active infection by two major malware families: Mirai and Gafgyt. Five classification models are benchmarked across all devices, with XGBoost selected as the final model following hyperparameter tuning via RandomizedSearchCV. Macro recall — the average recall across all three traffic classes — is the primary evaluation metric throughout.

---

## Dataset

**Source:** [N-BaIoT — UCI Machine Learning Repository](https://archive.ics.uci.edu/ml/datasets/detection_of_IoT_botnet_attacks_N_BaIoT)

The dataset captures network traffic statistics from 9 real IoT devices, each observed under three traffic conditions: normal (benign) operation, Mirai infection, and Gafgyt (Bashlite) infection. Combined across all devices, the dataset contains approximately **7,574,739 observations**.

**Devices covered:**

| Device | Type | Mirai Data |
|---|---|---|
| Danmini Doorbell | Doorbell | Yes |
| Ennio Doorbell | Doorbell | Yes |
| Ecobee Thermostat | Thermostat | Yes |
| Philips B120N10 Baby Monitor | Baby Monitor | Yes |
| Samsung SNH-1011N Webcam | Webcam | No |
| Provision PT-737E Security Camera | Security Camera | Yes |
| Provision PT-838 Security Camera | Security Camera | Yes |
| SimpleHome XCS7-1002WHT Security Camera | Security Camera | Yes |
| SimpleHome XCS7-1003WHT Security Camera | Security Camera | Yes |

**Traffic classes:**

| Class | Label | Description |
|---|---|---|
| 0 | Benign | Normal device network traffic |
| 1 | Mirai | Mirai botnet malware (SYN flood, UDP flood, ACK flood, scan, UDP plain) |
| 2 | Gafgyt | Gafgyt / Bashlite malware (TCP flood, UDP flood, scan, junk, combo) |

> Note: The Samsung SNH-1011N Webcam has no recorded Mirai attack data. For this device, classification is treated as a two-class problem (benign vs. Gafgyt).

**Class imbalance handling:** To prevent models from being biased toward majority classes, each device's training dataset was resampled to contain equal proportions of benign, Mirai, and Gafgyt traffic prior to model training.

---

## Methodology

### Data Preprocessing (`EDA.ipynb`)

- Loaded up to 15 CSV files per device (5 benign, 5 Mirai attack types, 5 Gafgyt attack types) into a unified dataframe
- Constructed a target variable `Class` (0 = benign, 1 = Mirai, 2 = Gafgyt) from directory-level labels
- Applied resampling to produce class-balanced training sets across all three traffic types
- Applied `StandardScaler` to all feature columns prior to model fitting, which proved critical — unscaled features produced near-zero recall on the first simple model

### Model Benchmarking (`Predict_IoT_Attacks.ipynb`)

Five models were trained and evaluated for each of the 9 IoT devices:

| Model | Notes |
|---|---|
| DummyClassifier | Majority-class baseline |
| Logistic Regression (FSM) | No train-test split, all defaults — confirms need for scaling and splitting |
| Logistic Regression | 70/30 train-test split, StandardScaler applied |
| K-Nearest Neighbours | Computationally expensive; lower accuracy than LR |
| Decision Tree | Fast; strong baseline |
| Random Forest | GridSearchCV tuning across depth and estimator count |
| XGBoost | RandomizedSearchCV hyperparameter tuning; selected as final model |

### Hyperparameter Tuning (XGBoost)

RandomizedSearchCV was run across the following parameter grid, optimising for macro recall:

```
learning_rate    : [0.05, 0.10, 0.15, 0.20, 0.25, 0.30]
max_depth        : [3, 4, 5, 6, 8, 10, 12, 15]
min_child_weight : [1, 3, 5, 7]
gamma            : [0.0, 0.1, 0.2, 0.3, 0.4]
colsample_bytree : [0.3, 0.4, 0.5, 0.7]
```

Best parameters selected: `learning_rate=0.05`, `max_depth=15`, `min_child_weight=1`, `gamma=0.4`, `colsample_bytree=0.7`

### Per-Device Model Serialisation

Each device's final trained model is pickled as a `.pkl` file, enabling inference without re-running the full training pipeline. The `IoT_Device` class in `src/model_prep.py` handles data loading, preprocessing, model fitting, and serialisation in a single instantiation.

---

## Results

XGBoost with hyperparameter tuning was the best-performing model across all 9 devices on both recall and accuracy.

**XGBoost (tuned) — Macro Recall across all 9 devices:**

| Metric | Value |
|---|---|
| Mean Macro Recall | 0.9997 |
| Minimum Macro Recall | 0.9990 |
| Maximum Macro Recall | 0.9998 |

**Random Forest (tuned) — Macro Recall across all 9 devices:**

| Metric | Value |
|---|---|
| Mean Macro Recall | 0.9996 |
| Minimum Macro Recall | 0.9990 |
| Maximum Macro Recall | 0.9998 |

XGBoost marginally outperformed Random Forest on macro recall while remaining more computationally efficient at inference time. Logistic Regression with StandardScaler also achieved strong results (~97% accuracy on Danmini Doorbell), confirming that the feature space is largely linearly separable after scaling.

---

## Limitations & Future Work

**Current Limitations:**

- Models are trained and evaluated per device — a single generalised model that works across all device types was not explored in this project
- The dataset was collected in a controlled lab environment; real-world traffic distributions may differ significantly, particularly for benign traffic
- The Samsung webcam has no Mirai data, which limits comparability of recall scores across devices
- Network traffic features are statistical aggregates (packet sizes, inter-arrival times) and do not include payload-level inspection, which could improve detection of novel variants

**Future Directions:**

- Train a unified cross-device model using device type as an additional categorical feature, enabling deployment as a single network-level classifier
- Evaluate model robustness against adversarial traffic crafted to mimic benign patterns
- Extend to real-time streaming inference using Kafka or similar pipeline, enabling detection latency benchmarking
- Explore deep learning approaches (autoencoders for anomaly detection, LSTM for sequential traffic modelling) as complements to the tree-based ensemble methods

---

## How to Run This Project

### Prerequisites

```bash
Python 3.8+
```

### 1. Clone the Repository

```bash
git clone https://github.com/d4h2nu8h/iot-botnet-detection.git
cd iot-botnet-detection
```

### 2. Install Dependencies

```bash
pip install -r requirements.txt
```

### 3. Download the Dataset

Download the N-BaIoT dataset from the [UCI Machine Learning Repository](https://archive.ics.uci.edu/ml/datasets/detection_of_IoT_botnet_attacks_N_BaIoT) and place the extracted folders in the `data/` directory, following the structure described in `Dataset.md`.

### 4. Run Exploratory Data Analysis

```bash
jupyter notebook EDA.ipynb
```

### 5. Train Models and Run Predictions

```bash
jupyter notebook Predict_IoT_Attacks.ipynb
```

Pre-trained model pickle files for all 9 devices are included in the repository. The notebook will load these automatically if they exist, skipping retraining.

---

## Tech Stack

| Category | Tools |
|---|---|
| Language | Python 3.8+ |
| Machine Learning | Scikit-learn, XGBoost |
| Data Processing | Pandas, NumPy |
| Visualisation | Matplotlib, Seaborn, Yellowbrick |
| Model Serialisation | Pickle |
| Notebook Environment | Jupyter Notebook |
| Dataset Source | UCI Machine Learning Repository |

---

## Author

**Dhanush Sambasivam**

[![GitHub](https://img.shields.io/badge/GitHub-d4h2nu8h-181717?style=flat&logo=github)](https://github.com/d4h2nu8h)

---

## License

This project is intended for academic and research purposes. The N-BaIoT dataset is publicly available via the UCI Machine Learning Repository.
