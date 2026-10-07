# Vulkan GPU memory and matrix path investigation

## Objective

Understand the Vulkan compute path in llama.cpp and measure its behavior on the RX 5700 XT and RX 9070 XT. Determine what the existing kernels already do, whether runtime measurements support a memory bandwidth limit, and what small experiment would test the proposed systolic-like staging idea.

Follow-up: measure both Vulkan GPUs together using larger local models and compare layer and tensor splitting. Save normalized JSON and CSV records.

## Constraints

- Do not use ROCm or HIP for this investigation.
- Use only model files already present under `/storage/default/llama-cpp-models`; do not download models.
- Benchmark each GPU separately before testing multi-GPU behavior.
- Do not change llama.cpp code during the investigation.
- Request elevated access for GPU and telemetry commands when the sandbox blocks them.
- Record device selection, model file, benchmark settings, telemetry interval, and results so the measurements can be repeated.
- Store build output in the repository's ignored, disk-backed `build-vulkan/` directory; do not build in `/tmp`.

## Steps

1. Confirm the working tree, GPU enumeration, Vulkan capabilities, installed build tools, and local model inventory.
2. Trace the Vulkan backend's matrix multiplication and quantized matrix multiplication shaders, their feature selection, and the dispatch path used by `llama-bench`.
3. Build `llama-bench` with Vulkan only if no suitable executable exists. Do not enable CUDA or ROCm.
4. Run a short device-list check, then benchmark the same local model on each GPU separately. Include decode and prompt processing.
5. Request all model layers on selected devices for capacity checks. Record model buffer sizes and driver VRAM/GTT counters. Do not treat `--no-host` as a strict ban on CPU or driver-managed memory use.
6. Sample AMDGPU driver telemetry during each benchmark. Compare activity and timing with an estimated weight-footprint read rate; label estimates separately from measured bandwidth.
7. Summarize which existing Vulkan features match the proposed pipeline, what is missing, what the measurements show, and the smallest useful follow-up experiment.

## Completion criteria

- The relevant Vulkan kernel and dispatch paths are described with file references.
- The build configuration and selected GPU are recorded.
- Results include benchmark settings, separate per-GPU telemetry observations, and capacity cases with limits stated.
- No model download, ROCm/HIP use, or source-code change occurred.
- The report states limits in the measurements and recommends a concrete next step.

## Multi-GPU follow-up

1. Use the local Qwen3.8-27B Q4_K_XL model, which exceeds one card's nominal VRAM, and request all layers across both devices.
2. Compare `layer` and `tensor` split modes with a 2:1 tensor split for the 9070 XT and 5700 XT.
3. Run a second larger local model across both devices, while monitoring available host memory.
4. Capture per-GPU activity and VRAM, then write normalized JSON and CSV results with units and settings.

All four steps are complete. The detailed records are in `build-vulkan/benchmark-results/multigpu_results.json` and `multigpu_results.csv`.
