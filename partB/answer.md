# answers.md — capacity reconciliation

Model: FLM-4B-Instruct, 1x L4 24GB, fp16. Verified by running the
calculations directly against bench_log.csv.

## B1
KV bytes/token = 2 × 28 layers × 8 KV-heads × 128 head_dim × 2 bytes(fp16)
              = 114,688 bytes (112.0 KiB)
Usable memory = 24GB × 0.92 = 22.08GB; minus weights (8.40GB) and overhead
(1.6GB) = 12.08GB KV budget → max cacheable tokens = 105,329 → max
concurrent 4096-token sequences ≈ 25.7.

Checked against the log (batch × 4096 = tokens in flight, prompt=3584 sweep):

| batch | tokens_in_flight | predicted_util | logged_util |
|---|---|---|---|
| 4 | 16,384 | 0.156 | 0.16 |
| 8 | 32,768 | 0.311 | 0.31 |
| 16 | 65,536 | 0.622 | 0.62 |
| 24 | 98,304 | 0.933 | 0.93 |
| 32 | 131,072 | 1.244 (impossible) | 0.97 (capped) |
| 48 | 196,608 | 1.867 (impossible) | 0.97 (capped) |

Predicted matches logged almost exactly through batch 24. Past that, the
math implies >100% utilization but the log caps at 0.97 — evidence the
scheduler is refusing admission/preempting rather than truly overflowing
memory.

## B2
reported_tok_s rises through batch 24 (1607.4) then falls at batch 32
(1384.0) and 48 (1298.5). preempted_seqs is 0 through batch 24, then jumps
to 7 at batch 32 and 23 at batch 48 — exactly where the ~25.7-sequence
ceiling from B1 is crossed. Once demand exceeds KV-cache capacity, the
scheduler must preempt sequences mid-generation; on resumption they must
re-prefill the full 3584-token prompt, wasting compute that produces no
new output. Both reported throughput and true goodput fall together as a
result (goodput: 200.9 → 173.0 → 162.3 tok/s from batch 24→32→48).

Proposed fix: cap max_num_seqs at 24-25 for this context length, keeping
the scheduler under the KV-cache ceiling (predicted: preempted_seqs → 0,
throughput scales linearly again). Alternative: fp8 KV-cache quantization
halves bytes/token to 57,344, roughly doubling capacity to ~51 sequences,
pushing the preemption cliff out past batch 48.

## B3
Misread column: reported_tok_s. Verified formula matches every row
exactly: (prompt_len + gen_len) × num_requests / wall_clock_s — conflates
one-time prefill tokens with real per-token decode output.

Honest goodput, batch=24/prompt=3584, two independent ways:
- gen_len × num_requests / wall_clock_s = 512 × 24 / 61.16 = 200.9 tok/s
- batch_size × (1000/itl_ms_p50) = 24 × (1000/96.07) = 249.8 tok/s

Both land near 200-250 tok/s, far below the reported 1607.4. The report's
batch-48 prediction of ~3200 tok/s is wrong in direction as well as
magnitude: actual batch-48 goodput is 162.3 tok/s — worse than batch 24,
not better. The report should have said throughput peaks around batch 24
(~200-250 tok/s real output) and degrades past that point due to
KV-cache exhaustion and preemption.

## B4
Pull kv_cache_util (already logged) alongside the scheduler's
preemption/eviction counter. Predicted value: kv_cache_util plateaus at
~0.93-0.97 exactly where preempted_seqs turns nonzero — the pattern
already visible in the batch-24-to-32 transition above, confirming cache
saturation (not e.g. network I/O) as the bottleneck.