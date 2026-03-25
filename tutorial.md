# Tutorial: Designing a Generic Automated Research Workflow

This tutorial explains how to design a generic, end-to-end workflow that can take a brand-new public health dataset and automatically produce a JAMA Network Open–style paper (including tables, figures, references, and a compiled PDF) with minimal human intervention.

This is intentionally beginner-friendly and self-contained. You do *not* need prior experience with multi-agent systems, LaTeX, or LLM orchestration to follow the design.

> **Tip:** Want this to look like a real webpage? This repo includes an optional GitHub Pages version (Section 9) that renders this tutorial from `docs/` with a left sidebar.

## Authors

- Johnny Zhao
- Yicheng He
- Junhao Luo

## What you’ll learn

By the end, you should be able to:

- Break an end-to-end “paper generator” into **small stages** with explicit inputs/outputs
- Design **dataset-agnostic** logic (type-driven, validated assumptions)
- Add **guardrails** so the system fails softly and still produces a coherent draft
- Debug quickly by inspecting the exact artifact that a stage produced

## What this workflow produces

At a high level, the workflow produces four kinds of things:

- **Artifacts** (JSON): machine-readable stage outputs you can inspect and rerun from
- **Tables** (CSV): analysis results in a format you can open anywhere
- **Figures** (PNG): bounded-size plots that don’t explode on large datasets
- **Paper** (TeX + PDF): a compiled manuscript with references

---

## At a glance

| What you get | Where it lands |
|---|---|
| Intermediate “stage artifacts” (debuggable JSON) | `exam_paper/intermediate/` |
| Tables (CSV) and figures (PNG) | `exam_paper/output/tables/`, `exam_paper/output/figures/` |
| LaTeX source and compiled paper | `exam_paper/tex/paper.tex`, `exam_paper/output/paper.pdf` |
| “Stop after a stage” debugging | `--stop-after <stage>` |

---

## Table of contents

