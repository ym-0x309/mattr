# MATTR 포맷 명세(v0.0.1)

> - 포맷명: MATTR
> - 정식 명칭: Mesh Attribute & Topology Transfer Representation
> - 분리형 파일:
>     - model.mattr.json
>     - model.mattr.bin
> - 통합형 파일(현재 미지원, 향후 계획):
>     - model.mattr

## 1. 포맷의 목적

Blender에서 제작한 메쉬를, attribute 데이터를 보존한 상태로 자체 엔진이나 툴로 가져오기 위한 포맷이다.

일반적인 OBJ, glTF, FBX처럼 렌더링에 필요한 결과만 전달하는 것이 아니라, Blender `Mesh` 데이터 블록에 존재하는 다음 domain의 attribute를 가능한 한 손실 없이 보존하는 것을 핵심으로 한다.


```text
POINT domain attribute
EDGE domain attribute
FACE domain attribute
CORNER domain attribute
```

이 포맷은 완전한 Blender scene, modifier stack, material graph 또는 animation 데이터를 보존하는 것을 목적으로 하지 않는다.

---

## 2. 용어 정리

### Vertex

메쉬의 공간상 점이다. `positions` 배열의 하나의 element에 대응한다.

### Edge

두 vertex를 연결하는 요소이다. 각 edge는 두 vertex index를 저장한다.

### Face

메쉬의 polygon 요소이다. Vertex나 edge 배열을 직접 소유하지 않고,
`face_offsets`를 통해 corner 배열의 연속된 범위를 참조한다.

### Face Corner

특정 face에서 특정 vertex가 사용되는 위치이다.
Corner는 face 간에 공유되지 않는다. 같은 vertex가 여러 face에서 사용되는 경우에도 각 face는 별도의 corner를 가진다.

### Domain

Attribute가 대응하는 메쉬 요소의 종류이다.

- `POINT`: vertex
- `EDGE`: edge
- `FACE`: face
- `CORNER`: face corner

### Element

배열의 논리적 항목 하나이다.

예:

- 하나의 position
- 하나의 edge
- 하나의 UV 좌표

### Component

하나의 element를 구성하는 scalar 값이다.

예를 들어 F32×3 position은 F32 component 세 개로 구성된다.

### Component type

각 component가 binary에 저장될 때 사용하는 scalar 자료형이다.

예: `F32`, `U32`

### Byte offset

Binary buffer 시작점으로부터 데이터 배열 시작점까지의 바이트 거리이다.

### Topology

Vertex, edge, face, corner 사이의 연결 관계를 나타내는 필수 메쉬 데이터이다.

### Attribute

특정 domain의 각 element에 대응하는 추가 데이터 배열이다.

---

## 3. 포맷 규칙

- 모든 binary scalar 값은 `little-endian` 방식으로 저장한다.
- 모든 데이터 배열은 4바이트 정렬을 사용한다.
    - 각 데이터 배열의 `byte_offset`은 4의 배수여야 한다.
    - 배열 사이에 padding이 필요한 경우 padding byte는 모두 `0`으로 기록한다.
    - Reader는 padding byte의 값을 데이터로 해석해서는 안 된다.
- JSON 파일은 UTF-8로 인코딩한다.
- 좌표계는 JSON의 `coordinate_system`에 명시된 규칙을 따른다.
- `coordinate_system`의 `up_axis`, `forward_axis`, `handedness`는
  world space 기준으로 정의된 축이다.
  Object별 local axis는 `transform`을 통해 world space로 변환된 뒤
  이 좌표계를 따른다.
- `positions`와 `transform`은 동일한 좌표계를 사용해야 한다.
- 행렬은 4×4 열 우선(Column-Major) 순서로 직렬화한다.
- 벡터는 열 벡터(Column Vector)로 취급하며, 변환은 `M × v` 순서로 적용한다.
- `object.transform`은 해당 object가 참조하는 mesh의 local space 좌표를 file world space 좌표로 변환하는 행렬이다.
- Reader는 자신이 알지 못하는 JSON 필드를 무시해야 한다.

---

## 4. 파일 구성

### 초기 파일 구조

초기 버전은 JSON과 Binary를 별개의 파일로 분리한다.

```text
test_mesh.mattr.json
test_mesh.mattr.bin
```

#### JSON 역할

JSON에는 메타 데이터를 저장한다.

```text
포맷 버전
오브젝트 이름
메쉬 이름
좌표계
데이터 개수
각 데이터 byte_offset
Attribute 이름/domain/type
```

저장 방식은 다음과 같다.

