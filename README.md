# morphomap

Predicting drug mechanism of action from cell images, with batch-aware evaluation (BBBC021).

Weekly progress notes: [Image-drug/docs/worklog.md](Image-drug/docs/worklog.md)

## What this project does

Cells treated with different drugs look different under a microscope. Drugs that work the same way tend to produce similar-looking cells. This project tests how well that holds: take images of drug-treated cells, turn them into numbers, and predict a drug's mechanism by finding the most similar other drug.

It replicates Ljosa et al. (2013), then adds a stricter evaluation the original paper mostly skipped: blocking matches between drugs imaged in the same week.

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

Images and CSVs are gitignored. Download them into `Image-drug/images/` and `Image-drug/csv/`.

## Pipeline

```
images → segment cells → measure each cell → normalize per plate against DMSO
       → average cells into one profile per well
       → median across the 3 replicate wells → one profile per treatment
       → cosine distance → nearest neighbour → predicted mechanism
```

## Evaluation

For each of the 103 treatments, find the closest profile from a different compound and take its mechanism. Reported two ways:

- **NSC** (not-same-compound) — the paper's rule
- **NSCB** (not-same-compound-or-batch) — also blocks matches from the same week

### Why the batch rule matters

The 55 plates were prepared over 10 weeks, each with a fresh batch of cells, freshly mixed dyes and its own microscope session. Small technical differences get baked into every image from a given week, and with 453 features per cell a model picks them up easily.

That matters here because every drug was imaged in exactly one week, with its three replicate wells spread across three plates within that week. So drug identity and week identity are entangled: two drugs sharing a week also share a technical fingerprint, and their profiles can end up close for reasons unrelated to what the drugs do. NSCB blocks that route, so whatever accuracy survives came from the cells rather than the session.

Two consequences:

- **The drop is large.** The BBBC page reports the factor-analysis result falling from 94% (NSC) to 77% (NSCB).
- **Some classes can't be scored at all.** All three kinase inhibitors are in week 7 and both cholesterol-lowering drugs are in week 9, so under NSCB every valid partner is hidden and those treatments are wrong by construction — 11 of 103, capping the achievable NSCB score. The paper states classes were distributed across batches to avoid biasing the classification; the released data doesn't bear that out for these two.

### Reference results (from the paper)

| Method | NSC |
|---|---|
| Means | 83% |
| KS statistic | 83% |
| SVM normal vector | 81% |
| Gaussian mixture | 83% |
| Factor analysis | 94% |

## Results

Not yet — pipeline in progress.

## What the data actually looks like

Established from the metadata (see the worklog for how):

- **Controls on every plate.** Exactly 6 DMSO wells, in fixed positions (B02, C02, D02, E11, F11, G11). Taxol at 0.3 µM is the positive control, also ~6 wells per plate, which gives it 333 wells where every other treatment has 3.
- **Fixed layout.** One drug per row, dose decreasing left to right, so dose is confounded with column position and all replicates share that position.
- **Replicates are plate groups.** Plates come in groups of 3 with identical layouts. Week 8 has one group of 2, which is why 7 treatments have 2 wells instead of 3.
- **One week per drug.** Only DMSO and taxol span all 10 weeks. Week sizes are uneven: week 3 holds 29 labelled treatments, week 6 holds 1.
- **Uneven classes.** 5–14 treatments and 2–4 drugs per class. Cholesterol-lowering and Eg5 have 2 drugs each, so each drug has exactly one possible correct match. Eg5's 12 treatments come from just 2 drugs (AZ-C contributes 7).
- **Dose ladder.** ×3 steps from 0.001 to 100, except statins and cycloheximide, which use their own steps. ALLN, MG-132 and proteasome inhibitor I skip middle doses — all three are protein degradation compounds, and the paper doesn't explain it.
- **Unlabelled compounds.** 113 imaged, 38 labelled. The excluded ones include doxorubicin and monastrol, whose mechanisms are well known, so exclusion wasn't purely about missing knowledge. Week 8 contains a compound named `UNKNOWN`.

## Repo layout

```
Image-drug/
  csv/          metadata CSVs and exploration notebook
  docs/         worklog
  images/       downloaded plate images (gitignored)
  src/          pipeline code
```
