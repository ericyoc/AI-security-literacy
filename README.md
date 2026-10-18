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

A pipeline can produce figures that look right and are not: a robustness curve
from a model that was never properly trained, or benchmark results attached to
the wrong categories because two files were joined by row position. Those are
the same kinds of failure the framework's most advanced dimension teaches
students to catch.

So the pipeline is built to make them structurally hard: attack evaluation is
gated on a clean-accuracy sanity check, joins are made on content rather than
position, blocked stages announce themselves rather than emitting empty output,
sampling is seeded, and every reported number is written to a machine-readable
results file instead of transcribed by hand. Reproducibility here is not
housekeeping — it is the subject matter.

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

Every dataset the pipeline analyzes, what it is, and why it is used.

### Tier 2 — public, downloaded automatically

| Dataset | What it is | Why it is used | Dim. | Source / access |
| --- | --- | --- | --- | --- |
| **RealToxicityPrompts** | ~99,000 sentence prompts, each split into a prompt half and a continuation and scored for toxicity by the Perspective API; parent sentences were sampled equally from four toxicity bands | Shows that a "toxic" label depends entirely on the chosen threshold, and that the opening fragment of a harmful sentence often looks harmless | D3 | [Hugging Face `allenai/real-toxicity-prompts`](https://huggingface.co/datasets/allenai/real-toxicity-prompts) · Apache-2.0 |
| **SMS Spam Collection** | 5,574 real SMS text messages, each labelled spam or legitimate | Measures the false-alarm vs. missed-scam trade-off of a scam detector at different thresholds, using the same multi-hot input representation as an in-browser scam classifier | D3 | [UCI ML Repository](https://archive.ics.uci.edu/dataset/228/sms+spam+collection) · CC BY 4.0 |
| **TruthfulQA** | 817 questions in 38 categories, each written so that a plausible-sounding false answer is the tempting one; generation and multiple-choice configurations | Shows that misconception-prone questions span many domains, and supplies the questions for a measured truthfulness rate | D4 | [Hugging Face `truthfulqa/truthful_qa`](https://huggingface.co/datasets/truthfulqa/truthful_qa) · Apache-2.0 |
| **GPT-2** (model) | OpenAI's 124M-parameter language model | Scored on TruthfulQA's multiple-choice questions to measure how often a fluent model picks the true answer, against a stated chance level | D4 | [Hugging Face `gpt2`](https://huggingface.co/gpt2) · MIT |
| **BBQ** | 58,492 question-answering items across 11 social-attribute categories (age, race, religion, and others) | Shows that bias in model answers can be broken down by attribute, and quantifies how unevenly the attributes are covered | D3, D4 | [Hugging Face `walledai/BBQ`](https://huggingface.co/datasets/walledai/BBQ) · open, see dataset card |
| **CIFAR-10** | 60,000 32×32 color images in 10 classes (50,000 train, 10,000 test) | Trains three small image classifiers and attacks them with single-step (FGSM) and ten-step (PGD) attacks, showing that a robustness number depends on the attack and on the particular trained model | D5 | `torchvision` · public |

### Tier 1 — survey micro-data, free registration, not redistributed

| Dataset | What it is | Why it is used | Dim. | Source / access |
| --- | --- | --- | --- | --- |
| **RAND American Teacher Panel, Fall 2025 (GAL1025T)** | Nationally representative, weighted survey of 2,012 U.S. public school teachers on careers and AI, fielded with Gallup | Measures whether schools have an AI policy and how clear it is, teachers' own AI use and support for it, and whether teachers expect their own vs. students' AI use to make teaching easier or harder | D1, D6 | [RAND Survey Panels data portal](https://www.rspdata.org) · free registration |
| **Pew American Trends Panel, Wave 142** (Feb 2024) | Nationally representative, weighted survey of U.S. adults | Measures norms of AI attribution (should AI credit its sources) and trust in information from ChatGPT | D1, D4 | [Pew datasets](https://www.pewresearch.org/datasets/) · free account |
| **Pew American Trends Panel, Wave 152** (Aug 2024) | Nationally representative, weighted survey of U.S. adults on AI | Measures concern about AI misusing personal information, impersonating people and spreading inaccurate information; perceived control over AI; and concern vs. excitement about AI | D2, D3, D4, D6 | [Pew datasets](https://www.pewresearch.org/datasets/) · free account |
| **Pew American Trends Panel, Wave 168** (Apr 2025) | Nationally representative, weighted survey of U.S. adults on online scams | Measures scam exposure, money lost to scams, self-assessed ability to protect oneself, and whether AI will make scams more common | D3 | [Pew datasets](https://www.pewresearch.org/datasets/) · free account |

Survey micro-data is **not** redistributed in this repository. Obtain it directly
from the providers under their own terms, and place the files as described in
[Tier 1 setup](#tier-1-setup).
