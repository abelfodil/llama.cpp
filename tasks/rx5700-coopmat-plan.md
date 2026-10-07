# RX 5700 XT Vulkan matrix support plan

## Objective

Determine whether the RX 5700 XT can run a useful Vulkan cooperative-matrix path, then implement the smallest path that is correct and improves llama.cpp workloads. Keep the normal Vulkan path available as a fallback.

Treat these as separate outcomes:

1. **API support:** RADV reports and correctly implements `VK_KHR_cooperative_matrix` for a defined set of operations and shapes.
2. **llama.cpp support:** the Vulkan backend selects a suitable kernel and returns correct results.
3. **Performance value:** repeatable llama.cpp workloads improve against the stock RX 5700 XT path.

Passing one outcome does not prove the next.

**Recommended order:** If the goal is faster llama.cpp inference, test a normal Vulkan compute kernel first. RADV extension work is justified only if the Vulkan API itself is also needed or if its compiler lowering can plausibly beat that kernel. A driver extension cannot add a native matrix unit to GFX10.

## Current evidence

- The RX 5700 XT is Navi10 / GFX10.1. The local Vulkan query did not report `VK_KHR_cooperative_matrix` for it. The checked Mesa feature list reports RADV support for GFX11 and newer.
- There is no runtime switch that can add the missing Vulkan extension. Driver support is required for the Vulkan cooperative-matrix API.
- llama.cpp has cooperative-matrix shader variants, but the current integer path also checks the reported device capability and limits its architecture path to RDNA3/RDNA4. Driver exposure alone will not enable the RX 5700 XT path.
- The RDNA3 WMMA guide does not establish native WMMA support on GFX10. A GFX10 implementation may need ordinary shader instructions. Its speed must be measured.
- A prior RX 9070 XT Q4_K experiment dispatched the cooperative-matrix kernel but showed no repeatable overall gain over its same-session control. That result is not a prediction for the RX 5700 XT.
- The current benchmark report shows low measured VRAM-busy values in the tested Q4_K_XL cases, including the RX 5700 XT. The new work must identify which prompt and decode kernels run and must not assume that cooperative matrices will fix a bandwidth limit.
- The current llama.cpp cooperative-matrix path is for matrix-matrix work. One-token decode normally uses a matrix-vector path, so prompt and decode results must be reported separately.

## Constraints

- Do not advertise an extension or property combination that the driver does not implement correctly.
- Do not remove the stock Vulkan fallback.
- Keep early experiments local and isolated. Build Mesa and llama.cpp under the disk-backed repository `build-vulkan/` directory, not `/tmp`, and do not replace the system Mesa installation.
- Use the existing local models. Do not download models for these comparisons.
- Keep driver changes and llama.cpp changes in separate patches or worktrees.
- Before implementation, read `skills/code-review/SKILL.md`, review current Mesa and llama.cpp issues/MRs for duplicate work, and confirm the contributor understands the selected design. Do not submit code or open a PR automatically.

## Plan

### Stage 1: Verify the capability and code paths

1. Record the GPU PCI IDs, ASIC generation, kernel driver, Mesa/RADV build identity, Vulkan loader, and device extension list for both cards.
2. Query cooperative-matrix properties only on devices that report the extension. Save the supported scopes, dimensions, component types, and saturation behavior.
3. Read the Vulkan extension requirements and RADV's current cooperative-matrix implementation and lowering path. Check what GFX10 instructions and NIR operations can implement the required signed INT8 x signed INT8 -> INT32 operation.
4. Trace the llama.cpp CM1 kernel from shader generation through pipeline registration, capability checks, dispatch, and output layout. Confirm the actual path with a debug dispatch trace on each workload.
5. Check current Mesa issues and merge requests for GFX10 cooperative-matrix work and current llama.cpp issues for RDNA1 Vulkan matmul work.

**Gate:** Continue with a RADV prototype only if the required Vulkan operation can be represented without falsely advertising unsupported properties and the driver change has a maintainable scope. Otherwise, test a llama.cpp shader that uses ordinary Vulkan compute operations.

### Stage 2: Measure a shader-only GFX10 baseline

1. Make a small local Vulkan compute benchmark for the exact 16x16x16 signed INT8 x signed INT8 -> signed INT32 operation used by llama.cpp CM1. The shader must use ordinary Vulkan compute operations and must not require `VK_KHR_cooperative_matrix`.
2. Compare its output to a CPU reference for aligned and edge tiles, multiple accumulations, and overflow-sensitive inputs.
3. Measure the shader alone and the current llama.cpp fallback with fixed inputs. Record dispatch time, data size, device clocks, and repeated samples.
4. Inspect generated GFX10 ISA to see whether the compiler uses useful packed or dot-product operations, or expands the work into slow scalar instructions.

**Gate:** If ordinary shader lowering is already too slow or has poor register use, do not spend time integrating it into llama.cpp. The RADV path can still proceed only if its compiler lowering can produce a materially better result.

### Stage 3: Prototype RADV support, only if Stage 1 passes

