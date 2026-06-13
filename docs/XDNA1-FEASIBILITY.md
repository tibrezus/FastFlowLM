# XDNA 1 (Phoenix / Hawk Point) Feasibility — Technical Analysis

**Status:** 🔴 Full LLM support **not implementable** from this codebase on XDNA 1.
**Author:** tibrezus (fork maintainer) — investigation done on a Ryzen 7 7840HS (Phoenix) running Arch Linux, kernel 7.0.11, `amdxdna` driver, NPU firmware `1.5.5.391`.

This document records *why* FastFlowLM is XDNA-2-only and exactly what "implementing
XDNA 1 compatibility" would and would not entail. It exists so that other Ryzen-AI-7000/8000
(XDNA 1) owners stop chasing a dead end, and so future contributors know where the real
boundary is.

---

## TL;DR

| Question | Answer |
|---|---|
| Is `flm` XDNA-2-only by choice or by necessity? | **Necessity.** The overlays are silicon-specific. |
| Can I just remove the `cols < 8` check and run on my 7840HS? | **No.** XRT will reject the XDNA2 overlay; wrong tile geometry + wrong AIE ISA. |
| Can the `.xclbin` kernels be recompiled for Phoenix here? | **No.** No kernel source and no AIE compiler flow exist in this repo. |
| If I had AMD's proprietary Vitis AIE compiler, would LLMs work on Phoenix? | **Marginal.** Phoenix is 5 columns / ~10 TOPS; the LLM kernels are sized for 8 columns. Throughput would be poor. |
| Is there a realistic local-LLM path on a 7840HS today? | **Yes — the iGPU** (Radeon 780M via ROCm/HIP), not the NPU. |

---

## 1. The hardware gap (why XDNA 1 ≠ XDNA 2)

The XDNA NPU is a **spatial dataflow accelerator**, not a programmable GPU. It cannot
run arbitrary code — every op must be *offline compiled, place-and-routed onto the AIE
tile array*, and shipped as an `.xclbin` overlay bitstream. The chip just streams data
through whatever overlay is loaded.

| | XDNA 1 — Phoenix / Hawk Point | XDNA 2 — Strix Point / Kraken / Strix Halo |
|---|---|---|
| CPU example | **Ryzen 7 7840HS**, 7940HS, 8040-series | Ryzen AI 300 / 400 series |
| AIE tile generation | **AIE-ML (AIE2)** | **AIE2P** |
| Columns | **5** | **8** |
| Peak | **~10 TOPS** | **~50 TOPS** |
| PCI device ID (NPU) | `1022:1502` | `1022:17F0` |

FastFlowLM's overlays are compiled for the 8-column AIE2P array on Strix. They are
**not portable** to the 5-column AIE2 array on Phoenix.

## 2. The software gap (the four-layer proprietary stack)

To run *any* model on this NPU you need a four-layer stack between the model and silicon:

1. **Operator compiler** (VAIP / Vitis AI) — ONNX/PyTorch subgraph → AIE instructions
2. **Precompiled overlays** (`.xclbin`) — the actual bitstreams the NPU loads
3. **Execution Provider** (`libonnxruntime_providers_ryzenai`) — ONNX-RT plugin routing to NPU
4. **Tuned, quantized model zoo** — models pre-sliced to fit the partition shape

**This whole stack is the secret sauce, and AMD built it Windows-first.** What is open on
Linux today is only the *bottom* layer — the `amdxdna` kernel driver (the dumb pipe). The
overlays, the compiler, and the EP above it are what AMD gates.

## 3. Evidence from this repository

The fork confirms the boundary concretely:

- **227 `.xclbin` binaries are committed**, every one `*-NPU2`:
  ```
  src/xclbins/Llama-3.2-1B-NPU2/{attn,dequant,layer,mm}.xclbin   # and 40+ more model dirs
  ```
- **No kernel source and no compiler flow.** There is no Vitis, no `aiecompiler`,
  no `v++`, no `xclbinutil`, no HLS/AIE-C++ anywhere in the tree. The `find` for kernel
  source returns only host build files (`Makefile`/`CMakeLists.txt`).