- [1) Why this workflow exists](#1-why-this-workflow-exists)
- [2) Getting started](#2-getting-started)
- [3) The system in action (a run walkthrough)](#3-the-system-in-action-a-run-walkthrough)
- [4) The building blocks](#4-the-building-blocks)
- [5) Workflow patterns that make it robust](#5-workflow-patterns-that-make-it-robust)
- [6) How we made it dataset-agnostic](#6-how-we-made-it-dataset-agnostic)
- [7) Customizing for your domain (how to extend safely)](#7-customizing-for-your-domain-how-to-extend-safely)
- [8) Appendix: file reference and troubleshooting](#8-appendix-file-reference-and-troubleshooting)
- [9) Publishing as a GitHub Pages site (optional)](#9-publishing-as-a-github-pages-site-optional)

> **Note:** If you’re in a hurry: start with Section 2 (run), then Section 3.3 (debug), then Appendix 8.2 (common failures).

---

## 1) Why this workflow exists

### 1.1 The problem (real-world reality)

An “automatic paper generator” fails in predictable ways when the dataset is unfamiliar:

- **Context mismatch**: the dataset is not the sample dataset, so any column-name assumptions break.
- **Quality drift**: even if it runs, the paper becomes vague (“the data shows…”) and the methods do not match the available variables.
- **Brittle plots**: large datasets can create unreadably large figures or crash image generation.
- **Hard-to-debug failures**: if everything is one big script, it’s impossible to locate what went wrong quickly.

Our solution is to treat paper generation like a production pipeline: **small stages + saved artifacts + verification + fallbacks**.

### 1.2 What makes our workflow different

We focus on four engineering principles:

1. **Artifact-driven pipeline**: each stage writes a structured output to disk (JSON/CSV/PNG/TeX). If a stage fails, you inspect the artifact and rerun just that stage.
2. **Type-driven logic instead of name-driven logic**: we select variables by their *data type* and structure (numeric/categorical/datetime) rather than expecting columns like “State” or “Date”.
3. **Adversarial QA loop (critic → fix)**: the initial paper draft is immediately reviewed and revised. The revision stage is designed to catch vague claims, missing limitations, and mismatched numbers.
4. **Graceful degradation**: when the “ideal” plan is not feasible, the workflow falls back to a simpler, still-valid output (e.g., descriptive analyses), so you still get a complete paper.

### 1.3 What you do vs what happens automatically

In a typical run, you do:

1. Put data into `exam_paper/data/`
2. Set `OPENAI_API_KEY`
3. Run one command

Everything else is automatic:

- file discovery
- dataset understanding
- research question + analysis plan
- statistical analysis
- table/figure generation
- paper drafting + revision
- references + PDF rendering

---

## 2) Getting started

### 2.1 Prerequisites

You need:

- Python 3
- Packages from `requirements.txt`
- An OpenAI API key available as an environment variable: `OPENAI_API_KEY`
- A LaTeX engine available on your machine (the workflow attempts PDF compilation)

If LaTeX is missing, the workflow is configured to try a minimal fallback so you still get a usable output.

### 2.1.1 Input expectations (what to put in `exam_paper/data/`)

The loader stage is designed around a simple idea: *discover tabular files + any documentation that helps interpret them*.

Recommended inputs:

- One or more CSV / Excel files (the “analytic tables”)
- A short `Data_Description.md` (or similar) describing columns and context

The workflow will still run if documentation is missing, but you’ll usually get a better research question + analysis plan if you include even a few bullets about what the dataset represents.

Quick setup commands (macOS/Linux shell):

```bash
# Install Python dependencies
python -m pip install -r requirements.txt

# Provide your API key
export OPENAI_API_KEY="YOUR_KEY_HERE"

# (Optional) sanity check the variable is set
python -c "import os; print('OPENAI_API_KEY set:', bool(os.getenv('OPENAI_API_KEY')))"
```

### 2.2 Repo structure (why it’s organized this way)

We separate reusable code from run outputs:

- `workflow/` — generic reusable pipeline
- `exam_paper/` — a run directory (inputs + intermediate artifacts + final output)
- `sample/` — sample dataset and example output

This prevents run artifacts from contaminating the reusable workflow and keeps debugging clean.

> **Tip:** Treat `exam_paper/` as disposable. If you want a fresh run, wipe `exam_paper/intermediate/` and `exam_paper/output/` and rerun end-to-end.

### 2.3 One-command run

From the repository root:

```bash
python workflow/scripts/main.py --config workflow/config/workflow_config.yaml
```

> **Tip:** If you only have time for one check: confirm `exam_paper/output/paper.pdf` exists after the run.

The primary output is:

- `exam_paper/output/paper.pdf`

Tip: you can point the workflow at a different dataset folder without moving files:

```bash
python workflow/scripts/main.py --data-dir path/to/new_dataset_folder
```

---

## 3) The system in action (a run walkthrough)

This section describes what happens when you run the workflow, and what files it produces.

### 3.1 Stage-by-stage flow

Conceptually (rendered as a diagram on GitHub, and on this repo’s GitHub Pages tutorial site):

```mermaid
flowchart LR
  A[Data folder] --> B[Load]
  B --> C[Understand]
  C --> D[Question]
  D --> E[Plan]
  E --> F[Analyze]
  F --> G[Tables/Figures]
  G --> H[Draft Paper]
  H --> I[Revise]
  I --> J[References]
  J --> K[Render PDF]
```

> **Note:** Mermaid diagrams render as diagrams on GitHub.com and on the GitHub Pages tutorial site under `docs/`. If you paste this Markdown into a different platform, Mermaid may show as a code block unless that platform supports Mermaid.

Plain-text version:

```text
Data folder → Load → Understand → Question → Plan → Analyze → Tables/Figures → Draft → Revise → References → Render
```

In code, the orchestrator is `workflow/scripts/main.py`, which executes each stage in order using the configured paths and guardrails.

### 3.2 Where outputs go

- Intermediate JSON artifacts: `exam_paper/intermediate/`
- Tables (CSV): `exam_paper/output/tables/`
- Figures (PNG): `exam_paper/output/figures/`
- LaTeX source: `exam_paper/tex/paper.tex`
- Final PDF: `exam_paper/output/paper.pdf`

If `exam_paper/output/paper.pdf` is missing but the render stage “completed”, check:

- `exam_paper/intermediate/render_report.json` (the compilation log)
- `exam_paper/tex/paper.pdf` (sometimes the PDF is produced in `tex/` even if copying fails)

### 3.2.1 What “artifact-driven” looks like in practice

During a run you’ll see a growing set of JSON files in `exam_paper/intermediate/`. These aren’t just logs — they’re **contracts** between stages.

Typical files include:

- `load_data_summary.json` → what files were discovered + basic schema info
- `dataset_summary.json` → the interpreted dataset description and variable roles
- `research_question.json` → a concrete, feasible question
- `analysis_plan.json` → the structured plan the executor will implement
- `analysis_results.json` → computed summaries/models + dataset metadata

If the paper looks “generic”, the fix is usually to improve or validate one of these upstream artifacts (especially `analysis_results.json` grounding).

### 3.2.2 Worked example: how artifacts become paper text

To make this concrete, here’s an excerpt from this repo’s `revised_paper_sections.json` (the revised manuscript draft), and how it maps back to earlier stages.

**Manuscript excerpt (from `revised_paper_sections.json`)**

> Title: “Distribution, Timing, and Financial Magnitude of Tracked NIH/HHS Award Disruptions in a Public Snapshot Dataset”
>
> Results (excerpt): “Among 5419 tracked disrupted award records, status categories were ‘Possibly Reinstated’ (2174 [40.12%]) …”

**Where those numbers came from**

- The dataset size (5419) comes from `analysis_results.json` → `selected_dataset.n_rows`.
- The status distribution comes from `analysis_results.json` → `descriptive_results.sample_characteristics.status.top_categories`.

This “traceability” is the main trick for avoiding fluffy LLM writing: the writing stage should *copy* computed values from artifacts instead of inventing them.

### 3.3 How to debug quickly when something breaks

Because each stage writes a dedicated artifact, debugging is usually:

1. Find the last successfully-written artifact in `exam_paper/intermediate/`
2. Inspect it for missing/invalid fields
3. Fix the upstream cause (often type detection or feasibility issues)
4. Rerun only the failed stage (or rerun end-to-end if time permits)

This is the main reason we avoid “single huge notebook” designs.

Practical commands that help:

```bash
# See what artifacts exist
ls -1 exam_paper/intermediate

# Quick sanity check: is the JSON valid?
python -m json.tool exam_paper/intermediate/analysis_plan.json > /dev/null && echo OK

# Skim the top of a large artifact
python -c "import json; p='exam_paper/intermediate/analysis_results.json'; d=json.load(open(p)); print(list(d.keys()))"
```

<details>
<summary><strong>Debugging cheat sheet (fast)</strong></summary>

- Last artifact written lives in `exam_paper/intermediate/`.
- If the next stage fails, inspect the artifact for missing keys / nonsense types.
- Use `--stop-after` to freeze earlier (Section 3.4).

</details>

### 3.4 Debug mode: stop after a stage

The orchestrator supports a `--stop-after` option so you can freeze the pipeline after a stage and inspect artifacts before moving on.

Examples:

```bash
# Stop after dataset understanding
python workflow/scripts/main.py --stop-after understand_data

# Stop after analysis planning (to inspect analysis_plan.json)
python workflow/scripts/main.py --stop-after plan_analysis
```

> **Warning:** `--stop-after` is intentionally “boring”: it exists so you can inspect artifacts before later stages stack errors on top of earlier mistakes.

This is useful when the research question is odd or the proposed method is infeasible—you can intervene earlier instead of waiting for later stages to fail.

---

## 4) The building blocks

Our workflow is intentionally split into five building blocks.

### 4.0 The “stage contract” table

Every stage has an explicit input and output. This makes the workflow testable and lets you rerun only the broken part.

| Stage | Script | LLM? | Reads (input) | Writes (output) |
|---|---|---:|---|---|
| 1. Load data | `load_data.py` | No | raw files in `exam_paper/data/` | `load_data_summary.json` |
| 2. Understand data | `understand_data.py` | Yes | `load_data_summary.json` + description docs | `dataset_summary.json` |
| 3. Research question | `generate_question.py` | Yes | `dataset_summary.json` | `research_question.json` |
| 4. Analysis plan | `plan_analysis.py` | Yes | `research_question.json` (+ schema context) | `analysis_plan.json` |
| 5. Run analysis | `run_analysis.py` | No | `analysis_plan.json` (+ data) | `analysis_results.json` |
| 6. Tables/figures | `make_tables_figures.py` | No | `analysis_results.json` (+ data) | tables + figures + `table_figure_manifest.json` |
| 7. Write paper | `write_paper.py` | Yes | results + manifest + template constraints | `paper_sections.json` |
| 8. Revise paper | `revise_paper.py` | Yes | `paper_sections.json` (+ results checks) | `revised_paper_sections.json` |
| 9. References | `generate_references.py` | Yes | paper sections + question context | `references.json` + BibTeX |
| 10. Render PDF | `render_paper.py` | No | revised sections + tables/figures + bib | `paper.tex` + `paper.pdf` |

The key idea is that “AI stages” are sandwiched between deterministic stages: the AI proposes structured objects, and deterministic code executes/validates.

#### 4.0.1 Concrete examples (real excerpts)

These shortened excerpts are from actual artifacts in this repository.

`dataset_summary.json` (what the system thinks the dataset *is*):

```json
{
  "dataset_summary": {
    "dataset_overview": {
      "analytic_files": ["nih_terminations.csv"],
      "documentation_files": ["Data_Description.md", "...pdf"]
    },
    "inferred_study_characteristics": {
      "likely_unit_of_observation": "A grant/award record ...",
      "likely_study_design": "Observational, descriptive ..."
    }
  }
}
```

`analysis_plan.json` (what will be computed):

```json
{
  "analysis_plan": {
    "analysis_overview": {
      "question_type": "descriptive",
      "overall_strategy": "Produce (1) overall and stratified tabulations ..."
    },
    "variable_mapping": {
      "outcome": ["status", "termination_date", "total_award"],
      "primary_exposure": ["org_type", "org_state", "activity_code"]
    }
  }
}
```

`analysis_results.json` (what actually happened):

```json
{
  "analysis_results": {
    "selected_dataset": {
      "table_name": "nih_terminations",
      "n_rows": 5419,
      "n_cols": 57,
      "resolved_required_variables": ["status", "termination_date", "org_state", "..."]
    },
    "descriptive_results": {
      "sample_characteristics": {
        "status": {"variable_type": "categorical", "n_total": 5419, "pct_missing": 0.0}
      }
    }
  }
}
```

Why this matters: once you have `analysis_results.json`, the writing stages can be prompted to *quote* numbers from it rather than inventing narrative.

### 4.1 Scripts: deterministic executors

The executable implementation lives in `workflow/scripts/`. Each script corresponds to a stage.

Examples:

- `load_data.py` reads CSV/XLSX, profiles columns, and saves a structured schema summary.
- `run_analysis.py` performs statistical analysis using deterministic Python libraries (e.g., statsmodels).
- `render_paper.py` converts sections + tables + figures into LaTeX and compiles a PDF.

Design rule: scripts should be able to run in isolation given the correct upstream artifact.

### 4.2 Prompts: strict contracts for LLM stages

LLM-dependent stages use prompt templates under `workflow/prompts/`. Prompts act like *contracts*:

- specify the required output format (often JSON)
- constrain the writing style
- prevent unsupported causal claims

We also keep temperatures low (see config) to reduce randomness and improve reproducibility:

- understand / question / plan / write: 0.20
- revise: 0.15 (stricter)

### 4.3 Skills: reusable “playbooks”

Skills live in `workflow/skills/`. They are not code; they are documentation that encodes:

- what a stage should accomplish
- common pitfalls
- how to judge quality

In practice, this makes prompts and scripts more consistent because the team has a shared target for “good output”.

### 4.4 Configuration: guardrails and limits

The pipeline is configured by `workflow/config/workflow_config.yaml`. This file sets:

- model name (`gpt-5.2`)
- environment variable for the API key (`OPENAI_API_KEY`)
- dataset profiling limits (max columns, category levels)
- analysis fallbacks (allow generic linear/logistic/poisson)
- rendering timeouts and minimal fallback behavior
- explicit guardrails, including:
  - no hard-coded variable names
  - avoid unjustified causal claims
  - preserve uncertainty

This matters because “generic” workflows fail when they quietly introduce hard-coded assumptions.

Practical tip: treat `workflow_config.yaml` as your **safety dial**. If a dataset is huge or messy, raising/lowering limits here is safer than adding special-cases to Python.

### 4.5 Templates: JAMA structure and LaTeX rendering

The paper is rendered using the JAMA-like LaTeX template (see `sample/tex/template.tex` as the reference structure). The workflow writes LaTeX into `exam_paper/tex/` and compiles the final PDF.

---

## 5) Workflow patterns that make it robust

This section is the heart of the tutorial: the patterns you can reuse when building other generic research pipelines.

### Pattern 1: “Plan-first” via a structured analysis plan artifact

Instead of jumping straight into modeling, we force a structured analysis plan (`analysis_plan.json`) that includes:

- the proposed outcome/exposure
- variable typing assumptions
- primary model
- fallbacks (Plan B / Plan C)
- what constitutes a failure and what to do next

Why it helps: it reduces wasted computation and prevents the model from inventing methods that don’t fit the data.

What to look for in a “good” plan artifact:

- The plan is explicit about **question type** (descriptive vs associational)
- It lists the exact variables it expects to use (and allows alternatives)
- It contains **field-specific denominators** (missingness-aware summaries)
- It includes **hard limits** (top K categories, bounded figure size)

### Pattern 2: Critic → fix loop (draft then revise)

We separate writing into two steps:

1. `write_paper.py` produces `paper_sections.json`
2. `revise_paper.py` produces `revised_paper_sections.json`

The revision stage is explicitly allowed to be “picky” and should:

- remove generic filler text
- force numerical grounding (e.g., report sample sizes, effect sizes)
- check internal consistency (numbers mentioned in Results must match `analysis_results.json`)
- strengthen limitations and uncertainty when causal claims are not justified

This avoids the common failure where the system “sounds good” but is not tied to computed results.

Here is the mental model:

```text
Draft writer (write_paper)  →  Critic/reviewer (revise_paper)
   |                             |
   v                             v
 paper_sections.json         revised_paper_sections.json
```

Even if both steps use an LLM, splitting them reduces self-justification: the revision step is explicitly prompted to be skeptical and to demand grounding.

If you want to make the revision step stronger, the easiest lever is: require that every major Results paragraph includes at least one number and names the artifact it came from (e.g., sample size from `analysis_results.json`).

### Pattern 3: Verification as a non-negotiable step

For academic outputs, “looks fine” is not enough. Verification means:

- figures must be generated without crashing
- tables must exist on disk and be readable
- LaTeX must compile (or a fallback must be produced)

We treat the compilation step as a test: if it fails, the pipeline should either fix it or degrade gracefully.

An easy “quality gate” checklist you can apply in 60 seconds:

- [ ] Does `exam_paper/output/paper.pdf` exist?
- [ ] Do `exam_paper/output/figures/` and `exam_paper/output/tables/` contain files?
- [ ] Does the Results section contain numbers (not just adjectives)?
- [ ] Are claims cautious when causality is not justified?

If one box fails, you can usually trace it to a single stage:

- Missing PDF → rendering/LaTeX stage
- Empty tables/figures → table/figure stage or upstream analysis results
- “No numbers” in Results → revise prompt needs stronger grounding rules

### Pattern 4: Graceful degradation (always produce a paper)

When the ideal method is not feasible (e.g., outcome type mismatch), the workflow should fall back to:

- descriptive statistics
- simple association models
- smaller model families that are guaranteed to run

The point is not to “hide failure” — it is to produce a coherent, honest paper even when the dataset is messy or time is limited.

In practice, degradation often means:

- If dates can’t be parsed reliably, summarize date *coverage* instead of trends
- If a candidate outcome is too sparse, pivot to descriptive distributions
- If a plot would be unreadable, limit to top K levels and label the choice

### Pattern 5: Type-driven visualization with hard bounds

Visualization is the easiest place to accidentally hard-code assumptions.

Our approach:

- Detect numeric/categorical/datetime columns and choose plots accordingly.
- Limit the number of categories plotted (e.g., top K).
- Cap figure sizes to avoid massive images.
- Subsample long time series for readability.

This pattern is what makes a plotting system truly portable across datasets.

---

## 6) How we made it dataset-agnostic

Here are the concrete rules we followed (and recommend).

### Rule A: Never key logic on a specific column name

Bad:

- “Use `State` as the grouping column.”

Good:

- “Find a low-cardinality categorical column that looks like an entity identifier.”

Practical heuristic used by generic pipelines:

- Prefer categorical columns with 5–50 unique values for grouping
- Avoid “ID-like” columns (very high cardinality) unless you’re deduplicating
- Treat free-text columns as optional unless you have a specific, simple text task

### Rule B: Prefer “detect and validate” over “assume and proceed”

For example, if a plan calls for logistic regression, validate that the outcome is binary and has enough events.

Validation examples:

- If a “numeric” column is mostly non-numeric strings, coerce + report conversion failures
- If a date column contains multiple formats, parse with a strict fallback and report unparseable counts
- If a categorical column has hundreds of levels, collapse to top K for plots/tables

### Rule C: Put hard limits everywhere

Examples of necessary limits:

- maximum number of columns profiled
- maximum category levels shown
- maximum plot size
- maximum time series points

These limits are not just performance improvements—they prevent runtime crashes.

One concrete example we hit during development: plotting a timeline with thousands of rows can silently create an image so tall that image formats reject it. The fix is architectural: always cap figure dimensions and/or sample points.

This is also why the pipeline prefers “summaries first”: a time-binned count plot (weekly/monthly) is usually safer than plotting every point.

### Rule D: Enforce honest claims

If the data do not support a causal design, the workflow should write an associational paper and explicitly state that limitation.

---

## 7) Customizing for your domain (how to extend safely)

The fastest way to break a generic workflow is to “just add one special case”. Instead, extend the system through well-defined hooks.

### 7.1 Add a new stage

If you want a new stage (e.g., automatic slide generation), follow the same artifact pattern:

1. Define a new output artifact (e.g., `slides.qmd` or `slides.pdf`).
2. Keep inputs explicit (which upstream JSON does it depend on?).
3. Add verification (does it render/compile?).

Concrete example: if you add a “slides” stage, treat the slide deck like the PDF stage — it should always either produce a file or produce a clear, minimal fallback.

### 7.2 Add a new “skill” document

Skills are documentation, but they matter: they keep the workflow coherent across team members.

Create a new file under `workflow/skills/` with:

- the purpose (“When should this be used?”)
- required inputs/outputs
- failure modes
- quality checklist

### 7.3 Adjust limits and guardrails

If a new dataset has thousands of columns or extremely high-cardinality fields, adjust limits in `workflow/config/workflow_config.yaml` instead of hard-coding exceptions in Python.

Examples of config knobs that commonly matter:

- `max_profile_columns`: if schema summaries are truncated too aggressively
- `max_categorical_levels`: if plots/tables explode in width
- `analysis.unsupported_methods_fail_soft`: keep this `true` for time-constrained or exploratory runs so you still get a coherent paper even when a fancy method fails

General guidance:

- When in doubt, prefer limits/config over code changes.
- Code changes should usually be reserved for new capabilities (not one-off exceptions).

---

## 8) Appendix: file reference and troubleshooting

### 8.1 Key files to know

If you only memorize five locations:

1. `exam_paper/data/` — input data
2. `exam_paper/intermediate/` — stage artifacts (debugging)
3. `exam_paper/output/figures/` — plots
4. `exam_paper/output/tables/` — tables
5. `exam_paper/output/paper.pdf` — final deliverable

### 8.2 Common failures and what to do

**Missing API key**

- Symptom: AI stages fail immediately.
- Fix: export `OPENAI_API_KEY` and rerun.

If you want to confirm it’s visible to Python:

```bash
python -c "import os; print(bool(os.getenv('OPENAI_API_KEY')))"
```

**Reference JSON parsing errors**

- Symptom: reference generation fails because the model outputs invalid JSON escaping.
- Fix: sanitize model output before parsing (this workflow includes defensive handling); if it still fails, fall back to placeholder references so rendering can proceed.

**LaTeX compilation fails**

- Symptom: no PDF produced.
- Fix: check LaTeX is installed; otherwise rely on fallback output and/or install a TeX distribution.

If you just need the source, the workflow also writes the TeX file to `exam_paper/tex/paper.tex`.

**Plots crash or become enormous**

- Symptom: errors about image dimensions or unreadable output.
- Fix: ensure plotting code has size caps and sampling; treat this as a required guardrail, not an optimization.

**Analysis plan selects an infeasible method**

- Symptom: modeling step throws type/convergence errors.
- Fix: fall back to descriptive analysis and document the limitation in the paper.

**The paper is too generic**

- Symptom: phrases like “the data shows” appear without concrete results.
- Fix: strengthen the revision step to (a) demand numbers and (b) cross-check against `analysis_results.json`.

A simple rule that works well in practice: every Results paragraph must contain at least one of:

- a sample size (N)
- a percentage
- a median (IQR) or mean (SD)

…and those numbers must be traceable to `analysis_results.json`.

---

## Final note: turning this into a blog post

This file is meant to be copied into your coding blog. The only required repo change is: add the published tutorial URL (and the tutorial video URL) into the root `Readme.md`.

If you tell me your blog platform (GitHub Pages, Hugo, Jekyll, Notion, etc.), I can restructure this into that platform’s preferred format (frontmatter, page title, navigation).

---

## 9) Publishing as a GitHub Pages site (optional)

This repo ships a ready-to-serve GitHub Pages site under `docs/` using Docsify (so you get a left sidebar that follows your scroll position).

### 9.1 What to commit

- `docs/index.html` (Docsify site entry)
- `docs/README.md` (the rendered tutorial content)
- `docs/_sidebar.md` (sidebar entry; section headings are generated automatically)
- `docs/.nojekyll` (ensures GitHub Pages serves `_sidebar.md` correctly)

### 9.2 How to enable it on GitHub

1. Push your repo to GitHub.
2. Go to **Settings → Pages**.
3. Under **Build and deployment**:
  - Source: **Deploy from a branch**
  - Branch: your default branch (often `main`)
  - Folder: **/docs**

After a minute, GitHub will show a URL like:

`https://<your-username>.github.io/<your-repo>/`

### 9.3 How to keep the repo tutorial and Pages tutorial in sync

Treat `tutorial.md` as your “source of truth” and periodically copy it into `docs/README.md`.
