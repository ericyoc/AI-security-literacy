# AI Security Literacy (AISL) — Grounding Pipeline

A reproducible analysis pipeline that grounds the six AI Security Literacy
competency dimensions in public datasets. Runs in Google Colab. Every artifact —
dataset cache, model checkpoint, figure, results table — persists to Google
Drive, and every stage is resumable.

---

## What it does

The pipeline runs in two tiers.

**Tier 2** grounds the technical dimensions from public sources. No configuration
needed — clone, open, run.

**Tier 1** grounds the attitudinal dimensions from educator/youth survey
micro-data that you download yourself. It stays gated until you supply the files
and a codebook mapping. See [Tier 1 setup](#tier-1-setup).

| | Dimension | Tier | Source |
| --- | --- | --- | --- |
| **D1** | Permission & Norms | 1 | RAND AEP, Harvard AI Policies |
| **D2** | Data & Privacy Security | 1 | RAND AEP |
| **D3** | Threat & Manipulation Recognition | 2 | RealToxicityPrompts, BBQ |
| **D4** | Trust Calibration & Verification | 2 | TruthfulQA, BBQ |
| **D5** | Adversarial & Technical Understanding | 2 | RobustBench, CIFAR-10 + FGSM |
| **D6** | Disposition & Agency | 1 | RAND AEP, Gallup |

---

## Quick start

Open `AISL_analysis_v2.ipynb` in Colab and run all cells top to bottom.

```
Runtime > Change runtime type > GPU     # optional; D5 training is ~10x faster
Runtime > Run all
```

First run downloads CIFAR-10 (~170 MB) and the Hugging Face corpora. **Both cache
to Drive and are never re-downloaded.** The run is safe to interrupt at any
point — see [Resuming a run](#resuming-a-run).

---

## Output layout

Everything persistent lives under `MyDrive/AISL_data/`. Nothing of value is
written to the ephemeral Colab disk.

```
AISL_data/
├── cache/
│   ├── hf/              # HF dataset cache (TruthfulQA, RTP, BBQ)
│   └── torch/           # CIFAR-10
├── checkpoints/
│   └── cifar10_cnn.pt   # written after every epoch, atomically
├── figures/             # 300-dpi JPGs
├── tables/              # CSV + JSON + TeX macro exports
├── logs/
│   └── status.json      # per-stage completion record
└── microdata/           # <- you place RAND / Harvard / Gallup files here
```

Results are exported in three formats: `aisl_results.json` (machine-readable),
per-stage `.csv` files, and `aisl_results.tex` (a set of `\newcommand` macros for
downstream typesetting).

---

## Stages

| Stage ID | Dim. | Dataset | Outputs |
| --- | --- | --- | --- |
| `d3_rtp` | D3 | `allenai/real-toxicity-prompts` | `d3_rtp_toxicity.jpg`, `d3_rtp_summary.csv` |
| `d4_truthfulqa` | D4 | `truthfulqa/truthful_qa` (`generation`) | `d4_truthfulqa_categories.jpg` + CSV |
| `d34_bbq` | D3, D4 | `walledai/BBQ` (11 attribute configs) | `d34_bbq_configs.jpg`, `d34_bbq_counts.csv` |
| `d5_train` | D5 | CIFAR-10 | `cifar10_cnn.pt` |
| `d5_fgsm` | D5 | CIFAR-10 + `torchattacks` | `d5_fgsm_sweep.jpg`, `d5_fgsm_sweep.csv` |
| `tier1` | D1, D2, D6 | Your survey micro-data | EFA loadings, α table, by-role figure |

Each stage records `ok` / `failed` / `skipped` in `logs/status.json` with a
timestamp and summary. A final cell prints the full run report.

---

## Resuming a run

Completed stages skip themselves. Training resumes from the last saved epoch.

```python
status_report()              # what has run, when, and what it produced
reset_stage('d5_fgsm')       # force one stage to re-run
reset_stage()                # force everything to re-run
```

Checkpoints and status writes go through `os.replace`, so an interrupted runtime
cannot leave a half-written file behind.

---

## Tier 1 setup

Tier 1 is gated on two inputs and produces nothing until both are present.

**1. Download the micro-data** into `AISL_data/microdata/`:

- [RAND American Educator Panels / American Youth Panel](https://www.rand.org/education-employment-infrastructure/survey-panels/aep/access-data.html) — registration; some files restricted
- Harvard *Educator AI Policies and Challenges*, Gallup *Teacher Workload and AI* — via the RAND survey catalog

**2. Complete `ITEM_MAP`** with exact variable names from the codebook:

```python
ITEM_MAP = {
  'D1_permission_norms':   ['TODO_var_policy_clarity', 'TODO_var_integrity_rules'],
  'D2_data_privacy':       ['TODO_var_data_consent'],
  ...
}
ROLE_VAR = 'TODO_var_role'
```

Until then the stage marks itself `skipped` and prints its blockers. It does
**not** emit an empty figure.

---

## Changes in v2

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
file order — to a seeded random sample, so the reported rate is an unbiased
estimate. The seed is recorded in the output.

---

## Reproducibility

- Global seed set for `random`, `numpy`, and `torch`.
- Dataset caches pinned to Drive; downloads happen once.
- Every reported number is written to `tables/aisl_results.json`.
- **Before publishing results:** pin package versions and record Hugging Face
  dataset revision hashes.

### Dependencies

Colab provides everything except `torchattacks`, which the notebook installs.
Tier 1 additionally requires `pingouin` and `factor-analyzer`.

---

## Datasets

| Dataset | Access |
| --- | --- |
| [RealToxicityPrompts](https://huggingface.co/datasets/allenai/real-toxicity-prompts) | Apache-2.0 |
| [TruthfulQA](https://huggingface.co/datasets/truthfulqa/truthful_qa) | Apache-2.0 |
| [BBQ](https://huggingface.co/datasets/walledai/BBQ) | Open — see dataset card |
| CIFAR-10 | Public |
| [RobustBench](https://robustbench.github.io/) | Public leaderboard — cited, not recomputed |
| RAND / Harvard / Gallup panels | Registration or application |

Survey micro-data is **not** redistributed in this repository. Obtain it directly
from the providers under their own terms.
