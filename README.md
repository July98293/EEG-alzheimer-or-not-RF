# EEG Alzheimer Classification — Random Forest on ds004504

Binary classification of **Alzheimer's Disease (AD) vs Healthy Controls (CN)** from resting-state EEG, using an full ERP-style preprocessing pipeline and band power features fed into a Random Forest classifier.

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1D0vBMbBruIPXp26kSDHao7Sq7Qb39vE9?usp=sharing)
![Python](https://img.shields.io/badge/python-3.10%2B-blue)
![License](https://img.shields.io/badge/license-MIT-green)

---

## Results

| Metric | Mean ± SD (5-fold CV) |
|---|---|
| Accuracy | 0.72 ± 0.08 |
| AUC-ROC | **0.85 ± 0.09** |
| F1 | 0.75 ± 0.07 |
| Precision | 0.76 ± 0.08 |
| Recall | 0.75 ± 0.13 |

### Ablation Study

Testing which feature groups actually matter:

| Features | AUC |
|---|---|
| Relative band power only | 0.833 ± 0.095 |
| Absolute band power only | 0.767 ± 0.093 |
| Both band powers | 0.824 ± 0.101 |
| ERP features only (amplitude, latency, SEF, Hjorth) | 0.706 ± 0.030 |
| All features without Hjorth | 0.848 ± 0.103 |
| **All features (full model)** | **0.857 ± 0.087** |

**Key finding:** relative band power alone (AUC 0.833) captures most of the discriminative signal. ERP-style features contribute marginally. The model is robust and not dependent on any single feature group.

---

## What this project does

Alzheimer's Disease produces a well-documented slowing of resting-state EEG, alpha power decreases, theta and delta power increase. This is consistent with cholinergic deficit and progressive cortical disconnection.

This pipeline operationalizes that biological signature into a machine learning classifier:

1. Downloads 65 subjects (36 AD, 29 CN) from OpenNeuro ds004504
2. Runs a full clinical-grade EEG preprocessing pipeline
3. Extracts band power and ERP-style features per subject
4. Trains a Random Forest and evaluates via stratified 5-fold CV

---

## Pipeline

```
Raw EEG (.set) ,  88 subjects, 19 channels, resting-state eyes-closed
        │
        ├─ 1.  Channel location       standard 10-20 montage
        ├─ 2.  Re-referencing         average reference
        ├─ 3.  Filtering              bandpass 1–100 Hz + notch 50 Hz
        ├─ 4.  Bad channel removal    MAD z-score > 5 → spherical interpolation
        │      Bad block removal      RMS z-score > 5 → annotate BAD_block
        │
        ├─ 5.  ICA decomposition      FastICA, 15 components
        ├─ 6.  Component removal      ocular (Fp1/Fp2 correlation). Almost all where ocular
        │                             cardiac (kurtosis + autocorrelation)
        │                             muscle (HF/LF power ratio)
        │
        ├─ 7.  Epoching               fixed 4-sec, skip BAD_block segments
        ├─ 8.  Epoch rejection        peak-to-peak > 150 µV dropped
        ├─ 9.  Bin assignment         label each epoch AD=1 / CN=0
        │
        ├─ 10. Epoch average          mean across valid epochs per subject
        ├─ 11. Feature extraction     band power (δ θ α β γ) absolute + relative
        │                             amplitude: mean, std, peak-to-peak
        │                             latency: alpha envelope peak (ms)
        │                             spectral: peak frequency, SEF95
        │                             complexity: Hjorth mobility + complexity
        │                             connectivity: mean alpha coherence
        │
        └─ 12. Random Forest          500 trees, balanced weights, 5-fold CV
```

---

## Feature extraction — band power

For each of the 19 channels, Welch PSD computed over 2-second windows. Power integrated over five canonical bands:

| Band | Range | AD vs CN |
|---|---|---|
| Delta (δ) | 0.5–4 Hz | ↑ in AD |
| Theta (θ) | 4–8 Hz | ↑ in AD |
| Alpha (α) | 8–13 Hz | ↓ in AD |
| Beta (β) | 13–30 Hz | ≈ |
| Gamma (γ) | 30–45 Hz | ≈ |

Both log-absolute and relative power extracted → **~290-dimensional feature vector** per subject.

---

## Repo structure

```
eeg-alzheimer-rf/
├── notebooks/
│   └── EEG_Alzheimer_Full_Pipeline.ipynb   ← full Colab pipeline
├── results/
│   ├── band_power.png
│   ├── erp_features.png
│   └── results_summary.png
└── README.md
```

---

## Quickstart

### Run in Colab (recommended)

Click the badge at the top. No setup needed.

### Run locally

```bash
# 1. Install
pip install -r requirements.txt

# 2. Download dataset (~2 GB, no account needed)
aws s3 sync --no-sign-request s3://openneuro.org/ds004504 ./ds004504 \
    --exclude "*" \
    --include "sub-*/eeg/*.set" \
    --include "participants.tsv"

# 3. Run
python src/pipeline.py

# 4. Test
pytest tests/
```

---

## Dataset

**OpenNeuro ds004504** — Miltiadous et al., *Data*, 2023

- 88 subjects: 36 Alzheimer's Disease, 29 Healthy Controls, 23 Frontotemporal Dementia
- 19 EEG channels, standard 10-20 system
- Resting-state eyes-closed, ~13 minutes per subject
- License: CC0 (public domain)

This project uses only AD and CN subjects (65 total). FTD subjects are excluded for binary classification.

---

## Citation

Please cite the original dataset:

```bibtex
@article{miltiadous2023dataset,
  title={A Dataset of Scalp EEG Recordings of Alzheimer's Disease,
         Frontotemporal Dementia and Healthy Subjects from Routine EEG},
  author={Miltiadous, Andreas and others},
  journal={Data},
  volume={8},
  number={6},
  pages={95},
  year={2023},
  doi={10.3390/data8060095}
}
```


MIT — see `LICENSE`.
