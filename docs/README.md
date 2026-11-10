# Documents

| File | What it is |
|---|---|
| [measurements.md](measurements.md) | every published number of this build, with how it was measured |

The plans and research notes of the build work are published with
[Volta Max uPLIft](https://github.com/ProprietaryLegal/Volta-Max-uPLIft-FP4-DeepSeek-V4.1-Flash-1017-NVFP4-8xV100-SXM2-PP2xTP4)
in its `docs/` folder. Read the original sources they cite instead of private copies:

- [dnv2003/v100-skinny](https://github.com/dnv2003/v100-skinny): the QPN kernels, their prepack and the Volta
  bandwidth results.
- [1CatAI/1Cat-vLLM](https://github.com/1CatAI/1Cat-vLLM): the SM70 kernels and pull requests (#742, #1277 and the
  others cited).
- [Volta sPLIt](https://github.com/ProprietaryLegal/Volta-sPLIt-DeepSeek-V4.1-Flash-1017-8xV100-SXM2-PP2xTP4): the
  production patches (the `v41k`, index and prefill kernels), the switch set and the harness.
- NVIDIA's Volta architecture whitepaper and the CUDA PTX ISA, for the sm_70 instruction limits.
