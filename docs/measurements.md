# Measurements

Every number here is MEASURED on 2026-11-09 unless its row or section gives another date or it is labelled
ESTIMATED (arithmetic shown) or UPSTREAM (the named source). The raw receipts are private build logs and are not
published; this page reduces them to tables and names the receipt each row comes from (paths under the build's
`logs/`).

This page is the part of Volta Max uPLIft's measurements page at commit `79dc164` that covers this build, which
was called the two-tier mode there. The tables keep that name. Nothing was measured when this repository was split
from Volta Max uPLIft.

## Results log

Newest first. A later arm is one more row at the top, with its details in a section below.

| Date | Build or profile | Test | Result | Receipt |
|---|---|---|---|---|
| 2026-11-11 | two-tier (this build's profile), the reference of Volta Max uPLIft's closeness test | 513 MMLU questions, greedy, thinking off | accuracy 83.8 % (95 % interval 80.7 % to 86.7 %), 38 without an answer counted as wrong. Repeat on 50 contexts and 50 questions in the same boot: identical on all of them, 1,475 MHz | published with [the test](https://github.com/ProprietaryLegal/Volta-Max-uPLIft-FP4-DeepSeek-V4.1-Flash-1017-NVFP4-8xV100-SXM2-PP2xTP4#how-close-to-nvfp4) in Volta Max uPLIft (`quality/closeness-test/`) |
| 2026-11-09 | two-tier, a later boot on the engine with the three-tier changes | short set, general | 18.3 tok/s over the whole request, with 178.95 ms per round and 3.32 tokens per round in decode | `baseline/t3-N2a-g1-tput.out` |
| 2026-11-09 | two-tier, Volta sPLIt | short sets, context rows, long prompts, harness-shaped test | the sections below | private boot receipts of the same day |

## How it was measured

- **Host and builds.** The 8-card host (8x V100-SXM2-32GB, two NVLink boards, PP2 x TP4). One build served at a time,
  and Volta sPLIt and this build were measured on the same day with the same prompts.
- **One request at a time**, greedy (T=0), streamed. A round is one verify pass, counted from the server's draft
  counters. "ms per round" is always decode time (first to last streamed token) divided by rounds.
- **Two measures of tok/s.** On the short sets, tok/s is the output tokens divided by the whole request time, the
  wait for the first token included. That is what `ab_t1_v2.py --tput` prints as its total. Tokens per round there
  is output tokens divided by rounds. So a short-set tok/s is lower than tokens per round divided by ms per round.
  Example (two-tier, general set, run 1): 18.8 tok/s over the three requests. The decode figures, 3.32 tokens per
  round at 174.64 ms per round, give 19.0 tok/s (arithmetic). The difference is mostly the three requests' wait
  for the first token. On the context rows, the harness-shaped tests and the tool loop, tok/s is decode only:
  output tokens after the first, divided by the time from the first to the last streamed token. There tok/s x ms per round gives the tokens per round.
- **Short sets** (`harness/run/measure_boot.sh`): the Volta sPLIt harness's `ab_t1_v2.py --tput` with its three
  general public prompts (an essay, a hash table, a Python module), and the same measurement over three generic
  legal-style public prompts (`harness/run/tput_prompts.py`; the prompt file is not published). Thinking off,
  `ignore_eos`, 1,024 tokens out. Run 1 follows a warm-up; run 2 repeats in the same boot without one.
- **Context rows** (`harness/run/ctx_tput.py`): a legal-style memorandum task after public filler text (Python
  standard-library source), 512 tokens out. "Read" is the time to the first token of a prompt nobody read before.
- **Long prompts** (`harness/run/longprompt.py`): public Python source, 48 tokens out, card memory sampled every 15 s
  (`harness/run/memwatch.sh`).
- **Harness-shaped tests**: `harness/run/harness_shape.py` (one chat, a 216K-token public prompt, thinking on, 300
  tokens, each follow-up adding the answer and 8,000 characters) and, for Volta sPLIt's first read,
  `harness/run/harness_loop.py` (a tool loop in the Anthropic Messages API format).
- **Single-card checks** ran on one card of a separate 4-card V100 test host at a 150 W cap, with no other compute
  process (each check refuses to run otherwise).

## Short sets

In this table tok/s is over the whole request, the first-token wait included. The ms per round is decode only, and
tokens per round is output tokens divided by rounds. So tok/s x ms per round gives less than the last column
([How it was measured](#how-it-was-measured)).

| Build | Set | Run 1: ms per round (decode), tok/s (whole request) | Run 2: the same | Tokens per round |
|---|---|---|---|---|
| Volta sPLIt | general | 21.48 ms, 136.9 tok/s | 21.57 ms, 137.9 tok/s | |
| Volta sPLIt | legal-style | 21.54 ms, 118.7 tok/s | 21.62 ms, 118.3 tok/s | |
| Two-tier | general | 174.64 ms, 18.8 tok/s | 163.37 ms, 20.1 tok/s | 3.32 |
| Two-tier | legal-style | 87.58 ms, 28.2 tok/s | 81.79 ms, 30.2 tok/s | 2.52 |

Within-boot noise on two-tier was 6.5 %. A later two-tier boot, on the engine with the three-tier changes, gave
18.3 tok/s (178.95 ms per round) on the general set (`baseline/t3-N2a-g1-tput.out`), 2.5 % per round from run 1 above.

Time to first token on the short sets: Volta sPLIt 0.37 to 0.45 s and two-tier 0.69 s.

## Context

| Context tokens | Volta sPLIt | Two-tier |
|---|---|---|
| 46 | 125.9 tok/s | 33.7 tok/s (79.07 ms) |
| 30,795 | 103.2 tok/s, read 11.8 s | 30.6 tok/s (82.54 ms), read 21.7 s |
| 106,531 | 101.0 tok/s, read 38.2 s | 31.5 tok/s (84.16 ms), read 67.9 s |
| 257,741 | 98.1 tok/s (26.85 ms), read 88.7 s | |
| 303,004 | | 30.2 tok/s (87.19 ms), read 190.0 s |

Prompt reading from these rows, in prompt tokens per second (arithmetic): Volta sPLIt 2,610 / 2,789 / 2,906;
two-tier 1,419 / 1,569 / 1,595 (303,004 tokens). The tool loop's first read of 216,245 tokens
([Harness-shaped tests](#harness-shaped-tests)) ran at 2,946 tok/s on Volta sPLIt. The README's prompt-reading row
covers these context rows and, for Volta sPLIt, that first read: 2,610 to 2,946 and 1,419 to 1,595 tok/s. Speed
tests stop at about 250K tokens of context. The longest prompt used since is 257,741 tokens. The two-tier row of 303,004 tokens was run before that limit.

## Long prompts and memory (two-tier)

| Boot | Card reserve (stage 0 / 1) | Prompt | Time | Notes |
|---|---|---|---|---|
| Calibration | 6 / 9 GiB | 106,496 tokens | 74.6 s (1,428 tok/s) | peak card memory 30,456 / 30,304 MiB |
| Calibration | 6 / 9 GiB | 498,099 tokens | 274.9 s | no out-of-memory error |
| Final | 4.5 / 7.5 GiB | 104,049 tokens | 72.2 s (1,441 tok/s) | |
| Final | 4.5 / 7.5 GiB | 498,099 tokens | 326.6 s (1,525 tok/s) | peak card memory 32,086 / 31,356 MiB, limit about 32,494 |

An earlier boot with one 6 GiB reserve for both stages ran out of memory on a 100K-token prompt on stage 1, which
holds the drafter and the head. KV pool on every boot: 2,076,870 tokens at max-model-len 524,288 (the engine reports
3.96x concurrency for 524,288-token requests).

## Residency at load

| Mode | Stage | Free per card after dense load | Expert budget | On the cards | Host | Free after |
|---|---|---|---|---|---|---|
| Two-tier | 0 (layers 0-19) | 29.30 GiB | 24.80 GiB | 5,463 | 2,217 | 3.81 GiB |
| Two-tier | 1 (layers 20-39) | 28.86 GiB | 21.36 GiB | 4,704 | 2,976 | 6.89 GiB |

Counts are expert slices per rank set (each of the four TP ranks of a stage holds the same experts); 7,680 per stage.
Decode route shares by the private tally (aggregates only): host-RAM experts 0.841 % of routes (stage 0) and
1.392 % (stage 1). Boot to ready: 288 s.

## Harness-shaped tests

Two-tier, one chat, 216K-token public prompt, thinking on, 300 tokens per request:

| Request | Context tokens | First token | Decode |
|---|---|---|---|
| 0 | 215,901 | 136.0 s | 38.6 tok/s |
| 1 | 217,985 | 134.3 s | 30.6 tok/s |
| 2 | 219,903 | 137.3 s | 31.4 tok/s |
| 3 | 221,827 | 140.0 s | 43.9 tok/s |

Every follow-up re-read the whole prompt. That is the test's own doing, not an engine defect: the engine keeps a
prompt only when the next request repeats the previous sequence, generated tokens included, to within 124 tokens of
its end, and this test drops the reasoning of earlier turns. Volta sPLIt on the same test also re-read (first token
78.0, 74.9 and 76.5 s).

Volta sPLIt, tool loop (`harness_loop.py`, Anthropic Messages API format, thinking on): first read 216,245 tokens in
73.4 s (2,946 tok/s). This build did not run the tool loop.

## Single-card checks (test host, one card, 150 W)

| Check | Result |
|---|---|
| Pack fold (CPU) | 44,696,862,720 values bit-equal to the NVFP4 formula in FP32, relative L2 0; 887.9 s |
| Scale rebase (CPU) | 61,440 expert records, 0 failures, widest exponent span 7 (W13) and 8 (W2), kernel limit 29 |
| Expert layer accuracy | worst relative L2 4.78e-4 against FP32 references from the raw NVFP4 weights (layers 5, 12, 27; 1, 6, 12 rows; ranks 0 and 3) |
| Expert layer time, CUDA graph | 1 row 130.05 to 131.07 us; 6 rows 279.55 to 348.16 us |
| Expert layer time, eager, 6 rows | 337.15 to 364.42 us |
| Graph replay | bit-equal to eager on two routes, 1 to 6 rows |
| Hot and host split | bit-equal to all experts on the card (natural and forced host routes, 1/6/12 rows, prompt chunks of 64 and 512 rows with up to 118 host experts staged) |
| Host expert, zero copy | one host expert per token at 1 row: 1,127.81 us against 152.45 us all on the card (eager); about 1.0 to 1.4 ms per host expert per layer at 6 rows |
| Split overhead with no host route (graph) | +63.5 us per layer at 1 row, +101.4 us at 6 rows |
| Miss path: copy-engine DMA | 743.7 us per expert (6.6 GB/s), linear to 6 experts; host-issued copies cannot be captured in a graph |
| Miss path: GPU gather | 1,014.8 us (1 expert) to 7,788 us (6) |
| Miss path: AVX2 host compute | 2,953.7 us per expert across 4 TP slices (10 threads); activation round trip 57.1 us |
| Prompt staging, 512 rows, every expert staged | 65 to 93 ms against 5.2 to 5.5 ms all on the card |
| MXFP4 Engram rows | bit-equal to FP32 in FP16 on 2 x 4,098 rows; 0.91 to 1.01 M rows/s against 4.2 to 4.6 M rows/s for IQ4_NL (eager) |

ESTIMATED from the miss-path rows and the tally (6 tokens x top-6 per round, independent tokens, 1.33 distinct
host experts per layer per round on average): all experts on the cards 36.1 ms per round; zero copy 96.9; DMA 69.8 to
79.8; GPU gather 84.8 to 94.8; host compute 190 to 200. No path comes within about 10 ms per round of all on the cards.
