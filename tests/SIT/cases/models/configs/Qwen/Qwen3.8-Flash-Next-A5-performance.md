# Qwen3.8 Flash Next A5 performance measurements

These AISBench measurements were collected on November 26–27, 2026, using the
Qwen3.8 Flash Next W4A8 MXFP checkpoint on one eight-device Ascend A5 server.
They provide measured operating points for the
[A5 prefill](Qwen3.8-Flash-Next-w4a8-mxfp-PD-Prefill-A5.yaml) and
[A5 decode](Qwen3.8-Flash-Next-w4a8-mxfp-PD-Decode-A5.yaml) nightly cases.

## Test environment

- Basic 1P1D with four devices per role and DP2/TP2/EP4/PP1 on each side. Tiered KV pooling was disabled.
- Both roles used the A5 openEuler r177 image with CANN 9.3.0, vLLM v0.34.0, and the
  vLLM-Ascend wheel built from `4f49ae332de1b1e572587bb9162cfb991eadf2ed`.
- The maximum context length was 143,360 tokens. Prefill used Fastokens 0.3.5.
  Decode used Static Kernel with Slice excluded, SuperKernel disabled, and MTP with three speculative tokens.
- The 90% single-prefix cases used one shared prefix covering 90% of each input. Prefill-only requests
  went directly to P. End-to-end and decode-only requests went through the 1P1D proxy.
  Decode-only runs used warmed prompts with P-side prefix caching. D-side prefix caching was disabled.
- Fixed-rate tests sent requests at uniform intervals. A configured request rate of zero means
  no pacing limit. Warmup and profiling requests were excluded from the measured rounds.

## Workloads and selected rounds

Each case ran three measured rounds with zero failed requests. The tables use one
representative round per case. For prefill and end-to-end, it is the middle round
when sorted by full-run mean TTFT. For decode, it is the middle round when sorted
by steady-state mean TPOT, with steady-state QPS ascending as the tie-breaker.

| ID | Workload | P prefix cache | Selected round | Mean input tokens | Mean output tokens | Requests issued | Max concurrency | Measured concurrency | Configured req/s |
| --- | --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| A | 128K/1K end-to-end, 90% single prefix | On | 3 | 131072 | 1024 | 160 | 40 | 34.0422 | 4.966642 |
| B | 128K/1 prefill, no shared prefix | Off | 1 | 131084.1750 | 1 | 40 | 10 | 4.3138 | 0.341371 |
| C | 128K/1 prefill, 90% single prefix | On | 3 | 131072 | 1 | 128 | 32 | 17.9828 | 3.498213 |
| D | 128K/1K decode, 150-concurrency retest | On | 1 | 131072 | 1024 | 600 | 150 | 150 | 0 |
| E | 16K/1K end-to-end, no shared prefix | On | 2 | 16396.1558 | 1024 | 1252 | 313 | 26.1168 | 2.018750 |
| F | 16K/1 prefill, no shared prefix | Off | 2 | 16396.0898 | 1 | 256 | 64 | 4.6038 | 2.669779 |
| G | 16K/1K decode | On | 2 | 16396 | 1024 | 1000 | 250 | 250 | 0 |

The decode rows report the AISBench steady-state slices of 450 requests for D and
750 for G. Requests issued are full-run counts. The other rows use full-run metrics.

## Latency

| ID | Mean TTFT (ms) | TTFT P90 (ms) | Mean TPOT (ms) | TPOT P90 (ms) | Mean ITL (ms) | Mean E2E latency (s) |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| A | 6655.9 | 8105.4 | 6.8 | 7.8 | 27.7 | 13.6309 |
| B | 14718.0 | 21312.0 | — | — | 0.1 | 14.7181 |
| C | 6644.4 | 10617.9 | — | — | 0.0 | 6.6445 |
| D | 23783.6 | 25856.5 | 8.5 | 9.1 | 39.5 | 32.4380 |
| E | 1970.2 | 2071.6 | 11.0 | 17.0 | 22.9 | 13.1933 |
| F | 1742.7 | 1842.8 | — | — | 0.0 | 1.7428 |
| G | 13132.9 | 14046.2 | 13.0 | 13.3 | 47.4 | 26.3934 |

TPOT is undefined for the single-output-token cases. Their ITL values are included
as AISBench reports them.

## Throughput

Token throughputs are in tokens/s. Per-device throughput divides by eight for
end-to-end and by four for prefill-only and decode-only.

| ID | QPS | QPM | Input | Input/device | Output | Output/device | Total | Total/device |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| A | 2.4974 | 149.84 | 327343.7231 | 40917.9654 | 2557.3728 | 319.6716 | 329901.0960 | 41237.6370 |
| B | 0.2931 | 17.59 | 38420.1621 | 9605.0405 | 0.2931 | 0.0733 | 38420.4552 | 9605.1138 |
| C | 2.7064 | 162.38 | 354737.6095 | 88684.4024 | 2.7064 | 0.6766 | 354740.3159 | 88685.0790 |
| D | 4.6484 | 278.90 | 609271.0429 | 152317.7607 | 4759.9300 | 1189.9825 | 614030.9730 | 153507.7432 |
| E | 1.9796 | 118.78 | 32457.1314 | 4057.1414 | 2027.0668 | 253.3834 | 34484.1982 | 4310.5248 |
| F | 2.6416 | 158.50 | 43311.7351 | 10827.9338 | 2.6416 | 0.6604 | 43314.3767 | 10828.5942 |
| G | 10.0046 | 600.28 | 164035.1354 | 41008.7838 | 10244.6925 | 2561.1731 | 174279.8279 | 43569.9570 |

## Speculative acceptance

Acceptance uses the D-side Prometheus counter increases across the full
measurement round, summed over both DP ranks. The rate is accepted draft tokens
divided by proposed draft tokens.

| ID | Accepted draft tokens | Proposed draft tokens | Acceptance |
| --- | ---: | ---: | ---: |
| A | 117752 | 138939 | 84.75% |
| D | 446622 | 505746 | 88.31% |
| E | 666920 | 1849473 | 36.06% |
| G | 743697 | 844968 | 88.01% |

The A5 decode nightly YAML uses a fully repeated prompt. Benchmark comparisons
should match its workload, serving configuration, and measurement window.
