# X98 Magnetic Surface Image binary format

Status: working draft 0.2, 2026-09-05  
Byte order: little-endian  
Provisional extension: `.x98`

Normative terms MUST, MUST NOT, REQUIRED, SHOULD, SHOULD NOT, and MAY are interpreted as requirement levels.

## 1. Design summary

An X98 file is an immutable object container:

```text
fixed header
object directory
object payloads (independently compressed)
optional metadata/provenance objects
```

`SURF` describes one cylinder/head and maps its ordered revolutions to magnetic objects. `CELL` is the authoritative decoded transition bitmap. `PHAS` refines each set bit in `CELL` to a sub-cell transition time. Identical objects may be referenced by several revolutions. `CDLT` and `PDLT` express one-level deltas from a full object.

The directory is authoritative for object location. Object IDs are file-local unsigned 64-bit identifiers; zero means “no object”.

## 2. Primitive types

- `u8`, `u16`, `u32`, `u64`: unsigned little-endian integers.
- `i8`, `i16`: two's-complement signed integers.
- FourCC: four ASCII bytes compared byte-for-byte.
- UUID: 16 opaque bytes in RFC 4122 network byte order.
- SHA-256: 32 bytes in normal digest byte order.
- CRC-32C: Castagnoli CRC using reflected polynomial `0x82F63B78`, initial value `0xFFFFFFFF`, reflected input/output, and final XOR `0xFFFFFFFF`. The check value for ASCII `123456789` is `0xE3069283`. It is calculated over uncompressed payload bytes.

All reserved fields MUST be written as zero and ignored by a reader unless a later compatible revision assigns them. Draft minor 2 assigns the revolution-descriptor field at `+0x28`; minor-1 readers continue to ignore it.

## 3. Fixed header — 128 bytes

| Offset | Size | Field | Meaning |
|---:|---:|---|---|
| 0x00 | 8 | magic | ASCII `X98IMG\r\n` |
| 0x08 | 2 | major | `0` for this draft |
| 0x0A | 2 | minor | `2` for writers using `flux_record_duration_ps`; readers may retain minor-1 compatibility |
| 0x0C | 4 | header_size | `128` |
| 0x10 | 4 | flags | global flags |
| 0x14 | 4 | directory_entry_size | `64` |
| 0x18 | 4 | directory_entry_count | number of entries |
| 0x1C | 4 | surface_count | number of `SURF` objects |
| 0x20 | 8 | directory_offset | absolute byte offset |
| 0x28 | 8 | declared_file_size | exact file size when finalized |
| 0x30 | 8 | created_unix_ns | informational; zero if unknown |
| 0x38 | 16 | image_uuid | immutable image identifier |
| 0x48 | 8 | metadata_object_id | `META` object or zero |
| 0x50 | 8 | provenance_object_id | `SRCE` object or zero |
| 0x58 | 4 | minimum_profile | 1 compact-cell, 2 fidelity-cell |
| 0x5C | 4 | max_delta_depth | MUST be `1` |
| 0x60 | 16 | directory_digest | first 16 bytes of SHA-256 over directory bytes |
| 0x70 | 4 | header_crc32c | CRC-32C of bytes 0x00–0x7F with this field zero |
| 0x74 | 12 | reserved | zero |

Global flag bit 0 means the file is finalized. Bit 1 means at least one compressed object exists. Bit 2 means at least one delta object exists. Other bits are zero in 0.2.

A reader MUST verify magic, supported major version, sizes, file bounds, header CRC, and directory digest before using payload offsets.

## 4. Object directory entry — 64 bytes

| Offset | Size | Field | Meaning |
|---:|---:|---|---|
| 0x00 | 8 | object_id | unique, nonzero within the file |
| 0x08 | 4 | type | `SURF`, `CELL`, `PHAS`, `CDLT`, `PDLT`, `SECT`, `META`, or `SRCE` |
| 0x0C | 2 | codec | 0 none, 1 raw DEFLATE, 2 Zstandard |
| 0x0E | 2 | flags | object flags; zero unless specified |
| 0x10 | 2 | cylinder | physical cylinder, or 0xFFFF if not applicable |
| 0x12 | 1 | head | physical head, or 0xFF |
| 0x13 | 1 | revolution | zero-based revolution, or 0xFF |
| 0x14 | 4 | logical_order | stable ordering key within the same type/surface |
| 0x18 | 8 | payload_offset | absolute stored-payload offset |
| 0x20 | 8 | stored_size | compressed/stored bytes |
| 0x28 | 8 | uncompressed_size | exact decoded payload bytes |
| 0x30 | 4 | payload_crc32c | CRC-32C of uncompressed payload |
| 0x34 | 4 | reserved0 | zero |
| 0x38 | 8 | content_tag | first 8 bytes of SHA-256 over uncompressed payload |

