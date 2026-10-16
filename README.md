# Coin Detection, Classification and Counting

**Image Processing and Computer Vision — Assignment Module #1**
University of Bologna, MSc in Artificial Intelligence

Given a photograph of euro coins on a table, find every coin, identify its denomination and sum
the money in the frame — using traditional computer vision only. No learned detector, no CNN, no
pretrained weights: every decision is made by a geometric, photometric or template-matching rule
whose parameters are set by hand and justified in the notebook.

---

## Results

Measured on the full target set: 142 images, 328 annotated coins, 322 of them carrying a
hand-made denomination label.

| | result |
|---|---|
| images with the exact coin count | **137 / 142** (96.5%) |
| mean absolute count error | 0.049 coins/image |
| coins detected | 323 (6 missed, 1 spurious) |
| denomination accuracy, per coin | **65.8%** over 322 labels |
| &nbsp;&nbsp;— colour + diameter only | 60.2% |
| &nbsp;&nbsp;— colour + design only | 56.5% |
| total on the table | 134.04 € predicted |

Accuracy is strongly conditioned on how many coins share the frame — 58.2% on single-coin images
against 68.3% on images with five or more — because the diameter cue needs companions to
calibrate against. That gap is a limit of the information in the photograph, not of the method;
section 8 of the notebook argues the point.

A held-out split (71 dev / 71 test, fixed seed) is reported in section 7.3 for the two detection
parameters that can be swept from cached scores. The operating threshold sits on a plateau:
anything from 0.56 to 0.72 gives 65 or 66 exact counts out of 71 on dev.

## The problem, and what makes it hard

The two sets are photographed under very different conditions, which is the point of the exercise:

- **Reference set** — 8 images, one clean, well-lit specimen per denomination.
- **Target set** — 142 images, noisy, with the **red and blue channels swapped**. Copper coins
  read as blue. The notebook measures the degradation before correcting it: Immerkaer's estimator
  gives σ ≈ 0.4 on the reference set against σ ≈ 17 on the target set, and the channel order is
  established from the physics of bimetallic coins rather than assumed.

Template matching across the two sets is therefore much weaker than it looks. One photograph per
denomination means a differently worn specimen has no counterpart: SIFT on the coin crops performs
**at chance**, and the notebook shows this is not a noise artefact — the same crop self-matches
with 117 inliers after degradation, against 9 across specimens.

## Method

**Detection — six phases**

1. Non-local means denoising and the red/blue correction
2. Hough circle candidates, four parameter sets, union deduplicated
3. Rim verification — completeness and alignment of the radial gradient, 180 samples per circle
4. Duplicate suppression by disc IoU
5. A physical size band (the largest coin cannot exceed the smallest by more than 25.75/16.25)
6. Rejecting circles that contain bare surface rather than a coin

**Classification — three cues combined per coin**

- **Colour** — grey-world normalisation, then the warmth step between core and ring for
  bimetallic coins and a b\*/a\* goldness ratio for the rest. The sign of the warmth step separates
  1 € from 2 € with no overlap at all over 74 coins.
- **Diameter** — one pixels-per-millimetre scale per image, recovered by minimising the total
  squared relative error against the official diameters of all coins in that frame.
- **Design** — Steger shape-based matching on sub-pixel refined crops, polarity-invariant,
  with a 3° rotation search.

A margin rule decides, per coin, whether the design or the diameter arbitrates.

## Repository contents

```
assignment_module_one.ipynb    the whole pipeline, with all outputs stored
coin_dataset/
  reference_set/               1cent.jpg 2cent.jpg 5cent.jpg 10cent.jpg
                               20cent.jpg 50cent.jpg 1euro.jpg 2euro.jpg
  target_set/                  image_1.jpg ... image_142.jpg
README.md
```

Everything needed to reproduce the results is here — clone the repo and run the notebook, no
other download and no path to edit.

The notebook is **self-contained** in the other sense too: the ground truth is inlined in a single
data cell near the top (collapse it — it is 117 lines of annotation, not logic), so nothing is
read from or written to an external `.json`. Running it creates two caches, `denoised_cache/` and
`candidates.json`, both regenerated from scratch if absent and both safe to delete.

## Running it

```
pip install opencv-python numpy scipy matplotlib tqdm
jupyter lab assignment_module_one.ipynb
```

Run the cells in order. The first execution is dominated by the non-local means denoising of 142
images; every run after that reuses `denoised_cache/`. The candidate cache is keyed by a signature
covering every parameter of phases 1–3, so changing a threshold invalidates it automatically —
`SCORING_VERSION` is bumped by hand when the scoring code itself changes.

Only two images are plotted, set by `EXAMPLES = [96, 44]` in the setup cell; change it to inspect
others. Full-dataset passes report through a single tqdm bar.

A caveat on exact reproducibility: the figures above were produced with the OpenCV version the
notebook was developed against. Other versions differ slightly in sub-pixel interpolation, which
moves per-coin accuracy by a few tenths of a point. The detection figures are stable.

## Notebook structure

| section | content |
|---|---|
| 1–2 | Setup, support functions, ground truth |
| 3 | Characterising the degradation: noise level, channel order |
| 4 | Detection, phases 1–6, plus a visual check |
| 5 | Classification: sub-pixel radius, colour, diameter, design, combination |
| 6 | Results, and sensitivity to the one free threshold |
| 7 | Validation and ablation — including **19 variants implemented, measured and rejected** |
| 8–9 | Limitations and the required per-image output |

Section 7.4 is worth a look: it lists every approach that was built and did not survive
measurement, with the reason each one fails. The pipeline keeps only what paid for itself.

## Output format

```
image_1.jpg - 3 coin(s) found:
  Coin 1 {value: 0.020}
  Coin 2 {value: 0.050}
  Coin 3 {value: 0.020}
  Partial Amount {value: 0.090}
...
Total Amount: 134.040 €
```

---

Academic work submitted for assessment. Please do not reuse as your own submission.
