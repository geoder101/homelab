# Live model/configuration benchmark scorecard

Updated: 2026-11-08T06:08:28.919508+00:00

Queue state: **complete**

| Batch | Model / variant | State | Context × slots | KV | Warm tok/s | Code time (s) | Code | Tools | Repairs | Native | Vision |
| --- | --- | --- | --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| first | baseline | complete | 262144 × 2 | q4_0 | 71.66 | 462.46 | 21/21 | 9/9 | 6/6 | 6/6 | 6/6 |
| first | qwen3-coder | complete | 262144 × 1 | q4_0 | 243.6 | 17.78 | 11/21 | 9/9 | 3/6 | 6/6 | — |
| first | glm47 | complete | 202752 × 2 | q4_0 | 235.66 | 412.5 | 15/21 | 9/9 | 4/6 | 2/6 | — |
| first | gemma4 | complete | 262144 × 2 | q4_0 | 178.41 | 376.83 | 19/21 | 0/9 | 6/6 | 6/6 | 6/6 |
| first | qwen36 | complete | 262144 × 2 | q4_0 | 214.88 | 431.32 | 20/21 | 9/9 | 6/6 | 6/6 | 6/6 |
| first | ornith | complete | 262144 × 2 | q4_0 | 231.3 | 225.17 | 21/21 | 9/9 | 6/6 | 4/6 | 6/6 |
| batch2 | baseline | complete | 262144 × 2 | q4_0 | 71.67 | 462.43 | 21/21 | 9/9 | 6/6 | 6/6 | 6/6 |
| batch2 | flint | complete | 262144 × 2 | q4_0 | 194.75 | 407.37 | 21/21 | 9/9 | 5/6 | 6/6 | — |
| batch2 | nemotron35 | complete | 262144 × 2 | q4_0 | 333.31 | 246.58 | 20/21 | 9/9 | 5/6 | 6/6 | — |
| batch2 | devstral2 | complete | 131072 × 1 | q8_0 | 72.23 | 57.13 | 12/21 | 9/9 | 3/6 | 6/6 | 6/6 |
| batch2 | occamy | complete | 262144 × 2 | q4_0 | 269.89 | 222.47 | 18/21 | 9/9 | 5/6 | 5/6 | 6/6 |
| batch2 | muse | complete | 131072 × 2 | q4_0 | 65.69 | 351.28 | 19/21 | 9/9 | 6/6 | 6/6 | 6/6 |
| batch2 | gptoss | complete | 131072 × 2 | q4_0 | 261.37 | 221.23 | 15/21 | 7/9 | 4/6 | 6/6 | — |
| tuning | baseline | complete | 262144 × 2 | q4_0 | 71.65 | 462.46 | 21/21 | 9/9 | 6/6 | 6/6 | 6/6 |
| tuning | qwen38-q3 | complete | 262144 × 2 | q4_0 | 84.81 | 454.62 | 19/21 | 9/9 | 6/6 | 6/6 | 6/6 |
| tuning | qwen38-q4-128k | complete | 131072 × 2 | q4_0 | 71.68 | 462.47 | 21/21 | 9/9 | 6/6 | 6/6 | 6/6 |
| tuning | qwen38-q8-128k | complete | 131072 × 2 | q8_0 | 74.72 | 524.29 | 21/21 | 9/9 | 6/6 | 6/6 | 6/6 |
| tuning | qwen38-newengine | complete | 262144 × 2 | q4_0 | 71.96 | 442.69 | 21/21 | 9/9 | 6/6 | 6/6 | 6/6 |
| tuning | qwen38-nvfp4 | complete | 163840 × 1 | fp8_e4m3 | 68.69 | 747.67 | 20/21 | 9/9 | 6/6 | 6/6 | 6/6 |
| tuning | ornith-q5 | complete | 262144 × 2 | q4_0 | 261.3 | 231.39 | 21/21 | 9/9 | 6/6 | 6/6 | 6/6 |
| tuning | qwen3-coder-q4 | complete | 262144 × 1 | q4_0 | 252.6 | 18.28 | 13/21 | 9/9 | 4/6 | 6/6 | — |
| tuning | gptoss-vllm | complete | 131072 × 2 | auto | 293.5 | 101.83 | 18/21 | 8/9 | 3/6 | 5/6 | — |

Counts remain partial while a configuration is running. Startup failures and incomplete
runs are not silently counted as passes. Different context/slot capacities, sampling
recipes, output budgets and formats must be considered before comparing throughput.
These are local fixtures and read-only native-agent probes, not SWE-bench scores.
The original Qwen precision selection is restored after each candidate; final promotion
requires review and deployed API verification.