Directory entries MUST be sorted by `object_id`. Payload extents MUST NOT overlap the header, directory, or each other. Duplicate object content SHOULD be represented by one object referenced multiple times, not duplicate entries.

Unknown object types MAY be skipped when they are not referenced by a REQUIRED object. Unknown codecs or unknown referenced object types MUST cause a clean unsupported-format error.

## 5. `SURF` payload

One `SURF` represents a physical cylinder/head. Fixed prefix:

| Offset | Size | Field |
|---:|---:|---|
| 0x00 | 2 | cylinder |
| 0x02 | 1 | head |
| 0x03 | 1 | revolution_count (1–32) |
| 0x04 | 4 | flags |
| 0x08 | 4 | nominal_cell_ps |
| 0x0C | 4 | source_time_quantum_ps; zero if unknown |
| 0x10 | 8 | declared_rpm_milli; zero if absent/unknown |
| 0x18 | 8 | sector_cache_object_id; zero if absent |
| 0x20 | 32 | reserved |

The prefix is followed immediately by `revolution_count` descriptors of 48 bytes:

| Offset | Size | Field |
|---:|---:|---|
| +0x00 | 8 | index_duration_ps |
| +0x08 | 8 | cell_object_id (`CELL` or `CDLT`) |
| +0x10 | 8 | phase_object_id (`PHAS` or `PDLT`); zero only for compact-cell |
| +0x18 | 8 | capture_sequence; monotonically increasing when known |
| +0x20 | 4 | flags |
| +0x24 | 4 | source_revolution_number |
| +0x28 | 8 | flux_record_duration_ps; complete captured flux-record advance, separately from INDEX |

`nominal_cell_ps` is the converter's magnetic-cell grid, not a sector data rate declaration. It MUST be constant within a surface. A converter SHOULD choose a grid that represents the source without changing clock/data patterns. For PC-98 2HD MFM, 1,000,000 ps is a typical magnetic-cell period, but it is not a format constant.

`declared_rpm_milli` is source metadata. Actual rotation timing always comes from each `index_duration_ps`.

`index_duration_ps` locates the INDEX observation. `flux_record_duration_ps` is the advance of the complete source revolution record and MUST include the interval that crosses or follows that INDEX observation. `CELL`/`PHAS` contain every transition from the record. Replay concatenates record timelines using `flux_record_duration_ps` while presenting INDEX from `index_duration_ps`; INDEX does not reset data-separator phase. Minor-2 writers MUST write a nonzero record duration. A minor-1 reader may treat the record duration as `index_duration_ps` for compatibility with old files.

## 6. `CELL` payload

| Offset | Size | Field |
|---:|---:|---|
| 0x00 | 8 | bit_count |
| 0x08 | 8 | transition_count |
| 0x10 | 4 | packing | MUST be 1 (MSB-first) |
| 0x14 | 4 | flags |
| 0x18 | 8 | reserved |
| 0x20 | ceil(bit_count/8) | bitmap |

Bit zero is the first nominal magnetic cell at the index boundary. In each byte, the earliest cell is bit 7. A set bit means one magnetic transition is assigned to that cell; a clear bit means none.

The unused low bits of the final byte MUST be zero. `transition_count` MUST equal the population count of the bitmap. `bit_count * nominal_cell_ps` should approximate `flux_record_duration_ps`; the exact record duration remains authoritative. When a record has transitions, `bit_count` MUST be `floor(flux_record_duration_ps / nominal_cell_ps) + 1`, so a transition exactly at the record endpoint has a representable cell. Such an endpoint transition is retained; it is not clamped or discarded.

Version 0.2 permits at most one transition per nominal cell. A converter encountering two transitions that cannot be represented separately MUST select a finer nominal cell grid or fail; it MUST NOT discard a transition.

## 7. `PHAS` payload

`PHAS` has exactly one signed residual for each set bit of the corresponding expanded `CELL`, in ascending cell order.

| Offset | Size | Field |
|---:|---:|---|
| 0x00 | 8 | transition_count |
| 0x08 | 4 | units_per_cell | MUST be 256 |
| 0x0C | 4 | flags |
| 0x10 | 16 | reserved |
| 0x20 | transition_count | `i8` residual array |

For a transition assigned to cell `n`, its reconstructed index-relative time is:

```text
t_ps = ((n * 256 + 128 + residual_i8) * nominal_cell_ps) / 256
```

The nominal position is the cell center. Residual range -128…127 covers the complete cell without ambiguity. Ordering of reconstructed transitions MUST be strictly increasing. If it is not, the file is corrupt.

At one-microsecond cells this gives approximately 3.90625 ns phase resolution. The converter still records the actual source quantum; precision metadata must not imply accuracy beyond the source.

## 8. Delta objects

Delta objects reduce repeated revolutions without changing their expansion.

### 8.1 `CDLT`

Prefix:

