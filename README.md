# Volta Full NVFP4: DeepSeek-V4.1-Flash-1017 on eight V100-SXM2 32 GB cards with every routed expert at exactly NVIDIA's NVFP4 precision, the most-used on the cards and the rest read from host RAM (PP2 x TP4)

Built by **Nick Mermiges, Esq.**, founder of [PLI Labs](https://proprietarylegal.ai/labs.html) · [proprietarylegal.ai](https://proprietarylegal.ai)

Weights: every routed expert is from [nvidia/DeepSeek-V4.1-Flash-1017-NVFP4](https://huggingface.co/nvidia/DeepSeek-V4.1-Flash-1017-NVFP4)
(MIT, revision `480550d3`), NVIDIA's NVFP4 pack of [deepseek-ai/DeepSeek-V4.1-Flash-1017](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash-1017)
(MIT), folded back to MXFP4 bit for bit. The attention, dense and shared-expert tensors and the IQ4_NL Engram tables
are the [ISTA-DASLab GSQ-RCO GGUF](https://huggingface.co/ISTA-DASLab/DeepSeek-V4.1-Flash-1017-GSQ-RCO-GGUF) (MIT),
whose routed-expert records the loader drops before any read; the config, the tokenizer and the DSpark drafter (FP8)
are DeepSeek's. There is no prebuilt package of this build: you build the packs from NVIDIA's files with the scripts
here. This repository holds no weights, no packs and no compiled libraries, and it downloads nothing.

This repository is the build's recipe and record: the engine changes as two new files and three patches in
[engine/](engine/), the serving profile and launcher in [engine/configs/](engine/configs/), the pack builder, its
bit-equality check, the single-card gates and the CPU tests in [harness/benchmarks/nvfp4_8r/](harness/benchmarks/nvfp4_8r/),
the measurement scripts in [harness/run/](harness/run/), and every measurement in
[docs/measurements.md](docs/measurements.md). Which experts stay on the cards comes from a decode routing tally
(aggregate counts per layer and expert), which is private and not published.

The engine is PLI Labs' Volta sPLIt engine: [1CatAI/1Cat-vLLM](https://github.com/1CatAI/1Cat-vLLM), the SM70 fork of
vLLM, with the DeepSeek-V4.1-Flash package of [#1277](https://github.com/1CatAI/1Cat-vLLM/pull/1277) and Volta sPLIt's
patch series and production patches. This build adds the routed-expert path only. Its experts run on the `sm70_skinny`
QPN backend of [#742](https://github.com/1CatAI/1Cat-vLLM/pull/742) with Dennis Vertzyas'
[v100-skinny](https://github.com/dnv2003/v100-skinny) kernels. Attention, the dense layers, the shared expert, the
Engram tables, the drafter, the two-stage split and the switch set are served as the engine serves them.

Every figure on this page is labelled or sits under a heading that labels it. **MEASURED** is a log, a probe or a
bench on our hardware, with its date. **ARITHMETIC** is computed from measured or read values. **ESTIMATE** is a model,
with its basis. **UPSTREAM** is someone else's number, with its source. The card power and clock limits of each run
are given once, under [Results](#results).

What it does:

- Eight V100-SXM2 32 GB cards as two pipeline stages of four tensor-parallel cards on two NVLink boards: layers 0 to 19
  and the Engram lookup on the first board, layers 20 to 39, the head and the DSpark drafter on the second.
- Every routed expert at exactly NVIDIA's NVFP4 values at all times. No routed expert runs from a lower-precision
  record, in decode or in prompt reading.
- The most-used experts on the cards, the rest in pinned host RAM. After the dense weights load, each stage measures
  its free card memory, keeps a reserve per card for the KV cache, the CUDA graphs, the drafter and prompt staging, and
  fills the rest with experts in order of use: 5,463 of 7,680 on the first stage's cards and 4,704 of 7,680 on the
  second's (MEASURED, final boot). In decode the cards read the host-RAM experts in place over PCIe; in prompt reading
  the ones a chunk needs are staged into card buffers at the same precision.
- 524,288 tokens of context a request, a KV pool of 2,076,870 tokens, one request at a time.
- DSpark speculative decoding with DeepSeek's drafter, five draft tokens.
- vLLM's OpenAI-compatible API server with DeepSeek's tool-call and reasoning parsers, thinking on by default.
- A launcher that refuses to start when a server already listens or a card has a compute process, and a loader that
  requires every routed-expert key of the profile, with no default.

## Second release (2026-11-11, commit `b543e49`): what changed

No code changed. This build's own multiple-choice accuracy was measured, with this repository's profile, as the
reference arm of a closeness test (MEASURED 2026-11-11, conditions B). The test, its rules, receipts and scores are
published with the build that used this one as its reference (see [Related builds](#related-builds)).

| | First release | Second release | Change | Measured in |
|---|---|---|---|---|
| Multiple choice, 513 MMLU questions, greedy, thinking off | not measured | 83.8 % (95 % interval 80.7 to 86.7 %), 38 questions without an answer counted as wrong | | [results log](docs/measurements.md#results-log) |
| Repeatability | three public prompts of 1,024 tokens repeated token for token on a second capture in the same boot (2026-11-09) | a repeat in the same boot on 50 contexts and 50 questions identical on all of them | | results log |

What the figure says: how often this build picks the right answer on a fixed public question set, with every routed
expert at NVIDIA's NVFP4 values. It does not by itself say how close the build is to the unquantized model, or that its
answers are good enough for a given kind of work.

## First release (2026-11-10, commit `fabfca1`): what it held

Published on its own on 2026-11-10, after its first publication as the two-tier mode of a related PLI Labs build (see
[Related builds](#related-builds)), with the engine source and patches and the harness as released then, the two-tier
profile and its launcher, the measurements of 2026-11-09 that cover this build, and the fold with its bit-equality proof.
Nothing was measured for the release itself. Every number is from the measurements of 2026-11-09 (conditions A), and
every one is in [docs/measurements.md](docs/measurements.md).

The steps of 2026-11-09, from the first boots to the profile of the release. The calibration boot set the card
reserves from measured peaks; the final boot runs the reserves of the release.

| | Calibration boot | Final boot | Change | Measured in |
|---|---|---|---|---|
| Card reserve, first / second stage | 6 / 9 GiB | 4.5 / 7.5 GiB (4,831,838,208 / 8,053,063,680 bytes) | -1.5 GiB each | [long prompts and memory](docs/measurements.md#long-prompts-and-memory-two-tier) |
| A prompt of about 100K tokens | 106,496 tokens in 74.6 s (1,428 tok/s) | 104,049 tokens in 72.2 s (1,441 tok/s) | | long prompts and memory |
| A prompt of 498,099 tokens | 274.9 s, no out-of-memory error | 326.6 s (1,525 tok/s), no out-of-memory error | | long prompts and memory |
| Highest card memory, first / second stage | 30,456 / 30,304 MiB | 32,086 / 31,356 MiB, limit about 32,494 | | long prompts and memory |
| Experts on the cards, first / second stage | | 5,463 / 4,704 of 7,680 each; 2,217 / 2,976 in host RAM | | [residency at load](docs/measurements.md#residency-at-load) |
| Boot to ready | | 288 s | | residency at load |

**The per-stage reserve.** An earlier boot with one 6 GiB reserve for both stages ran out of memory on a 100K-token
prompt on the second stage, which holds the drafter and the head. Sampling card memory every 15 s during 100K- and
500K-token prompts on the calibration boot showed about 4.0 GiB needed on the first stage and 6.9 GiB on the second;
the release keeps those plus about 0.5 GiB of margin. The loader takes the reserve as a map from a stage's first layer
to bytes.

**The first boots** each failed fast and were fixed at their cause: a host mapping requested on card 0 for every
worker; prompt staging buffers kept per layer instead of per card (out of memory); and one shared reserve for both
stages (out of memory on a 100K prompt).

**The fold.** NVIDIA's pack stores the routed experts in NVFP4, an FP8 scale per 16 values; its card says the
conversion from DeepSeek's MXFP4 was lossless. The builder folds them back to MXFP4, one power-of-two scale per 32
values, and checks every value: 44,696,862,720 values bit-equal to the NVFP4 formula in FP32, relative L2 error 0, in
887.9 s on CPU (all 384 experts x 3 projections x 4 tensor-parallel ranks in layers 5, 12 and 27, and experts 0, 191
and 383 in the other 37 layers). The packs take 299,473,305,600 bytes, W2 padded from K=576 to 640 for the kernel.

**The native expert path.** It prepacks experts for the QPN kernel into card memory or into a page-aligned pinned host
arena registered with CUDA, and runs the routed experts as one grouped QPN launch per projection and bank, with V4.1's
limited SwiGLU and the weighted sum in FP32, CUDA-graph safe for 1 to 6 rows. Prompt chunks stage host experts into
two shared 32-expert buffers per card on a copy stream.

What was left out: the three-tier and no-host-tier profiles and every measurement of them, and the plans and research
notes. They stay with the related build.

Contributors: the NVFP4 pack and its accuracy table are NVIDIA's; its experts run through Peuqui's
[#742](https://github.com/1CatAI/1Cat-vLLM/pull/742) backend with Dennis Vertzyas'
[v100-skinny](https://github.com/dnv2003/v100-skinny) kernels; the engine descends from yangzhuxinyzx's
[#1277](https://github.com/1CatAI/1Cat-vLLM/pull/1277); the non-expert tensors are ISTA-DASLab's. The fold, the native
expert path, the residency, the per-stage reserve, the harness scripts and the measurements are ours. Every source is
in [CREDITS.md](CREDITS.md).

## Results

All numbers are ours, MEASURED with our own measurement scripts, one request at a time, public prompts only, greedy
(T=0), streamed. The legal-style short set is a file of three public prompts we do not publish.

| Conditions | Runs | Cards, power and clock |
|---|---|---|
| A | every eight-card measurement of 2026-11-09 | the eight-card host, clock 1,522 MHz |
| B | the multiple-choice measurement of 2026-11-11 | the eight-card host, clock 1,475 MHz |
| One card | the single-card checks of 2026-11-09 | one card of a separate four-card V100 test host at a 150 W cap, no other compute process (each check refuses to run otherwise) |

### Speed

Short sets: three public prompts each, thinking off, `ignore_eos`, 1,024 tokens out. Run 1 follows a warm-up; run 2
repeats in the same boot without one. Here tok/s is the output tokens over the whole request time, the wait for the
first token included, as the short-set program (`ab_t1_v2.py --tput` of the Volta sPLIt harness) prints it; ms per round
is decode only, from the first to the last streamed token, divided by the rounds. A round is one target forward pass that
checks the drafter's proposals.

| Set | Run 1: tok/s (whole request), ms per round (decode) | Run 2: the same | Tokens per round | Measured in |
|---|---:|---:|---:|---|
| Legal-style | 28.2 tok/s, 87.58 ms | 30.2 tok/s, 81.79 ms | 2.52 | [short sets](docs/measurements.md#short-sets) |
| General | 18.8 tok/s, 174.64 ms | 20.1 tok/s, 163.37 ms | 3.32 | short sets |

Time to the first token on the short sets: 0.69 s. Within-boot noise was 6.5 %. A later boot on the engine with
the three-tier changes gave 18.3 tok/s on the general set (178.95 ms per round), 2.5 % per round from run 1 above.

Decode after context: a legal-style memorandum task after public filler text (Python standard-library source), 512
tokens out. Here tok/s is decode only: output tokens after the first over the time from the first to the last streamed
token. "Read" is the time to the first token of a prompt nobody read before; prompt reading is the prompt's tokens over
that time (ARITHMETIC).

| Context tokens | Decode | ms per round | Read | Prompt reading | Measured in |
|---:|---:|---:|---:|---:|---|
| 46 | 33.7 tok/s | 79.07 | | | [context](docs/measurements.md#context) |
| 30,795 | 30.6 tok/s | 82.54 | 21.7 s | 1,419 tok/s | context |
| 106,531 | 31.5 tok/s | 84.16 | 67.9 s | 1,569 tok/s | context |
| 303,004 | 30.2 tok/s | 87.19 | 190.0 s | 1,595 tok/s | context |

How to read these figures:

- **Decode hardly changes with context.** 30.2 to 33.7 tok/s from 46 to 303,004 tokens. Its main cost is reading
  host-RAM experts over PCIe, not attention over a long context (ESTIMATE, from the single-card costs below).
- **General text decodes at about half the legal-style speed** (174.64 against 87.58 ms per round, run 1). The
  experts kept on the cards were chosen from a private routing tally, and general text routes to host-RAM experts more
  often. By the tally, 0.841 % of decode routes go to host-RAM experts on the first stage and 1.392 % on the second;
  each such read costs about 1.0 to 1.4 ms per layer at 6 rows (one card).
- **The two measures of tok/s differ.** A short-set tok/s is lower than tokens per round divided by the decode time per
  round. For example, general set, run 1: 3.32 tokens per round at 174.64 ms gives 19.0 tok/s (ARITHMETIC) against
  18.8 tok/s over the whole requests; the difference is mostly the wait for the first token.
- **Speed tests stop at about 250K tokens of context.** The 303,004-token row was run before that limit.

### Long context

| Check | Result | Measured in |
|---|---|---|
| Context | 524,288 tokens a request (`max-model-len`); KV pool 2,076,870 tokens on every boot (the engine reports 3.96x concurrency for 524,288-token requests) | [long prompts and memory](docs/measurements.md#long-prompts-and-memory-two-tier) |
| One cold read of 498,099 tokens, final boot | 326.6 s (1,525 tok/s), no out-of-memory error | long prompts and memory |
| One cold read of 104,049 tokens, final boot | 72.2 s (1,441 tok/s) | long prompts and memory |
| Highest card memory, final boot | 32,086 MiB on the first stage, 31,356 MiB on the second, against a limit of about 32,494 MiB: about 0.4 and 1.1 GiB of margin | long prompts and memory |

One chat shaped like an agent conversation: a 216K-token public prompt, thinking on, 300 tokens a request, each
follow-up adding the answer and 8,000 characters of new public text.

| Request | Context tokens | First token | Decode | Measured in |
|---:|---:|---:|---:|---|
| 0 | 215,901 | 136.0 s | 38.6 tok/s | [harness-shaped tests](docs/measurements.md#harness-shaped-tests) |
| 1 | 217,985 | 134.3 s | 30.6 tok/s | harness-shaped tests |
| 2 | 219,903 | 137.3 s | 31.4 tok/s | harness-shaped tests |
| 3 | 221,827 | 140.0 s | 43.9 tok/s | harness-shaped tests |

Every follow-up read the whole prompt again. That is the test's own doing, not an engine defect: the engine keeps a
prompt only when the next request repeats the previous sequence, generated tokens included, to within 124 tokens of
its end, and this test drops the reasoning of earlier turns. This build did not run the tool-loop test.

### Gates

The checks refuse to run under the wrong conditions and say INCONCLUSIVE rather than pass.

- **The fold.** 44,696,862,720 values bit-equal to the NVFP4 formula in FP32, relative L2 0. The pack builder refuses to
  start without the `ALL-VERIFIED` log of the weight copy, which checks every pack file against its Hugging Face sha256.
- **Scale range on Volta.** The engine's FP16 rebase accepted all 61,440 expert records (40 layers x 4 ranks x 384), 0
  failures; widest exponent span 7 (W13) and 8 (W2) against a kernel limit of 29 (CPU).
- **Expert layer accuracy** (one card): worst relative L2 4.78e-4 against FP32 references built from the raw NVFP4
  weights, layers 5, 12 and 27 at 1, 6 and 12 rows, ranks 0 and 3.
- **Graph replay** (one card): bit-equal to eager on two routes, 1 to 6 rows.
- **Hot and host split** (one card): bit-equal to all experts on the card, for natural and forced host routes at 1, 6
  and 12 rows, and prompt chunks of 64 and 512 rows with up to 118 host experts staged.
- **Repeatable output.** On three public prompts (1,024 tokens each, greedy, thinking off) the output repeated token for
  token on a second capture in the same boot (2026-11-09); on 50 contexts and 50 questions in the same boot, identical on
  all of them (2026-11-11).
- **Multiple choice.** 83.8 % on 513 MMLU questions (95 % interval 80.7 to 86.7 %), 38 without an answer counted as
  wrong (conditions B).
- **The eight-card boot.** Ready in 288 s, and it answers; no out-of-memory error on the 498,099-token prompt.

The routed experts are bit-equal to DeepSeek's official MXFP4 experts; everything else is served as the engine serves
it. NVIDIA's own accuracy table for the pack, measured with vLLM on 4x GB300, is within run noise of the MXFP4 source,
for example GPQA-D 92.93 against 92.77 and AA-LCR 80.19 against 80.41 (UPSTREAM, NVIDIA's model card, not this build).
Answer quality in use is not established by these figures.

### Where the time goes (one card)

| Check | Result | Measured in |
|---|---|---|
| Expert layer, CUDA graph | 1 row 130.05 to 131.07 us; 6 rows 279.55 to 348.16 us (eager at 6 rows 337.15 to 364.42 us) | [single-card checks](docs/measurements.md#single-card-checks-test-host-one-card-150-w) |
| One host expert per token, zero copy, 1 row | 1,127.81 us against 152.45 us all on the card (eager); about 1.0 to 1.4 ms per host expert per layer at 6 rows (about 3.5 to 4.9 GB/s over PCIe) | single-card checks |
| Split overhead with no host route (graph) | +63.5 us per layer at 1 row, +101.4 us at 6 rows | single-card checks |
| Other ways to serve a host expert | copy-engine DMA 743.7 us per expert (6.6 GB/s), linear to 6 experts, and host-issued copies cannot be captured in a graph; GPU gather 1,014.8 us (1 expert) to 7,788 us (6); AVX2 host compute 2,953.7 us per expert across 4 tensor-parallel slices (10 threads), activation round trip 57.1 us | single-card checks |
| Prompt staging, 512 rows, every expert staged | 65 to 93 ms against 5.2 to 5.5 ms all on the card | single-card checks |
| MXFP4 Engram rows from NVIDIA's pack | bit-equal to FP32 in FP16 on 2 x 4,098 rows; 0.91 to 1.01 M rows/s against 4.2 to 4.6 M rows/s for the IQ4_NL table (eager) | single-card checks |

ESTIMATE from the rows above and the tally (6 tokens x top-6 routes per round, independent tokens, 1.33 distinct host
experts per layer per round on average), in ms per round: all experts on the cards 36.1; zero copy 96.9; DMA 69.8 to
79.8; GPU gather 84.8 to 94.8; host compute 190 to 200. No path comes within about 10 ms per round of all experts on the
cards.

### Run-time inputs

| Input | Value | Source |
|---|---|---|
| Card power and clock | conditions A, B and one card above | Results |
| Card reserves | 4,831,838,208 bytes on the first stage and 8,053,063,680 on the second (4.5 / 7.5 GiB), keyed by the stage's first layer | [serve-nvfp4-pp2.yaml](engine/configs/serve-nvfp4-pp2.yaml) |
| Residency at load, first stage | 29.30 GiB free per card after the dense load, expert budget 24.80 GiB: 5,463 on the cards, 2,217 in host RAM, 3.81 GiB free after | [residency at load](docs/measurements.md#residency-at-load) |
| Residency at load, second stage | 28.86 GiB free, budget 21.36 GiB: 4,704 on the cards, 2,976 in host RAM, 6.89 GiB free after | residency at load |
| How experts are counted | expert slices per rank set (each of the four tensor-parallel ranks of a stage holds the same experts), 7,680 per stage | residency at load |
| Routing tally | aggregate decode routing counts per layer and expert, private; point `nvfp4_8r_tally` at your own | [engine/README.md](engine/README.md) |
| NVFP4 pack | `nvidia/DeepSeek-V4.1-Flash-1017-NVFP4` revision `480550d3`, every file checked against its Hugging Face sha256 | [harness/run/copy_weights.sh](harness/run/copy_weights.sh) |
| Host memory | 363 GiB available with no server running (MEASURED); the host-RAM experts take about 101 GB (ESTIMATE from the tier counts) | [figures from the front page](docs/measurements.md#figures-from-the-front-page-2026-11-19) |
| The experts against the official model | NVIDIA's NVFP4 expert values equal DeepSeek's official MXFP4 experts; the card says the conversion was lossless | UPSTREAM, NVIDIA's model card; the fold above |

## The served settings

[serve-nvfp4-pp2.yaml](engine/configs/serve-nvfp4-pp2.yaml), started by [run_serve.sh](engine/configs/run_serve.sh).
vLLM's API server runs with pipeline parallel size 2 and tensor parallel size 4 over the eight cards,
`max-model-len 524288`, `max-num-seqs 1`, `max-num-batched-tokens 8192`, `kv-cache-memory-bytes 268435456`,
`gpu-memory-utilization 0.96`, prefix caching and chunked prefill on, CUDA graphs for 1 to 6 rows in full mode, the GGUF
loader for the non-expert tensors, the IQ4_NL Engram table, DSpark speculative decoding with the FP8 drafter and five
draft tokens, the `deepseek_v41` tool-call parser and the `deepseek_v4` reasoning parser, thinking on by default (a
request can pass `chat_template_kwargs {"thinking": false}` or `reasoning_effort`). The routed experts come from the
loader keys:

```yaml
nvfp4_8r_packs: $LANE/packs
nvfp4_8r_tally: <your tally>.npz
nvfp4_8r_reserve_bytes: {0: 4831838208, 20: 8053063680}
nvfp4_8r_residency: two_tier
nvfp4_8r_prefill_warm: native_staged   # prompt reading stages a host expert's NVFP4 record into the card
```

The loader parses the keys once and requires all of them. The launcher sets the engine's production switch set (listed
in the script) and checks the profile's `max-num-seqs` against `NVFP4_SLOTS` (default 1). The engine can also read
MXFP4 Engram rows from NVIDIA's pack (144-byte rows, 110,595,290,400 bytes); they were built and checked but are not the
default, because the IQ4_NL table decodes 4 to 5 times more rows per second in eager mode.

## Requirements

- Eight Tesla V100-SXM2 32 GB cards on two NVLink boards of four. Tensor parallelism stays inside a board; the pipeline
  hand-off between boards goes over PCIe. On our host four PLX switches each sit on their own x16 Gen3 root port with
  two cards at Gen3 x8 (MEASURED from sysfs).
- Host RAM for the experts that do not fit on the cards, pinned: about 101 GB with our tally (ESTIMATE). Our host had
  363 GiB available with no server running.
- Volta has FP16 `mma.sync.m8n8k4` only: no FP4 or FP8 tensor cores, no `cp.async`, no `ldmatrix`. Every 4-bit format
  is decoded in software into FP16 operands, so 4-bit experts pay by reading fewer bytes, not by faster math.
  v100-skinny measured a read ceiling of about 879 GB/s on a V100-SXM2 and its QPN kernel at 77 % of it at 1 row
  (UPSTREAM). At the official precision the routed experts take about 269 GiB (ESTIMATE: 384 x 35.4M weights x 40 x
  4.25 / 8 bytes), against 256 GiB of card memory in total before attention weights, KV cache and the drafter.
- The Volta sPLIt engine (1Cat-vLLM #1277 with Volta sPLIt's patch series and production patches) in its own source tree
  and venv (Python 3.12), with this repository's two new files and three patches applied, and its compiled libraries
  (`v41k.so`, `v41idx.so`, `v41pidx.so`, the sparse prefill library) built from that engine's source.
- NVIDIA's NVFP4 pack, the ISTA-DASLab GGUF, DeepSeek's config, tokenizer and drafter, room for the 299,473,305,600
  bytes of packs, and a decode routing tally of your own.
- For the measurement scripts, a checkout of the Volta sPLIt repository: they call its harness scripts read-only.

## Running it

[RUNNING.md](RUNNING.md) gives the steps: the engine, the weights and their check, the tally and the packs, the run
directory, the profile, the start and the boot checks, the stop, and the measurement scripts.
[engine/README.md](engine/README.md) describes every engine file, patch and config the steps use.

## What did not work or is not yet good

- **Reading experts from host RAM is the cost of this build.** One host-RAM expert per token at 1 row adds about 975 us
  to a layer, and at 6 rows each costs about 1.0 to 1.4 ms per layer. Copy-engine DMA, a GPU gather and AVX2 host compute
  were measured as alternatives on one card, and none comes within about 10 ms per round of all experts on the cards
  (ESTIMATE). The short sets give 18.8 to 30.2 tok/s and decode after context 30.2 to 33.7 tok/s.
- **General text is slower.** The experts on the cards come from a private routing tally, and general text misses them
  more often: 174.64 against 87.58 ms per round on the legal-style set (run 1).
- **Prompt reading is slow.** 1,419 to 1,595 tok/s from 30K to 300K tokens: every expert that is not on the cards is
  staged from host RAM while reading (65 to 93 ms per 512-row chunk with every expert staged, against 5.2 to 5.5 ms all on
  the card). A 216K-token prompt takes about 136 s to its first token, and a 498,099-token prompt 326.6 s.
- **The first token after a short prompt takes 0.69 s.**
- **The memory margin is small.** At the final reserves the highest card memory seen was 32,086 MiB on the first stage
  and 31,356 MiB on the second against a limit of about 32,494 MiB, about 0.4 and 1.1 GiB of margin.
- **The first boots failed** on a host mapping requested on card 0 for every worker, prompt staging buffers kept per
  layer, and one reserve shared by both stages. Each was fixed at its cause.
- **One request at a time.** The profile has one prompt slot; a follow-up reads the whole prompt again when it does not
  repeat the previous sequence to within 124 tokens of its end.
- **Not established**: closeness to the unquantized model, answer quality in use, sampling above temperature 0, and the
  tool-loop test, which this build did not run.

## Related builds

Other PLI Labs builds of DeepSeek-V4.1-Flash-1017 on V100 cards: [Volta Max uPLIft](https://github.com/ProprietaryLegal/Volta-Max-uPLIft-FP4-DeepSeek-V4.1-Flash-1017-NVFP4-8xV100-SXM2-PP2xTP4) ·
[Volta 12X AmPLIfy](https://github.com/ProprietaryLegal/Volta-12X-AmPLIfy-DeepSeek-V4.1-Flash-1017-NVFP4-12xV100-SXM2-PP3xTP4) ·
[Volta sPLIt](https://github.com/ProprietaryLegal/Volta-sPLIt-DeepSeek-V4.1-Flash-1017-8xV100-SXM2-PP2xTP4). The Volta
architecture and kernel research they stand on: [volta-v100-research](https://github.com/ProprietaryLegal/volta-v100-research).

## Work with PLI Labs

[PLI Labs](https://proprietarylegal.ai/labs.html) is the open research arm of Proprietary Legal Intelligence. PLI builds
AI around an organization's own work product: retrieval over its documents, staged workflows, validation of every output
and attorney review before anything goes out. This stack is how we run that privately, on hardware we own. If you want
private, on-prem LLM serving of your own, from choosing the hardware to the measured stack and the workflows on top, we
consult on exactly that: [proprietarylegal.ai](https://proprietarylegal.ai).

## Licence

Original documentation and code are Apache-2.0 ([LICENSE](LICENSE)); see [NOTICE](NOTICE). Patches against third-party
code keep that code's licence. No model weights, packs, wheels, compiled libraries, routing tallies or prompt corpora are
distributed here.

## Credits

This build stands on other people's work: DeepSeek (the model, its official MXFP4 experts, the DSpark drafter and the
Engram design), NVIDIA (the NVFP4 pack whose experts this build serves, its MXFP4 Engram tables and its accuracy table),
1CatAI and the authors of 1Cat-vLLM's SM70 work (yangzhuxinyzx's [#1277](https://github.com/1CatAI/1Cat-vLLM/pull/1277),
Peuqui's [#742](https://github.com/1CatAI/1Cat-vLLM/pull/742) backend), Dennis Vertzyas
([v100-skinny](https://github.com/dnv2003/v100-skinny), the QPN kernels and the Volta bandwidth findings), ISTA-DASLab
(the GSQ-RCO GGUF), ggml (the GGUF and IQ formats), the vLLM project, and PLI Labs' Volta sPLIt build with every project
it credits. Every source, with what we took from it, is in [CREDITS.md](CREDITS.md). If you build on this repository,
please credit them as well.
