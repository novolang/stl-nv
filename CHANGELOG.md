# Changelog

All notable changes to stl-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.1.0] — 2026-11-23

### Added

- `stlread`, a reader of binary and ASCII STL that tells the form by the size of the file.
- `stlmesh`, the mesh, its bounding box, surface area and signed volume, and the normal a triangle's corners imply.
- `stlwrite`, a writer of binary STL that rounds to 32-bit floats and refuses numbers a 32-bit float cannot hold.
- `stlf32`, the conversion between four little-endian bytes and a `Float`, and `stlerr`, the faults.
