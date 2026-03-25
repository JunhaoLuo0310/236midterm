# Tutorial: Designing a Generic Automated Research Workflow

This tutorial explains how to design a generic, end-to-end workflow that can take a brand-new public health dataset and automatically produce a JAMA Network Open–style paper (including tables, figures, references, and a compiled PDF) with minimal human intervention.

This is intentionally beginner-friendly and self-contained. You do *not* need prior experience with multi-agent systems, LaTeX, or LLM orchestration to follow the design.

> **Tip:** This page is designed to render as a GitHub Pages site with a left sidebar table of contents.

## Authors

- Johnny Zhao
- Yicheng He
- Junhao Luo

## What you’ll learn

- How to split an end-to-end “paper generator” into small stages with explicit inputs/outputs
- How to keep logic dataset-agnostic (type-driven, validated assumptions)
- How to add guardrails so failures degrade gracefully
- How to debug quickly by inspecting the exact artifact a stage produced

---

## At a glance

| What you get | Where it lands |
|---|---|
| Intermediate “stage artifacts” (debuggable JSON) | `exam_paper/intermediate/` |
| Tables (CSV) and figures (PNG) | `exam_paper/output/tables/`, `exam_paper/output/figures/` |
| LaTeX source and compiled paper | `exam_paper/tex/paper.tex`, `exam_paper/output/paper.pdf` |
| “Stop after a stage” debugging | `--stop-after <stage>` |

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
3. **Adversarial QA loop (critic → fix)**: the initial paper draft is immediately reviewed and revised.
4. **Graceful degradation**: when the “ideal” plan is not feasible, the workflow falls back to a simpler, still-valid output.

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

Quick setup commands (macOS/Linux shell):

```bash
python -m pip install -r requirements.txt
export OPENAI_API_KEY="YOUR_KEY_HERE"
python -c "import os; print('OPENAI_API_KEY set:', bool(os.getenv('OPENAI_API_KEY')))"
```

### 2.1.1 Input expectations (what to put in `exam_paper/data/`)

- One or more CSV / Excel files (analytic tables)
- Optional documentation like `Data_Description.md` (strongly recommended)

### 2.2 Repo structure (why it’s organized this way)

We separate reusable code from run outputs:

- `workflow/` — generic reusable pipeline
- `exam_paper/` — a run directory (inputs + intermediate artifacts + final output)
- `sample/` — sample dataset and example output

### 2.3 One-command run

From the repository root:

```bash
python workflow/scripts/main.py --config workflow/config/workflow_config.yaml
```

> **Tip:** If you only have time for one check: confirm `exam_paper/output/paper.pdf` exists after the run.

---

## 3) The system in action (a run walkthrough)

### 3.1 Stage-by-stage flow

```text
Data folder → Load → Understand → Question → Plan → Analyze → Tables/Figures → Draft → Revise → References → Render
```

In code, the orchestrator is `workflow/scripts/main.py`, which executes each stage in order.

### 3.2 Where outputs go

- Intermediate JSON artifacts: `exam_paper/intermediate/`
- Tables (CSV): `exam_paper/output/tables/`
- Figures (PNG): `exam_paper/output/figures/`
- LaTeX source: `exam_paper/tex/paper.tex`
- Final PDF: `exam_paper/output/paper.pdf`

### 3.3 How to debug quickly when something breaks

Because each stage writes a dedicated artifact, debugging is usually:

1. Find the last successfully-written artifact in `exam_paper/intermediate/`
2. Inspect it for missing/invalid fields
3. Fix the upstream cause
4. Rerun only the failed stage

Practical commands that help:

```bash
ls -1 exam_paper/intermediate
python -m json.tool exam_paper/intermediate/analysis_plan.json > /dev/null && echo OK
```

<details>
<summary><strong>Debugging cheat sheet (fast)</strong></summary>

- Last artifact written lives in `exam_paper/intermediate/`.
- If the next stage fails, inspect the artifact for missing keys / nonsense types.
- Use `--stop-after` to freeze earlier.

</details>

### 3.4 Debug mode: stop after a stage

```bash
python workflow/scripts/main.py --stop-after understand_data
python workflow/scripts/main.py --stop-after plan_analysis
```

---

## 4) The building blocks

### 4.0 The “stage contract” table

| Stage | Script | LLM? | Reads (input) | Writes (output) |
|---|---|---:|---|---|
| 1. Load data | `load_data.py` | No | raw files in `exam_paper/data/` | `load_data_summary.json` |
| 2. Understand data | `understand_data.py` | Yes | `load_data_summary.json` + description docs | `dataset_summary.json` |
| 3. Research question | `generate_question.py` | Yes | `dataset_summary.json` | `research_question.json` |
| 4. Analysis plan | `plan_analysis.py` | Yes | `research_question.json` | `analysis_plan.json` |
| 5. Run analysis | `run_analysis.py` | No | `analysis_plan.json` | `analysis_results.json` |
| 6. Tables/figures | `make_tables_figures.py` | No | `analysis_results.json` | outputs + manifest |
| 7. Write paper | `write_paper.py` | Yes | results + manifest | `paper_sections.json` |
| 8. Revise paper | `revise_paper.py` | Yes | draft sections | `revised_paper_sections.json` |
| 9. References | `generate_references.py` | Yes | sections + context | `references.json` + BibTeX |
| 10. Render PDF | `render_paper.py` | No | revised sections + assets | `paper.tex` + `paper.pdf` |

#### 4.0.1 Concrete examples (short excerpts)

`analysis_results.json` captures grounding facts like dataset size and resolved variables:

```json
{
	"analysis_results": {
		"selected_dataset": {"n_rows": 5419, "n_cols": 57}
	}
}
```

---

## 5) Workflow patterns that make it robust

### Pattern 1: Plan-first

Force a structured analysis plan (`analysis_plan.json`) with fallbacks.

### Pattern 2: Critic → fix loop

Draft first, then revise with a skeptical reviewer pass.

### Pattern 3: Verification

Treat PDF compilation and artifact existence as “tests”.

If the paper reads too generic, strengthen the revision step to require that each Results paragraph contains at least one number traceable to `analysis_results.json`.

---

## 6) How we made it dataset-agnostic

- Never key logic on specific column names.
- Prefer detect-and-validate over assume-and-proceed.
- Put hard limits everywhere (columns, categories, plot sizes).
- Enforce honest claims (associational unless causal design is justified).

---

## 7) Customizing for your domain (how to extend safely)

- Add new stages by defining new artifacts.
- Add new `workflow/skills/` playbooks.
- Adjust limits in `workflow/config/workflow_config.yaml`.

---

## 8) Appendix: file reference and troubleshooting

### 8.1 Key files to know

1. `exam_paper/data/` — input data
2. `exam_paper/intermediate/` — stage artifacts
3. `exam_paper/output/figures/` — plots
4. `exam_paper/output/tables/` — tables
5. `exam_paper/output/paper.pdf` — final deliverable

### 8.2 Common failures and what to do

- Missing API key → export `OPENAI_API_KEY`.
- LaTeX fails → install a TeX distribution or rely on fallback.
- Plots too large → ensure size caps and sampling.

---

## 9) Publishing as a GitHub Pages site (optional)

1. Push your repo to GitHub.
2. Go to **Settings → Pages**.
3. Source: **Deploy from a branch**
4. Branch: your default branch
5. Folder: **/docs**

You’ll get a URL like `https://<your-username>.github.io/<your-repo>/`.
