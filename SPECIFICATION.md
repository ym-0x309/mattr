# Topolyx Format Specification (v1.0.0)

> [!IMPORTANT]
> - Format name: **Topolyx** (**Topo**logy, **Poly**gon, e**X**change)
> - Full name: Mesh Attribute & Topology Interchange Format
> - File: `model.tlyx` (single unified file)
> - Magic bytes: `54 4C 59 58` (ASCII `TLYX`)

> [!WARNING]
> The original version of this document is in Korean(`ko_KR`), and the translation may not be accurate.

## 1. Purpose of the Format

This is a format for bringing meshes created in Blender into a custom engine or tool while preserving their attribute data.

Rather than transmitting only the results needed for rendering, as OBJ, glTF, or FBX typically do, the core goal of this format is to preserve, as losslessly as possible, attributes of the following domains that exist on a Blender `Mesh` data block:

```text
POINT domain attribute
EDGE domain attribute
FACE domain attribute
CORNER domain attribute
```

This format is not intended to preserve a complete Blender scene, modifier stack, material graph, or animation data.

---

## 2. Scope of the Format

This format stores the following information:

- Object Transform
- Vertex / Edge / Face / Corner Topology
- Mesh Attributes
- Coordinate System

The following information is **not included** in the scope of Topolyx up through `1.x` (the scope of `2.x` is undecided):

- Material definitions
- Textures and image files
- Shaders (WGSL, GLSL, etc.)
- Animation
- Armature / Bone
- Modifier Stack
- Scene composition
- Camera
- Light
- Render Settings

Textures, material graphs, and other data not directly related to geometry and attributes are not included in the scope of this format. Where needed, such data should be linked from an external system using attributes such as `MATERIAL_INDEX`.

---

## 3. Terminology

### Vertex

A spatial point of the mesh. Corresponds to one element of the `positions` array.

### Edge

An element that connects two vertices. Each edge stores two vertex indices.

### Face

A polygon element of the mesh. It does not directly own vertex or edge arrays; instead, it references a contiguous range of the corner array via `face_offsets`.

### Face Corner

The position at which a specific vertex is used within a specific face. Corners are not shared between faces — even when the same vertex is used by multiple faces, each face has its own separate corner.

### Domain

The kind of mesh element an attribute corresponds to.

- `POINT`: vertex
- `EDGE`: edge
- `FACE`: face
- `CORNER`: face corner

### Element

A single logical item in an array.

Examples:

- one position
- one edge
- one UV coordinate

### Component

A scalar value that makes up one element.

### Byte offset

The byte distance from the start of the BIN chunk's `chunk_data` to the start of a data array.

### Topology

The required mesh data representing the connectivity between vertices, edges, faces, and corners.

### Attribute

An additional data array corresponding to each element of a specific domain.

---

## 4. File Structure

> [!IMPORTANT]
>
> ```text
> test_mesh.tlyx
> ```
>
> Starting from `1.x.y`, the unified file format (`.tlyx`) replaces the earlier split file format (`.json` + `.bin`).

### Container Structure

> [!TIP]
>
> This structure is modeled after the glTF format's (`.glb`) storage layout.

1. Header: 12 bytes

    | offset | name | format | description |
    |-|-|-|-|
    | 0 | magic | 4-byte ASCII characters | `TLYX` (`54 4C 59 58`) |
    | 4 | version | U32 | major version integer. Currently `1` |
    | 8 | total_length | U32 | total byte length of the file (including header and all chunks) |

2. Chunk 0: JSON

    | name | format | description |
    |-|-|-|
    | chunk_length | U32 | length of `chunk_data`, including padding; a multiple of 4 |
    | chunk_type | 4-byte ASCII characters | `JSON` |
    | chunk_data | UTF-8 JSON | JSON padded at the end with `0x20` (space) to a 4-byte alignment |

3. Chunk 1: BIN

    | name | format | description |
    |-|-|-|
    | chunk_length | U32 | length of `chunk_data`, including padding; a multiple of 4 |
    | chunk_type | 4-byte ASCII characters | `BIN\0` |
    | chunk_data | binary | binary padded at the end with `0x00` to a 4-byte alignment |

