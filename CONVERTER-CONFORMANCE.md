# X98 converter and reader conformance plan

## 1. Required deliverables

The Floppy State Viewer project should implement these components independently:

1. One or more source-image input adapters.
2. X98 writer.
3. X98 reader that does not reuse writer structures as its parser.
4. X98 inspector showing header, directory, surfaces, revolution count, index duration, declared versus measured RPM, object sharing/deltas, compression ratio, and integrity results.
5. Comparison command for a source image versus generated X98.

The emulator reader should be a second implementation and consume the generic X98 surface API. It must not call source-format-specific code paths after open.

## 2. Conversion stages

```text
source header/index observations
  -> validated flux intervals per track/revolution
  -> index-aligned transition timeline
  -> nominal CELL grid + PHAS residuals
  -> content deduplication / one-level deltas
  -> per-object compression
  -> directory, digests, finalized header
```

Every stage reports counts and failures. There is no “best revolution” stage.

## 3. Static format tests

Test at minimum:

- smallest one-surface/one-revolution file;
- 168 surfaces × 32 revolutions directory boundary;
- empty/no-transition revolution;
- transitions in first and last cell;
- full residual range -128 and 127;
- identical object deduplication;
- valid CDLT/PDLT expansion;
- cell-count-changing delta requiring full PHAS;
- codec none and raw DEFLATE;
- unknown optional object skipped;
- unsupported required codec/profile rejected;
- all integer-overflow and extent-overlap cases;
- truncated header/directory/payload;
- bad header CRC, directory digest, payload CRC;
- duplicate IDs, missing references, wrong types, self/cyclic/depth-two deltas;
- decompression overrun and trailing compressed bytes.

## 4. Source fidelity comparison

For every source track/revolution:

- source and X98 revolution counts and order match;
- source index duration and X98 `index_duration_ps` differ by no more than declared conversion rounding;
- transition count matches after the documented source decoder;
- every reconstructed transition differs from the source timeline by no more than `max(source_quantum_ps / 2, nominal_cell_ps / 512)` plus integer rounding;
- index-relative ordering and cross-index last/first transition intervals match within the same tolerance;
- the expanded X98 object is identical before and after compression/delta decoding.

The comparison report must list the worst timing error and its cylinder/head/revolution/transition index. A pass/fail summary without maxima is insufficient.

## 5. Magnetic/FDC semantic tests

Use synthetic tracks, not software-specific exceptions, for:

- standard FM and MFM;
- mixed FM/MFM areas;
- invalid ID CRC and invalid data CRC;
- deleted data marks;
- duplicate and nonmonotonic C/H/R/N;
- missing ID/data marks;
- long gap and no-transition regions;
- deliberately marginal preambles;
- one-bit and multi-bit revolution variation;
- a sector present in only later revolutions;
- 3-, 5-, 16-, and 32-revolution wraparound;
- READ DATA, READ DELETED DATA, READ ID, READ TRACK, multi-sector continuation, terminal count, DMA/IRQ timing, index wait, and missing-ID timeout.

For every command, compare the ordered bytes, ST0/ST1/ST2, CHRN result, DRQ/TC completion boundary, selected revolution, index-relative start phase, and elapsed emulated clocks between the source-image reference path and fidelity-cell X98.

## 6. Real-image acceptance corpus

Keep original source images external and read-only. Convert test copies and record SHA-256. The initial corpus should include format-neutral samples covering:

- ordinary media with a small revolution count;
- high-revolution-count media;
- weak or unstable regions;
- invalid or missing CRC and address marks;
- duplicated or nonmonotonic sector identifiers;
- runtime media exchange;
- both standard and nonstandard track layouts.

This corpus is integration evidence only. It must not influence conversion rules or binary semantics.

## 7. Performance and size measurements

Measure rather than assume:

- source-image size;
- X98 uncompressed object total;
- X98 stored size by object type;
- deduplication savings;
- delta savings;
- compression savings;
- cold open time;
- first-track latency;
- cached-track latency;
- peak reader memory;
- sustained FDD-heavy audio/frame behavior in the emulator.

Test `none`, raw DEFLATE, and optional Zstandard separately. Do not use whole-file ZIP as the primary result because it prevents independent random track access.

Suggested targets, not conformance requirements:

- open without reading payload objects;
- peak cache configurable below 32 MiB;
- ordinary track/revolution load below one video frame on the target PC;
- high-revolution-count X98 meaningfully smaller than its source image while retaining fidelity-cell equivalence.

## 8. Reproducibility record

Each conversion report records:

- source path label, size, SHA-256, and stable-before/after check;
- converter binary SHA-256 and version;
- complete options and profile;
- number of surfaces and per-track revolution counts;
- source declared RPM and measured per-revolution RPM separately;
- output size and SHA-256;
- integrity verification after reopening the completed X98 file;
- fidelity maxima and every failure.

## 9. Release gate

Do not freeze version 1.0 until:

- the viewer's writer and independent reader pass all tests;
- the emulator's reader passes the same binary vectors;
- source-reference and X98 FDC traces match for the synthetic corpus;
- the accepted real-image corpus shows no regression;
- malformed-input fuzzing runs under memory and time limits;
- format documents and test vectors can implement a reader without consulting any source-image implementation.
