# White Blood Cell Classification Using Deep Learning
**Student: Yanir Karpis (318933371)**

---

## 1. Introduction

### 1.1 Background
White blood cell (WBC) identification is currently performed manually by specialists—a slow, expensive, and error-prone process. Accurate automated classification is critical for detecting infections, leukemia, and other medical conditions.

### 1.2 Problem Description
- **Input:** Microscopic JPG image of a blood cell
- **Output:** Classification into one of four classes: Neutrophil, Monocyte, Lymphocyte, or Eosinophil

### 1.3 Dataset & Preprocessing
The Blood Cell Images dataset from Kaggle (Paul Mooney, 2018) was used, containing ~12,500 balanced RGB images.

**Preprocessing:**
- Resized to 256px, center-cropped to 224×224 (standard for ResNet50)
- Normalization using ImageNet statistics: mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]
- **Data Augmentation (training only):** random horizontal/vertical flips, ±15° rotations, color jitter
- **Data Split:** 80% training (7,965), 20% validation (1,992), test set (2,487)
- Random seed: 42 for reproducibility

---

## 2. Related Work

| Paper | Year | Method | Accuracy |
|-------|------|--------|----------|
| Shahin et al. | 2019 | Modified VGG16 | 92% |
| Kutlu et al. | 2020 | Transfer Learning (AlexNet + CNNs) | 94% |
| Acevedo et al. | 2020 | Multi-method approach | Highlighted morphological similarity between Eosinophils and Neutrophils as main challenge |

---

## 3. Methods

### 3.1 Transfer Learning Justification
With only ~10k images, training a deep CNN from scratch would overfit. ResNet50, pretrained on 1M+ ImageNet images, already knows general features (edges, textures). We leverage this to adapt to medical imagery.

### 3.2 Architecture & Models Tested

**Model 1: Frozen Feature Extractor**
- Backbone: ResNet50 (all layers frozen)
- Custom head: Linear classifier (2048 → 4 classes)
- Rationale: Quick baseline with minimal training

**Model 2: Fine-tuned ResNet50**
- Frozen: Layers 0–2 (preserve general features)
- Unfrozen: Layer3 & Layer4 (learn cell-specific morphology)
- Custom head: Dropout(0.5) → Linear(2048 → 4)
- Rationale: Deeper layers adapt to domain-specific features while early layers remain general

### 3.3 Training Protocol

| Parameter | Model 1 | Model 2 | Rationale |
|-----------|---------|---------|-----------|
| **Optimizer** | Adam | Adam | Standard for deep learning |
| **Learning Rate** | 0.001 | 0.0001 | Lower for fine-tuning to preserve pretrained weights |
| **Batch Size** | 32 | 32 | Balance between memory and gradient stability |
| **Epochs** | 20 | 20 | With early stopping (patience=5) |
| **Loss Function** | Cross-Entropy | Cross-Entropy | Standard for multi-class classification |
| **Regularization** | None | Dropout(0.5) + Weight Decay(0.0001) | Combat overfitting in fine-tuned layers |
| **LR Scheduler** | None | ReduceLROnPlateau (factor=0.5, patience=3) | Reduce LR by 50% when validation loss plateaus for 3 epochs |

### 3.4 Hyperparameter Selection
Batch size (32), dropout (0.5), and learning rates were selected based on:
- Literature review of medical image classification
- Initial testing and validation curve stability
- Model 2's LR (0.0001) specifically chosen to avoid disrupting fine-tuned weights

---

## 4. Results and Analysis

### 4.1 Performance Summary

| Model | Test Accuracy | Macro F1 | Best Epoch |
|-------|---------------|----------|-----------|
| Model 1 (Frozen) | 65.50% | 0.6584 | Early stopping at epoch 9 |
| Model 2 (Fine-tuned) | 85.93% | 0.8641 | Early stopping at epoch 11 |
| **Improvement** | **+20.43%** | **+0.2057** | — |

### 4.2 Per-Class Performance (Model 2)

| Class | Precision | Recall | F1-Score | Confusion Issues |
|-------|-----------|--------|----------|------------------|
| Eosinophil | 0.84 | 0.83 | 0.83 | Confused with Neutrophil (granulocytes, similar morphology) |
| Lymphocyte | 0.98 | 0.99 | 0.98 | Highest F1—distinct large round nucleus is highly discriminative |
| Monocyte | 0.88 | 0.84 | 0.86 | Occasional confusion with Lymphocyte (both large nuclei) |
| Neutrophil | 0.75 | 0.78 | 0.76 | Most difficult; significant overlap with Eosinophil |

