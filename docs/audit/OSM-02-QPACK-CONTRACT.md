# OSM-02 — Swiftlet qpack compatibility contract

Upstream baseline: `leonickson1/Swiftlet@909c04213c9deb369dac0679d0872512cf3ab32e`

This note records the exact upstream behavior that ORBI StreamMoE must preserve for qpack v1 interoperability.

## Container shape

```text
<model>.qpack/
  manifest.json
  config.json
  model.safetensors
  tokenizer / chat-template artifacts
  packed_experts/
    layout.json
    layer_00.bin
    layer_01.bin
    ...
```

### manifest.json

Swiftlet writes:

- `magic = "QPACK"`
- `version = 1`
- `modelName`
- `sourceCheckpoint`
- `quantBits`
- `quantGroupSize`
- `files`: relative file path -> exact byte size

Important: `quantBits` and `quantGroupSize` describe the **packed routed expert tensors**, not necessarily the dense checkpoint default. Upstream issue #30/#35 hardened mixed-precision handling specifically to prevent a container from recording the wrong expert precision.

### packed_experts/layout.json

Fields:

- `expertCount`
- `layerCount`
- `expertStride`
- `sections[]`
  - `name`
  - `dtype`
  - `shape`
  - `offset`
  - `size`
- `linearLayers[]`

The stride is produced with 16 KiB alignment by the upstream repacker.

## Expert placement

Each layer has one file:

`packed_experts/layer_XX.bin`

For expert `e`:

```text
blob_offset = e * expertStride
```

The blob contains the gate/up/down expert projection payloads and quantization side tensors at fixed section offsets.

The upstream runtime reads one expert with exactly one positional read of `expertStride` bytes.

## Batched read behavior

For several cache misses in the same layer, Swiftlet:

1. opens/reuses one layer descriptor;
2. issues one positional read per expert;
3. uses distinct offsets and destination buffers;
4. allows the reads to execute concurrently;
5. treats a short read as corruption/failure.

This behavior is the target for ORBI's later Windows/Android storage backends.

## Quantization corruption guard

For a quantized expert projection, upstream validates that the logical input dimension inferred from the packed weight agrees with the logical input dimension inferred from scales:

```text
from_weight = weight_last_dim * (32 / bits)
from_scales = scales_last_dim * group_size
require from_weight == from_scales
```

This catches a manifest that claims, for example, 8-bit expert weights when the packed expert bytes are actually 4-bit.

ORBI StreamMoE OSM-02 reproduces this guard.

## Correctness oracle

Upstream `QpackTests.repackRoundTrip` verifies that an expert read from qpack reproduces the corresponding original checkpoint expert slice byte-for-byte.

That is the semantic target for ORBI interoperability:

```text
same qpack
same (layer, expert)
same expertStride bytes
```

GPU execution is intentionally outside this gate.