1. Use an isolated Mesa checkout and build under `build-vulkan/mesa-radv-gfx10/`, with its install prefix under `build-vulkan/mesa-radv-gfx10-prefix/`.
2. Implement a local proof of concept for the smallest operation set llama.cpp needs. Reuse existing RADV/NIR cooperative-matrix code where possible; add a GFX10 lowering only where the existing path cannot handle the operation.
3. Keep unsupported shapes, types, or scopes out of the reported property list. Do not add a user setting that merely claims support.
4. Select the isolated driver for tests without changing the system installation. Capture build logs, compiler output, and generated ISA.
5. Run the relevant Vulkan CTS cooperative-matrix cases and a small CPU-reference Vulkan test. Fix all failures for every property combination the driver reports.

**Gate:** Stop the RADV path if correct property reporting needs a broad driver rewrite, CTS exposes unsupported behavior that cannot be fixed locally, or generated code is not competitive with the shader-only baseline. Keep the result as a local experiment and document the reason.

### Stage 4: Integrate with llama.cpp

1. Add a capability-based selection path for the tested operation and shape. Remove or extend the RDNA3/RDNA4 architecture restriction only after the driver prototype passes its checks.
2. Add a GFX10-specific operand layout only if the current RDNA3 layout is not valid on GFX10. Do not reuse the RDNA3 layout by assumption.
3. Keep a runtime fallback to the existing Vulkan kernel for unsupported types, shapes, dimensions, or devices.
4. Compare outputs against the fallback for each supported quantization type, tile boundary, batch size, and prompt shape. Check for Vulkan validation errors and device faults.
5. Confirm through dispatch logging that the intended cooperative-matrix pipeline runs for prompt processing. Confirm separately whether decode uses that pipeline.

**Gate:** Do not benchmark performance until correctness and dispatch selection are confirmed.

### Stage 5: Benchmark and decide whether to keep the path

1. Rebuild the same llama.cpp revision with the stock RADV and experimental RADV builds. Use the same Vulkan loader, model, settings, and GPU selection.
2. Start with a model and quantization that fit in RX 5700 XT VRAM. Select a local model whose prompt path is confirmed to dispatch the tested kernel. Keep the Qwen3.8-27B Q4_K_XL case as a separate spill/capacity workload, not the primary kernel comparison.
3. Benchmark prompt processing and one-token decode separately. Use at least five measured repetitions and alternate control/experimental order to reduce clock and temperature bias.
4. Record prompt tok/s, decode tok/s, overall tok/s using `(prompt_tokens + decode_tokens) / (prompt_tokens / prompt_tok_s + decode_tokens / decode_tok_s)`, GPU busy, VRAM busy, GTT use, host memory, clocks, temperature, and exact driver/kernel IDs. Label activity percentages separately from byte-rate measurements.
5. Repeat the 9070 XT control only as a regression check if the shared llama.cpp change could affect its path. Keep the dual-GPU case separate because it includes split and synchronization costs.

**Keep criteria:** CTS and numerical checks pass; dispatch uses the intended path; the RX 5700 XT shows a repeatable improvement in the target workload without a material regression in decode or fallback cases. If API support works but performance does not improve, document it as functional support with no demonstrated llama.cpp performance benefit. If neither path helps, keep no production code change.

## Expected work areas

- Mesa/RADV: cooperative-matrix capability reporting, NIR lowering, GFX10 code generation, and Vulkan CTS coverage.
- llama.cpp: `ggml/src/ggml-vulkan/ggml-vulkan.cpp`, `ggml/src/ggml-vulkan/vulkan-shaders/mul_mmq_cm1.comp`, generated shader registration in `ggml/src/ggml-vulkan/vulkan-shaders/vulkan-shaders-gen.cpp`, and targeted Vulkan tests or benchmark tooling.
- Results: append comparable runs and limitations to `results.md`; keep build and raw telemetry files under ignored `build-vulkan/benchmark-results/`.

## Main risks

- GFX10 may not have instructions that make the required matrix operation efficient. A correct driver lowering can still be slower than the current path.
- The current llama.cpp architecture gate and operand layout may encode assumptions beyond extension presence.
- Prompt processing can use matrix-matrix kernels while decode uses another path, so an improvement may apply only to prompt processing.
- GTT use, model placement, clocks, and multi-GPU synchronization can hide or reverse a kernel-level gain.
- Mesa driver work is a separate project and may need maintainer review before a broader implementation is justified.

## References

- [Mesa Vulkan feature list at the checked revision](https://cgit.freedesktop.org/mesa/mesa/tree/docs/features.txt?id=01d17481308a74c347b68136bb25d9a8f6624ab6)
- [AMD GPUOpen: WMMA on RDNA 3](https://gpuopen.com/learn/wmma_on_rdna3/)
- [Vulkan `VK_KHR_cooperative_matrix` reference](https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_cooperative_matrix.html)
- [Khronos Vulkan CTS repository](https://github.com/KhronosGroup/VK-GL-CTS)
- Local benchmark and dispatch evidence: `results.md`
