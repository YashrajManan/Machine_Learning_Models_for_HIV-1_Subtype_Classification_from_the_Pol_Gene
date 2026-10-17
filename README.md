# HIV-1 Subtype Classification (*pol* gene)

**Alignment-free deep learning and classical ML tell HIV-1 subtypes and CRFs apart directly from raw *pol* gene
sequence, and a plain TF-IDF linear SVM matches or beats three CNN architectures while staying far more robust
when the input is short, fragmented, or ambiguity-coded — exactly the conditions a real clinical PCR amplicon
produces.** This repository holds the data, notebooks, and results behind that finding, as published in the
manuscript and its supplementary materials.

---

## 1. Research question

Can a classifier tell which HIV-1 subtype or circulating recombinant form (CRF) a sequence belongs to directly
from the raw *pol* gene sequence — without multiple-sequence alignment, without hand-built phylogenetic
features — and does that hold up under the conditions a real diagnostic sample actually presents: a short PCR
amplicon instead of the full gene, ambiguity-coded bases from imperfect Sanger sequencing, and a dataset where
near-identical sequences from the same outbreak or passage lineage must not leak between training and test.

## 2. Biological background

HIV-1 is classified into major group M subtypes (A, B, C, D, F, G, H, J, K) and circulating recombinant forms
(CRFs, e.g. 01_AE, 02_AG, 07_BC, 08_BC) that arise when two subtypes co-infect the same host and recombine.
Subtype identity matters clinically and epidemiologically: it correlates with differences in transmission
route, disease progression, and — most directly relevant here — with baseline susceptibility to specific
antiretroviral drugs and with which drug-resistance mutation pathways a virus is likely to take under treatment
pressure. The *pol* gene (encoding protease, reverse transcriptase, and integrase) is the standard target for
both subtyping and drug-resistance genotyping in clinical practice, because it is the gene actually sequenced
when a patient is tested for resistance — so a subtyping tool that works directly on the *pol* fragment a clinic
already has, rather than requiring the full genome or a separate alignment step, is the practically useful kind.

Alignment-free classification matters because standard subtyping tools (e.g. REGA, COMET) rely on phylogenetic
placement against a reference alignment, which is sensitive to alignment quality, computationally heavier, and
can struggle with recombinants and short fragments. A sequence-only classifier that works on raw, unaligned
nucleotide sequence sidesteps that dependency entirely.

## 3. Dataset

**Source.** The Los Alamos National Laboratory (LANL) HIV Sequence Database (`hiv.lanl.gov`), the standard
public, curated repository of annotated HIV sequences used throughout the field. Sequences were retrieved
through LANL's Sequence Search Interface (a curated web query interface over its sequence collection, not a
programmatic REST API) as a multi-subtype *pol* FASTA export, with subtype/CRF labels taken from each record's
LANL header field.

**Size and labels.** After filtering to subtypes/CRFs with at least 100 sequences and removing exact-duplicate
sequences: **24,072 sequences across 10 classes** — B, C, A1, A6, D, G, 01_AE, 02_AG, 07_BC, 08_BC.

**Files in `data/`:**

| File | Contents |
|---|---|
| `prepared_sequences.csv.gz` | Prepared dataset before near-duplicate clustering. |
| `prepared_sequences_clean.csv.gz` | The cleaned dataset used throughout the study. Columns: `sequence`, `subtype`, `cluster` (MinHash near-duplicate cluster id, Jaccard ≥ 0.90, 11,295 clusters). |
| `prepared_sequences_clean.fasta.gz` | The same cleaned dataset as FASTA (`>seq{idx}\|subtype={subtype}\|cluster={cluster}`). |

