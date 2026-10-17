# HIV-1 pol Gene Subtype Classification

Code, data, and results behind the manuscript *"Machine Learning Models for HIV-1 Subtype
Classification from the Pol Gene"* (ten-class HIV-1 subtype/CRF classification, alignment-free,
leakage-controlled benchmark of a linear SVM, a random forest, and two 1D CNNs — ResNet-18 and
VGG-16 — over TF-IDF-weighted k-mer features).

This repository is private while the manuscript is under review. It will be made public on
acceptance.

## Repository layout

```
data/         prepared, cluster-labelled sequence data (gzip-compressed)
notebooks/    one subfolder per experiment, numbered to match the order they're described
              in the manuscript's Methods section
results/      the CSV outputs each notebook produces, same numbering as notebooks/
```

## Data

| File | Description |
|---|---|
| `data/prepared_sequences.csv.gz` | All 26,292 sequences surviving initial QC, with the raw (uncleaned) subtype label and the MinHash near-duplicate `cluster` id (Jaccard ≥ 0.90). Output of `notebooks/00_data_preparation`. |
| `data/prepared_sequences_clean.csv.gz` | The final, paper-ready dataset: 24,072 sequences across the ten clean subtype/CRF classes (B, C, A1, A6, D, G, 01_AE, 02_AG, 07_BC, 08_BC), each with its MinHash `cluster` id. This is the file every downstream notebook loads. |
| `data/prepared_sequences_clean.fasta.gz` | The same 24,072 sequences as FASTA, header `>seq{row index}\|subtype={subtype}\|cluster={minhash cluster id}`. Generated directly from `prepared_sequences_clean.csv.gz` for this repository, so it is guaranteed to match it row-for-row. |

The *primary* grouping variable used throughout the paper's main results is **CD-HIT-97%-identity
clustering**, not the MinHash `cluster` column — CD-HIT clusters are computed fresh inside each
notebook that needs them (`cd-hit-est -c 0.97 -n 10 -M 16000 -T 4 -d 0`) rather than stored in the
data files, since CD-HIT output depends on installed version/build. The MinHash `cluster` column is
kept in the data because it's the grouping method the project started with, and it's used directly
by the sensitivity-check notebooks in `03_split_ablations/`.

## Notebooks and results, mapped to the manuscript

| Folder | Experiment | Manuscript item(s) | Models |
|---|---|---|---|
| `00_data_preparation` | Parse raw LANL download, filter rare classes, MinHash-cluster near-duplicates | Section 2.1, Table 2 | — |
| `01_baselines_and_clustering` | Leakage-delta diagnostic, CD-HIT-vs-random split calibration, one-hot CNN baseline, head-to-head vs. published tools (CASTOR/Kameris-equivalent, COMET), bootstrap methodology check, PR+RT sanity checks, ambiguity robustness, training/prediction timing | Section 2.2–2.3, Table 8, Supplementary Table S6/S7/S17 | SVM, RF |
| `02_fragmentation_sweep` | macro-F1 vs. fragment length, 7 lengths × 3 seeds | **Table 3**, Supplementary **Table S4**, **Figure 4** | SVM, RF, VGG-16 (`T18_*.ipynb`); ResNet-18 — **no GaussianNoise**, see note below (`ResNet18_full_reverification_no_gaussian_noise.ipynb`, Section A) |
| `03_split_ablations` | Random split vs. CD-HIT-97% (leakage demonstration); MinHash-grouped vs. CD-HIT-97% split | Supplementary **Table S13**, **Table S14** | SVM, RF, VGG-16 (`T23_cdhit_grouped_split.ipynb`); ResNet-18 — no-noise follow-up (`ResNet18_S13_S14_followup_no_gaussian_noise.ipynb`) |
| `04_nine_seed_robustness` | 9-seed replicate at 3 short lengths, confirming the 3-seed main results aren't a seed artifact | Supplementary **Table S11** | SVM, RF, VGG-16 (`T19_*.ipynb`); ResNet-18 no-noise Section B |
| `05_encoding_comparison` | Raw 6-mer / TF-IDF 6-mer / Word2Vec 6-mer / Word2Vec-TF-IDF, all 4 models | Supplementary **Table S5** | SVM, RF, VGG-16 (`T25_*.ipynb`); ResNet-18 no-noise Section C |
| `06_pr_rt_amplicon` | The real clinical PR+RT genotyping amplicon, intact and with a simulated 200bp dropout | **Table 6**, Supplementary **Table S8/S9** | SVM, RF, VGG-16 (`T21_amplicon_mcnemar.ipynb`, `T22_pr_rt_alone_cdhit97.ipynb`); ResNet-18 no-noise Section D |
| `07_ambiguity_injection` | Injected IUPAC ambiguity codes at 0/0.5/1/2/5/10% | **Table 7** | SVM, RF, VGG-16 (`T24_*.ipynb`); ResNet-18 no-noise Section E |
| `08_statistics` | McNemar exact test + Holm-Bonferroni correction (fragmentation and PR+RT amplicon), percentile bootstrap 95% CI, MinHash-split SVM/RF rerun | **Table 5**, Supplementary **Table S8/S9/S10/S12/S14** | SVM vs. ResNet-18 (no-noise) |

## ResNet-18: no GaussianNoise

The manuscript's ResNet-18 architecture is a plain 1D ResNet-18 — an input layer straight into a
`Conv1D(32, k=7, s=2)` stem, four residual stages with graduated stochastic depth, and a global-
average-pool classifier head. **It does not include a `GaussianNoise` input-perturbation layer.**
Every ResNet-18 result in this repository (Table 3/4/6/7, Supplementary Tables S4/S5/S8–S14) comes
from `ResNet18_full_reverification_no_gaussian_noise.ipynb` and its small follow-up,
`ResNet18_S13_S14_followup_no_gaussian_noise.ipynb` — both run on Kaggle, GPU required. SVM, RF, and
VGG-16 results are unaffected by this and come from their original notebooks (`T18`–`T25` series).

The exact architecture is documented in full in Supplementary Table S1 of the manuscript, and
reproduced in Figure 2.

## Known gaps / honesty notes

- **Inception-v1** was part of the original study design but was dropped from the published
  manuscript (consistently erratic, often below both the SVM and RF baselines — see the
  manuscript's Methods for the full reasoning). Its result files are not included here, though
  some of the `t_*`/`results_T*` raw CSVs still contain an `inception` column or rows from when it
  was still part of the grid — these are left as-is for transparency about what each notebook run
  actually produced, but Inception is not reported anywhere in the manuscript or supplementary
  materials.
- **Supplementary Table S14** (MinHash-grouped vs. CD-HIT-97% split): VGG-16 was not rerun under
  the MinHash-grouped split for the current revision of the manuscript — see the table's own
  footnote. SVM and Random Forest were rerun directly (`08_statistics/svm_rf_minhash_fulllength_raw.csv`).
- The `t_*` / `results_T*` naming in `results/` reflects the internal experiment numbering used
  during development (T1–T25); it does not match the manuscript's own table numbering, which is why
  this README's mapping table exists.

## Reproducing a notebook

Each notebook is self-contained: it installs CD-HIT if needed, loads `prepared_sequences_clean.csv`
(point `DATA_PATH` at your own copy, e.g. the decompressed `data/prepared_sequences_clean.csv.gz`),
and writes its own result CSVs incrementally (safe to interrupt and resume — each saves after every
seed/length/condition). GPU is required for every notebook that trains ResNet-18, VGG-16 or
Inception; the SVM/RF/baseline sections run on CPU alone.