The file contains exactly these two chunks, and their order is fixed as JSON → BIN.

### JSON

> [!IMPORTANT]
>
> The JSON stores metadata and is encoded in UTF-8.
>
> ```text
> format version
> object name
> mesh name
> coordinate system
> element counts
> byte_offset for each data array
> attribute name / domain / type
> ```

#### Storage Method and Field-Level Constraints

##### Basic File Information

- `header`: format information

    - `format`: `Topolyx`
    - `version`: `x.y`

##### Coordinate System Information

- `coordinate_system`: defines the world coordinate system. `up_axis`, `forward_axis`, `handedness`, and `winding` are fixed values that directly follow Blender's world coordinate system, and they are identical across every Topolyx file.

    - `up_axis`: always `+Z`. A different value makes the file invalid.
    - `forward_axis`: always `+Y`. A different value makes the file invalid.
    - `handedness`: always `RIGHT`. A different value makes the file invalid.
    - `winding`: always `CCW`. A different value makes the file invalid.
    - `meters_per_unit`: the length of 1 meter in this coordinate system. This is the only value that may differ from file to file.
        - Must be a valid number greater than 0 (NaN and Infinity are not allowed).

> [!NOTE]
>
> Because `up_axis`, `forward_axis`, `handedness`, and `winding` are fixed across every Topolyx file, no axis/handedness conversion between files is necessary.
> However, `meters_per_unit` may differ between files, and converting values between different unit scales is not guaranteed by this format — it is the responsibility of the writer/reader implementation.

##### Object Information

- object: contents of the `objects` array

    - `name`: the object's name.
    - `type`: currently only `MESH` is supported.
    - `index`: the index into the array assigned per `type`. For example, when `type` is `MESH`, `index` refers to the index into the `meshes` array.
    - `transform`: the 4x4 transform matrix applied to the source mesh, in column-major order.

        - Vectors are treated as column vectors, and the transform is applied in the order `M × v`.
        - Each mesh's local axes are converted to world space via `transform`.
        - The exact way `transform` is applied to the topology (`positions`) and to each attribute semantic follows the "Object Transform Application Rules" section below.

Parenting is currently not supported.

##### Attribute Information

- attribute data: fields shared in common between `topology` data and `attributes`

    - `byte_offset`: the byte offset of the data array, measured from the start of the BIN chunk's `chunk_data`.
        - Must satisfy the following condition (4-byte alignment):
            ```text
            byte_offset % 4 == 0
            ```
    - `byte_length`: the total byte length occupied by the data array.
        - Calculated as follows:
            ```text
            byte_length
            = byte size of component_type
            * component_count
            * element_count
            ```
        - Must satisfy the following condition, and the calculation must not overflow an integer:
            ```text
            byte_offset + byte_length <= chunk_length of the BIN chunk
            ```
    - `component_type`: the binary scalar type used to store each component. The currently supported components are as follows.

        | Component type |    size | meaning |
        | --------------- | ------: | ------- |
        | `F32`           | 4 bytes | IEEE 754 32-bit floating point |
        | `I32`           | 4 bytes | signed 32-bit integer |
        | `I8`            | 1 byte  | signed 8-bit integer |
        | `U32`           | 4 bytes | unsigned 32-bit integer |
        | `U8`            | 1 byte  | unsigned 8-bit integer |
        | `BOOL`          | 1 byte  | 1-byte boolean value. `0x00` is `false`, `0x01` is `true`; any other value is invalid. |

    - `component_count`: the number of components that make up one element. Must be at least 1.
    - `element_count`: the total number of elements stored in the array. Except for `face_offsets` (described later), this must match the `element_count` of the corresponding domain.

##### Mesh Information

