# Vulkan benchmark results

Last updated: 2026-10-06

This file records llama.cpp Vulkan measurements for comparison across models and hardware configurations. Add new runs as new rows. Keep the settings and measurement limits with each result.

## Test system

| Item | Value |
|---|---|
| llama.cpp commit used for the build | `51ce9c11a` |
| CPU | AMD Ryzen 9 3900X 12-Core Processor |
| Linux kernel | `7.2.8-2-cachyos` |
| Backend | Vulkan, RADV |
| Vulkan0 | AMD Radeon RX 9070 XT, 16,304 MiB VRAM, `VK_KHR_cooperative_matrix` |
| Vulkan1 | AMD Radeon RX 5700 XT, 8,176 MiB VRAM, no cooperative-matrix extension reported |
| Device selection | `VK_LOADER_LAYERS_DISABLE=VK_LAYER_AEJS_DeviceChooserLayer` |
| Other compute backends | CUDA and HIP disabled; no ROCm or HIP backend used |
| Model source | Local files; no downloads during these runs |

The RADV driver reported that it is not conformant and is intended for testing. Keep this driver condition in mind when comparing these results with other Vulkan drivers.

## Current memory-bandwidth finding

| Case | Result | Reading |
|---|---|---|
| RX 9070 XT, IQ2_XXS one-token probe | 47.86 tok/s across four probes; 56% median VRAM busy; about 65% of the calibrated read rate by model-size estimate | No sustained local GDDR bandwidth ceiling is shown during isolated decode. |
| RX 9070 XT, IQ2_XXS plus Vulkan read stream | 3.52 tok/s; 96% median VRAM busy | A near-saturating memory workload sharply reduces decode speed. |
| RX 9070 XT, IQ2_XXS plus cache-resident compute load | 5.16 tok/s; 0% median VRAM busy; 100% GPU busy | Compute contention also sharply reduces decode speed. |
| Q4_K_XL single and dual GPU | 4-33% median VRAM busy across the measured GPUs; single-GPU runs use GTT | No local GDDR saturation is shown. GTT and PCIe limits remain unknown. |

These measurements argue against sustained local VRAM bandwidth saturation during the tested isolated decode runs. They show that both memory and compute competition reduce throughput. They do not identify exact llama.cpp DRAM traffic, memory-latency stalls, or GTT/PCIe traffic.

## Benchmark settings

| Run group | Devices and placement | Model layers | Prompt / decode | Batch / micro-batch | KV cache | Repetitions |
|---|---|---|---|---|---|---:|
| Single GPU | One device per run; `Vulkan1` or `Vulkan0` | `-ngl -1` (automatic fitting) | 1024 prompt / 256 decode tokens | 512 / 128 | Q4_0 / Q4_0 | 3 |
| Single GPU, Qwen Q4 placement comparison | One device per run; `Vulkan0` or `Vulkan1` | 999 requested; all 66 layers assigned | 512 prompt / 128 decode tokens | 512 / 128 | Q4_0 / Q4_0 | 3 |
| Two GPUs | `Vulkan0` + `Vulkan1`; tensor split ratio 2:1 | 999 requested; all model layers were assigned | 512 prompt / 128 decode tokens | 512 / 128 | Q4_0 / Q4_0 | 3 |

All runs used llama-bench. Throughput values below are the mean and standard deviation across the three repetitions. Prompt processing and decode are separate workloads. The Qwen Q4 placement runs used the same offline settings and `--no-host` on one GPU at a time and on both GPUs. They requested all model layers with `-ngl 999`. This option does not guarantee that every model byte is in dedicated VRAM. The earlier IQ2_XXS single-GPU runs used `-ngl -1`.

## Throughput

One row represents one model and hardware configuration. Throughput values are tokens per second; each mean and standard deviation uses three repetitions.

### Run configurations

| Run ID | Model | Model file bytes | GPUs | Split mode | Tensor split | Prompt tokens | Decode tokens | Batch | Ubatch | KV K / V | Repetitions |
|---|---|---:|---|---|---|---:|---:|---:|---:|---|---:|
| `rx5700-long` | Qwen3.8-27B IQ2_XXS | 7266070528 | RX 5700 XT (`Vulkan1`) | Single GPU | N/A | 1024 | 256 | 512 | 128 | Q4_0 / Q4_0 | 3 |
| `rx9070-long` | Qwen3.8-27B IQ2_XXS | 7266070528 | RX 9070 XT (`Vulkan0`) | Single GPU | N/A | 1024 | 256 | 512 | 128 | Q4_0 / Q4_0 | 3 |
| `qwen27b-q4-rx9070` | Qwen3.8-27B Q4_K_XL | 17559178144 | RX 9070 XT (`Vulkan0`) | Layer, one GPU | N/A | 512 | 128 | 512 | 128 | Q4_0 / Q4_0 | 3 |
| `qwen27b-q4-rx5700` | Qwen3.8-27B Q4_K_XL | 17559178144 | RX 5700 XT (`Vulkan1`) | Layer, one GPU | N/A | 512 | 128 | 512 | 128 | Q4_0 / Q4_0 | 3 |
| `qwen27b-q4-dual-layer` | Qwen3.8-27B Q4_K_XL | 17559178144 | RX 9070 XT + RX 5700 XT | Layer | 2:1 | 512 | 128 | 512 | 128 | Q4_0 / Q4_0 | 3 |
| `qwen27b-q4-dual-tensor` | Qwen3.8-27B Q4_K_XL | 17559178144 | RX 9070 XT + RX 5700 XT | Tensor | 2:1 | 512 | 128 | 512 | 128 | Q4_0 / Q4_0 | 3 |
| `agentworld35b-q4-dual-layer` | Qwen-AgentWorld-35B-A3B Q4_K_XL | 22324804864 | RX 9070 XT + RX 5700 XT | Layer | 2:1 | 512 | 128 | 512 | 128 | Q4_0 / Q4_0 | 3 |

### Throughput

Overall tok/s is `(prompt tokens + generated tokens) / (mean prompt time + mean decode time)`, using llama-bench phase times. It excludes model loading and warmup and is a derived rate, not a separate benchmark run.

| Run ID | Prompt mean tok/s | Prompt standard deviation | Decode mean tok/s | Decode standard deviation | Overall tok/s |
|---|---:|---:|---:|---:|---:|
| `rx5700-long` | 125.820542 | 0.263259 | 22.481611 | 0.054750 | 65.55 |
| `rx9070-long` | 1122.987650 | 23.491790 | 52.287744 | 0.139562 | 220.38 |
| `qwen27b-q4-rx9070` | 496.244476 | 5.355830 | 12.710903 | 0.328297 | 57.62 |
| `qwen27b-q4-rx5700` | 16.818317 | 0.021351 | 0.735933 | 0.000758 | 3.13 |
| `qwen27b-q4-dual-layer` | 344.263107 | 0.199130 | 19.829472 | 0.043287 | 80.58 |
| `qwen27b-q4-dual-tensor` | 218.828748 | 0.473399 | 12.931816 | 0.021453 | 52.30 |
| `agentworld35b-q4-dual-layer` | 979.754658 | 10.350666 | 54.732600 | 2.547443 | 223.41 |

### Qwen Q4_K_XL individual samples

