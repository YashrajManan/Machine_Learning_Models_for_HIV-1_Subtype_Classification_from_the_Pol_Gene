# HIV-1 Subtype Classification (pol gene)

Alignment-free deep learning for HIV-1 subtype/CRF classification from the *pol* gene — the data, notebooks,
and results behind the manuscript and its supplementary materials.

**One architecture, no exceptions.** Every model in this repository — SVM, Random Forest, VGG-16 (1D),
Inception-v1 (1D), and ResNet-18 (1D) — uses no input-noise augmentation of any kind in its final, published
form. An earlier development version of ResNet-18 included a `GaussianNoise(0.05)` layer on its input; that
layer has been removed from every notebook in this repository (6 notebooks affected: the ones now numbered
11, 13, 14, 15, and 16). VGG-16, Inception-v1, SVM, and RF never had such a layer, so their numbers are
unaffected throughout.

## Layout

```
data/        prepared sequence data (gzipped CSV + FASTA)
notebooks/   17 notebooks, numbered in the order you'd actually run them
results/     one folder per notebook (same number), holding that notebook's output CSVs
```

## Data

| File | Contents |
|---|---|
| `prepared_sequences.csv.gz` | Original prepared dataset before near-duplicate cleaning. |
| `prepared_sequences_clean.csv.gz` | Cleaned dataset used throughout the study (24,072 sequences, 10 classes: B, C, A1, A6, D, G, 01_AE, 02_AG, 07_BC, 08_BC). Columns: `sequence`, `subtype`, `cluster` (MinHash near-duplicate cluster id, Jaccard ≥ 0.90, 11,295 clusters). |
| `prepared_sequences_clean.fasta.gz` | The same cleaned dataset as FASTA (`>seq{idx}\|subtype={subtype}\|cluster={cluster}`). |

The primary leakage-control split used everywhere in this study is **CD-HIT at 97% identity** (run from
inside the relevant notebooks, not pre-computed into a file); the `cluster` column above is a *separate*,
independent MinHash-based grouping used only for the cross-check in notebook 12.

## Notebooks → results → manuscript map

| # | Notebook | Models | Feeds |
|---|---|---|---|
| 01 | Data preparation | — | Methods: data preparation |
| 02 | Leakage delta, random vs. CD-HIT split | SVM, RF, ResNet-18 | Table 7 / leakage-delta discussion |
| 03 | Clustering threshold calibration | — | CD-HIT identity threshold choice |
| 04 | One-hot encoding CNN baseline | 1D-CNN | Encoding baseline comparison |
| 05 | Tool head-to-head comparison | SVM, RF | Tool comparison section |
| 06 | Bootstrap significance baseline | ResNet-18 | Bootstrap CI methodology |
| 07 | Real amplicon primer test | — | Amplicon realism discussion |
| 08 | Ambiguity robustness baseline | — | Ambiguity robustness discussion |
| 09 | Inference timing benchmark | ResNet-18 | Timing/efficiency discussion |
| 10 | CD-HIT grouped split baseline | SVM, RF, VGG-16, ResNet-18 | Table 7, Supplementary Table S12 |
| 11 | **Fragment length sweep, all models** | SVM, RF, VGG-16, Inception-v1, ResNet-18 | Table 3, Supplementary Table S4, Figure 4 |
| 12 | **Split strategy ablation, all models** | SVM, RF, VGG-16, ResNet-18 | Supplementary Tables S12, S13, S14 |
| 13 | **Nine-seed robustness, all models** | SVM, RF, VGG-16, Inception-v1, ResNet-18 | 9-seed robustness table |
| 14 | **Encoding comparison, all models** | SVM, RF, VGG-16, Inception-v1, ResNet-18 | Encoding-scheme comparison table |
| 15 | **PR/RT amplicon realism, all models** | SVM, RF, VGG-16, Inception-v1, ResNet-18 | Table 6, amplicon McNemar test |
| 16 | **Ambiguity injection, all models** | SVM, RF, VGG-16, Inception-v1, ResNet-18 | Ambiguity-robustness table |
| 17 | Statistics (McNemar, Holm-Bonferroni, bootstrap) | — | Independently reproduces the manuscript's significance tests and CIs directly from raw predictions |

Notebooks 11, 12, 13, 14, 15, and 16 (bold above) are each a single, complete notebook covering **every**
model used for that experiment — there is no separate "ResNet-18 only" notebook anywhere in this repository.
Two of them (12 and 15) are visibly built from two real sub-notebooks merged into one file, each clearly
marked with a "Part A" / "Part B" heading; the rest are a single original notebook with one line removed
from the ResNet-18 builder cell.

## "Legacy resnet" files in `results/`

A few raw/summary CSVs are suffixed `_legacy_resnet` or named `..._and_legacy_resnet_raw.csv`. These are the
**original** output files from notebooks 13, 14, 15, and 16, from before the GaussianNoise layer was removed.
They're kept, not deleted, for a transparent paper trail. Inside each such file:

- The **SVM / RF / VGG-16 / Inception-v1 rows are unaffected and still authoritative** — those models never
  had a GaussianNoise layer, so nothing about their numbers changed.
- The **ResNet-18 rows in these files are superseded.** The current, published, no-noise ResNet-18 numbers
  for the same experiment are in the matching `resnet18_..._nonoise_raw.csv` / `_summary.csv` file in the
  same folder.

This is why, for example, `results/13_nine_seed_robustness_all_models/` has both
`nineseed_deep_raw_legacy_resnet.csv` (old, ResNet-18 rows superseded) and
`resnet18_nineseed_nonoise_raw.csv` (current).

## Known gap

Supplementary Table S14 in the manuscript currently prints numbers under a column header that doesn't match
how the split was actually run (a known, pre-existing mismatch, not introduced by this repository). The
*correct* MinHash-grouped-split numbers for SVM, RF, and ResNet-18 are the real, verified results sitting in
`results/12_split_strategy_ablation_all_models/svm_rf_minhashsplit_fulllength_raw.csv` and
`resnet18_minhashsplit_s14_raw.csv` / `_summary.csv`. VGG-16 was not rerun under the MinHash split (only SVM,
RF, and ResNet-18 were). The manuscript table itself has not yet been corrected to match.

## Reproducibility note

Notebooks 11–16 train ResNet-18/VGG-16/Inception-v1 on GPU. GPU-trained deep learning is not bit-reproducible
even with a fixed random seed (cuDNN convolution algorithms are non-deterministic by default, and
floating-point summation is not associative across different thread-scheduling orders), so rerunning these
notebooks will land close to, but not necessarily identical to, the numbers in the committed CSVs. Notebook
17 (statistics) and the SVM/RF cells throughout are CPU-based and deterministic given the same library
versions, so those reproduce exactly.
