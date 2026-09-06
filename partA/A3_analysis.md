# A3_analysis.md — corrected cross-language analysis

Corpus: FLORES-200 dev, 997 parallel sentences, eng/hin/tam/tel.
Settings: all A2 fixes applied (no-lowercase, ws-split, grapheme-chars, micro-average).
Tokenizers: gpt2 (tiktoken), muril (google/muril-base-cased, HF).
Denominators: word, grapheme, byte, sentence.

## Full results

| tokenizer | denom | eng | hin | tam | tel |
|---|---|---|---|---|---|
| gpt2 | word | 1.2285 | 7.7957 | 24.6165 | 20.4810 |
| gpt2 | grapheme | 0.2056 | 2.3279 | 4.2043 | 4.5623 |
| gpt2 | byte | 0.2055 | 0.5946 | 0.9959 | 0.9907 |
| gpt2 | sentence | 25.8185 | 192.4052 | 398.3581 | 336.6520 |
| muril | word | 1.2582 | 1.2455 | 1.7225 | 1.9547 |
| muril | grapheme | 0.2106 | 0.3719 | 0.2942 | 0.4354 |
| muril | byte | 0.2104 | 0.0950 | 0.0697 | 0.0946 |
| muril | sentence | 26.4443 | 30.7412 | 27.8746 | 32.1304 |

## Cross-language ratios (relative to English)

| tokenizer | denom | hin/eng | tam/eng | tel/eng |
|---|---|---|---|---|
| gpt2 | word | 6.35 | 20.04 | 16.67 |
| gpt2 | grapheme | 11.32 | 20.45 | 22.19 |
| gpt2 | byte | 2.89 | 4.85 | 4.82 |
| muril | word | 0.99 | 1.37 | 1.55 |
| muril | grapheme | 1.77 | 1.40 | 2.07 |
| muril | byte | 0.45 | 0.33 | 0.45 |

## Which single number should drive the routing decision, and why

Tokens-per-UTF-8-byte, using the tokenizer actually deployed. Bytes are the
one denominator invariant to script and morphological typology (unlike
"word", which conflates real tokenizer inefficiency with the fact that
Tamil/Telugu words are agglutinative and naturally longer than English
words — see A2's conceptual-bug finding). Word- and grapheme-based ratios
overstate the apparent gap (gpt2 tam/eng = 20x on words vs 4.85x on bytes)
because they measure two different things at once: real compression AND
morphological word-length differences.

## Headline finding

gpt2 achieves near-zero compression on Tamil/Telugu (~0.99 tokens/byte,
almost 1 token per byte) versus ~0.21 tokens/byte on English — roughly 5x
more tokens for identical user content. MuRIL inverts this entirely
(0.07-0.095 tokens/byte on Hindi/Tamil/Telugu, more efficient than its own
English number). Switching Indic-language traffic to an Indic-aware
tokenizer would cut token counts, and therefore serving cost, by roughly
5-10x for those languages.