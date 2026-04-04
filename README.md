# Topology-Guided 2.5D Ensemble Fusion for Kidney Tumor Segmentation

A resource-efficient two-stage cascade architecture for automated kidney tumor segmentation from contrast-enhanced CT volumes, achieving competitive performance on the KiTS19 benchmark using only **8 GB consumer-grade GPU memory**.

## Overview

Accurate kidney tumor segmentation is critical for surgical planning in renal cell carcinoma. While 3D deep learning methods achieve high accuracy, they require 32+ GB GPU memory, limiting accessibility. This work proposes a 2.5D approach that achieves **93% of 3D reference performance at 25% of the computational cost**.

### Pipeline

```
CT Volume → 3D Kidney Localization (nnU-Net) → ROI Extraction → 
  ├── V5: Baseline (EfficientNet-B4 + UNet++ + ASPP + CBAM + EDL)
  ├── V6: + SliceAttentionGate + TumorAwareHead
  └── V7: + scSE + Focal Tversky + Curriculum Learning
→ Topology-Guided Ensemble Fusion → Post-Processing → Segmentation + Uncertainty Map
```

## Key Contributions

- **SliceAttentionGate**: Learnable module that adaptively weights each contextual slice based on diagnostic relevance
- **TumorAwareHead**: Auxiliary branch enforcing explicit tumor encoding at the deepest representation level
- **Topology-Guided Ensemble Fusion**: Uses high-precision predictions as spatial anchors, allowing expansion only where anatomical connectivity exists
- **Dual-Level Uncertainty**: Voxel-level evidential calibration (ECE: 0.012) + case-level reliability via inter-model agreement

## Results on KiTS19 (42 held-out validation cases)

| Metric | Score |
|--------|-------|
| Kidney Dice | 0.964 |
| Tumor Dice (Mean) | 0.792 |
| Tumor Dice (Median) | 0.870 |
| Tumor HD95 | 5.32 mm |
| Tumor ASSD | 1.01 mm |
| ECE (Calibration) | 0.012 |

### Comparison with Published Methods

| Method | Year | Type | GPU Memory | Tumor Dice |
|--------|------|------|-----------|------------|
| Prior 2.5D baseline | 2019 | 2.5D | 11 GB | 0.740 |
| **Proposed (Ours)** | **2025** | **2.5D** | **8 GB** | **0.792** |
| MSS 3D U-Net | 2019 | 3D | High | 0.805 |
| 3D Reference (nnU-Net) | 2021 | 3D | ≥32 GB | 0.851 |

## Architecture

**Stage 1:** 3D nnU-Net for kidney localization → binary kidney mask → ROI extraction with 10-voxel padding

**Stage 2:** Three architecturally diverse 2.5D networks:
- **Encoder:** EfficientNet-B4 (ImageNet pretrained, modified for 7-channel 2.5D input)
- **Decoder:** UNet++ with dense skip connections
- **Bottleneck:** ASPP (dilation rates: 1, 6, 12, 18) + CBAM attention
- **Uncertainty:** Evidential Deep Learning (Dirichlet prior)

## Tech Stack

- **Framework:** PyTorch 2.x
- **Segmentation:** segmentation-models-pytorch, nnU-Net
- **Hardware:** NVIDIA RTX 4060 (8 GB VRAM), 16 GB RAM
- **Training Time:** ~14 hours per variant (~42 hours total)
- **Dataset:** KiTS19 (208 cases, 80/20 stratified split)

## Project Structure

```
├── NB01_data_exploration.ipynb           # Dataset loading, voxel spacing analysis, HU clipping, normalization
├── NB07a_stratified_split.ipynb          # 80/20 train/val split stratified by tumor volume difficulty
├── NB07b_nnunet_retrain_80_20.ipynb      # Stage 1: Retrain nnU-Net kidney localizer on 166 training cases
├── NB07c_roi_extraction_80_20.ipynb      # ROI extraction from nnU-Net predictions (anisotropic dilation)
├── NB07d_v5_baseline_80_20.ipynb         # Stage 2: V5 baseline (ENet-B4 + UNet++ + ASPP + CBAM + EDL)
├── NB07e_v6_slicegate_tumorhead.ipynb    # Stage 2: V6 + SliceAttentionGate + TumorAwareHead
├── NB07f_v7_scse_focaltversky.ipynb      # Stage 2: V7 + scSE decoder + Focal Tversky + Curriculum Learning
├── NB05h_final_evaluation.ipynb          # Topology-guided ensemble fusion + full evaluation (Dice, HD95, ASSD, ECE)
└── README.md
```

### Pipeline Order

| Step | Notebook | Purpose |
|------|----------|---------|
| 1 | `NB01` | Explore dataset, compute preprocessing statistics |
| 2 | `NB07a` | Create stratified 80/20 split (208 → 166 train / 42 val) |
| 3 | `NB07b` | Train nnU-Net Stage 1 kidney localizer |
| 4 | `NB07c` | Extract kidney ROIs using nnU-Net predictions |
| 5 | `NB07d` | Train V5 baseline segmentation model |
| 6 | `NB07e` | Train V6 with attention mechanisms |
| 7 | `NB07f` | Train V7 with recall-biased loss and curriculum |
| 8 | `NB05h` | Ensemble fusion, evaluation, uncertainty analysis, figures |

> **Note:** Model weights (.pth) and dataset files (.nii.gz) are not included due to size constraints.  
> Download KiTS19 dataset from: https://github.com/neheller/kits19

## Training Configuration

| Parameter | Value |
|-----------|-------|
| Optimizer | AdamW (lr=1e-4, weight decay=1e-4) |
| LR Schedule | Cosine annealing (T_max=100) |
| Epochs | 100 per variant |
| Batch Size | 8 |
| Input Size | 7 × 256 × 256 (2.5D) |
| Fusion Channels | 48 |
| Augmentation | Random flip, rotation (±15°), scale (0.9–1.1) |

## Clinical Deployment

An interactive clinical dashboard built with **Gradio** provides:
- Segmentation overlay with adjustable opacity
- Confidence heatmap from evidential uncertainty
- CBAM spatial attention visualization
- Clinical advisory panel with descriptive findings

The entire system processes a case in under 3 minutes with TTA on consumer-grade hardware.

## Citation

```
Vejeya Prasad C, Ramya S. "Topology-Guided 2.5D Ensemble Fusion with 
Calibrated Uncertainty for Kidney Tumor Segmentation." 
SASTRA Deemed University, 2025. (Manuscript under review)
```

## License

This project is part of an M.Tech thesis at SASTRA Deemed University, Thanjavur, India.
