# Volta Full NVFP4

## Every routed expert at exactly NVIDIA's NVFP4 precision on 8x V100 from 2018 · the most-used on the cards, the rest from host RAM over PCIe · 28 to 30 tok/s legal-style, 19 to 20 general · slower than Volta Max uPLIft and Volta sPLIt

[![legal-style: 28-30 tok/s](https://img.shields.io/badge/legal--style-28--30%20tok%2Fs-1F6FEB?style=for-the-badge)](#tokens-per-second)
[![general: 19-20 tok/s](https://img.shields.io/badge/general-19--20%20tok%2Fs-8250DF?style=for-the-badge)](#tokens-per-second)
[![routed experts below NVFP4: none](https://img.shields.io/badge/routed%20experts%20below%20NVFP4-none-2EA043?style=for-the-badge)](#what-this-is)
[![KV pool: 2.08M tokens](https://img.shields.io/badge/KV%20pool-2.08M%20tokens-555555?style=for-the-badge)](docs/measurements.md#long-prompts-and-memory-two-tier)

Built by **Nick Mermiges, Esq.**, founder of [PLI Labs](https://proprietarylegal.ai/labs.html) · [proprietarylegal.ai](https://proprietarylegal.ai)

[![proprietarylegal.ai](https://img.shields.io/badge/proprietarylegal.ai-website-0B3D91)](https://proprietarylegal.ai) [![PLI Labs](https://img.shields.io/badge/PLI%20Labs-research-1F6FEB)](https://proprietarylegal.ai/labs.html) [![License: Apache-2.0](https://img.shields.io/badge/license-Apache--2.0-blue)](LICENSE) [![Hardware: 8x V100-SXM2-32GB](https://img.shields.io/badge/hardware-8x%20V100--SXM2--32GB-76B900)](#hardware) [![Related build: Volta Max uPLIft](https://img.shields.io/badge/related%20build-Volta%20Max%20uPLIft-555555)](https://github.com/ProprietaryLegal/Volta-Max-uPLIft-FP4-DeepSeek-V4.1-Flash-1017-NVFP4-8xV100-SXM2-PP2xTP4) [![Earlier build: Volta sPLIt](https://img.shields.io/badge/earlier%20build-Volta%20sPLIt-555555)](https://github.com/ProprietaryLegal/Volta-sPLIt-DeepSeek-V4.1-Flash-1017-8xV100-SXM2-PP2xTP4)

Volta Full NVFP4 serves DeepSeek-V4.1-Flash-1017 from NVIDIA's
[DeepSeek-V4.1-Flash-1017-NVFP4](https://huggingface.co/nvidia/DeepSeek-V4.1-Flash-1017-NVFP4) pack on eight
V100-SXM2-32GB cards (two 4-card NVLink boards, pipeline parallel 2 x tensor parallel 4), with every routed expert at
exactly that pack's precision. NVFP4 folds bit for bit to MXFP4 (4.25 bits per weight), which the v100-skinny QPN
kernel runs as W4A16. The most-used experts stay on the cards. The rest stay in pinned host RAM, and the cards read
them over PCIe in decode and in prompt reading. No routed expert runs below NVFP4 precision at any time. That costs
speed: 28 to 30 tok/s on legal-style text and 19 to 20 tok/s on general text (MEASURED 2026-11-09, 1,522 MHz), slower
than [Volta Max uPLIft](https://github.com/ProprietaryLegal/Volta-Max-uPLIft-FP4-DeepSeek-V4.1-Flash-1017-NVFP4-8xV100-SXM2-PP2xTP4),
which keeps every routed expert on the cards and runs the less-used ones from compressed records, and slower than
[Volta sPLIt](https://github.com/ProprietaryLegal/Volta-sPLIt-DeepSeek-V4.1-Flash-1017-8xV100-SXM2-PP2xTP4), which
runs every routed expert from a compressed record. Use this build when no routed expert may run below NVFP4
precision at any point, for example as the all-NVFP4 output to compare a faster build against.

This repository publishes the recipe and the measurements we have: the exact fold and its bit-equality proof, the
engine source and patches, the serving profile and launcher, the measurement scripts, how to reproduce the build,
and what did not work. Nothing was measured for this page. Every number comes from the measurements of 2026-11-09,
first published in Volta Max uPLIft's release at commit `79dc164`, where this build was called the two-tier mode.
The [measurement log](#measurement-log) lists them.

## Tokens per second

MEASURED on the 8-card host on 2026-11-09 at 1,522 MHz, the same day for both builds, one request at a time, public
prompts, greedy (T=0), thinking off, 1,024 tokens out for the short sets and 512 for the context rows. "Two runs" are
two runs in one boot. How each row was measured: [docs/measurements.md](docs/measurements.md).

| | Volta sPLIt (earlier build) | Volta Full NVFP4 |
|---|---|---|
| Routed experts | IQ2-class records, all on the cards | all at NVFP4 precision, the most-used on the cards and the rest in host RAM |
| Short prompts, legal-style set (two runs) | 118.7 / 118.3 tok/s (21.54 / 21.62 ms per round) | 28.2 / 30.2 tok/s (87.58 / 81.79 ms) |
| Short prompts, general set (two runs) | 136.9 / 137.9 tok/s (21.48 / 21.57 ms) | 18.8 / 20.1 tok/s (174.64 / 163.37 ms) |
| Decode after 46 tokens of context | 125.9 tok/s | 33.7 tok/s |
| Decode after 30,795 tokens | 103.2 tok/s | 30.6 tok/s |
| Decode after 106,531 tokens | 101.0 tok/s | 31.5 tok/s |
| Decode after 257,741 tokens (Volta Full NVFP4: 303,004) | 98.1 tok/s | 30.2 tok/s |
| Prompt reading, 30K to 300K tokens (the context rows, and for Volta sPLIt also the tool loop's first read) | 2,610 to 2,946 tok/s | 1,419 to 1,595 tok/s |
| First token, short prompt | 0.37 to 0.45 s | 0.69 s |

A "round" is one target forward pass that checks the DSpark drafter's proposals. The table carries two measures of
tok/s. On the two "Short prompts" rows, tok/s is the output tokens divided by the whole request time, the wait for
the first token included, as the short-set program (`ab_t1_v2.py --tput` of the Volta sPLIt harness) prints it. The
ms per round beside it is decode time only (first to last streamed token) divided by the rounds. So a short-set
tok/s is lower than tokens per round divided by the decode time per round. On the "Decode after" rows, tok/s is
decode only: output tokens after the first, divided by the time from the first to the last streamed token.

From 46 tokens of context to the longest prompt, decode goes from 125.9 to 98.1 tok/s on Volta sPLIt and from 33.7
to 30.2 tok/s on this build, which changes little with context. Its main cost is reading host-RAM experts over
PCIe (ESTIMATED, [What did not work](#what-did-not-work-or-is-not-yet-good)).

This build also read a 498,099-token prompt: 274.9 s on the calibration boot (card reserves 6 / 9 GiB) and 326.6 s
on the final boot (4.5 / 7.5 GiB), with no out-of-memory error. Its KV pool holds 2,076,870 tokens at a
max-model-len of 524,288.

## What this is

**The question.** DeepSeek-V4.1-Flash-1017 has 384 routed experts in each of 40 layers. At the official MXFP4
precision they take about 269 GiB (ESTIMATED: 384 x 35.4M weights x 40 x 4.25 / 8 bytes). Eight 32 GB V100s have
256 GiB of HBM in total, before attention weights, KV cache and the drafter. So not every expert fits on the cards
at the official precision. This build keeps them all at that precision and puts the overflow in host RAM.

**The pack.** NVIDIA's NVFP4 pack stores the routed experts in NVFP4 (FP8 scale per 16 values). Its card says the
conversion from DeepSeek's MXFP4 was lossless. We fold it back to MXFP4 (one power-of-two scale per 32 values) and
check every value: 44,696,862,720 values bit-equal to the NVFP4 formula in FP32, relative L2 error 0 (MEASURED: all
384 experts x 3 projections x 4 TP ranks in layers 5, 12 and 27, and experts 0, 191 and 383 in the other 37 layers).
The packs take 299,473,305,600 bytes, W2 padded from K=576 to 640 for the kernel. The builder and the check are
[build_packs.py](harness/benchmarks/nvfp4_8r/build_packs.py) and
[accept_p1.py](harness/benchmarks/nvfp4_8r/accept_p1.py), run as
[run_p1.sh](harness/benchmarks/nvfp4_8r/run_p1.sh) does.

**Two tiers.** The experts that route most often stay on the cards. The rest stay in pinned host RAM and the cards
read them over PCIe as mapped host memory. Which experts route most often comes from a decode routing tally
(aggregate counts per layer and expert, private, not published). After the dense weights load, each pipeline stage
measures its free HBM, keeps a per-card reserve for KV, graphs, the drafter and prompt staging, and fills the rest
with the most-used experts. Final boot (MEASURED): stage 0 keeps 5,463 experts on the cards and 2,217 in host RAM,
stage 1 keeps 4,704 and 2,976 (of 7,680 per stage). In prompt reading, the host-RAM experts a chunk needs are staged
into the cards at the same precision.

**Everything else is Volta sPLIt's.** Attention, the dense layers, the shared expert, the Engram tables, the DSpark
drafter, the PP2 x TP4 split and the switch set are served exactly as in Volta sPLIt (same engine, same GGUF for the
non-expert tensors). Only the routed-expert path is new.

<!-- pli:blurb:start -->

> Published by [PLI Labs](https://proprietarylegal.ai/labs.html), the open research arm of [Proprietary Legal Intelligence](https://proprietarylegal.ai). PLI builds AI systems around a legal organization's own work product, combining retrieval, staged workflows, validation, and attorney review. Visit [proprietarylegal.ai](https://proprietarylegal.ai) to explore our work and discuss what your organization needs.

<!-- pli:blurb:end -->

## What runs

| | |
|---|---|
| GPUs | 8x V100-SXM2-32GB on two 4-card SXM2 boards, NVLink inside each board. Four PLX switches, each on its own x16 Gen3 root port, two cards per switch at Gen3 x8 (MEASURED from sysfs) |
| Host RAM | 363 GiB available with no server running (MEASURED). The host-RAM experts take about 101 GB (ESTIMATED from the tier counts) |
| Routed experts | all of them from [nvidia/DeepSeek-V4.1-Flash-1017-NVFP4](https://huggingface.co/nvidia/DeepSeek-V4.1-Flash-1017-NVFP4) @ `480550d3`, folded to MXFP4 (E8M0 scale per 32) packs per layer and TP rank |
| Other weights | as Volta sPLIt: [ISTA-DASLab GSQ-RCO GGUF](https://huggingface.co/ISTA-DASLab/DeepSeek-V4.1-Flash-1017-GSQ-RCO-GGUF) for attention, dense and shared-expert tensors (Q5_K 4.76 GiB, Q8_0 0.36 GiB, BF16 0.26 GiB, F32 0.15 GiB by tensor bytes, MEASURED from the file header) and the IQ4_NL Engram tables. The loader drops that file's routed-expert records before any read. Config, tokenizer and DSpark drafter (FP8) from [deepseek-ai/DeepSeek-V4.1-Flash-1017](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash-1017) |
| Expert kernel | v100-skinny's QPN kernel as packaged in 1Cat-vLLM ([#742](https://github.com/1CatAI/1Cat-vLLM/pull/742)): `skinny_moe_qpn_sm70`, E8M0 scale mode, W4A16, FP16 `mma.sync`. V4.1's limited SwiGLU and the weighted sum in FP32. CUDA graphs for 1 to 6 rows |
| Host-RAM experts | prepacked once into a page-aligned host arena, registered with CUDA and read in place by the kernel (zero copy) in decode. In prompt reading, staged into two 32-expert HBM buffers on a copy stream, one staging pair per card |
| Engine | the Volta sPLIt engine (1Cat-vLLM #1277 with its patch series and production patches) plus two new files and three changed files ([engine/](engine/)), as released in Volta Max uPLIft at commit `79dc164` |
| Layout | PP2 x TP4: layers 0-19 and the Engram lookup on one board, layers 20-39, the head and the drafter on the other |
| Card reserve | per stage, from measured peaks plus about 0.5 GiB: 4.5 GiB per card on stage 0, 7.5 GiB on stage 1 |
| Context | max-model-len 524,288, KV pool 2,076,870 tokens, one request at a time (`max-num-seqs 1`) |
| Serving profile | [serve-nvfp4-pp2.yaml](engine/configs/serve-nvfp4-pp2.yaml) (`nvfp4_8r_residency: two_tier`, one slot, reserves 4.5 / 7.5 GiB), started by [run_serve.sh](engine/configs/run_serve.sh) |

## What worked

All MEASURED on 2026-11-09. Single-card results come from one card of a separate 4-card test host at 150 W.

| Item | Result |
|---|---|
| Exact NVFP4 to MXFP4 fold | 44,696,862,720 values bit-equal, relative L2 0 ([above](#what-this-is)) |
| Scale range on Volta | the engine's FP16 rebase accepted all 61,440 expert records (40 layers x 4 ranks x 384). Widest exponent span 8, kernel limit 29 |
| QPN expert layer, one card | worst relative L2 4.78e-4 against an FP32 reference built from the raw NVFP4 weights (layers 5, 12 and 27 at 1, 6 and 12 rows). CUDA graphs replay bit-equal to eager. Graph time 131 us per layer at 1 row, 280 to 348 us at 6 rows |
| Hot and host split | bit-equal to all experts on the card, for natural and forced host routes, decode and prompt reading |
| MXFP4 Engram from NVIDIA's pack | packed to 144-byte rows (110,595,290,400 bytes). Sampled rows equal the checkpoint, and decoded rows are bit-equal to FP32 in FP16. Built and checked, not the default (the IQ4_NL table decodes 4 to 5 times more rows per second in eager mode) |
| Eight-card boot | ready in 288 s, and it answers. KV pool 2,076,870 tokens |
| Per-stage reserve | one reserve for both stages ran out of memory on a 100K prompt. Reserves set per stage from peaks sampled every 15 s (stage 0 peak 30,456 MiB, stage 1 30,304 MiB on the calibration boot) fixed it, including a 498,099-token prompt |
| Decode flat with context | 30.2 to 33.7 tok/s from 46 to 303,004 tokens |

## What did not work or is not yet good

All MEASURED on 2026-11-09 unless a row is marked ESTIMATED.

| Item | Result |
|---|---|
| Slower than Volta sPLIt and Volta Max uPLIft | on the short sets this build reaches 14 to 26 % of Volta sPLIt's tok/s (arithmetic from the table above). Volta Max uPLIft is faster too: its page gives its speeds |
| Prompt reading | 1,419 to 1,595 tok/s, 54 to 56 % of Volta sPLIt's rate at 30,795 and 106,531 tokens (arithmetic). Every expert that is not on the cards is staged from host RAM while reading |
| Slow first token | short prompts: 0.69 s against 0.37 to 0.45 s. A 216K-token prompt read in full: about 136 s against 73.4 to 78 s |
| Reading experts from host RAM | one host-RAM expert per token at 1 row adds about 975 us to a layer. At 6 rows each costs about 1.0 to 1.4 ms per layer (about 3.5 to 4.9 GB/s over PCIe). Other miss paths on one card: copy-engine DMA 743.7 us per expert, GPU gather 1,014.8 us, AVX2 host kernel 2,953.7 us across 4 TP slices. ESTIMATED from those and the tally: no path comes within about 10 ms per round of all experts on the cards. That is why Volta Max uPLIft keeps every routed expert on the cards |
| General text | the most-used set comes from a private routing tally (not published), and general text misses it more often. General text decodes at about half the legal-style speed (174.64 against 87.58 ms per round, run 1). The cause is the host-RAM reads: with no experts in host RAM the gap goes away ([Volta Max uPLIft, What worked](https://github.com/ProprietaryLegal/Volta-Max-uPLIft-FP4-DeepSeek-V4.1-Flash-1017-NVFP4-8xV100-SXM2-PP2xTP4#what-worked)) |
| Memory margin | at the final reserves the highest card memory seen was 32,086 MiB (stage 0) and 31,356 MiB (stage 1) against a limit of about 32,494 MiB, about 0.4 and 1.1 GiB of margin |
| First boots | each failed fast and was fixed at its cause: a host mapping requested on card 0 for every worker, prompt staging buffers kept per layer instead of per card (out of memory), and one shared reserve for both stages (out of memory on a 100K prompt) |

## Quality

**Value-exact experts.** The routed experts are bit-equal to DeepSeek's official MXFP4 experts
([What this is](#what-this-is)). Everything else is served as in Volta sPLIt.

**Repeatable output.** On three public prompts (1,024 tokens each, greedy, thinking off) this build's output
repeated token for token on a second capture in the same boot (MEASURED 2026-11-09).

**Upstream accuracy.** NVIDIA's own accuracy table for the pack, measured with vLLM on 4x GB300 (UPSTREAM), is within
run noise of the MXFP4 source, for example GPQA-D 92.93 against 92.77 and AA-LCR 80.19 against 80.41. Those figures
are NVIDIA's, on NVIDIA's serving stack, not this build. This build's own multiple-choice accuracy, measured as the
reference of Volta Max uPLIft's closeness test, is a dated row of the [measurement log](#measurement-log).

## Measurement log

One row per measurement, newest first. Nothing was measured when this repository was split from Volta Max uPLIft.
Details: [docs/measurements.md](docs/measurements.md).

| Date | Build or profile | Test | Result | Basis |
|---|---|---|---|---|
| 2026-11-11 | Volta Full NVFP4 (its profile `serve-nvfp4-pp2.yaml`), the reference of Volta Max uPLIft's closeness test | 513 MMLU questions, greedy, thinking off | accuracy 83.8 % (95 % interval 80.7 % to 86.7 %), 38 questions without an answer counted as wrong. A repeat in the same boot on 50 contexts and 50 questions was identical on all of them ([the test](https://github.com/ProprietaryLegal/Volta-Max-uPLIft-FP4-DeepSeek-V4.1-Flash-1017-NVFP4-8xV100-SXM2-PP2xTP4#how-close-to-nvfp4)) | MEASURED, 1,475 MHz |
| 2026-11-09 | Volta Full NVFP4, a later boot on the engine with the three-tier changes | short set, general | 18.3 tok/s (178.95 ms per round) against 18.8 tok/s (174.64 ms) on the first run: 2.5 % per round, inside the 6.5 % within-boot noise | MEASURED |
| 2026-11-09 | Volta Full NVFP4 | 498,099-token prompt | 274.9 s (reserves 6 / 9 GiB), 326.6 s (4.5 / 7.5 GiB), no out-of-memory error | MEASURED |
| 2026-11-09 | Volta Full NVFP4 | one chat, 216K-token public prompt, four requests | first token 134.3 to 140.0 s. Every follow-up re-read the whole prompt, which is the test's own doing ([details](docs/measurements.md#harness-shaped-tests)) | MEASURED |
| 2026-11-09 | Volta sPLIt, Volta Full NVFP4 | short sets, context rows, prompt reading | the [tokens-per-second table](#tokens-per-second) | MEASURED, same day |
| 2026-11-09 | Volta Full NVFP4 | single-card checks: fold, scale range, expert layer, hot and host split, miss paths | [What worked](#what-worked), [docs/measurements.md](docs/measurements.md#single-card-checks-test-host-one-card-150-w) | MEASURED, one card of the test host |

## Hardware

Eight V100-SXM2-32GB on two NVLink quad boards, the same host and settings as the Volta sPLIt measurements of the
same day. Each of the four PCIe switches has its own x16 Gen3 root port and two cards at Gen3 x8. Volta has FP16
`mma.sync.m8n8k4` only: no FP4 or FP8 tensor cores, no `cp.async`, no `ldmatrix`. Every 4-bit format is decoded in
software into FP16 operands, so low-bit experts pay by reading fewer bytes, not by faster math. v100-skinny measured
a read ceiling of about 879 GB/s on a V100-SXM2 and its QPN kernel at 77 % of it at 1 row (UPSTREAM).

## Reproduce

1. Build the Volta sPLIt engine as its
   [Reproduce section](https://github.com/ProprietaryLegal/Volta-sPLIt-DeepSeek-V4.1-Flash-1017-8xV100-SXM2-PP2xTP4#reproduce)
   describes. Then apply [engine/patches/](engine/patches/) in the engine source root (`patch -p1`, each header gives
   the sha256 of the base file and of the result) and copy the two new files from [engine/src/](engine/src/) into
   the same paths of the vllm tree. See [engine/README.md](engine/README.md).
2. Download NVIDIA's NVFP4 pack (revision `480550d3`), the ISTA-DASLab GSQ-RCO GGUF and the DeepSeek config,
   tokenizer and DSpark drafter. [harness/run/copy_weights.sh](harness/run/copy_weights.sh) checks every pack file
   against the Hugging Face sha256. The pack builder refuses to start without its `ALL-VERIFIED` log.
3. Provide a decode routing tally: routing counts per layer and expert from your own traffic, in the array layout
   that `choose_residency` reads (see the engine files). Ours is private and not published.
4. Build and check the packs on CPU: `build_packs` then `accept_p1` (bit equality), as in
   [harness/benchmarks/nvfp4_8r/run_p1.sh](harness/benchmarks/nvfp4_8r/run_p1.sh).
5. Lay out the run directory. The scripts were measured in this layout and find each other there, not in this
   repository's folders. `$LANE` is this build's root (with `engine/venv/`, `engine/src/`, `models/` and `packs/`),
   `$PROD` the Volta sPLIt checkout (its bench5 harness scripts). From this repository's root:

   ```bash
   export LANE=/path/to/build PROD=/path/to/volta-split
   mkdir -p "$LANE/run/state" "$LANE/run/cache" "$LANE/logs/baseline" "$LANE/engine/src/benchmarks/nvfp4_8r"
   cp engine/configs/* harness/run/* "$LANE/run/"
   cp harness/benchmarks/nvfp4_8r/* "$LANE/engine/src/benchmarks/nvfp4_8r/"
   ```

   Copy Volta sPLIt's run tools into `$LANE/run/tools/` (`env.sh` puts them on the Python path), and replace `$LANE`
   in the profile with your path. When `LANE` or `PROD` is not set, the shell scripts that read it stop with a
   message and the Python files stop with a `KeyError` naming the variable.
6. Start with `cd "$LANE/run" && PROFILE=serve-nvfp4-pp2.yaml CUDA_VISIBLE_DEVICES=... bash run_serve.sh nvfp4` (the last word names the server's log and pid files).
7. Measure with `$LANE/run/`[measure_boot.sh](harness/run/measure_boot.sh) (short sets, long prompts, repeat) and
   [ctx_tput.py](harness/run/ctx_tput.py) (context rows). The legal-style prompt file is not published. Any file of
   one public prompt per line works, saved as `$LANE/logs/baseline/legal-public-prompts.txt`.

## Repository layout

| Path | What it is |
|-|-|
| `docs/` | [measurements](docs/measurements.md) and its [index](docs/README.md) |
| `engine/src/` | the two new engine files: the native expert path and the tier planner |
| `engine/patches/` | one patch per changed engine file (loader, GGUF MoE method, Engram reader), against the Volta sPLIt engine |
| `engine/configs/` | launcher, environment and the serving profile |
| `harness/run/` | measurement scripts: boots, short sets, long prompts, context rows, harness-shaped tests, card memory |
| `harness/benchmarks/nvfp4_8r/` | pack builder, acceptance checks, single-card benches and CPU tests |
| `tools/` | `check_data_boundary.sh` (pre-publication scan) |
| `.publication/SOURCE.json` | what each file came from and what was left out |

`engine/` and `harness/` are Volta Max uPLIft's release at commit `79dc164`, file for file, with two exceptions:
`engine/configs/` holds only this build's profile, and `engine/README.md` lists only that profile. The engine and
the harness also carry the code of Volta Max uPLIft's three-tier step (the tier planner and its checks). With this
build's profile the loader takes only the residency type from the planner's file.

## Related builds

Same model, same eight cards, same engine.

| Build | Routed experts | Serving profile |
|---|---|---|
| Volta Full NVFP4 (this repository) | every routed expert at exactly NVIDIA's NVFP4 precision, the most-used on the cards and the rest served from host RAM over PCIe | `serve-nvfp4-pp2.yaml` in [engine/configs/](engine/configs/) |
| [Volta Max uPLIft](https://github.com/ProprietaryLegal/Volta-Max-uPLIft-FP4-DeepSeek-V4.1-Flash-1017-NVFP4-8xV100-SXM2-PP2xTP4) | as much of the routed work at NVIDIA's NVFP4 precision as fits, every routed expert on the cards in decode, the rest from IQ2-class records. Faster than this build | `hot_iq.yaml.template` and `serve_hot_iq.sh`, in that repository's `engine/configs/` from commit [`c672630`](https://github.com/ProprietaryLegal/Volta-Max-uPLIft-FP4-DeepSeek-V4.1-Flash-1017-NVFP4-8xV100-SXM2-PP2xTP4/commit/c6726309d03171f6fa77076371c9ac11879d086a) |
| [Volta sPLIt](https://github.com/ProprietaryLegal/Volta-sPLIt-DeepSeek-V4.1-Flash-1017-8xV100-SXM2-PP2xTP4) | every routed expert from a compressed IQ2-class record, all on the cards. The fastest of the three | in its own repository |

This repository's engine is Volta Max uPLIft's as released at commit `79dc164`. From commit [`c672630`](https://github.com/ProprietaryLegal/Volta-Max-uPLIft-FP4-DeepSeek-V4.1-Flash-1017-NVFP4-8xV100-SXM2-PP2xTP4/commit/c6726309d03171f6fa77076371c9ac11879d086a) on, Volta Max
uPLIft carries the current engine, which adds the reader for its package's tier map.

## Work with PLI Labs

[PLI Labs](https://proprietarylegal.ai/labs.html) is the open research arm of Proprietary Legal Intelligence. PLI builds
AI around an organization's own work product: retrieval over its documents, staged workflows, validation of every output
and attorney review before anything goes out. This stack is how we run that privately, on hardware we own. If you want
private, on-prem LLM serving of your own, from choosing the hardware to the measured stack and the workflows on top, we
consult on exactly that: [proprietarylegal.ai](https://proprietarylegal.ai).

## Licence

Original documentation and code are Apache-2.0 ([LICENSE](LICENSE)). See [NOTICE](NOTICE). Patches against third-party
code keep that code's licence. No model weights, packs, wheels, compiled libraries, routing tallies or prompt corpora are
distributed here.

## Credits

This build stands on other people's work:

- **DeepSeek**: DeepSeek-V4.1-Flash-1017, its official MXFP4 experts, the DSpark drafter and the Engram design.
- **NVIDIA**: the DeepSeek-V4.1-Flash-1017-NVFP4 pack whose experts this build serves, and its accuracy table.
- **1CatAI / 1Cat-vLLM**: the Volta vLLM fork. [#1277](https://github.com/1CatAI/1Cat-vLLM/pull/1277) (yangzhuxinyzx),
  the DeepSeek-V4.1 package Volta sPLIt ports. [#742](https://github.com/1CatAI/1Cat-vLLM/pull/742) (Peuqui), the
  `sm70_skinny` QPN MoE backend this build calls.
- **Dennis Vertzyas (dnv2003), [v100-skinny](https://github.com/dnv2003/v100-skinny)**: the QPN kernels and the Volta
  bandwidth findings.
- **ISTA-DASLab**: the GSQ-RCO GGUF used for the non-expert tensors. **ggml**: the GGUF and IQ formats.
- **The vLLM project**: the serving engine.
- **Volta sPLIt** (PLI Labs), and every project it credits: the engine, switch set and harness underneath.

Every source, with what we took from it, is in [CREDITS.md](CREDITS.md). If you build on this repository, please credit
them as well.
