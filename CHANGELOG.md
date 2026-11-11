# Changelog

Dates are UTC. Every number is MEASURED on the 8-card host unless labelled otherwise. Details in
[docs/measurements.md](docs/measurements.md).

## 2026-11-11: multiple-choice accuracy as the closeness reference

- **Measured as the reference of Volta Max uPLIft's closeness test** (MEASURED 2026-11-11, this repository's profile,
  1,475 MHz): 83.8 % (95 % interval 80.7 % to 86.7 %) on 513 MMLU questions, greedy, thinking off, with 38 questions
  without an answer counted as wrong. A repeat in the same boot on 50 contexts and 50 questions was identical on all
  of them. The test, its rules, receipts and scores are in [Volta Max uPLIft](https://github.com/ProprietaryLegal/Volta-Max-uPLIft-FP4-DeepSeek-V4.1-Flash-1017-NVFP4-8xV100-SXM2-PP2xTP4#how-close-to-nvfp4).
- **Related builds** names Volta Max uPLIft's commit `c672630`, the first with the `hot_iq` launcher and profile.

## 2026-11-10: first release as its own repository

- **What this is.** The two-tier mode of Volta Max uPLIft's release at commit `79dc164` (2026-11-10), published on
  its own as Volta Full NVFP4: every routed expert at exactly NVIDIA's NVFP4 precision, the most-used on the cards
  and the rest served from host RAM over PCIe.
- **Carried over.** The engine source and patches and the harness as released at `79dc164`, the two-tier profile and
  its launcher, the measurements of 2026-11-09 that cover this build, the fold recipe and its bit-equality proof.
- **Left out.** The three-tier and no-host-tier profiles, Volta Max uPLIft's plans and research notes, and every
  measurement of those profiles. They stay in Volta Max uPLIft.
- **Not measured.** Nothing was measured for this release. The history of the shared code before this date is in
  Volta Max uPLIft's
  [CHANGELOG](https://github.com/ProprietaryLegal/Volta-Max-uPLIft-FP4-DeepSeek-V4.1-Flash-1017-NVFP4-8xV100-SXM2-PP2xTP4/blob/main/CHANGELOG.md).
