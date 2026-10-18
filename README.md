# AI Security Literacy (AISL) — Grounding Pipeline

A reproducible analysis pipeline that grounds the six AI Security Literacy
competency dimensions in public benchmarks and nationally representative survey
data. Runs in Google Colab. Every artifact — dataset cache, model checkpoint,
figure, results table — persists to Google Drive, and every stage is resumable.

---

## Why this exists

AI arrived in schools faster than any guidance about how to use it safely.
Students are handed black-box systems and expected to intuit what those systems
can be trusted to do. Teachers are split between adopting AI and fearing it.
Districts are writing acceptable-use policies while the tools underneath them
change every few months.

The literacy frameworks that exist teach students to *understand* and *use* AI.
The cybersecurity frameworks that exist were written before AI-specific threats
existed. Neither treats **operating securely in an AI-mediated environment** as a
competency in its own right — with its own dimensions, its own developmental
progression, and its own teachable skills.

That competency is what AISL defines, across six dimensions spanning the whole
school ecosystem rather than students alone.

### Why grounding it matters

A competency framework that nobody has tested against data is just a taxonomy.
Anyone can assert that students should "recognize manipulation" or "calibrate
trust." The harder question is whether those phrases name **real, measurable
phenomena** — and if so, what the measurements actually show.

That is what this pipeline is for. It asks what public data can establish about
each dimension, and the answers are specific:

- **A detection label is a threshold decision, not a property of text.** Moving
  a toxicity cutoff from 0.3 to 0.7 swings the share of items labelled toxic by
  more than twenty percentage points. On real scam text messages, the same move
  cuts false alarms from 15 to 2 while doubling missed scams from 18 to 36. No
  threshold removes both errors; someone has to choose.
- **Harm often sits in the part of the text you have not read yet.** In a corpus
  built to contain toxicity in equal measure across four bands, most *prompt
  halves* score in the lowest band.
- **Fluent is not the same as reliable.** A small language model picks the true
  answer on a misconception benchmark only modestly above chance.
- **A robustness number describes the evaluation, not just the model.** The same
  classifiers attacked two ways give two very different verdicts, and the spread
  across independently trained models is several times wider than the sampling
  interval on the test set.
- **The policy vacuum is measurable.** In a nationally representative 2025 survey,
  about a third of U.S. public school teachers report that their school has an AI
  policy, and a quarter do not know whether it does. Teachers expect their own AI
  use to make teaching easier and students' use to make it harder.
- **The surrounding population is exposed and uneasy.** Most U.S. adults call
  online scams a major problem, receive scam texts weekly, and are concerned about
  AI misusing personal information — while reporting little control over AI in
  their lives.

Each of those is a lesson a learner can be walked through with real models and
real data, which is the point: the framework is meant to be taught, not just
cited.

### Why the engineering is part of the point

An earlier version of this pipeline produced a clean-looking adversarial
robustness curve from a model that was never trained, and a later one labelled
benchmark results with the wrong categories because two dataset configurations
did not share row order. Both outputs looked plausible.

Those failures are the same kind the framework's most advanced dimension teaches
students to catch, so the pipeline is built to make them structurally hard to
repeat: attack evaluation is gated on a clean-accuracy sanity check, joins are
made on content rather than position, blocked stages announce themselves rather
than emitting empty output, sampling is seeded, and every reported number is
written to a machine-readable results file instead of transcribed by hand.
Reproducibility here is not housekeeping — it is the subject matter.

---

## What it does

The pipeline runs in two tiers.

**Tier 2** grounds the technical dimensions from public sources. No configuration
needed — open and run. Every dataset downloads automatically.

