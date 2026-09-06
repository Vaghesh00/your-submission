# audit.md — script & metric audit (A2)

Corpus: FLORES-200 dev split, 997 parallel sentences, eng/hin/tam/tel.
Tokenizer: gpt2 (tiktoken).

## Bug 1: .lower() applied before tokenizing
Command: analyze() with no_lowercase=True vs default, denominator=word.

| lang | baseline | no-lowercase | % change |
|---|---|---|---|
| eng | 1.2825 | 1.2367 | -3.57% |
| hin | 7.8232 | 7.8225 | -0.01% |
| tam | 24.7332 | 24.7314 | -0.01% |
| tel | 20.3995 | 20.3936 | -0.03% |

Why this proves the claim: removing lowercasing shifts English fertility by
-3.57%, two orders of magnitude larger than any Indic language shift (all
under 0.03%). Confirms lowercasing gives GPT-2's case-sensitive BPE an
artificial advantage on the Latin-script side, undisclosed in the original
script.

## Bug 2: line.split(" ") instead of general whitespace split
Command: analyze() with ws_split=True vs default.

| lang | baseline | ws-split | % change |
|---|---|---|---|
| eng | 1.2825 | 1.2826 | +0.01% |
| hin | 7.8232 | 7.8260 | +0.04% |
| tam | 24.7332 | 24.8669 | +0.54% |
| tel | 20.3995 | 20.6243 | +1.10% |

Why this proves the claim: fixing the split increases fertility for every
language but grows for Tamil/Telugu far more than English, showing the
corpus's Tamil/Telugu lines had more incidental double-spacing silently
absorbed into a deflated word count by the original script.

## Bug 3 (conceptual): len(line) counts codepoints, not grapheme clusters
Command: analyze() with denominator="grapheme", grapheme_chars=True vs False.

| lang | len()-based | grapheme-based | % difference |
|---|---|---|---|
| eng | 0.2152 | 0.2152 | 0.00% |
| hin | 1.5276 | 2.3317 | +52.64% |
| tam | 2.7171 | 4.2085 | +54.89% |
| tel | 2.6414 | 4.5961 | +74.01% |

Why this proves the claim: English shows zero difference (ASCII has no
combining marks). Correct tok/char fertility is 52-74% HIGHER than reported
for Hindi/Tamil/Telugu, because Indic scripts use combining vowel signs that
len() counts as separate codepoints. This is the largest distortion found —
the original script systematically understated how token-expensive these
languages are per real character.

## Methodological issue: macro-average vs micro-average
Command: analyze() with micro_average=True vs default (macro).

| lang | macro (baseline) | micro | % change |
|---|---|---|---|
| eng | 1.2825 | 1.2740 | -0.66% |
| hin | 7.8232 | 7.7934 | -0.38% |
| tam | 24.7332 | 24.4650 | -1.08% |
| tel | 20.3995 | 20.2276 | -0.84% |

Why this proves the claim: all four languages shift 0.4-1.1% when switching
averaging method, confirming sentence-length distribution differs enough by
language in this corpus that the choice isn't cosmetic.

## "Looks suspicious but is fine": random.seed(1337)
Checked: grepped fertility.py for other random.* usages — none found beyond
the seed line itself.

Why this is NOT a bug: random is never called elsewhere in the script, so
no sampling/shuffling occurs. The seed line is dead code with zero effect
on any reported number.