> [!TIP]
>
> Based on the array structure described in the [official Blender Mesh documentation](https://developer.blender.org/docs/features/objects/mesh/mesh/).

 - mesh: contents of the `meshes` array

    - `name`: the mesh's name.

    - `element_counts`

        - `vertices`: vertex count
        - `edges`: edge count
        - `faces`: face count
        - `corners`: face corner count

    - `topology`: required mesh data.

        - `positions`: attribute data

            - the local-space position of each vertex
            - `component_type`: `F32`
            - `component_count`: 3
            - `domain`: `POINT`
            - The conversion rule applied when the object's `transform` is applied follows the "Object Transform Application Rules" section below.

        - `edges`: attribute data

            - the indices of the two vertices that make up each edge
            - `component_type`: `U32`
            - `component_count`: 2
            - `domain`: `EDGE`
            - The order of the two vertex indices does not imply a direction for the edge.
            - Duplicate edges and self-edges are not allowed.

        - `corner_vertices`: attribute data

            - the index of the vertex referenced by each face corner
            - `component_type`: `U32`
            - `component_count`: 1
            - `domain`: `CORNER`
            - Corners belonging to the same face must be stored in the order in which they traverse the polygon, and the traversal direction follows the `winding` value in the JSON file.

        - `corner_edges`: attribute data

            - the index of the edge that leads from each face corner toward the next corner of the same face
            - `component_type`: `U32`
            - `component_count`: 1
            - `domain`: `CORNER`

        - `face_offsets`: attribute data

            - the starting index, within the corner array, of the range of corners used by each face
            - `component_type`: `U32`
            - `component_count`: 1
            - `domain`: `FACE`
            - **`element_count` must equal `faces + 1`.**
            - The range of corners used by face `i` is:

                ```text
                [face_offsets[i], face_offsets[i + 1])
                ```
            - The first value of `face_offsets` must be 0, and the last value must equal the total number of corners.
            - Each face must have at least 3 corners, so the following condition must hold:
                ```text
                face_offsets[i + 1] - face_offsets[i] >= 3
                ```
            - The triangulation and rendering of non-planar n-gons is outside the scope of this format; handling this is the responsibility of the reader/engine implementation.

    - attribute: general attributes stored in the `attributes` array

        - `name`: the attribute's name.
        - `domain`: one of `POINT`, `EDGE`, `FACE`, `CORNER`.
        - `semantic`: assigned to attributes that require a standardized interpretation rule. The currently supported values are as follows.

            |semantic|component type|component count|meaning|
            |-|-|-|-|
            |`POSITION`|`F32`|3|3D position information.|
            |`DIRECTION`|`F32`|3|3D direction information.|
            |`NORMAL`|`F32`|3|3D normal information. Its value format is the same as `DIRECTION`, but its conversion rule differs — e.g. the inverse transpose matrix is used when applying the object `transform`. See the "Object Transform Application Rules" section for details.|
            |`ROTATION`|`F32`|4|Quaternion rotation information, stored as (x, y, z, w).|
            |`TANGENT`|`F32`|4|Tangent information, stored as tangent (x, y, z) plus handedness (w). The bitangent is reconstructed as `cross(normal, tangent.xyz) * tangent.w`, where `normal` refers to the `NORMAL` semantic attribute present in the same mesh.|
            |`COLOR`|`F32`, `U8`|4|RGBA color|
            |`NONE`|Any|Any|any other attribute|

        - `data`: attribute data

###### Object Transform Application Rules

When applying an object's `transform` (local → world, 4x4) to a mesh's topology and attribute values, follow the rules below based on semantic.

Let `L` denote the linear (3x3) part of `transform`, and `t` denote the translation part.

| target | how it is applied |
|-|-|
| `positions` (topology), `POSITION` | `L · p + t` |
| `DIRECTION` | `L · v`. Translation is not applied, and the result is not re-normalized. |
| `NORMAL` | `transpose(inverse(L)) · n`. Translation is not applied; the result is re-normalized to a unit vector afterward. |
| `TANGENT` | The `xyz` part is applied the same way as `DIRECTION` and then re-normalized. If `det(L) < 0`, the sign of the `w` component is flipped. |
| `ROTATION` | `q' = quat(R) ⊗ q`. `R` is the pure rotation component extracted from `L`. |
| `COLOR`, `NONE` | Not transformed. |

- If `L` is a singular matrix (`det(L) = 0`), the transform above is undefined, which is a violation of the Transform Validity condition in Section 5.
- When `L` includes a reflection (`det(L) < 0`) and is used together with a `ROTATION` semantic attribute, the specific method used to extract the pure rotation component `R` from `L` (e.g. polar decomposition) is outside the scope of this format and is the responsibility of the reader/writer implementation.

<details><summary>Example storage of Blender's Default Cube (chunk_data of the JSON chunk)</summary>

```json
{
    "header": {
        "format": "Topolyx",
        "version": "1.0"
    },

    "coordinate_system": {
        "up_axis": "+Z",
        "forward_axis": "+Y",
        "handedness": "RIGHT",
        "winding": "CCW",
        "meters_per_unit": 1.0
    },

    "objects": [
        {
            "name": "object1",
            "type": "MESH",
            "index": 0,
            "transform": [
                1, 0, 0, 0,
                0, 1, 0, 0,
                0, 0, 1, 0,
                0, 0, 0, 1
            ]
        }
    ],

    "meshes": [
        {
            "name": "mesh1",

            "element_counts": {
                "vertices": 8,
                "edges": 12,
                "faces": 6,
                "corners": 24
            },

            "topology": {
                "positions": {
                    "byte_offset": 0,
                    "byte_length": 96,
                    "component_type": "F32",
                    "component_count": 3,
                    "element_count": 8
                },

                "edges": {
                    "byte_offset": 96,
                    "byte_length": 96,
                    "component_type": "U32",
                    "component_count": 2,
                    "element_count": 12
                },

                "corner_vertices": {
                    "byte_offset": 192,
                    "byte_length": 96,
                    "component_type": "U32",
                    "component_count": 1,
                    "element_count": 24
                },

                "corner_edges": {
                    "byte_offset": 288,
                    "byte_length": 96,
                    "component_type": "U32",
                    "component_count": 1,
                    "element_count": 24
                },

                "face_offsets": {
                    "byte_offset": 384,
                    "byte_length": 28,
                    "component_type": "U32",
                    "component_count": 1,
                    "element_count": 7
                }
            },

            "attributes": [
                {
                    "name": "sharp_face",
                    "domain": "FACE",
                    "semantic": "NONE",
                    "data": {
                        "byte_offset": 412,
                        "byte_length": 6,
                        "component_type": "BOOL",
                        "component_count": 1,
                        "element_count": 6
                    }
                },
                {
                    "name": "UVMap",
                    "domain": "CORNER",
                    "semantic": "NONE",
                    "data": {
                        "byte_offset": 420,
                        "byte_length": 192,
                        "component_type": "F32",
                        "component_count": 2,
                        "element_count": 24
                    }
                },
                {
                    "name": "custom_attribute",
                    "domain": "EDGE",
                    "semantic": "ROTATION",
                    "data": {
                        "byte_offset": 612,
                        "byte_length": 192,
                        "component_type": "F32",
                        "component_count": 4,
                        "element_count": 12
                    }
                }
            ]
        }
    ]
}
```
</details>

### BIN Chunk Data

> [!IMPORTANT]
>
> ```text
> positions
> edges
> corner_vertices
> corner_edges
> face_offsets
> attribute values
> ```
>
> - The BIN chunk's `chunk_data` stores each array consecutively.
>
> - A reader must read data based on the `byte_offset` recorded in the JSON, and must not rely on the actual storage order of the arrays.

#### Storage Method

- All binary scalar values are stored in `little-endian` order.

- All data arrays use 4-byte alignment.
    - The `byte_offset` of each data array must be a multiple of 4.

- Unused regions between stored arrays are allowed, but a reader must not interpret bytes that are not referenced by any `byte_offset`/`byte_length` as data.

    - <details><summary>Unused region of the binary file corresponding to the JSON example</summary>

        ```text
        offset 412

        BOOL attribute
        6 bytes

        unused space
        2 bytes

        offset 420

        next array
        ```
        </details>

    - Additionally, the bytes of unused regions must all be stored as 0.

---

## 5. Validity Conditions

> [!NOTE]
>
> Constraints that check relationships between multiple fields, plus common constraints.

### Container Validity

- The 4-byte `magic` value of the file must exactly match ASCII `TLYX` (`54 4C 59 58`). A reader must compare this as a raw byte sequence without converting it to an integer, and must reject the file if it does not match.
- The container `header.version` (U32) must equal the major version supported by the reader. If it differs, the reader must not read the file (see Section 6).
- The container `header.version` must match the `x` of the JSON `header.version` (`x.y`).
- `total_length` must exactly equal the actual byte length of the entire file.
- There must be exactly two chunks, and their order is fixed as JSON chunk → BIN chunk.
- Each chunk's `chunk_type` must exactly match the specification (`JSON`, `BIN\0`).
- Each chunk's `chunk_length` must be a multiple of 4 and must not exceed the range representable by a U32 (the maximum size of a single chunk is limited to about 4 GiB).
- Only `0x20` is allowed as the padding byte for the JSON chunk, and only `0x00` is allowed as the padding byte for the BIN chunk. A reader must not interpret padding bytes as data.

### Index Range

- All index values must fall within the following range (index value validity):
    ```text
    0 <= index < array length
    ```

### Name Constraints

- Names must not be empty strings.

- `object.name` must be unique within `objects`.
- `mesh.name` must be unique within `meshes`.
- `attribute.name` must be unique only within a single mesh.

### Attribute Data Constraints

- Fields of the `topology` struct, and attributes whose `semantic` is not `NONE`, must have `component_type` and `component_count` matching the specification when stored.

### Transform Validity

- The linear (3x3) part of `object.transform` must have a nonzero determinant (must be non-singular). If it is 0, the file is invalid.

### Corner and Edge Consistency

- For corner `c` within a face and its next corner `n`, the edge referenced by `corner_edges[c]` must connect the following two vertices:

    ```text
    corner_vertices[c]
    corner_vertices[n]
    ```

- The order of the two vertex indices stored in an edge does not matter.

- The corner following the last corner is the first corner of the same face's range.

- Not every vertex needs to be referenced by `edges` or `corner_vertices` (loose vertices are allowed).
- Not every edge needs to be referenced by `corner_edges` (loose edges are allowed).

### Empty Mesh

- An empty mesh of the following form is allowed:

    ```text
    vertices = 0
    edges = 0
    faces = 0
    corners = 0
    ```

- For an empty mesh, the required arrays must satisfy the following conditions:

    ```text
    positions.element_count == 0
    edges.element_count == 0
    corner_vertices.element_count == 0
    corner_edges.element_count == 0
    face_offsets == [0]
    ```

---

## 6. Versioning and Compatibility

> [!IMPORTANT]
>
> Versions use the `x.y.z` format in documents, the `x.y` format in the JSON `header.version`, and a U32 holding only the major version (`x`) in the container header.

- `x`
    - A breaking change incompatible with previous versions.
    - Removal or change in meaning of a required field.
    - A change to the binary representation or the basic topology structure.

- `y`
    - An addition of a feature that remains compatible with previous versions.
    - An optional field or feature added that an existing reader can ignore.

- `z`
    - A small fix that does not change semantics or the binary structure.
    - Clarification of a description, typo fix, or example fix.

- The `0.y.z` stage is an early experimental stage, so compatibility may not be guaranteed even across `y` changes.

- Since a `z` change is a fix that does not alter the format structure, it is not included in the JSON file.

- A reader must not read a file whose `x` version it does not support.

> [!NOTE]
>
> Version information appears in three places, each with a different level of precision.
>
> - Document version (e.g. CHANGELOG): `x.y.z`
> - JSON `header.version`: `x.y`
> - Container `header.version` (binary): a U32 holding only `x`
>
> The container version exists so that a reader can determine support at the binary level, before parsing the JSON.
> If the container version and the `x` of the JSON `header.version` differ, the file is invalid.

---