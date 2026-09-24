# OSM-01 — Swiftlet portability matrix

Status: **INITIAL AUDIT COMPLETE**

Baseline: `leonickson1/Swiftlet@909c04213c9deb369dac0679d0872512cf3ab32e`

## Executive result

The memory-saving mechanism is **not inherently Apple-only**. The model/container/cache strategy can be reproduced on Windows and Android, while the Metal execution layer must be rewritten.

The recommended ORBI architecture is therefore a clean portable C++20 runtime with an initially qpack-compatible container reader, a platform I/O abstraction and a Vulkan backend shared by Windows and Android.

## Component classification

| Upstream component | Role | Portability | ORBI action |
|---|---|---:|---|
| `ArchConfig.swift` | Qwen family constants / derived memory facts | High | Re-express as portable C++ structs and tests |
| `QwenConfig.swift` | Parse model config variants | High | Reimplement parser in C++ |
| `Qpack.swift` format structs | Container manifest/layout contract | High | Preserve compatibility first |
| `QpackExpertReader` | Fixed-stride expert reads | High concept / medium code | Implement `pread` on Android/Linux and overlapped `ReadFile` on Windows |
| `Checkpoint.swift` | Safetensors + quantization metadata | High | Portable C++ parser / validation |
| `Safetensors.swift` | Tensor metadata/slices | High | Portable C++ |
| `PrefillExpertUnionPlan.swift` | Deduplicate experts in prefill | High | Port algorithm |
| `ContextWindow.swift` | Context admission/accounting | High | Port semantics |
| `ExpertCache.swift` policy | Bounded LFU + recency cache | High | Reimplement platform-neutral policy |
| `ExpertCache.swift` buffers | Metal shared buffers | Low | Replace with backend-owned buffer slots |
| `MetalShardStore.swift` | Resident weight mapping/buffers | Low | Replace with cross-platform mapped/read storage + Vulkan upload/binding |
| `MetalEngine.swift` | GPU dispatch | None | Rewrite for Vulkan |
| `Kernels.metal.txt` | Quant GEMV, norms, attention, DeltaNet, MoE kernels | Math portable / code none | Translate and validate kernel-by-kernel against CPU oracle |
| `QwenMetalModel.swift` | Metal model execution/scheduling | Medium concept / low code | Rebuild around backend abstraction |
| `QwenCPUModel.swift` | CPU reference model | High conceptual value | Use as correctness oracle; implement ORBI CPU reference |
| `SwiftletSession.swift` | Session/chat orchestration | Medium | Rebuild after core correctness |
| `SwiftletServer` | OpenAI-compatible loopback service | Protocol portable | Reimplement after runtime works |
| `StreamingInstaller.swift` | Remote/install/repack pipeline | High concept | Port later; local container validation comes first |

## Critical evidence from source

### qpack

The upstream qpack contract packs the three routed expert projections and their quantization metadata into one fixed-stride blob per expert. Layout is page aligned at 16 KiB. The expert reader is explicitly designed around one `pread` per expert.

This is the key property to preserve because it decouples total model size from resident RAM.

### Expert cache

Upstream maintains:

- bounded cache capacity;
- lazy slot allocation;
- per-key frequency history;
- recency tie-break;
- protected slots for a multi-expert batch;
- batched concurrent reads for misses;
- invalidation of newly filled slots when a batch read fails.

ORBI should preserve those semantics before attempting optimization.

### Model graph

The frozen architecture config defines Qwen3-Next-80B-A3B as:

- hidden size: 2048
- layers: 48
- full-attention interval: 4
- routed experts: 512/layer
- experts selected per token: 10
- routed expert intermediate size: 512
- context ceiling in config: 262,144
- 36 Gated DeltaNet layers + 12 gated GQA layers

The active-expert property is therefore architecture-level, not a Metal feature.

## Platform mapping

### Windows

Storage path:
`qpack layer file -> asynchronous/overlapped ReadFile -> aligned host/staging slot -> Vulkan buffer -> expert kernel`

First implementation should favor correctness and observability over direct-I/O tricks. `FILE_FLAG_NO_BUFFERING` is a later benchmark gate, not an initial requirement.

### Android

Storage path:
`qpack layer file -> pread/preadv-style NDK I/O -> host-visible/staging slot -> Vulkan -> expert kernel`

The runtime belongs in C++ through the NDK. Kotlin/Java should remain app/UI/integration glue.

## Risks discovered

1. **GPU kernel work is substantial.** The memory idea is portable, but Qwen3-Next requires correct Gated DeltaNet, gated GQA, quantized GEMV and sparse MoE execution.
2. **Storage speed is device dependent.** A 42 GB 80B container can fit on some devices but throughput may be unacceptable on slower UFS/SSD.
3. **Thermals matter on Android.** Sustained decode may be limited by mobile GPU/SoC thermal budgets.
4. **4.3 GB is not a portable promise.** That figure is an upstream Apple measurement. ORBI must measure its own resident set, Vulkan allocations, page cache and driver overhead.
5. **qpack compatibility should precede a custom ORBI container.** Forking the format too early would make debugging harder.
6. **Correctness must be fixture-driven.** Cache placement must never alter model semantics.

## ORBI decision

Proceed with `orbi-streammoe` using:

- C++20 portable core;
- qpack v1 reader compatibility;
- CPU reference backend first;
- explicit storage abstraction;
- explicit accelerator abstraction;
- Vulkan as the first GPU backend;
- Windows first for easier instrumentation;
- Android second using the same core/backend contracts.

## Exit gate

OSM-01 is complete when `orbi-streammoe` contains a matching architecture contract and portable project skeleton.
