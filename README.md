# AI Team Intern Assignment — The Audit

Submission for the tokenizer + serving-capacity audit assignment.

## Structure

- `NOTEBOOK.md` — chronological lab notebook (hypothesis → experiment → result)
- `AI_USAGE.md` — where AI helped and how I verified it myself
- `partA/` — tokenizer audit
  - `partA.ipynb` — corpus prep, script audit ablations, corrected analysis
  - `corpus_real/` — FLORES-200 corpus (eng/hin/tam/tel, 997 parallel sentences)
  - `audit.md` — A2: bug-by-bug evidence (claim → command → before/after)
  - `A3_analysis.md` — A3: corrected cross-language analysis, 2 tokenizers × 4 denominators
  - `A4_memo.md` — A4: routing recommendation memo
- `partB/` — capacity reconciliation
  - `partB.ipynb` — KV-cache math, throughput anomaly analysis
  - `bench_log.csv` — serving load-test log (from starter kit)
  - `model_spec.md` — serving setup spec (from starter kit)
  - `answers.md` — B1–B4 written answers
- `partC/`
  - `memo.md` — decision memo: casual tone rollout across 6 Indic languages

## How to reproduce

1. `pip install tiktoken transformers sentencepiece regex pandas`
2. Open `partA/partA.ipynb`, run cells top to bottom (downloads FLORES-200 from Meta's public server, no auth required)
3. Open `partB/partB.ipynb`, run cells top to bottom (uses the included `bench_log.csv` and `model_spec.md`)
