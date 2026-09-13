# X98 v0.4 — protection and edited media (draft)

Review draft for protection and edited-media interoperability. Not version 1.0.
This document extends v0.3; unchanged container, directory, SURF and FLXS
fields retain their v0.3 definitions. It contains no acquisition guidance.

## Persistent write protection

The little-endian minor version at header offset 0x0A is 4.
Header flags at offset 0x10 reserve bit 3 (`0x00000008`) for WRITE_PROTECTED.
Bits 0–2 retain their existing meanings. All other bits remain reserved.

A reader/writer recognizing v0.4 must refuse simulated media WRITE and FORMAT
while WRITE_PROTECTED is set. It must not apply a separate write overlay to
circumvent protection. Filesystem access permission is independent of this flag.
Clearing the flag never grants host filesystem permission. An explicit metadata
editing operation may change protection and must rebuild affected header checks.
The bit is not valid in v0.1–v0.3; do not add it without the minor-version change.

## Standalone edited media

Successful image save produces a complete X98 with all current referenced data
inside that file. A reader or converter needs no private write-journal file.
The image retains the existing FLXS representation for edited transition data;
there is no new magnetic-data chunk type in this extension.

Partial writes must preserve transitions outside each actual write interval.
Captured observations must not simply be replaced by one reconstructed decoded
track. A physical edit is represented in every current observation; if its
position cannot be resolved without ambiguity, a writer must report the failure
rather than invent acquisition evidence. An explicit whole-surface format may
replace that surface with a generated timeline.

Shared source objects are immutable: allocate replacement objects and update
the edited SURF references. Preserve unmodified object contents. Recalculate
the container/directory sizes, offsets, digests and checksums. Unreferenced old
objects may remain; readers must follow current SURF references, not select an
older object merely because it appears first in the directory.

## Edit provenance metadata

A writer using this edit-provenance profile sets the header metadata reference to a new UTF-8
JSON META object. The original metadata is retained through
`previousMetadataObject`, a decimal string object identifier (including `"0"`
when no prior metadata exists). This avoids loss of precision for 64-bit IDs.

For full formatting:

```json
{"operation":"synthetic-full-format","captured":false,"physicalTracks":[3],"previousMetadataObject":"1"}
```

For ordered partial writes:

```json
{"operation":"synthetic-write-gates","captured":false,"previousMetadataObject":"1","writes":[{"physicalTrack":3,"observations":[{"startPs":100000000,"lengthPs":4160000000}]}]}
```

`physicalTrack` is cylinder × 2 + head. `observations` follows SURF observation
order. Values are integral picoseconds; start is relative to that observation's
index period. A gate crossing index wraps according to that period. `writes`
is ordered: a later overlapping write takes precedence. The FLXS data already
contains the final result; metadata is provenance, not a replay requirement.
`captured:false` describes the edit/generated content, not a claim that untouched
source observations were never acquired. Consumers must not relabel edits as
new measurements.

## Converter requirements

- Recognize minor 4 and preserve/report protection.
- Follow updated SURF references and decode their final FLXS objects.
- Preserve unrelated objects and provenance when producing X98 again.
- Do not require the producer's temporary overlay or journal.
- Clearly report target-format information loss, including inability to retain
  protection, timing, multiple observations, or edit provenance.

## Save safety and implementation limits

Use a verified recovery copy and the strongest commit operation provided by the
host. Verify the committed file by reopening it. This is not a universal claim
of power-loss atomicity. Keep unsaved edits on failure and reject stale-source
replacement if the target changed externally.

The current experimental writer has bounded-memory/record-count limits and may
reject ambiguous timing or unsupported source layouts. These implementation
limits are not new format restrictions. No claim of universal writable-image
coverage follows from accepting v0.4.