- **Overlays are loaded as opaque binaries** via XRT:
  ```cpp
  // src/include/npu_utils/npu_utils.hpp:16
  npu_app_manager* mvm_i8_xclbin = npu_mgr.register_xclbin("mvm_i8.xclbin");
  // src/test/whisper_npu/test.cpp:49
  xrt::device npu_device_global = xrt::device(0);
  ```
- **The column check is a hard gate, but a symptom not the cause:**
  ```cpp
  // src/src/main.cpp:328
  if (query_aie_metadata.cols < 8) {
      header_print_r("ERROR", "NPU ... does not have enough columns (... < 8)");
  ```
  Removing it changes nothing — the subsequent `register_xclbin()` would fail against the
  XDNA2 overlay on the 5-column device.

## 4. What "implementing XDNA 1 support" would actually require

There is no small patch. A real implementation needs:

1. **A Phoenix AIE2 compiler.** AMD's Vitis AIE / VoE flow, which is **not open-source and
   not redistributable** as part of an OSS project. (It ships inside the AMD Ryzen AI
   Software installer, AMD-account-gated.)
2. **Re-authoring every kernel** (attn / mm / dequant / layer) for the 5-column geometry —
   re-tile, re-place-and-route, re-validate numerics. Not a recompile; a re-design.
3. **A legally-shippable PHX `.xclbin`** to embed. None exists in the OSS ecosystem today.
   The only PHX overlays live in AMD's proprietary package and are Windows-oriented
   (`voe-4.0-win_amd64/1x4.xclbin`, referenced by `dahara1/llama3-8b-amd-npu`, which is
   itself Windows-only).
4. **Acceptance that throughput is poor.** 5 columns / ~10 TOPS vs the 8-column target the
   kernels were written for. For 8B-class models this is tile- and bandwidth-starved.

Even step 1 alone is a closed door for an open fork. This is why **no public tool runs
LLMs on the Phoenix NPU on Linux** — not for lack of trying, but because the closed overlay
compiler + the small silicon together make it not worthwhile.

## 5. What *is* feasible — and where this fork goes

Since an LLM runtime for XDNA 1 cannot be honestly built from FastFlowLM, this fork:

- **Pins the analysis above** so it is discoverable.
- **Will not ship a fake "XDNA1 support" patch** that just removes the column guard — that
  would build cleanly and then fail on the device, which is worse than a clear error.
- **Tracks** any future change in the closed-toolchain situation (e.g. AMD open-sourcing an
  AIE compiler, or publishing a redistributable PHX overlay).

### The realistic local-LLM path on a 7840HS

For local LLMs on this exact laptop under Linux, the viable route — confirmed by other
7840HS owners — is the **integrated GPU (Radeon 780M) via ROCm/HIP**, not the NPU:

```bash
export HSA_OVERRIDE_GFX_VERSION=11.0.0
export PYTORCH_ROCM_ARCH="gfx1100"
# llama.cpp built with -DLLAMA_HIPBLAS=ON (and -DLLAMA_HIP_UMA=ON for shared memory)
```

The iGPU shares system memory (UMA), so with a large BIOS UMA allocation (or
`Smokeless_UMAF`) you can fit multi-GB quantized models and run real tokens/s. The NPU
remains usable for what *does* have an overlay path — classic ONNX vision/NLP once a PHX
overlay is available — but not for FastFlowLM-style LLMs.

## 6. See also

- `tibrezus/xdna-npu-toolkit` — companion from-scratch toolkit that **detects, validates,
  and enables** the XDNA 1 NPU stack on Linux (kernel + firmware + XRT + memlock + power
  mode) and prints this same feasibility verdict for your specific machine.
- AMD Ryzen AI Software 1.7.1 release notes (lists Phoenix as supported for the *CNN/transformer
  ONNX* path via the proprietary VitisAI EP — the one path that legally touches Phoenix).
- `dkuku.github.io` — "Machine Learning on Ryzen 7840HS" (the iGPU ROCm fallback in practice).
