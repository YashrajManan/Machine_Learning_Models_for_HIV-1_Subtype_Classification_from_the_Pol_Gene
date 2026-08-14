# HIV-1 *pol* Subtype Classification with Deep Learning

Classifying HIV-1 subtypes and circulating recombinant forms (CRFs) from
partial *pol* gene sequences using classical ML baselines and 1D
convolutional neural networks (ResNet-18, VGG-16, InceptionNet-v1),
benchmarked under a leakage-free, cluster-aware evaluation protocol.

## Overview

HIV-1 is classified into genetic subtypes (A, B, C, D, G, ...) and
recombinant forms (e.g. 01_AE, 02_AG, 07_BC, 08_BC) that differ in
geographic distribution, transmissibility, and drug-resistance profiles.
Accurate subtyping from sequence data supports surveillance and treatment
decisions. This project benchmarks sequence-encoding + model combinations
for automated subtype classification directly from raw nucleotide sequence,
and stress-tests the best models against shorter, more realistic sequence
fragments.

**Data**: partial HIV-1 *pol* gene sequences sourced from the
[Los Alamos HIV Sequence Database](https://www.hiv.lanl.gov/), filtered down
to 10 well-defined subtypes/CRFs (~24,000 sequences after cleaning): B,
01_AE, C, A1, 07_BC, D, 08_BC, 02_AG, A6, G.

## Method

- **Sequence encodings compared**: raw/TF-IDF-weighted k-mer (6-mer)
  frequency vectors, Word2Vec k-mer embeddings, and chaos game
  representation (CGR) + TF-IDF.
- **Models**: a Linear SVM (TF-IDF 6-mer) as the classical baseline, and
  three 1D-CNN architectures adapted from computer vision — ResNet-18,
  VGG-16, and InceptionNet-v1 — trained directly on encoded sequences.
- **Evaluation protocol**: `StratifiedGroupKFold`, grouping near-duplicate
  sequences into the same fold (by cluster) so they can never leak across
  the train/test split, while keeping class proportions balanced. Runs are
  repeated across multiple seeds for robustness.
- **Per-class analysis**: precision/recall/F1 broken down per subtype for
  the three headline models (SVM baseline, ResNet-18, VGG-16), using the
  same leakage-free split.
- **Fragment-length stress test**: the same three models retrained/retested
  on randomly cropped sequence windows of decreasing length, to see which
  architecture degrades most gracefully as input gets shorter — a proxy for
  real-world partial/low-quality sequencing reads.

## Repository structure

```
hiv-subtype-classification/
├── notebooks/
│   ├── hiv_subtype_classification_data_prep.ipynb   # sequence loading & cleaning
│   ├── hiv-subtype-classification-resnet-18.ipynb   # ResNet-18 (1D) training/eval
│   ├── hiv-subtype-classification-vgg-16.ipynb      # VGG-16 (1D) training/eval
│   ├── hiv-subtype-classification-inception-net-v1.ipynb  # InceptionNet-v1 (1D)
│   ├── hiv-per-class-analysis.ipynb                 # per-subtype metrics, SVM vs ResNet-18 vs VGG-16
│   └── hiv-fragment-stress-test.ipynb               # performance vs. sequence fragment length
├── outputs/                                          # saved metrics/plots from each notebook run
│   ├── resnetoutput.zip
│   ├── vggoutput.zip
│   ├── inceptionoutput.zip
│   ├── perclassanalysis.zip
│   └── fragmentstresstest.zip
└── requirements.txt
```

## Data availability

The raw and prepared datasets (`hiv-db.fasta`, `prepared_sequences.csv`,
`prepared_sequences_clean.csv`) are not included in this repository because
of their size. They are available as a Kaggle dataset:
`yashrajrajarishi/subtype-classification` (referenced directly in the
notebooks), or can be rebuilt from the Los Alamos HIV Sequence Database
using `hiv_subtype_classification_data_prep.ipynb`.

## Reproducing

These notebooks were developed and run on Kaggle (GPU). To reproduce:

1. Attach the `subtype-classification` Kaggle dataset (or your own prepared
   CSV) as `/kaggle/input/...` and update `DATA_PATH` in each notebook's
   config cell.
2. Run `hiv_subtype_classification_data_prep.ipynb` first if you need to
   rebuild `prepared_sequences_clean.csv` from raw FASTA.
3. Run the model notebooks (`resnet-18`, `vgg-16`, `inception-net-v1`)
   independently — each trains and evaluates across multiple seeds.
4. Run `hiv-per-class-analysis.ipynb` and `hiv-fragment-stress-test.ipynb`
   for the deeper diagnostic comparisons.

## Status

Work in progress / pre-publication research code.

## License

TODO: add a license (e.g. MIT) if you intend this to be reused by others.
