# X98 v0.3 binary format update — review draft

Status: current review draft, 2026-09-13. Not a frozen 1.0 release.

Read this document together with [FORMAT-SPEC-v0.2.md](FORMAT-SPEC-v0.2.md).
The v0.2 container layouts remain the baseline; the additions and explicit
replay/version clarifications below take precedence. No implementation code
or media samples are distributed with this specification.

All integer fields below are little-endian unless stated otherwise.

This document defines an optional compact representation with backward-reading support
for X98 magnetic transition timing. It does not replace the v0.2 `CELL` and
`PHAS` representation. A writer may choose either representation per surface.

## Compatibility

- Container major version remains `0`; minor version is `3`.
- A v0.3 reader must continue to accept valid v0.1 and v0.2 files.
- A v0.2 reader must reject v0.3 instead of silently dropping timing data.
- `SURF`, index timing, record advance, integrity checks and immutable-object
  rules remain unchanged.
- The representation is source-format independent.

## FLXS object

`FLXS` stores the transition timelines for every revolution of one surface.
It is normally compressed as one raw-DEFLATE object (directory codec `1`).
Grouping a surface gives the compressor useful correlation across revolutions
and avoids repeated storage of a cell bitmap plus a phase byte per transition.

### Header (32 bytes)

| Offset | Size | Meaning |
|---:|---:|---|
| 0x00 | 4 | format version; `1` |
| 0x04 | 2 | cylinder |
| 0x06 | 1 | head |
| 0x07 | 1 | revolution count |
| 0x08 | 4 | transition time quantum in picoseconds |
| 0x0C | 4 | reserved; zero |
| 0x10 | 4 | revolution table offset; at least 32 |
| 0x14 | 12 | reserved; zero |

The time quantum must equal the `SURF.source_time_quantum_ps` value. A zero or
unknown source quantum cannot use `FLXS`.

### Revolution table entry (24 bytes)

| Offset | Size | Meaning |
|---:|---:|---|
| 0x00 | 4 | transition count |
| 0x04 | 4 | reserved; zero |
| 0x08 | 8 | byte offset of this revolution's stream in `FLXS` |
| 0x10 | 8 | byte length of the stream |

There is one entry for every revolution in the corresponding `SURF`, in the
same order. The first stream starts immediately after the table. Streams are contiguous,
in revolution order, and the last stream ends at the object end. Extents must
be in bounds, without overlap, gaps between streams, or trailing bytes.

### Transition stream

The stream contains exactly `transition_count` unsigned 64-bit LEB128 values (at most 10 bytes each). Truncated values,
overflow, zero deltas, and cumulative-position overflow must be rejected.
Writers emit the shortest encoding. Each
value is a strictly positive delta from the preceding transition position,
measured in `transition_time_quantum_ps` units from the start of the revolution
record. Cumulative positions must be strictly increasing and must not exceed
the revolution's record advance, allowing at most half a quantum for nearest
quantization.

When a `SURF` revolution uses `FLXS`:

- `cell_object_id` contains the shared `FLXS` object ID;
- `phase_object_id` is zero;
- every revolution of that surface refers to the same `FLXS` object.

## Fidelity rule

Conversion from `CELL` plus `PHAS` rounds each transition to the nearest
declared source-time quantum. This is lossless at the accuracy declared by the
source metadata; it must not claim finer accuracy. Nearest-quantum rounding
uses ties to even. A collision or a position rounded to zero cannot be encoded
as a positive delta and must cause FLXS selection to fail, not drop a transition. A converter must compare the
projected transition sequence before and after conversion and reject the output
if any transition count, quantum-bin position, index duration or record advance
differs.

No revolution may be removed merely because it resembles another revolution.
Deduplication is permitted only for byte-identical immutable objects.

## Writer selection

Writers should calculate both the existing v0.2 representation and `FLXS` per
surface, including directory and payload cost, and retain the smaller valid
choice. This prevents the compact extension from increasing files whose timing
distribution compresses better as `CELL` plus `PHAS`.



## Container and profile mapping

- The header minor field is 3; no new major version, global flag, codec, or
  profile number is introduced.
- Add `FLXS` to the supported directory object types. It uses the same
  stored-size, expanded-size, CRC-32C and content-tag rules as other objects.
- The FLXS directory cylinder/head match its surface; revolution is 0xFF.
- Profile 2 (the existing fidelity-cell numeric identifier) also covers FLXS
  fidelity timing. The historical profile name does not require a CELL bitmap
  when FLXS supplies the timeline. Profile 1 remains reduced-fidelity cell-only.
- A surface uses either CELL/CDLT with PHAS/PDLT, or one FLXS object for all
  its revolutions. Different surfaces in one file may choose differently.
- SURF nominal_cell_ps remains present; FLXS timestamps use the source quantum,
  not that nominal grid. SURF revolution descriptors retain index duration,
  record duration, capture sequence, flags and source revolution number.
- FLXS cylinder/head/count/quantum must match SURF. A FLXS reference requires
  minor 3, nonzero quantum, and phase_object_id zero. No FLXS delta type is
  defined. CDLT and PDLT still reference full CELL and PHAS respectively.
- Empty transition streams use count 0 and length 0. A revolution still
  retains its nonzero index and record durations.
- Writers use a table offset of 32; readers may accept a larger in-bounds
  offset. Reserved fields are zero on write.
- Existing compression codecs remain defined; raw DEFLATE is the portable
  compressed choice. A reader without an optional codec must reject required
  objects using it, not silently omit their data.

## Replay clarification (supersedes v0.2 section 12 where inconsistent)

For fidelity data, reconstruct transition times in the declared time units.
Keep index_duration_ps and flux_record_duration_ps separate. Concatenate
record timelines using the latter and expose the INDEX observation using the
former. Do not normalize measured timelines to declared RPM or stretch every
record to index duration. Converting picoseconds to a reader clock must retain
fractional remainder across boundaries; INDEX does not reset separator phase.

FLXS last-transition rounding may exceed record duration by at most half a
source quantum. This tolerance does not authorize changing the stored duration,
removing the transition, or repeatedly rounding boundary advances.

Preserve each whole revolution and its ordering. Abnormal fields, gaps, missing
or invalid CRC and marks are observations, not errors to repair during
conversion. Do not splice parts of a field from different revolutions.

## Version and validation clarification

Draft minor 3 adds a referenced representation, so an older reader is not
automatically forward-compatible. Readers must reject unsupported versions
or required representations rather than interpret FLXS as CELL or empty data.
A v0.3 reader retains v0.1/v0.2 support. For a minor-1 file with no independent
record duration, use index duration as the documented compatibility fallback;
minor-2/3 writers must store a nonzero independent record duration.

Validate all extents and arithmetic before allocation. Apply bounded object
and cache limits; exceeding supported limits produces a clear error. Do not
require the entire image in memory. A practical implementation may bound an
expanded object to 16 MiB; this is a declared implementation limit, not a new
field or an unlimited-memory promise.

Conversion validation must compare every source and expanded output surface,
revolution, transition count, quantum-bin position, index duration and record
advance, including last/first spacing across consecutive records. Reopen with
an independent reader, verify integrity, and report the worst timing error
and location. Runtime success alone is not proof of conversion fidelity.

No stable-looking revolution may be discarded. Exact immutable-object sharing,
one-level deltas and independent compression are storage optimizations only.
Choosing the smaller representation includes directory and payload costs;
no fixed compression ratio is guaranteed.

See [CONVERTER-CONFORMANCE.md](CONVERTER-CONFORMANCE.md) for the complete
validation plan and [DISCLAIMER.md](DISCLAIMER.md) for limitations.