| Run ID | Workload | Samples tok/s |
|---|---|---|
| `qwen27b-q4-rx9070` | Prompt | 497.442, 490.391, 500.900 |
| `qwen27b-q4-rx9070` | Decode | 12.3503, 12.7899, 12.9925 |
| `qwen27b-q4-rx5700` | Prompt | 16.8034, 16.8087, 16.8428 |
| `qwen27b-q4-rx5700` | Decode | 0.736526, 0.735078, 0.736194 |
| `qwen27b-q4-dual-layer` | Prompt | 344.445, 344.294, 344.050 |
| `qwen27b-q4-dual-layer` | Decode | 19.8332, 19.7845, 19.8708 |
| `qwen27b-q4-dual-tensor` | Prompt | 219.051, 219.150, 218.285 |
| `qwen27b-q4-dual-tensor` | Decode | 12.9315, 12.9105, 12.9534 |

The AgentWorld model is a 35B-A3B mixture-of-experts model with about 3B active parameters. Do not compare its decode rate directly with the dense Qwen3.8-27B results.

### Qwen3.8-27B Q4_K_XL placement comparison

| Placement | Prompt tok/s | Prompt change vs RX 9070 XT alone | Decode tok/s | Decode change vs RX 9070 XT alone | Overall tok/s | Overall change vs RX 9070 XT alone |
|---|---:|---:|---:|---:|---:|---:|
| RX 9070 XT alone | 496.244476 | Baseline | 12.710903 | Baseline | 57.62 | Baseline |
| RX 5700 XT alone | 16.818317 | -96.6% | 0.735933 | -94.2% | 3.13 | -94.6% |
| Both GPUs, layer split | 344.263107 | -30.6% | 19.829472 | +56.0% | 80.58 | +39.8% |
| Both GPUs, tensor split | 218.828748 | -55.9% | 12.931816 | +1.7% | 52.30 | -9.2% |

For this all-layers-requested setup, adding the RX 5700 XT with layer split reduced prompt speed and improved decode speed. Its combined rate was 80.58 tok/s, 39.8% above the RX 9070 XT alone. Tensor split reached 52.30 tok/s overall, 9.2% below the RX 9070 XT alone; its 1.7% decode difference is small relative to the three-run variation. These single-GPU results allow GTT spill and are not a comparison against a CPU/GPU auto-fit setup.

The two-GPU Qwen3.8-27B Q4_K_XL runs used the same model and settings. Layer split was faster than tensor split for both prompt processing and decode. The telemetry shows more paired GPU activity in the tensor-split capture, but the sampling interval is too coarse to explain the throughput difference on its own.

### Current bottleneck assessment

The current measurements do not show a local VRAM bandwidth bottleneck in the two-GPU Q4 decode runs. Decode-window VRAM-busy medians were 33% on the RX 9070 XT and 30.5% on the RX 5700 XT with layer split, and 24% / 17% with tensor split. The layer-split model-footprint estimates were about 217 GB/s on the RX 9070 XT and 131 GB/s on the RX 5700 XT. Those estimates are below the RX 9070 XT's calibrated 583 GB/s stream rate and the RX 5700 XT's published 448 GB/s peak. These are estimates; llama.cpp's exact DRAM and PCIe byte traffic was not measured.

The result instead points to limited useful GPU work and multi-GPU overhead or imbalance. With layer split, decode GPU-busy medians were 49% and 67.5%, and the RX 9070 XT plus RX 5700 XT produced 80.58 overall tok/s. Tensor split showed more paired GPU activity in its broad capture but produced only 52.30 overall tok/s. The RX 5700 XT also lacks the cooperative-matrix extension reported by the RX 9070 XT. These facts make device imbalance, synchronization/transfer costs, and the Vulkan execution path plausible limits; the available 250 ms activity samples do not identify which one dominates.

For a single RX 9070 XT running Q4_K_XL, the model allocation peaked at 4,232 MiB of GTT, so host-memory/PCIe traffic may limit that over-capacity case even though local VRAM busy was only 21% in the decode probe. In the two-GPU layer-split run, peak GTT was only 545 MiB on the RX 9070 XT and 16 MiB on the RX 5700 XT, so the stronger evidence there points away from GTT spill and toward multi-GPU execution efficiency. Direct PCIe traffic counters were unavailable.

### Row split attempt

| Model | Devices | Split | Result |
|---|---|---|---|
| Qwen3.8-27B Q4_K_XL | RX 9070 XT + RX 5700 XT | Row, 2:1 | Model loading failed before benchmarking: `device Vulkan0 does not support split buffers`. No throughput result was produced. |

