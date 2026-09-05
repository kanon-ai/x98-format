# X98 Magnetic Surface Image

Working documents for an original, multi-revolution, decoded magnetic-surface image format.

X98 is not intended to compete with or displace existing disk-image formats.
It addresses a practical gap: lightweight decoded images may omit magnetic
observations needed for faithful emulation, while complete flux captures can
be too large and I/O-intensive for routine use. X98 retains the observations
needed by an emulator in a random-access, independently compressed form while
the original capture remains the archival source.

Publication of this specification is for technical documentation,
preservation, research, and interoperability. It does not encourage or promote
unauthorized copying or distribution of software or media, nor the use of any
particular emulator. Users are responsible for ensuring that every source
image and use is lawful and properly authorized.

- [REQUIREMENTS.md](REQUIREMENTS.md): product and compatibility requirements
- [FORMAT-SPEC-v0.2.md](FORMAT-SPEC-v0.2.md): normative binary format draft
- [CONVERTER-CONFORMANCE.md](CONVERTER-CONFORMANCE.md): converter and reader validation plan
- [DISCLAIMER.md](DISCLAIMER.md): warranty, preservation, interoperability, and third-party-rights notice
- [LICENSE.md](LICENSE.md): CC BY 4.0 license notice and canonical terms

`X98` and the `.x98` extension are provisional names. The format is intentionally specified independently of every source-image format and implementation. Source formats are handled only by import/export adapters; they are not structural templates or normative authorities.

Version 0.2 is a review draft. Writers must not label files as stable release files until the format is frozen as version 1.0.

## License

The X98 format specification and accompanying documentation are licensed under
the [Creative Commons Attribution 4.0 International License](LICENSE.md)
(CC BY 4.0).

Implementations of the X98 format may be distributed under any license. The
documentation license does not require an implementation to use the same
license.

Copyright (c) 2026 kanon-ai contributors.

SPDX-License-Identifier: CC-BY-4.0

## Important notice

X98 is experimental. Keep original source images and physical media whenever
possible. See [DISCLAIMER.md](DISCLAIMER.md) before using X98 for preservation,
conversion, emulation, or physical-media writing.
