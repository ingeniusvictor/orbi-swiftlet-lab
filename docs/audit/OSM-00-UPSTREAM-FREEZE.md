# OSM-00 — Swiftlet upstream freeze

Status: **FROZEN / AUDIT BASELINE**

## Canonical upstream

- Repository: https://github.com/leonickson1/Swiftlet
- Branch: `main`
- Commit: `909c04213c9deb369dac0679d0872512cf3ab32e`
- Commit date: 2026-09-14T15:18:48Z
- Commit message: `Resolve the streaming installer's manifest quantization per module, as the repacker does (#35)`
- License declared by upstream: Apache-2.0

This SHA is the baseline for the first ORBI portability audit. Later upstream changes must not silently alter audit conclusions; they require an explicit rebase/re-audit gate.

## Verified upstream design facts

At the frozen revision:

1. Swiftlet targets macOS/iOS and is implemented in Swift + Metal.
2. The `.qpack` container stores dense/resident tensors separately from routed expert blobs.
3. Routed expert blobs are fixed-stride and page aligned.
4. A routed expert fetch is designed as one `pread` from its layer file.
5. The expert cache uses a bounded slot pool with LFU eviction plus recency tie-breaking.
6. Cache misses can be issued as a batch of concurrent reads.
7. The Qwen graph is represented independently enough to identify portable architecture constants.
8. The model family mixes Gated DeltaNet and gated GQA layers with sparse routed MoE.
9. Qwen3-Next-80B-A3B is configured as 48 layers, 512 routed experts/layer, top-10 experts/token, hidden size 2048.
10. Apple-specific execution is concentrated around Metal buffers, command encoders and Metal kernels.

## Provenance rule

ORBI will preserve upstream attribution for any reused or adapted Apache-2.0 source and will keep a third-party notices record. Model weights are not vendored into this repository.

## Exit gate

OSM-00 is complete when:

- this freeze record exists;
- the upstream SHA is machine-readable;
- OSM-01 classifies the implementation into portable, adaptable and rewrite-required components.
