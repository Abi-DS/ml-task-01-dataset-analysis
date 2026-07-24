# WikiArt Dataset Analysis

A production-quality structural analysis and data-quality audit of a 27-style
WikiArt painting corpus (**81,444 images**) and its two label files,
`classes.csv` and `wclasses.csv`.

The deliverable is a single, sequentially-executable notebook
(`notebooks/analysis.ipynb`) supported by a written report
(`reports/summary.pdf`). Every finding below was established by directly
inspecting the data — nothing is inferred from folder names alone.

---

## Repository structure

```
ml-task-01-dataset-analysis/
├── notebooks/
│   └── analysis.ipynb     # main deliverable — runs top-to-bottom
├── reports/
│   └── summary.pdf        # written findings with embedded figures
├── README.md
├── requirements.txt
└── .gitignore
```

The notebook is **self-contained** — all logic lives in small, typed functions
inside `analysis.ipynb`, so there are no helper modules to install. The dataset
is **not** included in this repository (see below).

---

## Dataset setup

This repository does **not** contain the WikiArt images or label files. Obtain
the dataset separately and place it anywhere on disk **outside** this repo. The
folder you point to must directly contain:

```
<your-dataset-folder>/
├── classes.csv
├── wclasses.csv
├── Abstract_Expressionism/
├── Action_painting/
├── ...                     # 27 style sub-folders in total
└── Ukiyo_e/
```

### Configuring the dataset path

The notebook locates the data through a single variable, `DATASET_ROOT`, at the
top of the configuration cell (Section 0). Set it in **either** way:

1. **Environment variable** (recommended — no code edits):

   ```bash
   # macOS / Linux
   export WIKIART_DATASET_ROOT=/path/to/wikiart

   # Windows (PowerShell)
   $env:WIKIART_DATASET_ROOT = "C:\path\to\wikiart"
   ```

2. **Edit one line** in the notebook's configuration cell:

   ```python
   DATASET_ROOT = Path(os.environ.get("WIKIART_DATASET_ROOT", "data/wikiart"))
   #                                                            ^^^^^^^^^^^^
   #                                    replace with your dataset path
   ```

If the path is wrong, the notebook fails fast in Section 1 with a clear message
telling you exactly what to fix.

---

## Installation

Requires **Python 3.11+**.

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
python -m pip install -r requirements.txt
```

---

## Running the notebook

```bash
# 1. Point the notebook at your dataset (see "Dataset setup" above)
export WIKIART_DATASET_ROOT=/path/to/wikiart

