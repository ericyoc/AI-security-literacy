# AI Security Literacy (AISL)

Empirical-grounding pipeline and manuscript source for **"AI Security Literacy:
Ecosystem-based Competency Framework and K-12 Learning Platform."**

The AISL framework defines six competency dimensions for operating securely in an
AI-mediated school environment. This repository contains the analysis that grounds
those dimensions in public data, and the LaTeX manuscript that consumes its output.

> **Scope.** No human-subjects data are collected or analyzed. NeuralFortify is
> described illustratively; no efficacy claims are made. See
> [Ethics](#ethics-and-governance).

---

## Contents

| Path | What it is |
| --- | --- |
| `AISL_analysis_v2.ipynb` | Grounding pipeline — checkpointed, resumable, Drive-persistent |
| `AISL_article.tex` | Manuscript source |
| `references.bib` | Bibliography |
| `figures/` | 300-dpi JPGs consumed by the manuscript |
| `tables/` | CSV exports + `aisl_results.tex` (auto-generated LaTeX macros) |

---

## The six dimensions

| | Dimension | Grounded by |
| --- | --- | --- |
| **D1** | Permission & Norms | Tier 1 — RAND AEP, Harvard AI Policies |
| **D2** | Data & Privacy Security | Tier 1 — RAND AEP |
| **D3** | Threat & Manipulation Recognition | Tier 2 — RealToxicityPrompts, BBQ |
| **D4** | Trust Calibration & Verification | Tier 2 — TruthfulQA, BBQ |
| **D5** | Adversarial & Technical Understanding | Tier 2 — RobustBench, CIFAR-10 + FGSM |
| **D6** | Disposition & Agency | Tier 1 — RAND AEP, Gallup |

**Tier 1** (attitudinal) requires survey micro-data you download yourself.
**Tier 2** (technical) runs from public sources with no manual setup.

---

## Quick start

Open `AISL_analysis_v2.ipynb` in Google Colab and run all cells top to bottom.
Tier 2 needs no configuration. The run is safe to interrupt — see
[Resuming](#resuming-a-run).

```
Runtime > Change runtime type > GPU     # optional; D5 training is ~10x faster
Runtime > Run all
```

First run downloads CIFAR-10 (~170 MB) and the Hugging Face corpora. **Both are
cached to Drive and never re-downloaded.**

### Drive layout

Everything persistent lives under `MyDrive/AISL_data/`. Nothing of value is
written to the ephemeral Colab disk.

```
AISL_data/
├── cache/
│   ├── hf/              # HF dataset cache (TruthfulQA, RTP, BBQ)
│   └── torch/           # CIFAR-10
├── checkpoints/
│   └── cifar10_cnn.pt   # written after every epoch, atomically
├── figures/             # 300-dpi JPGs for the manuscript
├── tables/              # CSVs + aisl_results.tex + aisl_results.json
├── logs/
│   └── status.json      # per-stage completion record
└── microdata/           # <- you place RAND / Harvard / Gallup files here
```

---

## Pipeline stages

| Stage ID | Dim. | Source | Outputs |
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

Reruns are cheap. Completed stages skip themselves; training resumes from the
last saved epoch.

```python
status_report()              # what has run, when, and what it produced
reset_stage('d5_fgsm')       # force one stage to re-run
reset_stage()                # force everything to re-run
```

Checkpoints and status writes are atomic (`os.replace`), so an interrupted
runtime cannot leave a half-written file behind.

---

## Manuscript integration

The notebook writes **`tables/aisl_results.tex`**, a file of LaTeX macros holding
every computed number:

```latex
\newcommand{\dThreeN}{19,877}
\newcommand{\dThreeMean}{0.087}
\newcommand{\dFiveClean}{74.1\%}
...
```

`AISL_article.tex` inputs it, so the manuscript quotes results rather than
transcribing them:

```latex
From a seeded random sample ..., $n = \dThreeN$ prompts carried non-null
Perspective toxicity scores. Mean prompt toxicity was \dThreeMean.
```

**To update the paper after a notebook run**, copy two things out of Drive:

```
AISL_data/figures/*.jpg           ->  figures/
AISL_data/tables/aisl_results.tex ->  tables/
```

Recompile. Every number updates. No manual transcription anywhere.

### The manuscript always builds

- Missing `aisl_results.tex` → macros fall back to a red `??`.
- Missing figure → `\aislfig` prints a boxed **MISSING FIGURE** placeholder
  naming the file and the stage that produces it.

Nothing absent can break the build, and nothing absent can pass unnoticed.

---

## Tier 1 setup

Tier 1 is gated on two inputs and will not produce output until both are present.

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

Until then the stage marks itself `skipped`, lists its blockers, and the
manuscript reports Tier 1 status as *not executed* — it does **not** emit an
empty figure.

---

## Changes in v2

Three defects in the original notebook produced results that looked valid and
were not.

**Figures were written to the ephemeral disk.** `FIGDIR` was a relative path, so
`savefig` wrote to the Colab VM while Drive sat mounted and unused. It printed a
success message every time. All output paths are now rooted at Drive, and
`savefig` verifies the file exists and is non-empty before returning.

**The FGSM sweep ran against an untrained model.** Clean CIFAR-10 accuracy was
12% — chance for ten classes — so the accuracy-vs-ε curve measured noise being
perturbed, not an attack effect. The model is now actually trained, and a sanity
gate raises rather than plotting if clean accuracy falls below `MIN_CLEAN_ACC`.

**BBQ never loaded.** The call requested `split='test'`, which `walledai/BBQ`
does not have — it partitions by social attribute. The exception was swallowed by
a bare `except`, so no BBQ result ever existed. The loader now enumerates the
eleven attribute configs and reports per-config failures individually.

Also: RTP sampling changed from `select(range(20000))` — the first 20k rows in
file order — to a seeded random sample, so the reported rate is an unbiased
estimate. The seed is recorded in the output.

---

## Reproducibility

- Global seed set for `random`, `numpy`, and `torch`.
- Dataset caches pinned to Drive; downloads happen once.
- Every reported number is written to `tables/aisl_results.json` and mirrored
  into the LaTeX macro file.
- **Before submission:** pin package versions and record Hugging Face dataset
  revision hashes.

### Dependencies

Colab provides everything except `torchattacks` (installed by the notebook).
Tier 1 additionally needs `pingouin` and `factor-analyzer`.

---

## Data availability

| Source | License / access |
| --- | --- |
| RealToxicityPrompts | Apache-2.0 |
| TruthfulQA | Apache-2.0 |
| BBQ | Open (see dataset card) |
| CIFAR-10 | Public |
| RobustBench | Public leaderboard — cited, not recomputed |
| RAND / Harvard / Gallup panels | Registration or application |

Survey micro-data is **not** redistributed in this repository.

---

## Ethics and governance

No human-subjects data are collected or analyzed in this work. NeuralFortify is
described illustratively; no efficacy or learning claims are made.

Any future validation study would require IRB approval and parental consent, and
would operate under COPPA/FERPA with de-identified, sandboxed data release. The
platform's username-only identity model is privacy-by-architecture: no PII is
collected from minors.

---

## Citation

```bibtex
@misc{neuralfortify2026,
  author       = {Yocam, Eric and Vaidyan, Varghese},
  title        = {NeuralFortify / Defend the Black Box: A Cloud-Based {AI}
                  Security Learning Platform for {K-12}},
  year         = {2026},
  howpublished = {Dakota State University},
  note         = {NSF STEM K-12 project}
}
```