### 4.3 ROC Analysis (Model 2)

Per-class AUC (Area Under Curve):
- Eosinophil: 0.92
- Lymphocyte: 0.98
- Monocyte: 0.94
- Neutrophil: 0.90

**Interpretation:** All classes show strong separation (AUC > 0.90), indicating the model reliably discriminates across decision thresholds. Lymphocyte achieves near-perfect separation.

### 4.4 Learning Curves Analysis

**Model 1:**
- Training accuracy rapidly increases to ~95%
- Validation accuracy plateaus at ~65–67%
- **Diagnosis:** Severe overfitting; frozen features inadequate for cell morphology

**Model 2:**
- Smooth convergence: training ~99%, validation ~86%
- Dropout and scheduler prevent overfitting
- **Diagnosis:** Proper generalization; fine-tuning allows adaptation to cell-specific features

### 4.5 Why Fine-tuning Works
Model 2's 20% improvement stems from:
1. **Layer3 & Layer4 adaptation:** These mid-to-high-level layers learn cell-specific textures and nuclear shapes
2. **Regularization:** Dropout(0.5) prevents memorization
3. **Controlled learning:** Low LR (0.0001) prevents catastrophic forgetting of pretrained weights
4. **Scheduler:** Allows precise convergence when progress stalls

---

## 5. Discussion

### 5.1 Key Findings
Fine-tuning deeper layers of ResNet50 yielded a 20% accuracy boost over simple feature extraction. Medical images require model layers to adapt to domain-specific morphology beyond generic ImageNet features.

### 5.2 Medical Insights
The primary difficulty is **Eosinophil↔Neutrophil confusion** (both granulocytes with multi-lobed nuclei and granular textures). Lymphocytes are easiest to classify due to their distinct morphology.

### 5.3 Future Work
1. Explore Vision Transformers (ViT) for potentially better feature extraction
2. Apply Class Activation Maps (CAM) to visualize which cell regions drive classification
3. Test ensemble methods combining multiple architectures
4. Investigate data augmentation strategies specific to cell morphology

---

## 6. References

- Acevedo, A., Merino, A., Alferez, S., Molina, Á., Boldú, L., & Rodellar, J. (2020). A dataset of microscopic peripheral blood cell images for development of automatic recognition systems. *Data in Brief*, 30, 105474.
- He, K., Zhang, X., Ren, S., & Sun, J. (2016). Deep residual learning for image recognition. *CVPR*.
- Kutlu, H., Avci, E., & Özyurt, F. (2020). White blood cells detection and classification based on regional convolutional neural networks. *Medical & Biological Engineering & Computing*, 58(5), 1155–1168.
- Mooney, P. (2018). Blood Cells Image Dataset. *Kaggle*.
- Shahin, A. I., Guo, Y., Amin, K. M., & Sharawi, A. A. (2019). White blood cells identification system based on convolutional deep neural learning. *Computer Methods and Programs in Biomedicine*, 168, 69–80.

---

## 7. Student Contribution

**Yanir Karpis (318933371)** completed this project independently:
- Dataset pipeline and preprocessing
- Model architecture design (PyTorch)
- Data augmentation strategy
- Hyperparameter optimization and selection
- Training loop implementation and optimization
- Performance evaluation (confusion matrix, ROC, F1-scores)
- Visualization and analysis

---

## 8. Repository & Code Execution

**Notebook:** `Medical_image_processing_final.ipynb`

**Execution Instructions:**
1. Open in Google Colab
2. Enable T4 GPU
3. Run all cells in sequence
4. Required packages install automatically: PyTorch, torchvision, scikit-learn, matplotlib, gdown
5. Dataset downloads automatically from Google Drive

**Code Structure:**
- Cells 1–3: Setup, dataset download, loading, and augmentation
- Cells 4–6: Model 1 (frozen) training and evaluation
- Cells 7–10: Model 2 (fine-tuned) training and evaluation
- Cells 11–17: ROC analysis and visualizations

**Reproducibility:** Random seed (42) ensures identical train/val splits and initialization across runs.