The run used the same prompt, decode, batch, KV-cache, and layer settings as the other Qwen Q4 placement tests. In row mode, the model loader requests a backend split-buffer type and raises this error when the backend does not provide one ([loader code](src/llama-model.cpp#L1155)). The current Vulkan backend does not provide that type, so row split is unsupported in this Vulkan build. Changing `--no-host` does not address this limitation.

## GPU and memory telemetry

AMDGPU telemetry was requested at 250 ms intervals. The observed median interval was 263-266 ms. Captures include model load and warmup, not only the measured benchmark interval. Memory activity is a utilization percentage, not a bandwidth measurement in GB/s. VRAM and GTT values are sampled peaks.

### Per-GPU telemetry

| Run ID | GPU | GFX activity mean percent | Memory activity mean percent | Peak VRAM MiB | Peak GTT MiB |
|---|---|---:|---:|---:|---:|
| `rx5700-long` | RX 5700 XT | 90.73 | 21.82 | 6846 | 16 |
| `rx9070-long` | RX 9070 XT | 84.38 | 43.87 | 9194 | 501 |
| `qwen27b-q4-rx9070` | RX 9070 XT | 72.18 | 23.89 | 16286 | 4232 |
| `qwen27b-q4-rx5700` | RX 5700 XT | 86.43 | 4.74 | 8167 | 7902 |
| `qwen27b-q4-dual-layer` | RX 9070 XT | 32.13 | 17.50 | 12671 | 545 |
| `qwen27b-q4-dual-layer` | RX 5700 XT | 37.19 | 16.30 | 6135 | 16 |
| `qwen27b-q4-dual-tensor` | RX 9070 XT | 45.06 | 16.35 | 13402 | 513 |
| `qwen27b-q4-dual-tensor` | RX 5700 XT | 55.48 | 12.07 | 5405 | 18 |
| `agentworld35b-q4-dual-layer` | RX 9070 XT | 16.31 | 3.34 | 15834 | 1629 |
| `agentworld35b-q4-dual-layer` | RX 5700 XT | 7.83 | 2.54 | 6884 | 16 |

### Qwen3.8-27B Q4_K_XL GPU activity summary

The whole-run means below include model load and warmup. Decode medians come from separate one-repetition probes. The whole-run activity fields and decode-window sysfs counters use different sources and time windows, so compare them only as separate measurements.

| Placement | GPU | Whole-run GFX activity mean | Whole-run memory activity mean | Decode GPU-busy median | Decode VRAM-busy median |
|---|---|---:|---:|---:|---:|
| Single GPU | RX 9070 XT | 72.18% | 23.89% | 100% | 21% |
| Single GPU | RX 5700 XT | 86.43% | 4.74% | 99% | 4% |
| Both GPUs, layer split | RX 9070 XT | 32.13% | 17.50% | 49% | 33% |
| Both GPUs, layer split | RX 5700 XT | 37.19% | 16.30% | 67.5% | 30.5% |
| Both GPUs, tensor split | RX 9070 XT | 45.06% | 16.35% | 46% | 24% |
| Both GPUs, tensor split | RX 5700 XT | 55.48% | 12.07% | 66% | 17% |

### Capture and host memory

| Run ID | Telemetry samples | Median interval ms | Paired sample count | Both GPUs at >=10% GFX | Minimum host memory available GiB |
|---|---:|---:|---:|---:|---:|
| `rx5700-long` | 268 | 265 | N/A | N/A | Not recorded |
| `rx9070-long` | 84 | 263 | N/A | N/A | Not recorded |
| `qwen27b-q4-rx9070` | 187 | 264 | N/A | N/A | 17.06 |
| `qwen27b-q4-rx5700` | 2774 | 266 | N/A | N/A | 10.20 |
| `qwen27b-q4-dual-layer` | 157 | 266 | 157 | 74 (47.1%) | 19.85 |
| `qwen27b-q4-dual-tensor` | 176 | 265 | 176 | 149 (84.7%) | 20.30 |
| `agentworld35b-q4-dual-layer` | 179 | 265 | 179 | 31 (17.3%) | 20.32 |

The paired-activity count requires both GPUs to report at least 10% GFX activity in the same sample. It does not show sub-millisecond overlap. AMDGPU did not provide direct average DRAM read/write bandwidth metrics for these captures.

### Model-footprint rate estimates

AMD lists peak memory bandwidth as up to 640 GB/s for the RX 9070 XT and up to 448 GB/s for the RX 5700 XT ([RX 9070 XT specs](https://www.amd.com/en/products/graphics/desktops/radeon/9000-series/amd-radeon-rx-9070xt.html), [RX 5700 XT specs](https://www.amd.com/en/support/downloads/drivers.html/graphics/radeon-rx/radeon-rx-5000-series/amd-radeon-rx-5700-xt.html)). The table estimates model-footprint rate as llama-bench model size multiplied by decode tokens/s. It assumes one model-size read for each generated token. This is not a DRAM counter measurement.

| Model and placement | Decode tok/s | Model-footprint rate estimate GB/s | Reference |
|---|---:|---:|---|
| Qwen3.8-27B IQ2_XXS, RX 5700 XT | 22.48 | 163.1 | 36.4% of RX 5700 XT published peak |
| Qwen3.8-27B IQ2_XXS, RX 9070 XT | 52.29 | 379.4 | 59.3% of RX 9070 XT published peak |
| Qwen3.8-27B Q4_K_XL, RX 9070 XT | 12.71 | 223.1 | 34.9% of RX 9070 XT published peak; GTT peak was 4,232 MiB |
| Qwen3.8-27B Q4_K_XL, RX 5700 XT | 0.736 | 12.9 | Not comparable to GDDR peak; GTT peak was 7,902 MiB |
| Qwen3.8-27B Q4_K_XL, both GPUs with layer split | 19.83 | 348.0 combined | Two separate memory pools; combined rate does not show either GPU's bandwidth use |
| Qwen3.8-27B Q4_K_XL, both GPUs with tensor split | 12.93 | 226.9 combined | Two separate memory pools; combined rate does not show either GPU's bandwidth use |

These estimates put the fitting IQ2_XXS runs below the cards' published peak rates. They assume one full model-tensor read per generated token and are not DRAM counter measurements. The Q4_K_XL single-GPU runs spill into GTT, so their estimates do not isolate GDDR bandwidth.

### RX 9070 XT OpenCL streaming-read cross-check

I ran a controlled OpenCL/Rusticl read test on the RX 9070 XT. It used a 512 MiB buffer and launched 512 kernels. Each kernel read the full buffer once. The 512 kernels requested 274.878 GB of reads in total. GPU event times were summed across the kernels. The test sampled `mem_busy_percent` every 20 ms. The buffer is larger than the GPU's 64 MiB Infinity Cache, so repeated passes cannot remain in that cache.

| Trial | Read GB | GPU time s | Read rate GB/s | `mem_busy_percent` samples | Busy median | Busy range |
|---|---:|---:|---:|---:|---:|---:|
| 1 | 274.878 | 0.474409 | 579.411 | 26 | 88% | 4-90% |
| 2 | 274.878 | 0.474896 | 578.818 | 25 | 89% | 1-92% |
| 3 | 274.878 | 0.474853 | 578.870 | 26 | 89% | 1-92% |

The median measured rate was 578.870 GB/s, or 90.4% of the RX 9070 XT's published 640 GB/s peak. Rusticl emitted a warning that patched Mesa libclc was absent; the checksum matched across all three runs and the rates were stable. This is an OpenCL stream test, not a llama.cpp measurement or a hardware byte counter. The busy readings are an activity signal, not a linear bandwidth scale. Raw samples are in [memory-bandwidth-calibration-rx9070.jsonl](build-vulkan/benchmark-results/memory-bandwidth-calibration-rx9070.jsonl).

### RX 9070 XT Vulkan streaming-read calibration

I repeated the stream test with a standalone compute dispatch using RADV, the Vulkan driver used by llama.cpp. Each run read a 512 MiB device-local buffer 512 times. The buffer is larger than the 64 MiB Infinity Cache. The shader read 64 bytes and wrote 4 bytes per work item. Vulkan timestamps measured the dispatch sequence. The reported byte counts are the shader's requested reads and writes, not hardware DRAM counters.

| Trial | Read GB | Write GB | GPU time s | Read GB/s | Total GB/s | `mem_busy_percent` samples | Busy median |
|---|---:|---:|---:|---:|---:|---:|---:|
| 1 | 274.878 | 17.180 | 0.471135 | 583.437 | 619.902 | 23 | 96% |
| 2 | 274.878 | 17.180 | 0.471388 | 583.125 | 619.570 | 24 | 96% |
| 3 | 274.878 | 17.180 | 0.471129 | 583.445 | 619.910 | 24 | 96% |

The median read rate was 583.437 GB/s, 91.2% of the published 640 GB/s peak. Reads plus writes reached 619.902 GB/s, 96.9% of that peak. The median `mem_busy_percent` was 96% in all three runs. The OpenCL result is within 0.8% of this Vulkan read rate. These tests provide a repeatable local stream reference, but neither measures llama.cpp's actual DRAM bytes. Raw samples are in [memory-bandwidth-calibration-rx9070-vulkan.jsonl](build-vulkan/benchmark-results/memory-bandwidth-calibration-rx9070-vulkan.jsonl).

### Read-rate comparison for decode

Qwen3.8-27B is dense (`n_expert=0` in the loader output). For these rows, the estimate is the model tensor size multiplied by decode rate. It approximates a full weight scan per token. The rate and busy samples come from separate benchmark runs, so the comparison is approximate.

| Workload and device | Decode rate used | Estimated read rate | Decode-window VRAM-busy median | Reference peak or calibration | Assessment |
|---|---:|---:|---:|---:|---|
| IQ2_XXS, RX 9070 XT | 47.79 tok/s probe | 346.7 GB/s | 56% | 59.4% of measured 583.4 GB/s | Below the calibrated streaming ceiling; no sustained full-bandwidth evidence |
| IQ2_XXS, RX 5700 XT | 22.68 tok/s probe | 164.5 GB/s | 39% | 36.7% of published 448 GB/s | Below published peak; no sustained full-bandwidth evidence |
| Q4_K_XL, RX 9070 XT | 8.18 tok/s probe | 143.5 GB/s | 21% | 24.6% of measured 583.4 GB/s | Local VRAM was not saturated; model used 4,232 MiB GTT at peak |
| Q4_K_XL, RX 5700 XT | 0.70 tok/s probe | 12.3 GB/s | 4% | Not comparable to GDDR peak | Model used 7,902 MiB GTT at peak; GTT/PCIe traffic remains unmeasured |
| Q4_K_XL, layer split, RX 9070 XT | 19.83 tok/s combined | About 216.8 GB/s | 33% | About 37.1% of measured 583.4 GB/s | Approximate per-device estimate from layer-buffer share; local VRAM was not saturated |
| Q4_K_XL, layer split, RX 5700 XT | 19.83 tok/s combined | About 131.1 GB/s | 30.5% | About 29.3% of published 448 GB/s | Approximate per-device estimate from layer-buffer share; local VRAM was not saturated |
| Q4_K_XL, tensor split, both GPUs | 12.93 tok/s combined | 226.9 GB/s combined | 24% / 17% | Per-device split was not reported | Low busy readings; cannot assign the combined estimate to either memory pool |

For IQ2_XXS on the RX 9070 XT, the three-repetition rate of 52.29 tok/s implies about 379.4 GB/s, or 65.0% of the Vulkan stream rate. The one-repetition probe implies 346.7 GB/s. The decode-window median was 56% busy, compared with 96% during the stream test. These estimates assume one full model-tensor read per generated token. They argue against a sustained local GDDR bandwidth ceiling during isolated IQ2 decode, but they are not measured model traffic.

The Q4_K_XL layer-split estimates use each GPU's reported model-buffer share to divide the combined model-footprint rate. They are approximate because allocated bytes do not prove the exact bytes fetched from each GPU per token. The single-GPU Q4 runs spill into GTT, so low VRAM-busy values rule against saturating local GDDR, but do not rule out a host-memory or PCIe bottleneck. No PCIe byte counter was available.

Across these tested decode runs, the evidence does not show sustained local VRAM bandwidth saturation. The RX 9070 XT Vulkan stream gives a direct read-rate reference, and the model estimates and busy samples remain below it. This does not rule out memory latency limits, short bursts, or GTT/PCIe limits in the over-capacity Q4 runs.

### RX 9070 XT IQ2_XXS contention comparison

I compared the same one-token prompt and 256-token IQ2_XXS decode on the RX 9070 XT by itself and while a second Vulkan process used the same GPU. The model tensor size was 7,255,074,816 bytes. All runs used `-ngl 999`, `Vulkan0`, `--no-host`, batch 128, ubatch 128, Q4_0 K/V cache, one repetition, and no warmup. The explicit `VK_LOADER_LAYERS_DISABLE=VK_LAYER_AEJS_DeviceChooserLayer` setting kept the device order stable: `Vulkan0` was the RX 9070 XT and `Vulkan1` was the RX 5700 XT. Busy values below are decode-window medians sampled at 100 ms intervals.

| Condition | Decode samples tok/s | Mean +/- sample SD tok/s | GPU-busy median | VRAM-busy median |
|---|---|---:|---:|---:|
| No competing workload | 47.897, 47.855, 47.854, 47.831 | 47.859 +/- 0.027 | 99% | 56% |
| 512 MiB Vulkan streaming reads | 3.578, 3.442, 3.553 | 3.524 +/- 0.072 | 100% | 96% |
| Cache-resident compute workload | 5.151, 5.171, 5.169 | 5.164 +/- 0.011 | 100% | 0% |

The streaming process read at 582.37-583.36 GB/s on its individual 512 MiB passes, with 619-620 GB/s including writes. The compute control used a 16 MiB buffer and repeated an FMA shader. It kept GPU busy at 100% while its median VRAM-busy value stayed at 0%; a standalone check reported a 0-7% busy range. This control tests competition for GPU execution with little sustained GDDR traffic.

Both competing workloads reduced decode speed sharply. The compute-only load reduced it by 89.2%; the memory-stream load reduced it by 92.6%. The stream condition was 31.8% slower than the compute-only condition. This is consistent with an additional cost from memory traffic, but it does not measure the exact share of decode time spent waiting on memory: the stress shaders differ, and no valid hardware counter reported llama.cpp's DRAM bytes. Together with the isolated 56% VRAM-busy median and the estimated 65.0% of the measured read ceiling, the results do not show a sustained local-bandwidth ceiling for isolated IQ2 decode.

An initial contention batch in `memory-contention-qwen-iq2-rx9070-20261006/` did not set `VK_LOADER_LAYERS_DISABLE`. In that batch, llama-bench saw only the RX 5700 XT as `Vulkan0`, while the stream process and sampler targeted the RX 9070 XT. Those runs are excluded. The corrected captures are in `memory-contention-qwen-iq2-rx9070-corrected-20261006/`.

### Decode-window VRAM-busy samples

These one-repetition probes sampled the kernel's `gpu_busy_percent` and `mem_busy_percent` sysfs values every 100 ms. The sampler wrote the start time after it observed the second `sched_reserve` log line, then wrote the end time after the second standalone `eval time` line. The IQ2_XXS and dual-GPU Q4_K_XL samplers checked the log every 25 ms; the new single-GPU Q4_K_XL samplers checked it every 100 ms. These markers approximate the decode interval. The prompts were one token. IQ2_XXS probes decoded 256 tokens; all Q4_K_XL probes decoded 128 tokens.

These one-repetition rates are probe results. Use the three-repetition table above for throughput comparisons.

| Run | Device during decode | Decode tok/s, one repetition | Window sec | Samples | GPU busy median | GPU busy range | VRAM busy median | VRAM busy range |
|---|---|---:|---:|---:|---:|---:|---:|---:|
| Qwen3.8-27B IQ2_XXS, single GPU | RX 9070 XT | 47.79 | 5.37 | 52 | 99% | 89-100% | 56% | 50-60% |
| Qwen3.8-27B IQ2_XXS, single GPU | RX 5700 XT | 22.68 | 11.31 | 111 | 99% | 51-99% | 39% | 17-46% |
| Qwen3.8-27B Q4_K_XL, single GPU | RX 9070 XT | 8.18 | 15.62 | 142 | 100% | 91-100% | 21% | 0-61% |
| Qwen3.8-27B Q4_K_XL, single GPU | RX 5700 XT | 0.70 | 182.83 | 1668 | 99% | 92-100% | 4% | 1-33% |
| Qwen3.8-27B Q4_K_XL, both GPUs, layer split | RX 9070 XT | 20.17 combined | 6.36 | 62 | 49% | 7-79% | 33% | 0-59% |
| Qwen3.8-27B Q4_K_XL, both GPUs, layer split | RX 5700 XT | 20.17 combined | 6.36 | 62 | 67.5% | 11-94% | 30.5% | 7-63% |
| Qwen3.8-27B Q4_K_XL, both GPUs, tensor split | RX 9070 XT | 13.38 combined | 9.57 | 93 | 46% | 38-53% | 24% | 19-33% |
| Qwen3.8-27B Q4_K_XL, both GPUs, tensor split | RX 5700 XT | 13.38 combined | 9.57 | 93 | 66% | 63-70% | 17% | 15-23% |

The Linux kernel defines `mem_busy_percent` as the SMU firmware's percentage estimate of how busy VRAM is. It is not a byte counter or a measurement in GB/s ([kernel documentation](https://docs.kernel.org/gpu/amdgpu/thermal.html)). The single-GPU Q4_K_XL probes had 99-100% GPU busy, but only 21% VRAM busy on the RX 9070 XT and 4% on the RX 5700 XT. The RX 5700 XT probe also spilled 7,902 MiB into GTT, so its VRAM-busy value does not measure host-memory or PCIe traffic. In the dual-GPU Q4_K_XL layer-split sample, median VRAM busy was 33% and 30.5%; with tensor split, it was 24% and 17%. These windows did not show sustained 100% VRAM busy. They cannot establish exact bandwidth use or rule out short periods at the limit.

For Q4_K_XL, the GPU engines were near fully busy in both single-GPU probes while the VRAM busy counters stayed low. This is evidence against sustained saturation of the local VRAM controller in those windows. Because both single-GPU runs spilled into GTT and PCIe byte counts are unavailable, the readings cannot distinguish a GTT/PCIe limit from another GPU-side limit.

The three-repetition Q4_K_XL benchmark showed a 56.0% decode gain for layer split over the RX 9070 XT alone. Tensor split was 1.7% faster than the RX 9070 XT alone, within the run variation. In the one-repetition probes, layer split was faster than tensor split and had higher VRAM-busy medians on both cards. The single-GPU Q4_K_XL runs spilled into GTT, so these comparisons do not isolate dedicated VRAM bandwidth or identify what caused the performance difference. The new single-GPU probes kept at least 22.08 GiB and 16.79 GiB of host memory available on the RX 9070 XT and RX 5700 XT, respectively; neither reached the 8 GiB stop limit.

### Direct GPU-memory counter capture

On 2026-10-06, I tried to collect RGP memory-byte counters through RADV's `MESA_VK_TRACE=rgp` path. Mesa documents RGP traces for RADV. Radeon GPU Profiler documents memory Fetch size and Write size counters for RDNA3 and newer; the RX 9070 XT is GFX12, while the RX 5700 XT is RDNA1 ([Mesa trace variables](https://docs.mesa3d.org/envvars.html), [RGP Events window](https://gpuopen.com/manuals/rgp_manual/events_windows/)).

| Local time | GPU | Outcome | Usable bandwidth data |
|---|---|---|---|
| 17:44 | RX 9070 XT (`Vulkan0`) | RADV canceled the trace after detecting a hang condition and requested `profile_peak`. `llama-bench` then lost the Vulkan device before model context setup. The kernel logged a graphics-ring timeout and recovered the device. | None. The saved 6,794-byte `.rgp` file is from the canceled attempt and is not a valid benchmark trace. |
| 17:47 | RX 5700 XT (`Vulkan1`) | During `llama-bench`, the kernel logged a GPU page fault, a graphics-ring timeout, and a failed ring reset. The RX 5700 XT then disappeared from the bus. The machine rebooted shortly after. | None. The run did not finish and produced no valid trace or benchmark output. |
| 17:58 | RX 9070 XT (`Vulkan0`) | After authorization, RADV was retried with `profile_peak` active for one 16-token IQ2_XXS decode. RADV still canceled the trace; the kernel logged a graphics-ring timeout and a successful GPU reset. The setting was restored to `manual`. | None. The saved 6,794-byte `.rgp` file is from the canceled attempt and is not a valid benchmark trace. |
| 19:11 | RX 9070 XT (`Vulkan0`) | A one-token prompt and 16-token IQ2_XXS decode completed, but the trigger watcher did not find its scheduler marker because verbose logging was off. No trigger file was created. | None. No RGP trace was produced and no reset was logged. |
| 19:12 | RX 9070 XT (`Vulkan0`) | The trigger was created after model loading and decode graph setup. The benchmark completed, but RADV produced no `.rgp` file. | None. No reset was logged; the `profile_peak` setting was restored to `manual`. |

The kernel evidence ties the RX 5700 XT failure to a GPU page fault and failed reset during the profiler attempt. It does not prove what caused the later system reboot. The RX 9070 XT `profile_peak` attempt also canceled and reset the GPU. Mesa 26.2.4 checks its trigger file in the Vulkan WSI present path, and RADV starts or stops this trace path from `vkQueuePresentKHR` ([Mesa 26.2.4 WSI source](https://gitlab.freedesktop.org/mesa/mesa/-/blob/mesa-26.2.4/src/vulkan/wsi/wsi_common.c#L2339), [RADV 26.2.4 trace source](https://gitlab.freedesktop.org/mesa/mesa/-/blob/mesa-26.2.4/src/amd/vulkan/layers/radv_sqtt_layer.c#L665)). `llama-bench` uses headless compute and does not present a frame, so its delayed trigger cannot start a capture. The per-submit method did not produce a usable trace and previously caused resets, so I stopped trying RGP on this workload. A post-reboot `amdgpu_top --gpu-metrics -J` sample returned null for `average_dram_reads` and `average_dram_writes` on both cards.

No direct DRAM byte counters were collected. The decode-window `mem_busy_percent` samples above provide a VRAM-busy signal but not measured bandwidth in GB/s. The RX 9070 XT is back in its original `manual` mode, and the RX 5700 XT remains in `auto`.

#### AMD GPUPerfAPI counter attempt

I also tried GPUPerfAPI 4.4 through an isolated AMDVLK driver because GPUPerfAPI does not support Mesa RADV on Linux. The llama.cpp throughput runs above remain RADV results. On the RX 9070 XT, GPUPerfAPI returned zero for `FetchSize`, `PcieBytes`, `LocalVidMemBytes`, and `MemUnitBusy`, including when profiling the full 4,041-node decode graph. `GPUTime` was nonzero, but the zero memory values are invalid for this workload and were not used. The RX 5700 XT attempt ended with `VK_ERROR_DEVICE_LOST` during queue submission and produced no counter values. I stopped GPA profiling on the RX 5700 XT after that failure.

| Device | Counter set / capture | Result | Used in analysis |
|---|---|---|---|
| RX 9070 XT | `FetchSize,GPUTime,PcieBytes`, 3 passes | `FetchSize=0`, `PcieBytes=0`, `GPUTime=21.7 ms` | No; memory counters were invalid |
| RX 9070 XT | `LocalVidMemBytes,PcieBytes`, one sample spanning the graph | Both zero | No |
| RX 9070 XT | `LocalVidMemBytes,PcieBytes`, 23 separate command-buffer samples | All values zero | No |
| RX 9070 XT | `MemUnitBusy`, one sample spanning the graph | Zero | No |
| RX 5700 XT | `LocalVidMemBytes,PcieBytes`, separate command-buffer samples | Device lost during queue submission | No; no result was produced |

The raw GPA logs are under `build-vulkan/benchmark-results/gpa-captures-20261006/`. They confirm that this counter path did not provide valid model traffic values. The standalone Vulkan and OpenCL streaming-read calibrations above provide repeatable stream references, but not model traffic counters.

### Other direct counter paths checked

AMD documents Linux Vulkan profiling with RGP through either a 25.10-based Radeon driver or RADV's integrated trace path. RDP does not capture Vulkan workloads through the newer 25.20-based driver stack. The 25.10.2 package lists RX 9070 XT support, but its supported Linux distributions do not include EndeavourOS/Arch, so a driver change is not a low-risk option on this host ([RGP Linux guidance](https://gpuopen.com/manuals/rgp_manual/), [25.10.2 release notes](https://www.amd.com/en/resources/support-articles/release-notes/radeon--software-for-linux--25-10-2-with-rocm-6-4-2-release-note.html)). AMD's documented ROCm memory-bandwidth analysis currently supports MI350 GPUs, not the RX 9070 XT ([ROCm memory-bandwidth analysis](https://rocm.docs.amd.com/projects/rocprofiler-compute/en/develop/how-to/membw_analysis.html)).

The kernel documents a `pcie_bw` sysfs counter that estimates PCIe traffic over the last second. It could help show PCIe traffic during GTT-heavy runs, but it does not isolate GTT traffic from other PCIe transfers ([kernel documentation](https://docs.kernel.org/gpu/amdgpu/driver-misc.html#pcie_bw)). The file is absent for both GPUs on this host. `perf list` shows a Zen 2 `nps1_die_to_dram` metric, but this kernel does not expose a `data_fabric` PMU under `/sys/bus/event_source/devices`; `perf stat` cannot collect it. That metric would also count total system DRAM traffic, not isolate GPU requests. No additional GPU-specific byte counter was available through the checked interfaces.

## Model placement and capacity checks

| Model | Device setup | Loader result | Notes |
|---|---|---|---|
| Qwen3.8-27B IQ2_XXS, 7,266,070,528-byte file | RX 5700 XT only | Explicit check loaded 65/65 layers; Vulkan model buffer 6,521.13 MiB; CPU-mapped buffer 397.85 MiB | 8,154 MiB free VRAM was reported before loading. The throughput run itself used automatic layer fitting. |
| Qwen3.8-27B Q2_K_XL, 9,828,981,664-byte file | RX 9070 XT only | 66/66 layers assigned; Vulkan model buffer 8,630.56 MiB | 13,932 MiB free VRAM was reported before loading. Sampled peak VRAM use was 11,302 MiB. |
| Qwen3.8-27B Q2_K_XL, 9,828,981,664-byte file | RX 5700 XT only | 66/66 layers assigned; Vulkan model buffer 8,630.56 MiB; CPU-mapped buffer 397.85 MiB | The reported free VRAM was 8,154 MiB. A one-token run completed even though the requested Vulkan model buffer exceeded that amount; this does not prove all weights remained in dedicated VRAM. |
| Qwen3.8-27B Q4_K_XL, 17,559,178,144-byte file | RX 9070 XT only | 66/66 layers assigned; Vulkan model buffer 15,718.47 MiB; CPU-mapped buffer 682.03 MiB | Sampled peak was 16,286 MiB VRAM and 4,232 MiB GTT. Minimum available host memory was 17.06 GiB. |
| Qwen3.8-27B Q4_K_XL, 17,559,178,144-byte file | RX 5700 XT only | 66/66 layers assigned; Vulkan model buffer 15,718.47 MiB; CPU-mapped buffer 682.03 MiB | Sampled peak was 8,167 MiB VRAM and 7,902 MiB GTT. Minimum available host memory was 10.20 GiB; the 8 GiB stop limit was not reached. |
| Qwen3.8-27B Q4_K_XL, 17,559,178,144-byte file | Both GPUs, layer split | 66/66 layers assigned; buffers: 9,795.39 MiB on RX 9070 XT, 5,923.08 MiB on RX 5700 XT, and 682.03 MiB CPU-mapped | Model file is larger than the RX 9070 XT's nominal VRAM. |
| Qwen3.8-27B Q4_K_XL, 17,559,178,144-byte file | Both GPUs, tensor split | 66/66 layers assigned; Meta buffer 10,524.96 MiB and CPU-mapped buffer 682.03 MiB | Per-device model-buffer sizes were not reported for the Meta buffer. |
| Qwen-AgentWorld-35B-A3B Q4_K_XL, 22,324,804,864-byte file | Both GPUs, layer split | 41/41 layers assigned; buffers: 14,058.44 MiB on RX 9070 XT, 6,706.36 MiB on RX 5700 XT, and 515.31 MiB CPU-mapped | Host-memory guard was set to 8 GiB available; the minimum observed was 20.32 GiB. |

Layer assignment and reported buffer sizes do not prove that every model byte resides in dedicated VRAM. `--no-host` skips a special GPU-host buffer, but CPU-mapped memory and driver-managed GTT can still be used.

## Source records

The build and raw captures are in the ignored `build-vulkan/benchmark-results/` directory.

- [Two-GPU normalized CSV](build-vulkan/benchmark-results/multigpu_results.csv)
- [Two-GPU normalized JSON](build-vulkan/benchmark-results/multigpu_results.json)
- [RX 9070 XT Q4_K_XL single-GPU JSONL](build-vulkan/benchmark-results/qwen27b-q4-rx9070.jsonl)
- [RX 5700 XT Q4_K_XL single-GPU JSONL](build-vulkan/benchmark-results/qwen27b-q4-rx5700.jsonl)
- [RX 9070 XT Q4_K_XL single-GPU status](build-vulkan/benchmark-results/qwen27b-q4-rx9070.status.json)
- [RX 5700 XT Q4_K_XL single-GPU status](build-vulkan/benchmark-results/qwen27b-q4-rx5700.status.json)
- [RX 9070 XT Q4_K_XL single-GPU telemetry](build-vulkan/benchmark-results/qwen27b-q4-rx9070.telemetry.jsonl)
- [RX 5700 XT Q4_K_XL single-GPU telemetry](build-vulkan/benchmark-results/qwen27b-q4-rx5700.telemetry.jsonl)
- [Qwen Q4_K_XL row-split attempt log](build-vulkan/benchmark-results/qwen27b-q4-dual-row.stderr)
- [Qwen Q4_K_XL row-split attempt status](build-vulkan/benchmark-results/qwen27b-q4-dual-row.status.json)
- [RX 5700 XT single-GPU llama-bench JSON](build-vulkan/benchmark-results/rx5700-long.json)
- [RX 9070 XT single-GPU llama-bench JSON](build-vulkan/benchmark-results/rx9070-long.json)
- [RX 5700 XT IQ2_XXS full-layer check](build-vulkan/benchmark-results/rx5700-iq2-all-bench.txt)
- [RX 9070 XT Q2_K_XL capacity check](build-vulkan/benchmark-results/rx9070-q2k-all-bench.txt)
- [RX 5700 XT Q2_K_XL capacity check](build-vulkan/benchmark-results/rx5700-q2k-full-capacity.txt)
- [GPU activity plot](build-vulkan/benchmark-results/gpu-activity.png)
- [RX 9070 XT Vulkan streaming-read calibration JSONL](build-vulkan/benchmark-results/memory-bandwidth-calibration-rx9070-vulkan.jsonl)
- [RX 9070 XT OpenCL streaming-read cross-check JSONL](build-vulkan/benchmark-results/memory-bandwidth-calibration-rx9070.jsonl)
- [RX 9070 XT cache-resident compute calibration JSON](build-vulkan/benchmark-results/perf-counter-tools/compute-continuous-calibration.json)
- [Corrected contention benchmark script](build-vulkan/benchmark-results/run-memory-contention-qwen-iq2.sh)
- [Corrected IQ2 baseline benchmark JSONL](build-vulkan/benchmark-results/memory-contention-qwen-iq2-rx9070-corrected-20261006/pair-c1-control/llama-bench.jsonl)
- [Corrected IQ2 memory-contention benchmark JSONL](build-vulkan/benchmark-results/memory-contention-qwen-iq2-rx9070-corrected-20261006/pair-1-stress/llama-bench.jsonl)
- [Corrected IQ2 memory-contention stream data](build-vulkan/benchmark-results/memory-contention-qwen-iq2-rx9070-corrected-20261006/pair-1-stress/streamer.jsonl)
- [Corrected IQ2 compute-contention benchmark JSONL](build-vulkan/benchmark-results/memory-contention-qwen-iq2-rx9070-corrected-20261006/pair-c1-compute/llama-bench.jsonl)
- [Corrected IQ2 compute-contention load data](build-vulkan/benchmark-results/memory-contention-qwen-iq2-rx9070-corrected-20261006/pair-c1-compute/streamer.jsonl)
- [RX 9070 XT canceled RGP probe output](build-vulkan/benchmark-results/radv-rgp/probe/qwen27b-iq2-rgp-probe.stderr)
- [RX 9070 XT canceled RGP trace](build-vulkan/benchmark-results/radv-rgp/probe/llama-bench_2026.10.06_17.44.14_submit0.rgp)
- [Prior-boot GPU reset log for RX 5700 XT](build-vulkan/benchmark-results/radv-rgp/probe/prior-boot-gpu-reset.log)
- [RX 9070 XT profile-peak probe output](build-vulkan/benchmark-results/radv-rgp/rx9070-profile-peak-20261006/llama-bench.stderr)
- [RX 9070 XT profile-peak benchmark output](build-vulkan/benchmark-results/radv-rgp/rx9070-profile-peak-20261006/llama-bench.jsonl)
- [RX 9070 XT profile-peak exit status](build-vulkan/benchmark-results/radv-rgp/rx9070-profile-peak-20261006/exit-status.txt)
- [RX 9070 XT profile-peak reset log](build-vulkan/benchmark-results/radv-rgp/rx9070-profile-peak-20261006/kernel-reset.log)
- [RX 9070 XT profile-peak canceled trace](build-vulkan/benchmark-results/radv-rgp/rx9070-profile-peak-20261006/llama-bench_2026.10.06_17.58.42_submit0.rgp)
- [RX 9070 XT profile-peak active mode readback](build-vulkan/benchmark-results/radv-rgp/rx9070-profile-peak-20261006/active-mode.txt)
- [RX 9070 XT restored mode readback](build-vulkan/benchmark-results/radv-rgp/rx9070-profile-peak-20261006/restored-mode.txt)
- [RX 9070 XT profile-peak benchmark command](build-vulkan/benchmark-results/radv-rgp/rx9070-profile-peak-20261006/run-as-user.sh)
- [RX 9070 XT delayed-trigger probe output](build-vulkan/benchmark-results/radv-rgp/rx9070-delayed-trigger-verbose-20261006/llama-bench.stderr)
- [RX 9070 XT delayed-trigger benchmark JSONL](build-vulkan/benchmark-results/radv-rgp/rx9070-delayed-trigger-verbose-20261006/llama-bench.jsonl)
- [RX 9070 XT delayed-trigger exit status](build-vulkan/benchmark-results/radv-rgp/rx9070-delayed-trigger-verbose-20261006/exit-status.txt)
- [RX 9070 XT delayed-trigger marker time](build-vulkan/benchmark-results/radv-rgp/rx9070-delayed-trigger-verbose-20261006/trigger-fired.txt)
- [RX 9070 XT IQ2_XXS decode-window busy samples](build-vulkan/benchmark-results/memory-busy-qwen-iq2-rx9070-repeat-20261006/memory-busy.csv)
- [RX 9070 XT IQ2_XXS one-repetition benchmark output](build-vulkan/benchmark-results/memory-busy-qwen-iq2-rx9070-repeat-20261006/llama-bench.jsonl)
- [RX 9070 XT IQ2_XXS decode start marker](build-vulkan/benchmark-results/memory-busy-qwen-iq2-rx9070-repeat-20261006/decode-start.txt)
- [RX 9070 XT IQ2_XXS decode end marker](build-vulkan/benchmark-results/memory-busy-qwen-iq2-rx9070-repeat-20261006/decode-end.txt)
- [RX 9070 XT IQ2_XXS sampling script](build-vulkan/benchmark-results/memory-busy-qwen-iq2-rx9070-repeat-20261006/run-as-user.sh)
- [RX 5700 XT IQ2_XXS decode-window busy samples](build-vulkan/benchmark-results/memory-busy-qwen-iq2-rx5700-20261006/memory-busy.csv)
- [RX 5700 XT IQ2_XXS one-repetition benchmark output](build-vulkan/benchmark-results/memory-busy-qwen-iq2-rx5700-20261006/llama-bench.jsonl)
- [RX 5700 XT IQ2_XXS decode start marker](build-vulkan/benchmark-results/memory-busy-qwen-iq2-rx5700-20261006/decode-start.txt)
- [RX 5700 XT IQ2_XXS decode end marker](build-vulkan/benchmark-results/memory-busy-qwen-iq2-rx5700-20261006/decode-end.txt)
- [RX 5700 XT IQ2_XXS sampling script](build-vulkan/benchmark-results/memory-busy-qwen-iq2-rx5700-20261006/run-as-user.sh)
- [Dual-GPU Q4_K_XL layer-split decode-window busy samples](build-vulkan/benchmark-results/memory-busy-qwen-q4-dual-layer-20261006/memory-busy.csv)
- [Dual-GPU Q4_K_XL one-repetition benchmark output](build-vulkan/benchmark-results/memory-busy-qwen-q4-dual-layer-20261006/llama-bench.jsonl)
- [Dual-GPU Q4_K_XL decode start marker](build-vulkan/benchmark-results/memory-busy-qwen-q4-dual-layer-20261006/decode-start.txt)
- [Dual-GPU Q4_K_XL decode end marker](build-vulkan/benchmark-results/memory-busy-qwen-q4-dual-layer-20261006/decode-end.txt)
- [Dual-GPU Q4_K_XL sampling script](build-vulkan/benchmark-results/memory-busy-qwen-q4-dual-layer-20261006/run-as-user.sh)
- [Dual-GPU Q4_K_XL tensor-split decode-window busy samples](build-vulkan/benchmark-results/memory-busy-qwen-q4-dual-tensor-20261006/memory-busy.csv)
- [Dual-GPU Q4_K_XL tensor-split one-repetition benchmark output](build-vulkan/benchmark-results/memory-busy-qwen-q4-dual-tensor-20261006/llama-bench.jsonl)
- [Dual-GPU Q4_K_XL tensor-split decode start marker](build-vulkan/benchmark-results/memory-busy-qwen-q4-dual-tensor-20261006/decode-start.txt)
- [Dual-GPU Q4_K_XL tensor-split decode end marker](build-vulkan/benchmark-results/memory-busy-qwen-q4-dual-tensor-20261006/decode-end.txt)
- [Dual-GPU Q4_K_XL tensor-split sampling script](build-vulkan/benchmark-results/memory-busy-qwen-q4-dual-tensor-20261006/run-as-user.sh)
- [RX 9070 XT single-GPU Q4_K_XL decode-window samples](build-vulkan/benchmark-results/memory-busy-qwen-q4-rx9070-20261006/memory-busy.csv)
- [RX 9070 XT single-GPU Q4_K_XL one-repetition benchmark output](build-vulkan/benchmark-results/memory-busy-qwen-q4-rx9070-20261006/llama-bench.jsonl)
- [RX 9070 XT single-GPU Q4_K_XL sampling script](build-vulkan/benchmark-results/memory-busy-qwen-q4-rx9070-20261006/run-as-user.sh)
- [RX 9070 XT single-GPU Q4_K_XL decode start marker](build-vulkan/benchmark-results/memory-busy-qwen-q4-rx9070-20261006/decode-start.txt)
- [RX 9070 XT single-GPU Q4_K_XL decode end marker](build-vulkan/benchmark-results/memory-busy-qwen-q4-rx9070-20261006/decode-end.txt)
- [RX 9070 XT single-GPU Q4_K_XL exit status](build-vulkan/benchmark-results/memory-busy-qwen-q4-rx9070-20261006/exit-status.txt)
- [RX 5700 XT single-GPU Q4_K_XL decode-window samples](build-vulkan/benchmark-results/memory-busy-qwen-q4-rx5700-20261006/memory-busy.csv)
- [RX 5700 XT single-GPU Q4_K_XL one-repetition benchmark output](build-vulkan/benchmark-results/memory-busy-qwen-q4-rx5700-20261006/llama-bench.jsonl)
- [RX 5700 XT single-GPU Q4_K_XL sampling script](build-vulkan/benchmark-results/memory-busy-qwen-q4-rx5700-20261006/run-as-user.sh)
- [RX 5700 XT single-GPU Q4_K_XL decode start marker](build-vulkan/benchmark-results/memory-busy-qwen-q4-rx5700-20261006/decode-start.txt)
- [RX 5700 XT single-GPU Q4_K_XL decode end marker](build-vulkan/benchmark-results/memory-busy-qwen-q4-rx5700-20261006/decode-end.txt)
- [RX 5700 XT single-GPU Q4_K_XL exit status](build-vulkan/benchmark-results/memory-busy-qwen-q4-rx5700-20261006/exit-status.txt)

## Fields to record for new runs

For each new comparison, record the exact model filename and byte size, model type, llama.cpp commit, backend and driver, GPU names and Vulkan indices, split mode and ratio, requested and loaded layer counts, prompt and decode token counts, batch and ubatch sizes, KV cache types, repetition samples and mean +/- standard deviation, per-GPU peak VRAM, telemetry sampling interval and activity values, and minimum available host memory. For decode-window `gpu_busy_percent` and `mem_busy_percent` samples, also record the marker method, sample count, median and range for each GPU. Keep activity percentages and model-footprint estimates separate from direct byte measurements.


## Vulkan cooperative-matrix experiment

### RX 9070 XT Q4_K prompt path

This used Qwen3.8-27B Q4_K_XL on Vulkan0 only, with 512 prompt tokens, 128 decode tokens, batch 512, ubatch 128, Q4_0 KV, `-ngl 999`, and three repetitions. The stock build skips the Q4_K INT8 cooperative-matrix pipeline on RDNA4. The experiment temporarily registered that existing CM1 pipeline for Q4_K only. Source and build settings were restored after the test.

| Run ID | Pipeline state | Prompt mean +/- SD (tok/s) | Decode mean +/- SD (tok/s) | Overall tok/s |
|---|---|---:|---:|---:|
| `qwen27b-q4-rx9070-control` | Stock RDNA4 gate | 206.206750 +/- 5.975521 | 8.902898 +/- 0.016547 | 37.96 |
| `qwen27b-q4-rx9070-coopmat` | Q4_K CM1 enabled | 209.432841 +/- 1.709509 | 8.873594 +/- 0.017102 | 37.94 |

| Run ID | Prompt samples (tok/s) | Decode samples (tok/s) |
|---|---|---|
| `qwen27b-q4-rx9070-control` | 207.786, 199.600, 211.234 | 8.88390, 8.91418, 8.91061 |
| `qwen27b-q4-rx9070-coopmat` | 210.068, 207.497, 210.734 | 8.85514, 8.87673, 8.88891 |

Overall tok/s uses `(512 + 128) / (512 / prompt mean tok/s + 128 / decode mean tok/s)`. The CM1 run changed prompt rate by +1.56%, decode by -0.33%, and combined rate by -0.05% versus the same-session stock control. There is no measurable overall gain in these three repetitions. The ordinary one-token decode path is separate from this CM1 matrix-matrix pipeline, so a decode improvement was not expected.

A debug trace confirmed the experiment created `matmul_q4_k_q8_1_2` and dispatched it 544 times during a 512-token prompt. The trace also shows a supported 16x16x16 signed INT8 x signed INT8 -> signed INT32 subgroup shape. The debug run is not included in the throughput table. [Dispatch evidence](build-vulkan/benchmark-results/qwen27b-q4-rx9070-coopmat-trace.txt). The existing shader uses `coopMatMulAdd` in [mul_mmq_cm1.comp](ggml/src/ggml-vulkan/vulkan-shaders/mul_mmq_cm1.comp); the Q4_K CM1 shader is generated by [vulkan-shaders-gen.cpp](ggml/src/ggml-vulkan/vulkan-shaders/vulkan-shaders-gen.cpp).

The saved earlier stock result for the same model and settings was 496.24 prompt tok/s, 12.71 decode tok/s, and 57.62 overall tok/s. The current stock control is much slower, so results from those separate sessions are not a valid kernel comparison. Repeat the stock baseline under the current system state before using the older rate as a target.

### RX 5700 XT capability check

The host Vulkan device-extension query found `VK_KHR_cooperative_matrix` on the RX 9070 XT and not on the RX 5700 XT. The RX 5700 XT was not part of the CM1 performance experiment. The Mesa feature list marks RADV cooperative matrices as available on GFX11 and newer, while Navi10 / RX 5700 XT is GFX10.1. This is not a setting that can be enabled for this card. Adding the extension would require RADV driver support and conformance testing; any implementation on GFX10 would need to lower matrix operations to its normal shader instructions because this generation has no AMD WMMA instructions. [Mesa Vulkan feature list](https://cgit.freedesktop.org/mesa/mesa/tree/docs/features.txt?id=01d17481308a74c347b68136bb25d9a8f6624ab6), [AMD WMMA guide for RDNA 3](https://gpuopen.com/learn/wmma_on_rdna3/).