**Tier 1** grounds the attitudinal dimensions from nationally representative
survey micro-data that you download yourself under free registration. Until the
files are present, its stages mark themselves `skipped`. See
[Tier 1 setup](#tier-1-setup).

| | Dimension | Tier 1 (surveys) | Tier 2 (public data) |
| --- | --- | --- | --- |
| **D1** | Permission & Norms | RAND ATP Fall 2025, Pew ATP W142 | — |
| **D2** | Data & Privacy Security | Pew ATP W152 | — |
| **D3** | Threat & Manipulation Recognition | Pew ATP W152, W168 | RealToxicityPrompts, SMS Spam Collection, BBQ |
| **D4** | Trust Calibration & Verification | Pew ATP W142, W152 | TruthfulQA + GPT-2 baseline, BBQ |
| **D5** | Adversarial & Technical Understanding | — | CIFAR-10 + FGSM/PGD, 3 seeds |
| **D6** | Disposition & Agency | RAND ATP Fall 2025, Pew ATP W152 | — |

---

## Quick start

Open `AISL_analysis_poc.ipynb` in Colab and run all cells top to bottom.

```
Runtime > Change runtime type > GPU     # optional; D5 training is ~10x faster
Runtime > Run all
```

The first run downloads CIFAR-10 (~170 MB), the Hugging Face corpora, GPT-2 and
the SMS Spam Collection. **All of them cache to Drive and are never
re-downloaded.** The run is safe to interrupt at any point — see
[Resuming a run](#resuming-a-run).

If you add survey files to Drive while a Colab session is open, restart the
session (`Runtime > Restart session`) before rerunning: Colab does not always see
files added after Drive was mounted.

---

## Output layout

Everything persistent lives under `MyDrive/AISL_data/`. Nothing of value is
written to the ephemeral Colab disk.

```
AISL_data/
├── cache/
│   ├── hf/                    # Hugging Face datasets and GPT-2
│   └── torch/                 # CIFAR-10
├── checkpoints/
│   └── cifar10_cnn_seed{N}.pt # one per seed, written after every epoch, atomically
├── figures/                   # 300-dpi JPGs
├── tables/                    # CSV + JSON + TeX macro exports
├── logs/
│   └── status.json            # per-stage completion record
└── microdata/
    ├── sms/                   # SMS Spam Collection (downloaded automatically)
    ├── pew/                   # <- you place the Pew ATP .sav files here
    └── rand/                  # <- you place the RAND ATP .dta file here
```

Results are exported in three formats: `aisl_results.json` (machine-readable),
per-stage `.csv` files, and `aisl_results.tex` (a set of TeX macros, one per
reported value, for downstream typesetting).

---

## Stages

| Stage ID | Dim. | Data | Outputs |
| --- | --- | --- | --- |
| `d3_rtp` | D3 | `allenai/real-toxicity-prompts` | `d3_rtp_toxicity.jpg`, `d3_rtp_summary.csv` |
| `d4_truthfulqa` | D4 | `truthfulqa/truthful_qa` (`generation`) | `d4_truthfulqa_categories.jpg` + CSV |
| `d4_baseline` | D4 | TruthfulQA (`multiple_choice`) + `gpt2` | `d4_truthfulqa_baseline.jpg` + CSV |
| `d34_bbq` | D3, D4 | `walledai/BBQ` (11 attribute configs) | `d34_bbq_configs.jpg`, `d34_bbq_counts.csv` |
| `d5_train` | D5 | CIFAR-10, 3 seeds | `cifar10_cnn_seed{N}.pt` |
| `d5_fgsm` | D5 | CIFAR-10 + `torchattacks` | `d5_fgsm_sweep.jpg` + CSV |
| `d3_threshold` | D3 | RealToxicityPrompts | `d3_threshold_sensitivity.jpg` + CSV |
| `d5_pgd` | D5 | CIFAR-10 + `torchattacks` | `d5_attack_comparison.jpg` + CSV |
| `d3_sms` | D3 | UCI SMS Spam Collection | `d3_sms_threshold.jpg` + CSV |
| `tier1_pew_inventory` | Tier 1 | Pew ATP files in `microdata/pew/` | `pew_item_inventory.csv` |
| `tier1_rand_inventory` | Tier 1 | RAND file in `microdata/rand/` | `rand_item_inventory.csv` |
| `tier1_mapped` | D1–D4, D6 | Pew ATP + RAND ATP | `tier1_survey_evidence.jpg`, `tier1_mapped_results.csv` |

Two further cells write summary tables (`grounding_matrix`, `alignment_matrix`)
and the TeX macro file, and a final cell prints the full run report. An optional
cell after the Tier 1 stage redraws `tier1_survey_evidence.jpg` for print from the
saved results table without recomputing anything.

Each stage records `ok` / `failed` / `skipped` in `logs/status.json` with a
timestamp and summary.

---

## Resuming a run

Completed stages skip themselves. Training resumes from the last saved epoch.

```python
status_report()              # what has run, when, and what it produced
reset_stage('d5_fgsm')       # force one stage to re-run
reset_stage()                # force everything to re-run
```

Checkpoints, status writes and the results file go through `os.replace`, so an
interrupted runtime cannot leave a half-written file behind.

---

## Tier 1 setup

Tier 1 uses four survey files, each free to download after registration. Place
them in the folders below; the stages find them automatically.

**Pew Research Center — American Trends Panel** (U.S. adults) → `microdata/pew/`

1. Create a free account at <https://www.pewresearch.org/datasets/>.
2. Download **Wave 142** (Feb 2024), **Wave 152** (Aug 2024) and **Wave 168**
   (Apr 2025).
3. Unzip each and place the SPSS `.sav` files in `microdata/pew/`.

**RAND American Educator Panels — American Teacher Panel** (U.S. public school
teachers) → `microdata/rand/`

1. Register at the RAND Survey Panels data portal, <https://www.rspdata.org>.
2. Open **2025 ATP Gallup Fall Survey (GAL1025T)**.
3. Download the **Stata** data file (not the CSV — the Stata file carries the
   question and answer labels), plus the codebook and survey instrument.
4. Place the `.dta` file in `microdata/rand/`. Do not also place the CSV there.

The inventory stages list every relevant item in each file with its weighted
distribution. The mapped stage then computes, for each item assigned to a
dimension, the weighted share giving the stated response and a 95% interval from
the Kish effective sample size. The response categories counted for each item are
fixed in the `SPEC` table in that cell and printed beside every estimate, so each
number can be checked against the questionnaire.

---

## Changes

### v3

- **D5 now replicates across three independently trained models** and evaluates
  on the full 10,000-image test set, with 95% Wilson intervals on every value.
  Attacks are seeded per model and cuDNN runs deterministically, so reruns are
  bit-stable.
- **D4 gained a measured failure rate.** GPT-2 is scored on TruthfulQA's
  multiple-choice configuration against a stated chance level. Categories are
  joined to questions by question text: the two configurations do not share row
  order, and joining by position had silently mislabelled every category.
- **D3 gained real scam text.** A threshold analysis on the UCI SMS Spam
  Collection, using the same multi-hot input representation as a deployable
  in-browser scam classifier.
- **Tier 1 now runs on real survey data** — three Pew American Trends Panel waves
  and the RAND American Teacher Panel Fall 2025 survey — replacing the earlier
  placeholder item map.
- **The TeX macro export keeps every stage's values.** Values written by the later
  stages survive the main export instead of being overwritten.

### v2

Three defects in the original notebook produced results that looked valid and
were not.

**Figures were written to the ephemeral disk.** `FIGDIR` was a relative path, so
`savefig` wrote to the Colab VM while Drive sat mounted and unused — printing a
success message every time. All output paths are now rooted at Drive, and
`savefig` verifies the file exists and is non-empty before returning.

**The FGSM sweep ran against an untrained model.** Clean CIFAR-10 accuracy was
12% — chance for ten classes — so the accuracy-vs-ε curve measured noise being
perturbed rather than an attack effect. The model is now actually trained, and a
sanity gate raises instead of plotting if clean accuracy falls below
`MIN_CLEAN_ACC`.

**BBQ never loaded.** The call requested `split='test'`, which `walledai/BBQ`
does not have — it partitions by social attribute. A bare `except` swallowed the
exception, so no BBQ result ever existed. The loader now enumerates the eleven
attribute configs and reports per-config failures individually.

Also: RTP sampling changed from `select(range(20000))` — the first 20k rows in
file order — to a seeded random sample, and a PGD comparison and a threshold
sensitivity analysis were added.

---

## Reproducibility

- Global seed set for `random`, `numpy` and `torch`; each D5 model and attack is
  seeded individually, and cuDNN runs in deterministic mode.
- Dataset caches pinned to Drive; downloads happen once.
- Every reported number is written to `tables/aisl_results.json` and exported as
  a TeX macro; none is transcribed by hand.
- Survey estimates are weighted with each survey's own weight variable, detected
  automatically and printed with the results.
- **Before archiving results:** pin package versions and record Hugging Face
  dataset revision hashes.

### Dependencies

Colab provides everything except `torchattacks`, which the notebook installs
(pinned to 3.5.1, without its dependencies so it cannot downgrade the
environment), and `pyreadstat`, which the Tier 1 stages install on first use to
read `.sav` and `.dta` files.

---

## Datasets

| Dataset | Source | Access |
| --- | --- | --- |
| [RealToxicityPrompts](https://huggingface.co/datasets/allenai/real-toxicity-prompts) | Hugging Face | Apache-2.0; downloaded automatically |
| [TruthfulQA](https://huggingface.co/datasets/truthfulqa/truthful_qa) | Hugging Face | Apache-2.0; downloaded automatically |
| [BBQ](https://huggingface.co/datasets/walledai/BBQ) | Hugging Face | Open — see dataset card; downloaded automatically |
| CIFAR-10 | `torchvision` | Public; downloaded automatically |
| [SMS Spam Collection](https://archive.ics.uci.edu/dataset/228/sms+spam+collection) | UCI ML Repository | CC BY 4.0; downloaded automatically |
| [GPT-2](https://huggingface.co/gpt2) (model) | Hugging Face | MIT; downloaded automatically |
| [Pew ATP Waves 142, 152, 168](https://www.pewresearch.org/datasets/) | Pew Research Center | Free account; terms prohibit redistribution |
| RAND ATP Fall 2025 (GAL1025T) | [RAND Survey Panels data portal](https://www.rspdata.org) | Free registration; terms prohibit redistribution |

Survey micro-data is **not** redistributed in this repository. Obtain it directly
from the providers under their own terms.
