# Running Volta Full NVFP4

How to build and start the two-tier build on eight V100-SXM2 32 GB cards, check that it booted, stop it, and measure
it. Every command here is the one the repository's scripts and READMEs document; nothing in this file was run for it.
The figures the build gives are in the [README](README.md#results); [engine/README.md](engine/README.md) describes every
engine file, patch and config.

There is no prebuilt package: you build the packs from NVIDIA's files and choose the experts kept on the cards from a
routing tally of your own.

Placeholders: `<build>` is this build's root on your host (the `LANE` of the scripts), `<volta-split>` a checkout of the
Volta sPLIt repository (the `PROD` of the scripts), `<engine>` the engine source tree. Use absolute paths.

## 1. The engine

1. Build the Volta sPLIt engine (1Cat-vLLM [#1277](https://github.com/1CatAI/1Cat-vLLM/pull/1277) with Volta sPLIt's
   patch series and production patches) as its
   [Reproduce section](https://github.com/ProprietaryLegal/Volta-sPLIt-DeepSeek-V4.1-Flash-1017-8xV100-SXM2-PP2xTP4#reproduce)
   describes, in its own venv (Python 3.12), with its compiled libraries (`v41k.so`, `v41idx.so`, `v41pidx.so`, the
   sparse prefill library). The launcher stops if one is missing.
2. Apply this repository's three patches in the engine source root. Each is a unified diff against the Volta sPLIt
   engine file as deployed on 2026-11-09, and its header gives the sha256 of that base file and of the result.

   ```bash
   cd <engine>
   for p in <this-repo>/engine/patches/*.patch; do patch -p1 < "$p"; done
   ```

3. Copy the two new files of [engine/src/](engine/src/) (`nvfp4_8r_native.py`, `nvfp4_8r_tiers.py`) into the same paths
   of the engine's `vllm` tree.

## 2. The cards and the weights

Eight V100-SXM2 32 GB cards on two NVLink boards of four. Tensor parallelism stays inside a board; the hand-off between
the two pipeline stages goes over PCIe. The host needs pinned RAM for the experts that do not fit on the cards (about
101 GB with our tally, ESTIMATE).

Download into `<build>/models/`:

- NVIDIA's pack, `nvidia/DeepSeek-V4.1-Flash-1017-NVFP4` at revision `480550d3`;
- the ISTA-DASLab GSQ-RCO GGUF (the profile reads its first shard and the IQ4_NL Engram file);
- DeepSeek's config and tokenizer, and the DSpark drafter.

Copy and check NVIDIA's pack. [harness/run/copy_weights.sh](harness/run/copy_weights.sh) copies it from a local
download, checks every file's sha256 against the Hugging Face record, stops at the first failure and prints
`ALL-VERIFIED`:

```bash
export LANE=<build>
mkdir -p "$LANE/logs"
WEIGHTS_SRC=<local download of the NVIDIA pack> bash harness/run/copy_weights.sh | tee "$LANE/logs/copy_weights.log"
```

It writes the pack to `$LANE/models/DeepSeek-V4.1-Flash-1017-NVFP4`. The pack readers of the next step take
`logs/copy_weights.log` as their verification log and refuse to touch a checkpoint shard without `ALL-VERIFIED` in it.

## 3. The tally and the packs

Provide a decode routing tally: routing counts per layer and expert from your own traffic, in the array layout that
`choose_residency` reads (see `nvfp4_8r_native.py`). Ours is private and not published.

Build and check the packs on CPU with [run_p1.sh](harness/benchmarks/nvfp4_8r/run_p1.sh), which runs three modules in
order from `$LANE` with the engine's Python through `uv run`, and no card (`CUDA_VISIBLE_DEVICES=''`):

| Step | Module | What it does |
|---|---|---|
| 1 | `probe_p1` | reads the NVFP4 checkpoint after the verification log |
| 2 | `build_packs` | folds every routed expert to MXFP4 packs per layer and tensor-parallel rank, `--workers 8` |
| 3 | `accept_p1` | checks the packs value for value against the NVFP4 formula in FP32 and writes `logs/p1-acceptance.json` |

The script reads the tally from `engine/private/p16-tally.npz` under `$LANE`; put yours there or change the path in it.
Its exit code goes to `logs/p1-job.log`. The packs take 299,473,305,600 bytes.

## 4. The run directory

The scripts were measured in one layout and find each other there, not in this repository's folders. `LANE` is this
build's root (with `engine/venv/`, `engine/src/`, `models/` and `packs/`), `PROD` the Volta sPLIt checkout (its harness
scripts). From this repository's root:

```bash
export LANE=<build> PROD=<volta-split>
mkdir -p "$LANE/run/state" "$LANE/run/cache" "$LANE/logs/baseline" "$LANE/engine/src/benchmarks/nvfp4_8r"
cp engine/configs/* harness/run/* "$LANE/run/"
cp harness/benchmarks/nvfp4_8r/* "$LANE/engine/src/benchmarks/nvfp4_8r/"
```

Copy Volta sPLIt's run tools into `$LANE/run/tools/` (`env.sh` puts them on the Python path). When `LANE` or `PROD` is
not set, the shell scripts stop with a message and the Python files stop with a `KeyError` naming the variable. Set
both; do not edit the scripts.

## 5. The profile

[serve-nvfp4-pp2.yaml](engine/configs/serve-nvfp4-pp2.yaml) is the profile. Before the first start:

- replace `$LANE` in it with your path (YAML does not expand it);
- point `nvfp4_8r_tally` at your tally;
- set `host` and `port` for your server; the profile listens on loopback.

The routed-expert keys are required and have no default:

```yaml
nvfp4_8r_packs: <build>/packs
nvfp4_8r_tally: <your tally>.npz
nvfp4_8r_reserve_bytes:        # per card, keyed by the stage's first layer
  0: 4831838208                # 4.5 GiB
  20: 8053063680               # 7.5 GiB: the second stage holds the drafter and the head
nvfp4_8r_residency: two_tier
nvfp4_8r_prefill_warm: native_staged
```

The reserves were set from peaks measured on our cards (about 4.0 and 6.9 GiB) plus about 0.5 GiB. One reserve for both
stages ran out of memory on a 100K-token prompt; if your peaks differ, sample card memory with `memwatch.sh` during
100K- and 500K-token prompts and set the two values from them.

## 6. Start and check

[run_serve.sh](engine/configs/run_serve.sh) checks first, before any CUDA call: it refuses if a server already listens
on the ports it checks or if any card has a compute process, and says INCONCLUSIVE if `nvidia-smi` fails. To run only
that check:

```bash
cd "$LANE/run" && bash run_serve.sh --precheck-only
```

Start:

```bash
cd "$LANE/run"
PROFILE=serve-nvfp4-pp2.yaml CUDA_VISIBLE_DEVICES=<first board's four indices>,<second board's four> bash run_serve.sh nvfp4
```

The last word names the server's log and pid files, `$LANE/run/state/nvfp4.log` and `.pid`. `env.sh` sets the card
order by PCI bus id and checks that vLLM imports from the build's venv with its compiled ops. The launcher sets the
engine's production switch set, checks the profile's `max-num-seqs` against `NVFP4_SLOTS` (default 1, which this
profile uses), and starts the server in its own process group.

Check the boot log. After the dense weights load, each stage logs its free card memory, its expert budget and how many
experts it keeps on the cards and in host RAM. Ours, with our tally:

| Stage | Free per card after dense load | Expert budget | On the cards | Host RAM | Free after |
|---|---:|---:|---:|---:|---:|
| First (layers 0 to 19) | 29.30 GiB | 24.80 GiB | 5,463 | 2,217 | 3.81 GiB |
| Second (layers 20 to 39) | 28.86 GiB | 21.36 GiB | 4,704 | 2,976 | 6.89 GiB |

The KV pool is 2,076,870 tokens at a `max-model-len` of 524,288. Our boot was ready in 288 s. With another tally the
counts on each stage are yours, and the speeds of the README are not established for them.

`VLLM_NVFP4_8R_LOAD_TRACE=1` logs card and allocator memory during the stage build, counts and bytes only. It was added
after the measurements of 2026-11-09 and is off by default.

## 7. Stop

```bash
kill -TERM -- -$(cat "$LANE/run/state/nvfp4.pid")
```

The pid file holds the server's process group, so this stops every worker.

## 8. Measure

The scripts send one request at a time, use public text only, record numbers only (no prompt or generated text) and stop
at the first failure. They call the Volta sPLIt harness scripts (`warmup.py`, `ab_t1_v2.py`) from `$PROD`; get them from
the Volta sPLIt repository's `harness/`. [harness/README.md](harness/README.md) lists every script.

| Script | What it measures |
|---|---|
| `measure_boot.sh URL LABEL [LONG]` | warm-up and the general set, the legal-style set, optional long prompts (`LONG=1`: 100K then 250K; `LONG=2`: 100K then 500K, for a build that changes card memory use), then both sets again |
| `tput_prompts.py URL PROMPTS.txt OUT.jsonl` | the same measurement over a prompt file, one prompt per line |
| `ctx_tput.py URL CHARS OUT [MAX_TOKENS]` | the context rows: decode speed and read time after CHARS characters of public text (512 tokens out by default) |
| `longprompt.py URL CHARS [MAX_TOKENS [SKIP]]` | one long public prompt (Python standard-library source) with a short answer, for prompt reading and card memory |
| `harness_shape.py URL CHARS N OUT.jsonl [THINK] [MAX_TOKENS]` | one chat shaped like an agent conversation: a cold read, then follow-ups that add the answer and about 8,000 characters of new public text |
| `memwatch.sh OUT SECONDS` | card memory every 15 s, index and MiB only |

For example:

```bash
cd "$LANE/run"
bash measure_boot.sh http://127.0.0.1:<port> my-boot 1
```

The legal-style prompt file is not published. Any file of one public prompt per line works, saved as
`$LANE/logs/baseline/legal-public-prompts.txt`. The prompt wording of `ctx_tput.py` and `harness_shape.py` was
generalized for publication, so their token counts differ by a few tokens from the measured ones.

## 9. Checks without the eight cards

The modules of `harness/benchmarks/nvfp4_8r/` run from the engine source tree as `python -m benchmarks.nvfp4_8r.NAME`.
The CPU tests (`test_p3_loader_wiring`, `test_p3_split_groups`, `test_t3_dispatch`, `test_t3_iq_bank`, `test_t3_quality`,
`test_t3_stage_plan`, `test_t3_tiers`) need no card; `test_residency` needs a routing tally. The single-card gates
(`accept_p2`, `accept_p3_layer`, `accept_p4`, `accept_t3_layer`, `accept_t3_stage`, `falsify_t3_iq_drop`,
`falsify_f_iq_prefill`, with their `run_*_testhost.sh` runners) need one card on a test host. Each checks the card first
(power cap, no other compute process, enough free host memory) and exits INCONCLUSIVE rather than running under the wrong
conditions; [harness/README.md](harness/README.md) gives the test-host layout they expect.

Before a commit, the repository's data-boundary scan refuses weights, archives, corpora, private directories, files over
1 MiB and credential patterns:

```bash
bash tools/check_data_boundary.sh --staged      # the index
bash tools/check_data_boundary.sh --tree HEAD   # every file in a commit
```
