# Topolyx Version History and Future Plans

> [!WARNING]
> The original version of this document is in Korean(`ko_KR`), and the translation may not be accurate.

## Future Plans

### After v1.0

- **Priorities:**
  - Begin development of a Rust importer
  - Split the repository (specification/Blender extension, Rust importer)
- Support for parenting
- Support for object types other than `MESH` (`CURVE`, `POINT_CLOUD`, `INSTANCE`, etc.)

---

## Change History

### v1.0.0

- Changed the format name to `Topolyx` and the file extension to `.tlyx`
- Defined a single unified file (model.tlyx), replacing the previous split file format
- Fixed the coordinate system stored in the file (Z+ up, Y+ forward, RIGHT handedness, CCW winding)
- Specified object transform application rules per semantic
- Added `NORMAL` to the semantic list

### v0.3.2

- Merged Chapters 6 and 7, and moved that content to Chapter 2 (`2. Scope of the Format`)
  - Clearly defined the supported scope of format version `1.x`

### v0.3.1

- Fixed typos

### v0.3.0

- Added `I8`, `U8` to `component_type`
- Introduced the `semantic` field for attributes (`POSITION`, `DIRECTION`, `ROTATION`, `TANGENT`, `COLOR`, `NONE`)
- Defined conversion rules for semantics that require coordinate system conversion
- Changed BOOL notation from `0b` to `0x`
- Added "Attribute Data Constraints" to the validity conditions (type/count constraints for attributes whose semantic is not NONE)

### v0.2.0

- Restructured the document: removed the "Format Rules" (former Section 3) and "Storage Structure" (former Section 5) sections → merged into per-field constraints
- Changed the `header.version` format from `x.y.z` to `x.y`
- Added explicit `domain` specification to each topology field
- Added `I32`, `BOOL` to `component_type`
- Added a constraint prohibiting NaN and Infinity for `coordinate_system`'s `meters_per_unit`
- Greatly simplified the validity conditions: consolidated fine-grained validation rules into generalized rules
    - Specified name constraints in detail: `object.name` unique within `objects`, `mesh.name` unique within `meshes`, `attribute.name` unique within a single mesh
- Specified rules prohibiting duplicate edges and self-edges in topology fields
- Removed the explicit mention of `F32`/`U32` from the supported scope → generalized to "fixed-length attributes"

### v0.1.0

> [!IMPORTANT]
>
> Since the versioning policy was not clearly defined during the v0.1 period, this version is recorded as 0.0.1 in the commit history.

- Wrote the initial draft
- Defined the split file structure (JSON + Binary)
- Defined the basic topology fields: positions, edges, corner_vertices, corner_edges, face_offsets
- Defined the coordinate system (`coordinate_system`) fields: up_axis, forward_axis, handedness, winding, meters_per_unit
- Defined the common attribute data fields: byte_offset, byte_length, component_type, component_count, element_count
- `component_type`: added support for `F32`, `U32`
- Defined the 4-byte alignment and little-endian binary rules
- Defined the rule allowing empty meshes
- Defined the validity conditions (index range, name constraints, corner-edge consistency, face offsets, data descriptors, required mesh data sizes)
- Defined the versioning policy (`x.y.z`: x = breaking change, y = feature addition, z = minor fix)
- Defined the supported scope and future extensions