# memo.md — casual tone in Indic languages (≤1 page)

**Chosen path:** (b) small (≤1B) inference-time rewriter model, applied
after the main model's response.

**Assumptions:**
- The main assistant model can be prompted to generate synthetic
  "casualized" rewrites of its own formal responses — this counts as
  in-house compute, not an external API, fitting the no-API-budget
  constraint.
- A ≤1B model can be fine-tuned (likely LoRA) on a few thousand pairs per
  language within hours, not days, on a single A100-80GB.
- "Casual" is primarily a lexical/register shift (contractions, particles,
  informal connectives) rather than a change in factual content, so a
  narrow rewriter model is sufficient without full instruction retraining.

**Back-of-envelope arithmetic:**
- Data: ~3,000-5,000 (formal, casual) pairs per language × 6 languages
  ≈ 20,000-30,000 pairs, generated via few-shot prompting the main model
  itself — estimated 1-2 days of generation + light filtering.
- Training: ≤1B model, LoRA fine-tune on ~25k examples — a few hours per
  run on the A100; budget 3-4 iteration rounds within the 2-week window,
  leaving slack for hyperparameter fixes.
- Reviewer throughput: 30 hours total (10h/week × 3 weeks), ~2 min/example
  review ≈ 900 reviewable examples — enough for real signal on Hindi +
  Kannada (~150 examples per checkpoint × 2-3 checkpoints), but zero
  native coverage for Tamil/Telugu/Bengali/Marathi. Those four rely on an
  automated proxy (informal-marker/contraction density, or a formality
  classifier) as a stopgap, explicitly flagged as unvalidated.

**Success metric:** ≥70% of sampled Hindi and Kannada rewriter outputs
rated "casual/conversational" (top-2-box on a 1-5 scale) by the native
reviewer, measured on a held-out 150-example set per language at the
final checkpoint.

**Kill criterion:** if by end of week 2, two iteration rounds still leave
the Hindi/Kannada approval rate below 60%, OR the rewriter introduces
meaning-altering errors (omission/hallucination) in more than 10% of
sampled outputs — abandon the rewriter path and fall back to (c)
prompt-engineering-only for the remaining week, since it's the only
option that can still ship inside the 3-week window from a cold stop.

**Day-1 experiment:** generate ~50 casualized Hindi + Kannada response
pairs by prompting the main model directly (no training yet), and get the
reviewer to rate them same day. This tests two things at once: whether
casualization via prompting alone is already "good enough" (arguing for
cheaper path (c) instead), and whether the synthetic data generation
approach itself produces usable training pairs before committing GPU
time to it.