# Investigation checklist

## Completed

- [x] Confirmed the repository is clean at commit `51ce9c11a`.
- [x] Confirmed both cards are AMDGPU devices and recorded PCI addresses: RX 5700 XT at `0000:25:00.0`, RX 9070 XT at `0000:2f:00.0`.
- [x] Found that the device chooser Vulkan layer hides the RX 9070 XT; disabling `VK_LAYER_AEJS_DeviceChooserLayer` exposes both cards.
- [x] Confirmed the RX 9070 XT exposes `VK_KHR_cooperative_matrix`; prior feature inspection did not show this extension on the RX 5700 XT.
- [x] Confirmed Vulkan lists the RX 9070 XT as GPU0 and the RX 5700 XT as GPU1 when the device chooser layer is disabled.
- [x] Found existing Vulkan cooperative-matrix shader variants and the `mul_mm` shader family.
- [x] Traced the matrix multiply dispatch: one-token decode normally takes the matrix-vector path; wider matrix work takes `ggml_vk_mul_mat_q_f16` and can use cooperative-matrix pipelines when supported.
- [x] Read the `llama-bench` options and output formats.
- [x] Listed current model files in the local cache, including Gemma E4B Q4_K_XL (4.22 GB) and Qwen3.8-27B IQ2_XXS (7.27 GB).
- [x] Found CMake 3.22.1 in the Android SDK. It is not on `PATH`.
- [x] Confirmed there is no `tasks/plan.md` or `tasks/todo.md` to preserve.
- [x] Configured and built `llama-bench` with Vulkan enabled and CUDA/HIP disabled. Build output is in ignored `build-vulkan/` on the repo's disk-backed filesystem.
- [x] Confirmed `llama-bench --list-devices` maps `Vulkan0` to the RX 9070 XT with `KHR_coopmat` and `Vulkan1` to the RX 5700 XT with no matrix-core path.
- [x] Benchmarked the same local Qwen3.8-27B IQ2_XXS model on both GPUs, with all layers offloaded, Q4_0 KV cache, prompt 1024, decode 256, and three repetitions.
- [x] Captured AMDGPU driver telemetry at about 263 ms intervals during each longer benchmark.
- [x] Compared token rates, utilization counters, VRAM use, and an explicitly labeled weight-footprint rate estimate.
- [x] Created a GPU and memory activity plot in `build-vulkan/benchmark-results/gpu-activity.png`.

## Capacity cases

- [x] Request all 66 layers of local Qwen3.8-27B Q2_K_XL (9.82 GB GGUF file) on the RX 9070 XT; record performance and telemetry.
- [x] Request all 66 layers of Qwen3.8-27B Q2_K_XL on the RX 5700 XT and compare requested Vulkan buffer size with the device memory limit.
- [x] Run local Qwen-AgentWorld-35B-A3B Q4_K_XL (22.32 GB file) across both GPUs with a host-memory stop threshold; all 41 layers were assigned across the cards.

## Final checks

- [x] Record the selected model's exact cache path and verify the file is local.
- [x] Read the Vulkan backend's matrix shader, quantized shader, feature-selection, and dispatch code in detail.
- [x] Write findings and a narrowly scoped next experiment; do not modify source code during this pass.

## Results

- Model file: `/storage/default/llama-cpp-models/huggingface/models--unsloth--Qwen3.8-27B-GGUF/snapshots/4ca720788d1e01f1bff70c033e0d0028fd02e502/Qwen3.8-27B-UD-IQ2_XXS.gguf` (7,266,070,528 bytes).
- RX 5700 XT: 125.82 +/- 0.26 tokens/s for prompt processing and 22.48 +/- 0.05 tokens/s for decode.
- RX 9070 XT: 1,122.99 +/- 23.49 tokens/s for prompt processing and 52.29 +/- 0.14 tokens/s for decode.
- During the RX 5700 XT capture, GFX activity averaged 90.7%, Memory activity averaged 21.8%, and peak VRAM use was 6,846 MiB.
- During the RX 9070 XT capture, GFX activity averaged 84.4%, Memory activity averaged 43.9%, and peak VRAM use was 9,194 MiB.
- AMDGPU's average DRAM read/write metrics were null. The activity percentages are not measured bandwidth in GB/s.
- Estimated model-footprint read rate for decode, assuming about one pass over model weights per token: 163.1 GB/s on the RX 5700 XT and 379.4 GB/s on the RX 9070 XT. These are estimates, not bus measurements.
- The initial same-model benchmark used `-ngl -1` (automatic layer fitting). A verbose explicit full-layer check of the 7.27 GB Qwen3.8-27B IQ2_XXS model logged 65/65 layers on the RX 5700 XT and a 6,521 MiB Vulkan model buffer, below the reported 8,154 MiB free before loading.
- The explicit full-layer Q2_K_XL check logged 66/66 layers and an 8,631 MiB Vulkan model buffer on both cards. RX 9070 XT had 13,932 MiB free before loading and reached 11,302 MiB total VRAM use in sampled telemetry. The RX 5700 XT reported only 8,154 MiB free, less than the requested Vulkan buffer, yet the one-token run completed. This is evidence of accepted layer assignment and possible memory overcommit, not proof that all weights stayed in dedicated VRAM.
- In llama.cpp, `--no-host` skips a special GPU-host buffer type. It does not prevent the CPU buffer fallback or driver-managed GTT use. The Q2_K_XL check also logged a 397.85 MiB CPU-mapped model buffer.
- The local Qwen-AgentWorld-35B-A3B Q4_K_XL file is 22.32 GB, larger than either card's nominal VRAM. It loaded across both cards with a host-memory guard; see `results.md` for the run settings, throughput, and buffer sizes.
- No llama.cpp source files were changed, and no models were downloaded.

## Multi-GPU follow-up

- [x] Benchmark Qwen3.8-27B Q4_K_XL (17.55 GB) across both cards with layer split and tensor split, using a 2:1 split ratio.
- [x] Benchmark Qwen-AgentWorld-35B-A3B Q4_K_XL (22.32 GB) across both cards with layer split and a host-memory guard.
- [x] Record per-GPU GFX activity, memory activity, VRAM use, and available host memory.
- [x] Write machine-readable summary files: `build-vulkan/benchmark-results/multigpu_results.json` and `multigpu_results.csv`.

### Multi-GPU results

- Qwen3.8-27B Q4_K_XL: layer split reached 344.26 +/- 0.20 tokens/s prompt processing and 19.83 +/- 0.04 decode. Tensor split reached 218.83 +/- 0.47 prompt processing and 12.93 +/- 0.02 decode.
- In the 250 ms telemetry samples, both GPUs showed at least 10% GFX activity in 74/157 layer-split samples and 149/176 tensor-split samples. The capture includes load and warmup; this does not establish sub-millisecond overlap.
- Qwen-AgentWorld-35B-A3B Q4_K_XL: layer split reached 979.75 +/- 10.35 prompt processing and 54.73 +/- 2.55 decode. The model is an MoE with about 3B active parameters, so do not compare its decode rate directly with the dense Qwen3.8 model.
- Both cards received model buffers for every run. The 22.32 GB model logged Vulkan buffers of 14,058 MiB on the RX 9070 XT and 6,706 MiB on the RX 5700 XT, plus a 515 MiB CPU-mapped model buffer.
- The JSON and CSV contain full benchmark samples, split mode, model bytes, loader buffer sizes, VRAM peaks, host-memory minima, and telemetry counters.
