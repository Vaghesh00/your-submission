# A4_memo.md — recommendation memo

**Corrected headline numbers:** gpt2 costs ~5x more tokens/byte than English
on Tamil/Telugu (near 1:1 token-to-byte, almost no compression); MuRIL is
5-10x more efficient than gpt2 on the same Indic-language content (full
table in A3_analysis.md).

**Routing recommendation:** route Hindi/Tamil/Telugu (and likely other
Indic-language) traffic through an Indic-aware tokenizer/model rather than
gpt2. The current setup means Indic users consume 5x+ more context/compute
per equivalent input than English users, directly inflating cost and
reducing effective context budget for those languages.

**Biggest caveat:** based on 997 FLORES sentences — formal, edited, written
text. Real user traffic (casual, code-mixed Hindi-English, etc.) may show
different fertility; recommend validating on a sample of real production
queries before committing to a routing change.

**Production monitoring metric:** tokens/UTF-8-byte per live request,
segmented by detected language. Alert if it drifts materially from this
benchmark — would catch e.g. a misrouted request hitting the wrong
tokenizer, or genuine domain shift making the benchmark stale.