# X98 requirements — working draft 0.2

## 1. Purpose

X98 stores a decoded magnetic surface for floppy-disk emulation. It targets the space between sector-oriented images and full flux-capture images:

- retain a caller-selected number of physical revolutions;
- retain nonstandard layout, missing marks, invalid CRC, deleted data marks, duplicated IDs, long/short gaps, weak or unstable areas, and revolution-to-revolution differences;
- retain enough transition timing for an emulator's VFO/data separator to observe marginal media behavior;
- avoid repeatedly decoding source flux on every FDC access;
- allow track/revolution random access without loading the whole image;
- make normal multi-revolution areas small through content sharing, delta encoding, and per-object compression.

X98 does not compete with or supersede either class of source format. It is an
operational format for cases where a compact decoded image lacks required
magnetic observations, but repeatedly storing and reading a complete flux
capture imposes excessive storage and I/O cost. The source capture remains the
preservation master.

X98 is not a software-specific compatibility database and must never contain filename-, application-, sector-, or fixed-byte-specific behavior.

X98 is an emulator-oriented derivative, not a replacement for archival source captures. Original source images should be retained. Conversion to X98 is intentionally irreversible because drive-level flux intervals are mapped to a documented magnetic-cell/timing model; X98 fidelity means equivalent reconstructed transitions and FDC observations within stated tolerances, not byte-for-byte regeneration of the source image.

## 2. Authoritative data model

The authoritative object is the rotating magnetic surface, not a list of sectors.

For every captured cylinder/head/revolution, the format stores:

1. the index-to-index duration;
2. an index-aligned magnetic-cell transition bitmap (`CELL`);
3. for the fidelity profile, a transition phase stream (`PHAS`) that refines each transition position within its nominal cell;
4. provenance and conversion parameters sufficient to reproduce the conversion.

Sector/mark tables may be stored only as derived caches. A reader must be able to ignore or rebuild them. If a cache conflicts with `CELL`/`PHAS`, the magnetic surface wins.

## 3. Compatibility requirements

### 3.1 Must preserve

- 1–32 revolutions per track; different tracks may have different counts.
- Individually measured index duration for every revolution.
- Index-relative phase and continuous ordering of adjacent revolutions.
- FM, MFM, mixed or undecodable regions without requiring a declared encoding.
- Exact magnetic clock/data-bit pattern after source-to-cell decoding.
- Revolution-specific variants; no synthetic random bit flipping.
- Missing/extra/duplicated sector IDs and arbitrary C/H/R/N values.
- Correct, incorrect, or absent ID/data CRC bytes as recorded.
- Deleted and ordinary data address marks as recorded.
- Data outside recognized sectors, including gaps and malformed marks.
- Unformatted/no-transition and partially readable tracks.
- Source-declared geometry/RPM and measured timing as separate metadata.

### 3.2 Must not do

- Do not infer actual RPM only from a source header declaration.
- Do not normalize track length, gaps, sync counts, marks, CRC, or sector order.
- Do not merge revolutions into a fabricated “best” revolution.
- Do not replace unstable source observations with PRNG output.
- Do not require all tracks to use one cell rate, one RPM, or one encoding.
- Do not require the complete file in memory.

### 3.3 Profiles

`compact-cell` stores `CELL` plus index timing. It preserves decoded magnetic patterns and multi-revolution variation but not sub-cell timing. It is suitable only when the user accepts reduced marginal-timing fidelity.

`fidelity-cell` stores `CELL`, `PHAS`, and index timing for every present revolution. This is the required profile when the source supplies sub-cell timing and is the default for flux-image conversion.

A file declares its minimum profile. Readers must reject a profile they cannot honor; silently dropping `PHAS` is forbidden.

## 4. Capacity and access requirements

- A reader opens only the fixed header and directory initially.
- A track/revolution must be independently readable and independently verifiable.
- Compression blocks must not span tracks.
- The writer must deduplicate byte-identical objects and should delta-code later revolutions against a base revolution when smaller.
- A reader may use a bounded LRU cache; the format must not require permanent expansion of all tracks.
- All offsets and sizes are 64-bit.
- Required implementation limits must include at least 168 cylinder/head slots, 32 revolutions per slot, 16 MiB uncompressed per object, and 16 GiB per image.
- Implementations must reject integer overflow, overlapping stored extents, out-of-file offsets, recursive delta chains, and decompression bombs.

## 5. Compression requirements

- Codec 0 (`none`) and codec 1 (`raw DEFLATE`) are mandatory for readers.
- Codec 2 (`Zstandard`) is optional and must be explicitly identified.
- CRC-32C is calculated over the uncompressed object.
- Compression changes storage only; it must not change decoded cells or timing.
- Delta depth is limited to one: a delta object may reference a full object but never another delta.

## 6. Conversion requirements

### 6.1 Flux image to X98

- Read all selected source revolutions and preserve their order.
- Derive actual revolution duration from each index record.
- Record the source time resolution and complete source SHA-256 in provenance metadata.
- Decode flux transitions to nominal magnetic cells with a documented, versioned conversion algorithm.
- In `fidelity-cell`, store residual transition phase fine enough that reconstructed transition-time error is no greater than half the source time quantum, or explicitly report the larger X98 quantization error.
- A conversion that omits, merges, repairs, or fabricates a revolution must fail unless the user explicitly requested a lossy profile.

### 6.2 Sources without sub-cell timing

- Import adapters may create `compact-cell` when the source has no sub-cell timing.
- Unknown source timing must remain unknown; do not invent measured RPM or analog precision.
- Source-specific status may be stored in metadata, but generic readers must not need it to replay the magnetic surface.

## 7. Write support boundary

Version 0.2 is a read-only base-image specification. Emulator writes belong in a separate overlay file. The base image remains immutable.

The overlay design must identify the base image by whole-file SHA-256 or immutable image UUID plus directory digest, and must replace complete affected track/revolution objects. Sector-only patches are insufficient for arbitrary magnetic layouts.

## 8. Success criteria

The format is ready for 1.0 only when:

- two independent readers parse the same corpus identically;
- source-image to X98 conversion passes the conformance checks;
- truncated, corrupt, overlapping, oversized, and cyclic-reference files are rejected safely;
- random track access works without whole-image loading;
- the accepted format-neutral regression corpus behaves the same with `fidelity-cell` X98 as with its source images;
- measured size and access latency are reported for ordinary and high-revolution-count media without claiming savings before measurement.