# 2. Launch Jupyter and run all cells top-to-bottom
jupyter lab notebooks/analysis.ipynb      # then Run All
```

The notebook executes sequentially from top to bottom with no hidden state and
works regardless of the directory Jupyter is launched from.

**Runtime note.** The exhaustive image-integrity scan decodes every image and is
the slow step. During development, set `INTEGRITY_SAMPLE` in the configuration
cell to an integer (e.g. `4000`) for a fast partial pass; leave it `None` for
the full scan.

The notebook is committed with its executed outputs — every table, figure,
statistic, and sample image — so reviewers can read the full analysis without
running it. `reports/summary.pdf` presents the same results as a written report.
Re-running top-to-bottom reproduces the committed outputs exactly.

---

## Source-of-truth hierarchy

The analysis follows one explicit, documented hierarchy:

1. **Filesystem (the 27 style folders) — authoritative image inventory.** The
   only artifact that exactly represents the images on disk. The folder name is
   a reliable style label: it equals the primary label in `classes.csv` for
   **every** overlapping row (0 mismatches, verified).
2. **`classes.csv` — primary human-readable metadata.** Artist name, pixel
   dimensions, perceptual hash (`phash`), the official `train`/`test` split, and
   style labels. Joined onto the inventory as an *enrichment*.
3. **`wclasses.csv` — validation / reconciliation only.** A numeric-ID encoding
   with **no legend file**; its artist/genre IDs cannot be resolved to names.
   Used to cross-check coverage. Inconsistencies are reported transparently,
   never silently discarded or overwritten.

---

## Key findings

| Metric | Value |
|---|---|
| Images on disk | **81,444** (all `.jpg`) |
| Style folders | **27** |
| Named artists (`classes.csv`) | **1,119** |
| `classes.csv` rows / columns | 80,042 / 9 |
| `wclasses.csv` rows / columns | 81,444 / 4 |
| Corrupt / unreadable images | **0** (full structural scan of all 81,444; 0 truncated in a 1,500-image full-decode sample) |
| Colour modes | 100% `RGB` |
| Duplicate basenames across styles | **1,349 names** (2,698 files) |
| Perceptual-hash duplicate groups | 22 groups / 44 images (6 span >1 style) |
| Median resolution / aspect ratio | 2.55 MP / 0.93 (slightly portrait) |
| Official split | 63,998 train · 16,000 test · 44 `uncertain artist` · 1,402 unlabelled |

### Reconciliation (filesystem ↔ metadata)

A **two-stage join** is used: an exact-path match first (79,347 images), then an
encoding-robust fallback key applied only to the still-unmatched residual.

- **`classes.csv` fully reconciles to disk.** Stage 2 rescues **695** accented
  filenames, so all 80,042 rows map to a real image — **0 orphan rows** — and
  the total enriched count equals the classes row count exactly.
- **Unambiguous.** Confining the fuzzy match to the residual yields **0 residual
  key collisions**, so no image is ever assigned the wrong metadata. (Duplicate
  basenames across style folders do not collide, because the key includes the
  folder.)
- **1,402 disk images have no `classes.csv` metadata** — they exist only in
  `wclasses.csv`. They keep their filesystem style label and are reported.
- **`wclasses.csv` is a 1:1 superset of the disk** (every image is present).

---

## Data quirks handled (all confirmed by inspection)

- **`classes.csv`'s `genre` column actually holds the *style*** (mislabelled at
  source), stored as a Python-list literal, e.g. `['Impressionism']`. It is
  parsed into a clean `style_label`.
- **`genre` means different things in the two files.** In `wclasses.csv`,
  `genre` is a genuine, separate 11-category concept — but unlabelled, so it is
  left as-is.
- **Filename encoding mismatch.** Non-ASCII artist names are mangled by *two
  different* mojibake encodings (one on disk, another in `classes.csv`). Exact
  joins silently drop every accented name. `normalize_key` reduces paths to
  lowercase ASCII alphanumerics so both corruptions collapse to the same key;
  Unicode NFKD folding is deliberately **avoided** because it transliterates the
  two corrupt byte sequences inconsistently and fails to reconcile them.
- **Multi-label styles.** 1,400 images carry 2–3 style labels (1,398 have two,
  2 have three).

---

## Methodology notes

- **Modular & typed.** Each stage is a small function with type hints and a
  docstring; the notebook executes sequentially with no hidden state.
- **Reproducible.** All random sampling uses a fixed seed (`RANDOM_SEED = 42`).
- **Efficient on large data.** Directory-level aggregation avoids loading
  pixels; the integrity scan is I/O-bound and runs across a thread pool.
- **Graceful corruption handling.** A two-stage `verify()` + `load()` check
  records every failure (path + error) and never raises.
- **Two notions of duplicate.** Duplicate *basenames* across style folders
  (required) and, as an optional enhancement, perceptual-hash near-duplicates
  from the provided `phash` column.

---

## Assumptions & limitations

- No ID legend exists for `wclasses.csv`; artist and genre integer IDs are left
  unresolved rather than guessed.
- The two-stage join is validated, not assumed: the fuzzy fallback is confined
  to unmatched images and produces **0** residual key collisions, verified in
  the notebook.
- `width`/`height` come from `classes.csv` metadata rather than re-measured
  pixels, for efficiency; images without that metadata are excluded from
  resolution statistics and reported as such.
- Perceptual-hash duplicate detection covers only images present in
  `classes.csv`; its 22 groups match the duplicate hashes in `classes.csv`
  itself, independent of the join.