**Leakage control.** Two independent grouping schemes are used so that near-identical sequences (the same
outbreak, the same patient sampled longitudinally, lab-passaged variants) never split across train and test:
**CD-HIT at 97% sequence identity** (run live, inside the relevant notebooks, via `cd-hit-est`) is the primary
split used everywhere in this study; a separate, independent **MinHash-based clustering** (Jaccard ≥ 0.90,
the `cluster` column above) is used as a cross-check in notebook 12. Both are compared against an ordinary
random split to quantify how much a naive split overstates accuracy (the "leakage delta").

## 4. Methods

Five models, spanning classical ML to three CNN families, all trained on identical splits for a fair
comparison:

| Model | What it is | Why it was chosen (vs. the realistic alternatives) |
|---|---|---|
| **Linear SVM** | `LinearSVC` over TF-IDF 6-mer features | The standard strong classical baseline for sequence classification; linear kernel chosen over RBF/polynomial because it scales to tens of thousands of sparse high-dimensional TF-IDF features without a kernel-matrix blowup, and because a linear decision boundary over k-mer frequencies is directly interpretable (feature weight = a k-mer's contribution to a subtype call) — useful for comparing against the CNNs, which are not. |
| **Random Forest** | Ensemble of decision trees over the same TF-IDF features | A non-linear classical baseline, chosen alongside the SVM rather than instead of it: it can capture feature interactions the linear SVM cannot, so the gap (or lack of one) between the two classical models tells you whether non-linearity itself is buying anything here, independent of whether deep learning is. |
| **VGG-16 (1D)** | 1D adaptation of the VGG-16 CNN architecture | A deep, uniform stacked-convolution architecture with no skip connections — the "plain deep CNN" reference point, included so ResNet-18's skip connections can be credited (or not) with any performance difference against an otherwise comparably deep plain network. |
| **Inception-v1 (1D)** | 1D adaptation of GoogLeNet's Inception module | Multi-scale: parallel convolutional branches at different kernel sizes let the network learn motifs at several nucleotide-window scales simultaneously, a biologically plausible prior for sequence motifs of varying length (compared to VGG's and ResNet's single kernel size per layer). |
| **ResNet-18 (1D)** | 1D ResNet-18 with skip connections, stochastic depth, spatial dropout | The deepest, most regularized CNN tested, included to check whether depth and residual learning — which help substantially on image tasks — transfer to a one-dimensional, much lower-information-density nucleotide sequence. All five models in this repository use the same, single architecture throughout — one ResNet-18, start to finish, with no input-noise augmentation. |

**Encoding.** TF-IDF over character 6-mers (`TfidfVectorizer`, 512 features) is the primary encoding for
every model; notebook 14 additionally benchmarks plain-frequency 6-mers, Word2Vec 6-mer embeddings, and a
Word2Vec+TF-IDF hybrid.

**Why Python (not R).** The project uses Python throughout rather than R because its scientific-computing
stack (`scikit-learn`, `TensorFlow/Keras`, `Biopython`) covers classical ML, deep learning, and sequence I/O in
one consistent ecosystem with GPU support (needed for the three CNNs) that R's equivalent packages do not
match as directly, and because `TfidfVectorizer`/`StratifiedGroupKFold` give exactly the leakage-safe,
feature-engineering building blocks this study's split design needed.

## 5. Analysis pipeline

1. **Data preparation** (notebook 01) — parse the LANL FASTA, filter to classes with ≥100 sequences, dedupe
   exact sequences, cluster near-duplicates by MinHash.
2. **Leakage-control calibration** (notebooks 02–03) — quantify the random-vs-grouped-split leakage delta;
   calibrate the CD-HIT identity threshold.
3. **Baseline checks** (notebooks 04–10) — one-hot CNN encoding baseline, head-to-head against external
   subtyping tools, bootstrap CI methodology, a real PR/RT amplicon extraction test, an ambiguity-code
   robustness baseline, inference timing, and a CD-HIT-grouped-split baseline.
4. **Main experiments, all five models together** (notebooks 11–16) — fragment-length sweep, split-strategy
   ablation, nine-seed robustness, encoding comparison, PR/RT amplicon realism, ambiguity-code injection.
5. **Statistics** (notebook 17) — McNemar significance tests (Holm-Bonferroni corrected) and bootstrap
   confidence intervals, computed directly from the per-sequence predictions saved above.

## 6. Results

Headline numbers (macro-F1, CD-HIT-97% split, mean over 3 seeds) from
`results/11_fragment_length_sweep_all_models/fragment_sweep_summary_cdhit97.csv`:

| Length (bp) | SVM | Random Forest | VGG-16 | Inception-v1 | ResNet-18 |
|---|---|---|---|---|---|
| 300 | **0.834** | 0.789 | 0.728 | 0.370 | 0.753 |
| 500 | **0.936** | 0.857 | 0.781 | 0.376 | 0.886 |
| 1000 | **0.972** | 0.946 | 0.896 | 0.557 | 0.954 |
| full | 0.979 | **0.988** | 0.974 | 0.780 | 0.976 |

The linear SVM holds a clear, statistically significant advantage (McNemar, Holm-Bonferroni corrected —
`results/17_statistics_mcnemar_bootstrap/`) over every CNN at short fragment lengths, where a real diagnostic
PCR amplicon actually lives; the gap narrows at full length, where Random Forest edges ahead. Every CSV behind
every manuscript table and supplementary table is in `results/`, one folder per notebook (same number).

## 7. Figures

The manuscript's figures (fragment-length-sweep curves, confusion matrices, architecture diagrams) are built
from the CSVs in `results/` and live in the manuscript and its supplementary materials rather than being
duplicated as image files in this repository; any figure can be regenerated directly from the corresponding
notebook's output CSV.

## 8. Limitations

- **Labels follow the LANL header, not independent confirmatory subtyping** — a small fraction of public
  database labels are known to be imperfect or recombinant-ambiguous.
- **CD-HIT-97% and MinHash (Jaccard ≥ 0.90) are proxies for epidemiological relatedness**, not a guarantee
  that every true transmission cluster is kept intact; a stricter identity threshold would shrink the
  effective dataset further.
- **Class imbalance**: 10 classes with very different sample sizes (subtype B and C dominate the public
  database); minority-class CRF performance has a wider confidence interval than the headline macro-F1 implies.
- **Simulated, not clinical, amplicons and ambiguity codes** — the PR/RT amplicon test and ambiguity-injection
  robustness test use primer-coordinate extraction and synthetic ambiguity-code injection on existing
  sequences, not sequences actually produced by Sanger resequencing of degraded clinical samples.
- **GPU training is not bit-reproducible** even at a fixed seed (see Reproducibility note below), so exact
  CNN numbers will drift slightly on rerun even though the conclusions are stable across seeds.
- No causal claim is made about *why* the linear SVM generalizes better at short length; the result is
  behavioral (measured robustness), not a mechanistic explanation of the CNNs' short-fragment degradation.
- **Table 11's PR-alone and RT-alone rows were not regenerated on the CD-HIT-97% split.** A
  coordinate-mapping bug meant the PR+RT-combined and dropout rows (Part A of notebook 15) had to be
  rerun; the PR-alone/RT-alone rows (Part B) still reflect an earlier run on the MinHash split, which
  is why their standard deviations (0.004–0.006) are visibly tighter than the two regenerated rows
  (0.022–0.029) in the same table. Notebook 15 documents this explicitly and provides the code to close
  the gap on a future rerun.
- **Results for notebooks 01–10 are not archived in `results/`** (only notebooks 11–17 have a
  `results/` subfolder, with notebook 03 as the one partial exception — see the layout note below).
  These are baseline/methodology notebooks whose numbers are reported directly in the manuscript text
  rather than re-saved as standalone CSVs in this repository; re-running any of them reproduces its
  numbers from the raw data.

## 9. Biological interpretation

A 6-mer TF-IDF linear SVM effectively counts short, subtype-diagnostic nucleotide motifs and weighs them
linearly — a good match for how subtypes actually differ: by an accumulation of many short, scattered
sequence signatures across the gene, each individually weak but collectively diagnostic, rather than by one
long-range structural pattern a CNN's deeper receptive field is needed to see. That is consistent with why the
SVM's advantage is largest exactly where there is the least sequence to count motifs from (300–500 bp) and
narrows once the full gene is available and every model has enough signal to work with. It also matches the
practical diagnostic setting directly: a clinical genotyping amplicon is short by necessity (PCR amplicon
length, sequencing read length), so a classifier that degrades gracefully rather than catastrophically at
short length — as the SVM does relative to the CNNs, most visibly Inception-v1 — is the more clinically useful
tool even where a CNN might win marginally at full length.

## 10. Reproducible code

Every notebook is self-contained and states, at the top, whether it needs a GPU and roughly how long it will
take on Kaggle's free tier. To reproduce:

1. Download the LANL multi-subtype *pol* FASTA (or use `data/prepared_sequences_clean.csv.gz` directly to skip
   re-downloading and re-clustering).
2. Run notebooks in numeric order; each is resume-safe (re-running skips any `(model, seed, …)` combination
   already present in its output CSV).
3. Install dependencies: scikit-learn, TensorFlow/Keras, Biopython, datasketch, statsmodels; CD-HIT-EST
   installed via `apt`/`conda` for the clustering steps.

**Reproducibility note.** Notebooks 11–16 train ResNet-18/VGG-16/Inception-v1 on GPU. GPU-trained deep
learning is not bit-reproducible even with a fixed random seed (cuDNN convolution algorithms are
non-deterministic by default, and floating-point summation is not associative across different
thread-scheduling orders), so rerunning these notebooks will land close to, but not necessarily identical to,
the numbers in the committed CSVs. Notebook 17 (statistics) and the SVM/RF cells throughout are CPU-based and
deterministic given the same library versions, so those reproduce exactly.

## 11. Layout, notebook map, and one-line pitch

**One-line pitch.** *A plain TF-IDF linear SVM matches or beats three CNN architectures at HIV-1 subtype/CRF
classification from raw* pol *gene sequence, and is far more robust than every CNN tested when the input is
short, fragmented, or ambiguity-coded — the exact conditions a real clinical genotyping amplicon produces.*

```
data/        prepared sequence data (gzipped CSV + FASTA)
notebooks/   17 notebooks, numbered in the order you'd actually run them
results/     one folder per notebook that has archived output CSVs (11-17, plus the partial
             exception of 03 below) — see the Limitations section for why 01-10 are mostly unarchived
```

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

Notebooks 11–16 (bold above) are each a single, complete notebook covering every model used for that
experiment — there is no separate "ResNet-18-only" or "SVM/RF-only" notebook among the main results.
Notebooks 01–10 are earlier, narrower-scope baselines and methodology checks, each answering one specific
question with only the model(s) that question needs (a split-threshold calibration or a timing benchmark
does not need all five models to answer its own question) — none of them compares models against each other
the way 11–16 do, so each uses only as much of the model roster as its own question requires. Every multi-model
comparison in 11–16 trains and evaluates every model included in it over the same number of seeds, so no
model is given an advantage of extra repeated trials the others don't have.

**CV / GitHub abstract.** *Built an alignment-free, leakage-controlled benchmark for HIV-1 subtype/CRF
classification from raw* pol *gene sequence (24k sequences, 10 classes), comparing a TF-IDF linear SVM,
Random Forest, and three 1D CNN architectures (VGG-16, Inception-v1, ResNet-18) under CD-HIT-97%-grouped,
MinHash-grouped, and random splits. Found that a simple linear SVM over 6-mer TF-IDF features matches or
beats all three CNNs and is substantially more robust under the short, fragmented, ambiguity-coded inputs a
real clinical genotyping amplicon produces — with every result backed by McNemar significance testing and
bootstrap confidence intervals.*