Blender의 Default Cube의 저장 예시이다.
```jsonc
{
    // 포맷 기본 정보
    "header": {
        "format": "MATTR",
        "version": "0.0.1"
    },

    "buffer": {
        "uri": "test_mesh.bin",
        "byte_length": 604  // Binary 파일 길이
    },

    // 좌표계 정보
    "coordinate_system": {
        "up_axis": "+Z",
        "forward_axis": "-Y",
        "handedness": "RIGHT",
        "winding": "CCW",
        "meters_per_unit": 1.0
    },


    // 오브젝트 정보. 현재 parenting 미지원.
    "objects": [
        {
            "name": "object1",
            "type": "MESH",
            "index": 0, // 뒤의 "meshes" 배열 인덱스
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

            // 버텍스/에지/페이스/코너 개수
            "element_counts": {
                "vertices": 8,
                "edges": 12,
                "faces": 6,
                "corners": 24
            },

            // 필수 메쉬 데이터 저장
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

            // 그 외 attribute 저장
            "attributes": [
                {
                    "name": "UVMap",
                    "domain": "CORNER",
                    "data": {
                        "byte_offset": 412,
                        "byte_length": 192,
                        "component_type": "F32",
                        "component_count": 2,
                        "element_count": 24
                    }
                }
            ]
        }
    ]
}
```

#### Binary 역할

```text
positions
edges
corner_vertices
corner_edges
face_offsets
attribute values
````

Binary에는 각 배열을 연속적으로 저장한다.

Writer는 위 순서를 권장 저장 순서로 사용할 수 있지만, reader는 JSON에 기록된 `byte_offset`을 기준으로 데이터를 읽어야 하며 배열의 실제 저장 순서에 의존해서는 안 된다.

Blender `Mesh`의 배열 중심 구조를 따라 SoA 방식으로 저장하며, 모든 scalar 값은 little-endian 방식으로 직렬화한다.

### 후기 파일 구조

통합형 파일(.mattr)은 v0.0.1의 지원 범위에 포함되지 않는다.
JSON과 Binary를 하나의 파일에 저장하는 chunk 기반 컨테이너 구조는
향후 버전에서 별도로 정의한다 (9절 참조).

v0.0.1은 분리형 파일 구조(model.mattr.json + model.mattr.bin)만 지원한다.

---

## 5. 저장 구조

### Attribute 기본 정보

필수 메쉬 데이터와 일반 attribute의 data descriptor에 공통으로 사용되는 필드이다.

- `byte_offset`: binary buffer 시작점 기준 데이터 배열의 byte offset
- `byte_length`: 데이터 배열이 차지하는 전체 byte 길이
- `component_type`: 각 component의 binary scalar 자료형
- `component_count`: 하나의 element를 구성하는 component 수
- `element_count`: 배열에 저장된 element의 총 개수

고정 길이 배열의 `byte_length`는 다음과 같이 계산한다.

```text
byte_length
= component_type의 byte 크기
× component_count
× element_count
````

v0.0.1에서 지원하는 component type은 다음과 같다.

| Component type |      크기 | 의미                             |
| -------------- | ------: | ------------------------------ |
| `F32`          | 4 bytes | IEEE 754 32-bit floating-point |
| `I32`          | 4 bytes | signed 32-bit integer        |
| `U32`          | 4 bytes | unsigned 32-bit integer        |

### 필수 메쉬 데이터

