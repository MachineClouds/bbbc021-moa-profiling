# Worklog

Newest entries at the top.

## Week 1 — Metadata exploration

Notebook: `csv/dataexploration.ipynb`

Goal: understand the dataset before touching any images. Loaded both CSVs, joined them, and worked out how the experiment was actually structured.

### What I did

- Loaded `BBBC021_v1_image.csv` (13,200 rows) and `BBBC021_v1_moa.csv` (104 rows), and checked both for missing values — there are none.
- Confirmed the label file is internally consistent: 38 drugs, 12 mechanisms, no drug carrying more than one mechanism.
- Checked DMSO coverage plate by plate.
- Joined the two files on compound + concentration (inner join) to get `final_data`.
- Counted doses per drug, and the ratio between consecutive doses, to see the dose ladder.
- Counted treatments and drugs per mechanism class.
- Extracted the week from the plate name and mapped which weeks each drug appears in.
- Built a plate-layout heatmap and compared plates within and across weeks.
- Loaded and displayed the first DNA, tubulin and actin images.

### What I found

**Controls**

- All 55 plates have exactly 6 DMSO wells — no plate is missing a normalization reference.
- DMSO sits in the same six positions on every plate: B02, C02, D02, E11, F11, G11.
- Taxol at 0.3 µM is on every plate as a positive control (333 wells total), while also being one of the 103 labelled treatments. Its other doses have 3 wells like everything else. This will need handling explicitly when building treatment profiles.

**Plate layout**

- 60 wells used per plate; the outer ring (row A, row H, columns 1 and 12) is empty, presumably because edge wells evaporate faster.
- One drug per row, dose decreasing left to right. Column 3 is always the highest dose.
- Dose is therefore confounded with column position, and all three replicates sit in the same position, so no replicate can average a position effect out.
- Plates come in groups of 3 sharing an identical layout — these are the replicates. Week 1 has two such groups: {22123, 22141, 22161} and {22361, 22381, 22401}.
- Week 8's second group has only 2 plates (38341, 38342), which explains the 7 treatments that have 2 wells rather than 3: PD-169316 (3, 10), PP-2 (3, 10), alsterpaullone (1, 3), bryostatin (0.3).

**Weeks**

- Every drug appears in exactly one week; only DMSO and taxol span all 10.
- Replicates are spread across plates but not across weeks, so a week-level problem hits all three copies identically.
- Week sizes are very uneven: week 3 holds 29 labelled treatments, week 6 holds 1.
- All three kinase inhibitors are in week 7; both cholesterol-lowering drugs are in week 9. Under a not-same-week rule these classes have no valid partner at all.
- The paper says classes were distributed across batches to avoid biasing the classification. For these two classes, the released data doesn't support that.

**Doses**

- Drugs have 1–7 labelled doses out of 8 tested. 7 drugs have a single labelled dose; AZ-C and vincristine have 7 each.
- Doses follow a ×3 ladder (0.001, 0.003, 0.01 … 100). Statins and cycloheximide use their own steps (1.5/5/15, 2/6/20, 5/15/50).
- ALLN (3 → 100), MG-132 (0.1 → 3) and proteasome inhibitor I (0.1 → 3) skip middle doses. All three are protein degradation compounds, which suggests something systematic rather than random. Unexplained in the paper. Worth checking against `image_meta` whether those middle doses were imaged at all.

**Classes**

- 5 to 14 treatments per class, 2 to 4 drugs per class.
- Cholesterol-lowering and Eg5 inhibitors have only 2 drugs, so each drug has exactly one possible correct match.
- Eg5's 12 treatments come from just 2 drugs, with AZ-C contributing 7 — a small class that looks large.

**Other**

- 113 compounds were imaged but only 38 labelled. The unlabelled set includes doxorubicin (DNA damage) and monastrol (Eg5), whose mechanisms are well known and whose classes exist in the label set, so exclusion wasn't purely about missing knowledge.
- Week 8 contains a compound named `UNKNOWN`.

**Images**

- 1024 × 1280, 16-bit, values roughly 200–7,500.
- Three separate grayscale files per photo spot. They need contrast stretching to view, but must be passed to segmentation and feature extraction as raw 16-bit.

### Open questions

- Were the skipped middle doses for the three protein-degradation compounds imaged?
- Why were doxorubicin and monastrol left out of the labelled set?
- Which treatments have a valid same-mechanism partner in a different week? (Needed to fix the NSCB ceiling precisely.)

### Next

- Compare DMSO against drugs from several mechanisms by eye, across all three channels
- Segment cells with Cellpose on the DNA channel
- Extract per-cell features
- Build the evaluation harness (NSC and NSCB)