| Offset | Size | Field |
|---:|---:|---|
| 0x00 | 8 | base_cell_object_id; MUST reference full `CELL` |
| 0x08 | 8 | expanded_bit_count |
| 0x10 | 4 | patch_count |
| 0x14 | 4 | flags |
| 0x18 | 8 | reserved |

Each patch is:

```text
u64 start_bit
u32 bit_length
u32 stored_byte_count = ceil(bit_length/8)
u8  xor_bitmap[stored_byte_count]  // MSB-first within patch
```

Patches MUST be sorted, non-overlapping, in bounds, and have zero unused tail bits. The expanded bitmap is first sized to `expanded_bit_count`: a longer base is truncated and a shorter base is zero-extended. Any unused low bits in the resized final byte MUST be zero before patches are applied. Expansion then XORs patches. The expanded population count is used to validate the matching `PHAS`/`PDLT`.

### 8.2 `PDLT`

Prefix:

| Offset | Size | Field |
|---:|---:|---|
| 0x00 | 8 | base_phase_object_id; MUST reference full `PHAS` |
| 0x08 | 8 | expanded_transition_count |
| 0x10 | 4 | patch_count |
| 0x14 | 4 | flags |
| 0x18 | 8 | reserved |

Each patch is `u64 start_transition`, `u32 count`, `u32 reserved`, followed by `count` replacement `i8` residuals. Patches are sorted and non-overlapping. A `PDLT` is legal only when the expanded `CELL` has the same transition count and transition ordinal mapping as the base `CELL`. If cell insertions/deletions change ordinals, the writer MUST emit a full `PHAS` object.

Delta depth MUST be one. Cycles, self-reference, or delta-to-delta references are corrupt.

## 9. Optional `SECT` cache

`SECT` is a performance/inspection cache containing detected marks, C/H/R/N, mark type, CRC bytes/result, data extent, and cell positions. Its detailed payload is intentionally deferred from draft 0.2.

Until defined, writers MUST omit `SECT`. Readers MUST derive FDC-visible content from the magnetic surface. This prevents an early sector schema from becoming an accidental limitation on protected layouts.

## 10. `META` and `SRCE`

Both payloads are UTF-8 JSON without BOM.

`META` is descriptive and MAY contain format-neutral annotations, tool versions, comments, media diameter, drive model, and user labels.

`SRCE` provenance SHOULD contain:

```json
{
  "sourceFormat": "source format identifier",
  "sourceSha256": "64 lowercase hex digits",
  "sourceSize": 0,
  "sourceDeclaredRpm": 360,
  "sourceTimeQuantumPs": 25000,
  "converter": {
    "name": "implementation name",
    "version": "version",
    "algorithm": "stable algorithm identifier"
  },
  "selectedRevolutions": "all",
  "profile": "fidelity-cell"
}
```

Declared and measured RPM/timing MUST remain separate fields. Metadata is not authoritative for replay.

## 11. Compression and integrity

Codec 1 is an RFC 1951 raw DEFLATE stream with no zlib or gzip wrapper. Codec 2 is one independent Zstandard frame. The dictionary feature is forbidden in 0.2.

Readers MUST enforce the directory's uncompressed size before allocation, stream decompression into a bounded buffer, reject trailing compressed data, and verify CRC-32C. A CRC failure is a corrupt-image error, not a simulated disk CRC error.

FDC CRC errors are magnetic content stored in `CELL`/`PHAS`; container CRC errors mean the file itself is damaged.

## 12. Replay model

For compact-cell replay, a reader places every set-bit transition at the cell center and scales the final continuous timeline to the recorded `index_duration_ps`.

For fidelity-cell replay, it applies `PHAS`, then scales only for exact rational conversion to the recorded index duration. A reader MUST retain the last transition state across the index boundary when supplying consecutive revolutions to a VFO. The index pulse identifies the boundary but MUST NOT force a data-separator reset.

Revolution selection follows elapsed spindle revolutions modulo the available captured count. A reader MUST return captured observations in stored order and MUST NOT fabricate alternating bits.

## 13. Error handling

A reader MUST reject:

- duplicate/zero object IDs;
- out-of-range or overlapping extents;
- unsupported required profile or codec;
- missing referenced objects or wrong referenced types;
- invalid CRC/digest;
- decompressed-size mismatch;
- invalid surface/revolution counts;
- transition-count mismatch;
- non-monotonic reconstructed transition times;
- illegal or recursive deltas;
- resource limits exceeded.

Errors must identify the object ID and reason. A corrupt track must not silently become an empty/unformatted track.

## 14. Versioning

Major-version changes may be incompatible. Minor-version changes may add unreferenced optional object types or assign reserved flags without changing existing object semantics.

Draft 0.x files are experimental. Version 1.0 will freeze magic/version policy, the required profiles, limits, and test vectors.