> [Blender 공식 Mesh 문서](https://developer.blender.org/docs/features/objects/mesh/mesh/)의 배열 구조를 기반으로 한다.

- `positions`
    - 각 vertex의 mesh local space 위치
    - 형식: `F32×3`
    - `element_count`는 `vertices`와 같아야 한다.

- `edges`
    - 각 edge를 구성하는 두 vertex의 index
    - 형식: `U32×2`
    - `element_count`는 `edges`와 같아야 한다.
    - 두 vertex index의 순서는 edge의 방향을 의미하지 않는다.

- `corner_vertices`
    - 각 face corner가 참조하는 vertex의 index
    - 형식: `U32`
    - `element_count`는 `corners`와 같아야 한다.
    - 같은 face에 속한 corner들은 polygon을 순회하는 순서대로 저장되어야 하며, 이때 순회 방향은 JSON 파일의 `winding`을 따른다.

- `corner_edges`
    - 각 face corner에서 같은 face의 다음 corner 방향으로 이어지는 edge의 index
    - 형식: `U32`
    - `element_count`는 `corners`와 같아야 한다.

- `face_offsets`
    - 각 face가 사용하는 corner 배열 범위의 시작 index
    - 형식: `U32`
    - `element_count`는 `faces + 1`이어야 한다.
    - face `i`가 사용하는 corner 범위는 다음과 같다.

        ```text
        [face_offsets[i], face_offsets[i + 1])
        ````

`face_offsets`의 마지막 값은 전체 corner 수와 같아야 한다.

### 일반 Attribute

일반 attribute는 특정 domain의 각 element에 대응하는 고정 길이 데이터 배열이다.

- `name`
    - Attribute의 이름이다.
    - 하나의 mesh 안에서 유일해야 한다.
    - 빈 문자열이어서는 안 된다.

- `domain`
    - Attribute가 대응하는 메쉬 요소를 지정한다.
    - 다음 값 중 하나여야 한다.

        ```text
        POINT
        EDGE
        FACE
        CORNER
        ````

- `data`

  - Attribute 배열의 binary 저장 정보를 나타낸다.
  - `byte_offset`, `byte_length`, `component_type`, `component_count`, `element_count`를 포함한다.

Attribute의 `element_count`는 domain에 따라 다음 값과 같아야 한다.

```text
POINT  → element_counts.vertices
EDGE   → element_counts.edges
FACE   → element_counts.faces
CORNER → element_counts.corners
```

---

## 6. 유효성 조건

파일은 다음 조건을 모두 만족해야 한다.

### JSON 및 buffer

- `header.format`은 해당 포맷에서 정의한 식별 문자열(MATTR)과 일치해야 한다.
- `header.version`은 `x.y.z` 형식이어야 한다.
- `buffer.uri`가 가리키는 binary 파일이 존재해야 한다.
- 실제 binary 파일의 크기는 `buffer.byte_length`와 같아야 한다.
- `handedness`는 `RIGHT` 혹은 `LEFT`여야 한다.
- `winding`은 `CW` 혹은 `CCW`여야 한다.
- `up_axis`, `forward_axis`는 `+X`, `-X`, `+Y`, `-Y`, `+Z`, `-Z` 중 하나이며, 서로 평행해서는 안 된다.
- 모든 object의 `index` 값은 `meshes` 배열의 유효한 index여야 한다.

### Data descriptor

- 모든 필수 메쉬 데이터와 일반 attribute의 data descriptor는 다음을 만족해야 한다.

    ```text
    byte_offset % 4 == 0
    ````

    ```text
    byte_offset + byte_length <= buffer.byte_length
    ```

- `byte_offset + byte_length` 계산은 정수 overflow를 발생시켜서는 안 된다.

- 다음 식이 성립해야 한다.

    ```text
    byte_length
    = component_size(component_type)
    × component_count
    × element_count
    ```

- `component_count`는 1 이상이어야 한다.

- `element_count`가 0이면 `byte_length`도 0이어야 한다.

### 필수 메쉬 데이터 크기

- 다음 조건이 성립해야 한다.

    ```text
    positions.component_type == F32
    positions.component_count == 3
    positions.element_count == element_counts.vertices
    ```

    ```text
    edges.component_type == U32
    edges.component_count == 2
    edges.element_count == element_counts.edges
    ```

    ```text
    corner_vertices.component_type == U32
    corner_vertices.component_count == 1
    corner_vertices.element_count == element_counts.corners
    ```

    ```text
    corner_edges.component_type == U32
    corner_edges.component_count == 1
    corner_edges.element_count == element_counts.corners
    ```

    ```text
    face_offsets.component_type == U32
    face_offsets.component_count == 1
    face_offsets.element_count == element_counts.faces + 1
    ```

### Index 범위

- 모든 edge의 두 vertex index는 다음 범위에 있어야 한다.

    ```text
    0 <= vertex_index < element_counts.vertices
    ```

- 모든 `corner_vertices` 값은 다음 범위에 있어야 한다.

    ```text
    0 <= corner_vertex < element_counts.vertices
    ```

- 모든 `corner_edges` 값은 다음 범위에 있어야 한다.

    ```text
    0 <= corner_edge < element_counts.edges
    ```

### Face offsets

- `face_offsets`는 다음 조건을 만족해야 한다.

    ```text
    face_offsets[0] == 0
    ```

    ```text
    face_offsets[i] <= face_offsets[i + 1]
    ```

    ```text
    face_offsets[element_counts.faces] == element_counts.corners
    ```

- v0.0.1에서 각 face는 최소 세 개의 corner를 가져야 한다.

    ```text
    face_offsets[i + 1] - face_offsets[i] >= 3
    ```

- 단, face가 없는 빈 mesh에서는 `face_offsets`가 `[0]`이어야 한다.

- Non-planar n-gon의 삼각화 및 렌더링 방식은 이 포맷의 범위 밖이며, 이를 처리하는 방식은 reader/엔진의 책임으로 한다.

### Corner와 edge의 일관성

- Face 안의 corner `c`와 그 다음 corner `n`에 대해, `corner_edges[c]`가 참조하는 edge는 다음 두 vertex를 연결해야 한다.

    ```text
    corner_vertices[c]
    corner_vertices[n]
    ```

- Edge에 저장된 두 vertex index의 순서는 무관하다.

- 마지막 corner의 다음 corner는 같은 face 범위의 첫 번째 corner이다.

- 모든 vertex가 `edges` 또는 `corner_vertices`에 의해
  참조될 필요는 없다 (loose vertex 허용).
- 모든 edge가 `corner_edges`에 의해 참조될 필요는 없다
  (loose edge 허용).

### 일반 Attribute

- `name`은 같은 mesh 안의 다른 일반 attribute 이름과 중복되어서는 안 된다.
- `domain`은 `POINT`, `EDGE`, `FACE`, `CORNER` 중 하나여야 한다.
- `component_type`은 `F32` 또는 `U32` 혹은 `I32`여야 한다.
- `component_count`는 1 이상이어야 한다.
- `element_count`는 domain에 따라 다음 값과 같아야 한다.

    ```text
    POINT  → element_counts.vertices
    EDGE   → element_counts.edges
    FACE   → element_counts.faces
    CORNER → element_counts.corners
    ```

### 빈 Mesh

- 다음과 같은 빈 mesh를 허용한다.

    ```text
    vertices = 0
    edges = 0
    faces = 0
    corners = 0
    ```

- 빈 mesh에서 필수 배열은 다음 조건을 만족해야 한다.

    ```text
    positions.element_count == 0
    edges.element_count == 0
    corner_vertices.element_count == 0
    corner_edges.element_count == 0
    face_offsets == [0]
    ```

---

## 7. 버전과 호환성

버전은 `x.y.z` 형식을 사용한다.

- `x`
    - 이전 버전과 호환되지 않는 중대한 변경
    - 필수 필드 삭제 또는 의미 변경
    - binary 표현 및 기본 topology 구조 변경

- `y`
    - 이전 버전과 호환되는 기능 추가
    - 기존 reader가 무시할 수 있는 선택적 필드 또는 기능 추가

- `z`
    - 의미나 binary 구조를 변경하지 않는 작은 수정
    - 설명 보완, 오탈자 수정, 예시 수정

`0.0.z` 단계는 초기 실험 단계이므로, `z` 변경에서도 호환성이 보장되지 않을 수 있다.

Reader는 자신이 지원하지 않는 `x` 버전의 파일을 읽어서는 안 된다.

---

## 8. 지원 범위

### v0.0.1에서 지원

- 하나의 JSON 파일과 하나의 binary 파일
- 하나 이상의 mesh
- 하나 이상의 mesh object
- 여러 object가 동일한 mesh를 참조하는 구조
- 4×4 object transform 행렬
- Vertex
- Loose vertex
- Edge
- Loose edge
- Triangle
- Quad
- N-gon
- `POINT` domain attribute
- `EDGE` domain attribute
- `FACE` domain attribute
- `CORNER` domain attribute
- `F32` component
- `U32` component
- 고정 길이 numeric attribute
- 빈 mesh
- Little-endian binary
- 4바이트 정렬

### v0.0.1에서 지원하지 않음

- Object parenting
- Collection
- Scene hierarchy
- Modifier stack
- Material
- Shader 및 material graph
- Texture 및 image
- Armature
- Vertex group
- Skin weight
- Shape key
- Animation
- Curve
- Point cloud
- Object instance
- Sparse attribute
- Binary compression
- 여러 binary buffer
- 64-bit index
- Face 내부의 hole
- 통합형 파일(.mattr) 바이너리 컨테이너

---

## 9. 향후 확장

다음 기능은 향후 버전에서 검토한다.

- JSON과 binary를 하나의 파일에 저장하는 chunk 기반 컨테이너
- 자연 정렬 또는 8바이트 정렬
- `F64`, `I32`, `U8`, `U64` component type
- Attribute semantic
- Material 및 material slot
- Vertex group 및 skin weight
- Shape key
- Object parenting
- Binary 압축
- MESH 외 다른 타입
