# morphomap

Predicting drug mechanism of action from cell images, with batch-aware evaluation (BBBC021).

## What this project does

Cells treated with different drugs look different under a microscope. Drugs that work the same way tend to produce similar-looking cells. This project tests how well that holds: take images of drug-treated cells, turn them into numbers, and predict a drug's mechanism by finding the most similar other drug.

It replicates Ljosa et al. (2013), then adds a stricter evaluation the original paper mostly skipped: blocking matches between drugs imaged in the same week, so the model can't succeed by recognising the batch instead of the biology.

Paper: https://doi.org/10.1177/1087057113503553 (free version: https://pmc.ncbi.nlm.nih.gov/articles/PMC3884769)

## Dataset

BBBC021: MCF7 breast cancer cells, 113 compounds at 8 doses in triplicate, stained for DNA, tubulin and actin, imaged over 10 weeks.

| | |
|---|---|
| Plates | 55 (49 hold labelled treatments) |
| Wells used per plate | 60 (rows B–G, columns 2–11) |
| Photo spots | 13,200 (4 per well, 3 channels each) |
| Images | 39,600 TIFFs, 1024 × 1280, 16-bit |
| Cells | ~2.2 million |
| Labelled treatments | 103 drug-dose pairs, 38 drugs, 12 mechanism classes |

Download: https://bbbc.broadinstitute.org/BBBC021

- `BBBC021_v1_image.csv` — one row per photo spot: plate, well, compound, dose, file names
- `BBBC021_v1_moa.csv` — one row per labelled drug-dose: its mechanism
- Image ZIPs, one per plate, ~800 MB each

Images and CSVs are gitignored. Download them into `images/` and `csv/`.

## Pipeline

```
images → segment cells → measure each cell → normalize per plate against DMSO
       → average cells into one profile per well
       → median across the 3 replicate wells → one profile per treatment
       → cosine distance → nearest neighbour → predicted mechanism
```

Scoring follows the paper: for each of the 103 treatments, find the closest profile from a different compound and take its mechanism. Reported two ways:

- **NSC** (not-same-compound) — the paper's rule
- **NSCB** (not-same-compound-or-batch) — also blocks same-week matches

Paper's results for reference: means 83%, KS statistic 83%, SVM 81%, Gaussian mixture 83%, factor analysis 94% (all NSC). The BBBC page reports the factor-analysis result dropping to 77% under NSCB.
