# Explainable AI-Based Network Intrusion Detection System (NIDS) using XGBoost

## Project Overview

A Machine Learning-based **Network Intrusion Detection System** that classifies network traffic into 15 categories — including various cyberattacks and benign traffic — using **XGBoost** with **SHAP-based Explainable AI** for model interpretability.

---

## Dataset — CICIDS2017

**Source:** Canadian Institute for Cybersecurity  
**Total Samples:** 2,830,743 rows × 79 columns (after merging 8 CSV files)  
**Features:** 78 network traffic flow features  
**Target:** `Label` (15 classes)

### Class Distribution

| Class | Count |
|---|---|
| BENIGN | 2,271,320 |
| DoS Hulk | 230,124 |
| PortScan | 158,804 |
| DDoS | 128,025 |
| DoS GoldenEye | 10,293 |
| FTP-Patator | 7,935 |
| SSH-Patator | 5,897 |
| DoS slowloris | 5,796 |
| DoS Slowhttptest | 5,499 |
| Bot | 1,956 |
| Web Attack – Brute Force | 1,507 |
| Web Attack – XSS | 652 |
| Infiltration | 36 |
| Web Attack – SQL Injection | 21 |
| Heartbleed | 11 |

> The dataset is **severely imbalanced** — BENIGN traffic accounts for ~80% of samples while rare attack classes like Heartbleed have only 11 samples.

---

## Pipeline

```
8 CSV Files
    → Merge (pd.concat)
    → Clean (remove inf/NaN, strip column names)
    → Label Encode
    → Stratified Train-Test Split (70:30)
    → XGBoost (Baseline)
    → XGBoost (Class-Weighted)
    → SHAP Explainability
```

---

## Models

### Baseline XGBoost (No imbalance handling)
- `objective='multi:softprob'`
- `tree_method='hist'` (histogram-based, fast on large data)
- No class weighting

### Weighted XGBoost
- Same architecture
- `compute_sample_weight(class_weight='balanced')` applied to training
- Improves recall on minority attack classes

---

## Results

### Baseline Model

| Metric | Score |
|---|---|
| Accuracy | ~1.00 |
| Weighted F1 | ~1.00 |
| **Macro F1** | **0.87** |
| **Macro Recall** | **0.85** |

### Weighted Model (Imbalance-Handled)

| Metric | Score |
|---|---|
| Accuracy | ~1.00 |
| Weighted F1 | ~1.00 |
| **Macro F1** | **0.89** |
| **Macro Recall** | **0.91** |

### Key Observations

- High-volume classes (BENIGN, DoS Hulk, PortScan, DDoS) achieve near-perfect precision and recall in both models
- Weighted model significantly improves recall on rare classes:
  - Heartbleed: recall improved from **0.45 → 1.00**
  - Bot: recall improved from **0.76 → 0.99**
- Macro Recall improved from **0.85 → 0.91** with sample weighting
- Rare classes (SQL Injection: 21 samples, Infiltration: 36 samples) remain challenging

---

## Explainable AI — SHAP Findings

SHAP (SHapley Additive exPlanations) was applied to the weighted XGBoost model to explain predictions globally and at the individual sample level across all 15 classes.

### Global Feature Importance

**Destination Port** is by far the most influential feature, contributing the highest mean absolute SHAP value across all classes. This makes intuitive sense — specific ports are strongly associated with specific attack types (e.g., port scanning, FTP/SSH brute force). The top 5 features globally:

1. **Destination Port** — dominant discriminator across nearly all attack classes
2. **Init_Win_bytes_backward** — TCP window size in the backward direction; distinguishes normal vs. attack handshake patterns
3. **Init_Win_bytes_forward** — forward TCP window initialization; low or zero values often indicate DoS/scanning behavior
4. **min_seg_size_forward** — minimum segment size in forward direction; abnormally small values indicate crafted/malicious packets
5. **Bwd Packets/s** — backward packet rate; high rates indicate flood-based attacks (DoS, DDoS)

Timing features (Fwd IAT Min, Flow IAT Min, Flow IAT Mean) and throughput features (Flow Bytes/s, Bwd Packet Length Max) also ranked in the top 20, reflecting that **both packet structure and timing behavior** are critical signals for intrusion detection.

### Individual Prediction — Waterfall Plot

For a single test sample, the model started from a base value of 0.5 and arrived at a final prediction score of -20.151, strongly ruling out a particular class. The top contributors driving this decision:

- **Fwd Packet Length Min = 34** → SHAP: -5.44 (strongest single feature)
- **Bwd Packet Length Min = 50** → SHAP: -4.83
- **Flow Duration = 348,095** → SHAP: -4.60
- **Fwd Packet Length Std = 0** → SHAP: -3.40

All top features pushed the prediction negatively, indicating the model was highly confident this sample did **not** belong to the target class. Zero standard deviation in forward packet lengths (all packets same size) combined with specific min packet lengths is a strong signature the model learned to associate with particular traffic types.

### Dependence Plot — Flow Duration

The SHAP dependence plot for **Flow Duration** reveals:

- **Short-duration flows (near 0)** show the widest variance in SHAP impact — both strongly positive and strongly negative — indicating they are the most ambiguous and class-discriminative region. Very short flows are associated with scanning and DoS attacks but also legitimate quick connections.
- **Long-duration flows** stabilize toward near-zero SHAP contribution, meaning the model becomes less reliant on flow duration as a signal for longer connections.
- The interaction variable `Init_Win_bytes_forward` (color axis) shows that **high window sizes (pink, ~25000) combined with short durations** push SHAP values higher, while **low window sizes (blue, ~0) with short durations** push SHAP values lower — revealing a meaningful interaction between TCP handshake behavior and flow duration in distinguishing attack types.

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Python | Core language |
| Pandas, NumPy | Data processing |
| Scikit-learn | Preprocessing, metrics, sample weights |
| XGBoost | Classification model |
| SHAP | Model explainability |
| Matplotlib, Seaborn | Visualization |
| Jupyter Notebook | Development environment |

---

## Author

**Akula Monish Ram**  
B.Tech CSE — Lovely Professional University  
Minor in AI — IIT Ropar
