# Estimating influenza vaccine effectiveness from Twitter/X

Code and de-identified data accompanying the paper on estimating influenza
vaccine effectiveness (VE) from social media.

> **Abstract.** Influenza vaccine effectiveness (VE) is estimated from a limited
> number of clinics using a test-negative design. These standard estimates face
> geographic, temporal, and operational constraints. Using Twitter/X data, we
> applied few-shot chain-of-thought prompting to identify self-reported
> vaccination status and influenza test results, then implemented a
> test-negative-like design to estimate VE. Our estimates fell within the range
> of interim reports and could complement current systems, improving
> feasibility, timeliness, and scalability.

## Purpose

This repository lets others reproduce the analysis in the paper: it takes
tweet-level classifier outputs (self-reported flu vaccination and flu-test
results) and CDC FluView surveillance data, builds user-level case/control
tables, and estimates influenza VE with a test-negative-like design (logistic
regression of test-positive status on vaccination status). It produces the
manuscript's descriptive tables, epidemic curves, and VE figures, including
sequential (cumulative-over-time) VE estimates.

It does **not** include the upstream tweet collection or the LLM prompting /
labeling pipeline; it starts from the classifier output. See
[Known limitations for reproducibility](#known-limitations-for-reproducibility).

## Repository structure

```
.
├── README.md                     This file
├── LICENSE                       MIT license (code)
├── CITATION.cff                  How to cite
├── requirements.txt              Python requirements (standard library only)
├── analysis/
│   └── vaccine_effectiveness_analysis.Rmd   Main analysis (all tables + figures)
├── scripts/
│   ├── label_uncertainty_probe.py   Robustness check: hedged-reasoning share
│   ├── prepare_public_data.py       How the public data was de-identified
│   └── unzip_data.sh                Unzip data/tweets/*.zip in place
├── environment/
│   └── install.R                 Installs the required R packages
├── data/
│   ├── README.md                 Data dictionary (read this)
│   ├── tweets/                   Tweet-level labels (.zip; unzip before running)
│   ├── user_level/               Derived user-level vaccine/test status
│   └── surveillance/             CDC FluView weekly surveillance
└── outputs/
    └── figures/                  Generated figures (+ manual_layout/ source)
```

## Software requirements

- **R** >= 4.1.0 with the packages listed in `environment/install.R`
  (tidyverse-style packages plus `here`, `egg`, `patchwork`, `cowplot`,
  `broom`, `rmarkdown`, `knitr`).
- **Python** >= 3.8 (standard library only; nothing to `pip install`).
- **unzip** (to expand the tweet archives) and a LaTeX-free PDF device is fine —
  figures are written with `ggsave`.

Exact package versions are not pinned. For a byte-for-byte reproducible R
environment we recommend [`renv`](https://rstudio.github.io/renv/):
`renv::init()` then `renv::snapshot()`, and commit the resulting `renv.lock`.

## Installation

```bash
# 1. Get the code
git clone <your-repo-url>
cd <repo>

# 2. R packages
Rscript environment/install.R

# 3. Python (no third-party packages; this just confirms the version)
python3 --version        # needs >= 3.8

# 4. Expand the tweet-level data
bash scripts/unzip_data.sh
```

## Reproducing the analysis

### 1. Main analysis, tables, and figures (R)

Knit the R Markdown document from the repository root:

```bash
Rscript -e 'rmarkdown::render("analysis/vaccine_effectiveness_analysis.Rmd")'
```

or open it in RStudio and click **Knit**. Paths are resolved with the `here`
package relative to the repository root, so it does not matter what your working
directory is, as long as you run it from inside the repository.

This regenerates the descriptive tables, the chi-square tests, the crude
(unadjusted) VE estimates, the tweet-volume epidemic curves, the sequential VE
plots, and Figure 1 panels into `outputs/figures/`.

### 2. Reasoning-uncertainty robustness check (Python)

```bash
python3 scripts/label_uncertainty_probe.py
```

This reports, per season and arm, the share of committed Positive/Negative/Past
labels whose model reasoning contains an epistemic-uncertainty ("hedge") marker,
plus a pooled total. It reads the `.csv` files if present, or the `.csv.zip`
archives directly if not.

## Expected inputs and outputs

**Inputs** (see `data/README.md` for the full data dictionary):
- `data/tweets/*.csv` — tweet-level classifier labels and reasoning.
- `data/user_level/*.csv` — derived user-level vaccine/test status (2025–26).
- `data/surveillance/*.csv` — CDC FluView weekly percent-positive.

**Outputs** written to `outputs/figures/`:

| File | Content |
|------|---------|
| `ve_tab.pdf` | Case/control counts by vaccine status and season. |
| `tw_weekly.pdf` | Weekly tweet and unique-user volume. |
| `plot_epicurve_mon.pdf` | Monthly epidemic curve of flu-related tweets. |
| `plot_ve.pdf` | Crude (unadjusted) VE point estimate with 95% CI. |
| `ve_sequential_tnd_week.pdf` / `_month.pdf` | Sequential (cumulative) VE over time. |
| `ve_weekly_counts_panel_{2425,2526}.pdf` | Cumulative-by-week case/control count tables. |
| `ve_weekly_ve_panel_{2425,2526}.pdf` | Cumulative-by-week VE forest panels. |
| `Fig1A.pdf` | Manuscript Fig 1A: cumulative VE point-ranges over time. |
| `Fig1B.pdf` / `Fig1B_with_legend.pdf` | Manuscript Fig 1B: tweet volume with CDC percent-positive overlay. |
| `Figure.png` | Final composed Figure 1 (Fig1A + Fig1B laid out manually). |

The final `Figure.png` was composed manually in Affinity Designer; the editable
source is `outputs/figures/manual_layout/Figure.afdesign`. The individual panels
(`Fig1A.pdf`, `Fig1B.pdf`) are fully regenerated by the R code.

## Method notes and assumptions

- **Test-negative-like design.** VE is estimated as `(1 - OR) * 100`, where the
  odds ratio comes from a logistic regression of influenza-test-positive status
  on vaccination status. Confidence intervals are transformed accordingly (the
  upper OR bound maps to the lower VE bound).
- **Vaccinated definition.** A user is `vaccinated = 1` if they report a prior
  ("Past") vaccination, or a current-season shot dated at least
  `vax_window_days` (default **14 days**) before their flu-test evidence date,
  to respect the immunity lag. This threshold is a configurable parameter at the
  top of the R Markdown.
- **De-duplication.** Where a user appears in more than one 2025–26 snapshot, the
  later snapshot is authoritative; user-level rows are enforced unique.
- **Sequential VE** is only fit once a cumulative window contains at least
  `min_flu_events` (default **100**) influenza cases, to avoid unstable estimates.

## Data availability and privacy

Tweet text is **not** redistributed. Per X's Developer Policy, only Post IDs and
pseudonymized User IDs are shared; the classifier labels and reasoning are the
authors' derived annotations. See `data/README.md` for the full de-identification
description and re-hydration instructions. Users intending to re-collect the
underlying tweets need their own X API access and must follow X's terms.

## Known limitations for reproducibility

These are flagged so reviewers and re-users know the boundaries of what this
repository reproduces:

1. **Upstream labeling not included.** The tweet collection and the few-shot
   chain-of-thought LLM prompting/inference that produced the labels are not in
   this repository; the analysis starts from the classifier output.
2. **2025–26 user-level derivation not included.** The 2024–25 season's
   user-level case/control table is built inside the R analysis from tweet-level
   files, but the 2025–26 season starts from pre-computed `user_level/` files
   whose derivation code is not included. As a result, the two seasons are not
   produced by an identical in-repo pipeline.
3. **No pinned dependency versions.** See the `renv` recommendation above.
4. **R execution not machine-verified here.** The Python probe was run and its
   output verified; the R Markdown was refactored by static review (paths,
   filenames, parameters) but should be knit once in your environment to confirm.
5. **Reasoning traces.** `*_reasoning` columns are model output that may echo
   tweet content; review before public release (see `data/README.md`).

## License and citation

Code is released under the [MIT License](LICENSE). Please cite the paper and this
repository (see [`CITATION.cff`](CITATION.cff)). Update the author list, version,
and DOI in `CITATION.cff` before release.
