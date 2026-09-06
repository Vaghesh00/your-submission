# NOTEBOOK.md — lab notebook

## Setup
- Reviewed starter kit: fertility.py, REPORT_v0.md, bench/model_spec.md,
  bench/bench_log.csv, corpus_sample/.
- Planned structure: partA/, partB/, partC/ folders, one notebook per part.

## Part A1 — corpus
- First attempt: tried datasets.load_dataset("facebook/flores", ...,
  trust_remote_code=True) — FAILED. FLORES-200 is gated on HF, and
  trust_remote_code is deprecated in current datasets library.
- Fix: downloaded FLORES-200 directly from Meta's public tarball
  (dl.fbaipublicfiles.com/nllb/flores200_dataset.tar.gz), no auth needed.
- Extracted eng_Latn, hin_Deva, tam_Taml, tel_Telu, dev split.
- Wrote corpus_real/{eng,hin,tam,tel}.txt inside partA/ folder.
- Verified: 997 parallel lines per language, matched across all four.

## Part A2 — script audit
- Rewrote fertility.py's logic as plain functions (read_lines, count_words,
  count_chars, analyze) runnable directly in notebook cells — the original
  script's argparse/CLI structure caused friction in Jupyter.
- Baseline (buggy) run on real corpus: eng 1.2825, hin 7.8232, tam 24.7332,
  tel 20.3995 (tokens/word, gpt2). Reproduced identically on a second,
  independent run in a fresh notebook — confirms determinism.
- Ablation: no-lowercase — eng shifts -3.57%, Indic languages shift <0.03%.
  Confirms lowercasing unfairly benefits English's case-sensitive BPE.
- Ablation: ws-split (proper whitespace split) — eng +0.01%, tam +0.54%,
  tel +1.10%. Confirms literal split(" ") creates spurious empty "words",
  effect stronger in tam/tel.
- Ablation: grapheme-chars — first attempt showed 0.00% change for all
  languages using default settings; realized this was because the default
  denominator is "word", so the char-counting method never gets exercised
  unless denominator is explicitly set to "grapheme". Reran with
  denominator="grapheme" forced: hin +52.64%, tam +54.89%, tel +74.01%,
  eng 0.00%. Largest single distortion found — len() badly undercounts
  "chars" for Indic scripts due to combining marks.
- Ablation: micro vs macro average — all four languages shift 0.4-1.1%,
  tam moves most (-1.08%). Confirms sentence-length distribution differs
  enough by language that averaging method isn't cosmetic.
- Checked random.seed(1337): grepped for other random.* calls in the
  script — none found. Confirmed dead code, not a real bug.

## Part A3 — corrected analysis
- Added MuRIL (google/muril-base-cased) as second, Indic-aware tokenizer.
- Ran full grid: 2 tokenizers × 4 denominators (word/grapheme/byte/sentence).
- Key finding: gpt2 tokens/byte on tam/tel is ~0.99 (near 1 token per byte,
  effectively no compression) vs ~0.21 on English — ~5x cost gap.
- MuRIL inverts this: 0.07-0.095 tokens/byte on hin/tam/tel, more efficient
  than its own English number.
- Decided byte-based fertility is the right routing metric: invariant to
  script/morphology, unlike "word" which conflates real inefficiency with
  Tamil/Telugu's agglutinative word structure.

## Part A4
- Wrote memo recommending routing Indic traffic through an Indic-aware
  tokenizer, ~5-10x token savings, with corpus-size/domain caveat and
  tokens/byte production monitoring metric.

## Part B — capacity reconciliation
- Hit a Windows path bug: single-backslash path string
  ("D:\PROJECT\task\...") got silently mangled because \t and \b were
  interpreted as escape characters (tab, backspace), producing an
  unreadable path and OSError. Fixed by placing bench_log.csv directly in
  the partB/ folder and using a plain relative path "bench_log.csv".
- B1: KV cache = 114,688 bytes/token (112 KiB). Max concurrent 4096-token
  sequences ≈ 25.7. Cross-checked against logged kv_cache_util — predicted
  utilization matches logged values almost exactly through batch 24
  (0.933 vs 0.93), diverges past the ceiling (predicted 1.24 vs logged
  capped at 0.97) — confirms scheduler admission cap.
- B2: reported_tok_s peaks at batch 24 (1607.4), falls at batch 32 (1384.0)
  and 48 (1298.5). preempted_seqs jumps 0→7→23 exactly at the predicted
  capacity ceiling. Mechanism: preempted sequences must re-prefill on
  resumption, wasting compute.
- B3: confirmed reported_tok_s = (prompt_len+gen_len)*num_requests/wall_
  clock_s exactly on every row — conflates prefill with decode. True
  goodput at batch 24 ≈ 200.9 tok/s (method 1) / 249.8 tok/s (method 2),
  both far below reported 1607.4. Batch-48 actual goodput = 162.3 tok/s,
  contradicting the report's ~3200 tok/s prediction.
- B4: kv_cache_util + preemption counter identified as the confirming
  metric, already visible in the log data itself.

## Part C
- Chose path (b), small rewriter model, given only 2 of 6 target
  languages (Hindi, Kannada) have reviewer coverage. Set explicit kill
  criterion (fallback to prompt-only) if approval rate <60% by week 2.

## Wrap-up
- Re-ran Part A and Part B calculations in fresh notebooks to confirm
  reproducibility — all numbers matched exactly across runs.
- Wrote this notebook and AI_USAGE.md.