# RX 5700 XT cooperative-matrix checklist

## Stage 1: capability and code-path check

- [ ] Record GPU, kernel driver, Mesa/RADV, loader, and Vulkan device identities.
- [ ] Query valid cooperative-matrix properties for each supporting GPU.
- [ ] Read the Vulkan extension requirements and RADV/NIR lowering code.
- [ ] Verify GFX10 instruction and compiler support for the required INT8 x INT8 -> INT32 operation.
- [ ] Trace the llama.cpp capability gate, shader layout, and dispatch for the selected model workload.
- [ ] Check current Mesa issues/MRs and llama.cpp issues for duplicate work.
- [ ] Decide whether a RADV prototype has a narrow, supportable scope.

## Stage 2: shader-only baseline

- [ ] Create an ordinary-compute Vulkan microbenchmark for the CM1 operation.
- [ ] Compare results with a CPU reference across full and edge tiles.
- [ ] Measure repeated dispatch times and inspect generated GFX10 ISA.
- [ ] Compare against the current llama.cpp fallback on fixed inputs.
- [ ] Stop if the shader-only path has no credible performance case.

## Stage 3: isolated RADV proof of concept

- [ ] Build a local Mesa checkout under `build-vulkan/mesa-radv-gfx10/`.
- [ ] Implement only the needed cooperative-matrix operation and property combinations.
- [ ] Keep unsupported properties out of device queries.
- [ ] Use an isolated install prefix; do not replace system Mesa.
- [ ] Run Vulkan CTS cooperative-matrix cases and CPU-reference tests on RX 5700 XT.
- [ ] Compare generated code and timing with the shader-only baseline.
- [ ] Stop if correctness, scope, or performance gates fail.

## Stage 4: llama.cpp integration

- [ ] Add capability-based selection for the verified GFX10 operation.
- [ ] Add a GFX10 operand layout only if tests require it.
- [ ] Preserve fallback behavior for unsupported cases.
- [ ] Check output correctness across quantization types, tile edges, and prompt sizes.
- [ ] Verify pipeline dispatch and whether decode uses the path.

## Stage 5: benchmark and report

- [ ] Select a local model that fits in RX 5700 XT VRAM and dispatches the target prompt kernel.
- [ ] Benchmark stock and experimental paths with matched settings and at least five repetitions.
- [ ] Record prompt, decode, overall tokens/s, GPU/VRAM busy, GTT, host memory, clocks, temperature, and exact software versions.
- [ ] Keep Qwen3.8-27B Q4_K_XL and dual-GPU results as separate capacity/split cases.
- [ ] Update `results.md` with raw artifact links, results, limitations, and a keep/reject decision.
- [ ] Keep or reject the production path using the criteria in `rx5700-coopmat-plan.md`.
