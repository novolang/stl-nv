# stl-nv

STL, for "stereolithography", is the triangle-mesh file format of 3D printing and of the mesh export of most CAD programs. A file is a list of triangles and nothing else: each triangle has a normal and three corners. This package reads both forms of the format, binary and ASCII, measures the mesh it reads, and writes binary STL.

## What it is

The format has two forms.

| Form | Layout |
| --- | --- |
| Binary | An 80-byte header, a 32-bit little-endian triangle count, then 50 bytes per triangle: twelve 32-bit little-endian floats (normal, corner A, corner B, corner C) and a 16-bit attribute word. |
| ASCII | `solid <name>`, then for each triangle `facet normal`, `outer loop`, three `vertex` lines, `endloop` and `endfacet`, then `endsolid <name>`. |

A binary file is exactly `84 + 50 * count` bytes long. A binary header may begin with the word `solid`, and several CAD programs write it so. The first word therefore does not tell the forms apart, and this package tells them by size: a file of exactly that length for the count in bytes 80 to 83 is binary, and any other file that begins with `solid` is ASCII.

STL stores no shared vertices and no units. The coordinates are plain numbers, and the unit is whatever the program that wrote the file assumed, usually millimetres.

## Install

```
novo pkg add stl-nv
```

## Example

```novo
use std.bytes
use stlmesh
use stlread

fn main() [io]
    let src = bytes.from_str("solid corner\nfacet normal 0 0 1\nouter loop\nvertex 0 0 0\nvertex 10 0 0\nvertex 0 10 5\nendloop\nendfacet\nendsolid corner\n")
    match stlread.read(src)
        Err(e) => println(e.message())
        Ok(m)  =>
            let box = stlmesh.bounds(m)
            println("${stlmesh.name_of(m)}: ${stlmesh.count(m)} triangle(s), height ${stlmesh.height_of(box)}")
            println("area ${stlmesh.surface_area(m)}")
```

The program prints `corner: 1 triangle(s), height 5` and `area 55.90169943749474`.

## What the package contains

| Module | Contents |
| --- | --- |
| `stlread` | `read` for either form, `read_binary` and `read_ascii` for a caller who knows which. |
| `stlmesh` | The mesh and its triangles; the bounding box, the surface area and the volume; the normal a triangle's corners imply. |
| `stlwrite` | `write_binary`. |
| `stlf32` | The conversion between a 32-bit float's four bytes and a `Float`. |
| `stlerr` | `StlFault`, with a message and a code. |

## The rules a user needs

1. A file's form is told by its size. A binary file with trailing bytes, or one whose count does not match its length, is a `StlSizeMismatch` fault that names both sizes. A file shorter than 84 bytes that is not ASCII is `StlTooShort`.
2. A binary corner or normal is a 32-bit float and is read exactly into a `Float`. A corner that is infinite or not a number is a fault naming the triangle. A normal that is infinite or not a number is read as the zero vector, because `normal_of` computes a normal from the corners.
3. An ASCII keyword may be in any letter case. A number is a decimal such as `1`, `-0.5`, `1.5e-3` or `1E+2`. `nan`, `inf`, a hexadecimal number and a decimal too large for a `Float` are syntax faults with the line they are on.
4. The name of an ASCII solid is the rest of the `solid` line. The name of a binary mesh is the header text up to its first NUL byte, without surrounding white space.
5. Text after the `endsolid` line is a fault. An ASCII file holds one solid.
6. The attribute word of a binary triangle is not kept and is written as zero.
7. `bounds` is the box of the corners. A mesh with no triangle has a box of zeros.
8. `signed_volume` sums the tetrahedron each triangle makes with the origin. It is positive when the corners wind counter-clockwise seen from outside, as STL requires, and negative for a mesh turned inside out. It is the volume of the solid only for a closed mesh with no self-intersection. `volume` is its magnitude.
9. `write_binary` rounds every number to the nearest 32-bit float, ties to even. A number that is infinite, not a number, or beyond 3.4028235e38 is a fault naming the triangle.
10. A header given to `write_binary` that begins with `solid`, after white space and in any letter case, is written after the words `binary `, so that other readers do not take the file for ASCII. A header longer than 80 bytes is cut at 80.

## Design

A mesh holds its coordinates as `Float`, which is 64 bits, in one list of twelve numbers per triangle. A binary file stores 32 bits, so a read is exact and a write rounds. A mesh built from an ASCII file therefore writes back to a binary file whose numbers are the 32-bit neighbours of the decimals it read.

Two choices have another reasonable answer.

- **The size decides the form.** A reader that decided by the first word would read a binary file with a `solid` header as ASCII. A reader that never looked at the word would refuse an ASCII file whose length happens to match a count, which is a coincidence of 1 in 4 billion bytes.
- **Strict length.** Some programs write extra bytes after the last triangle. This reader refuses them, because a count that disagrees with the length is the usual sign of a truncated or damaged file.

## What is not included

- **An ASCII writer.** Binary is smaller and exact to 32 bits. The shape of the ASCII form is documented above for a caller who needs it.
- **Colour.** The attribute word carries a colour in some programs and its layout is not standard.
- **Vertex welding.** The mesh stores each corner once per triangle. Finding that two corners are the same point is a decision about tolerance that the caller makes.
- **3MF, OBJ and PLY.** [zip-nv](https://novo-lang.org/packages/zip-nv) and [xml-nv](https://novo-lang.org/packages/xml-nv) read the container and the XML of a 3MF file.
- **Files.** The caller reads and writes the bytes. Every function declares no effect.

## Tests

```bash
novo test tests/stl_tests.nv         # both forms, the measurements, the writer
novo test tests/stl_edge_tests.nv    # 32-bit float conversion and every refusal
bash tests/coverage.sh               # line coverage over src/, merged across the suites
```

The binary fixture was written by Python's `struct` module, an encoder independent of this package, and the expected volume and area were computed apart from it. The 32-bit conversion is checked against the bit patterns IEEE 754 assigns: one, the largest and smallest normal numbers, the smallest subnormal, signed zeros, and the ties-to-even case. A sample of 60,000 bit patterns spread over every exponent is converted to a `Float` and back to the same bits.

## Licence

Apache-2.0. See `LICENSE`.
