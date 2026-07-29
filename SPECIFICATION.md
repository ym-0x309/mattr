# Topolyx 포맷 명세(v1.0.0)

> [!IMPORTANT]
> - 포맷명: **Topolyx** (**Topo**logy, **Poly**gon, e**X**change)
> - 정식 명칭: Mesh Attribute & Topology Interchange Format
> - 파일: `model.tlyx` (단일 통합 파일)
> - Magic bytes: `54 4C 59 58` (ASCII `TLYX`)
>

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

## 2. 포맷의 범위

본 포맷은 다음 정보를 저장한다.

- Object Transform
- Vertex / Edge / Face / Corner Topology
- Mesh Attribute
- Coordinate System

다음 정보는 Topolyx `1.x`까지의 범위에 **포함되지 않는다**(`2.x` 범위는 미정).

- Material 정의
- Texture 및 이미지 파일
- Shader(WGSL, GLSL 등)
- Animation
- Armature / Bone
- Modifier Stack
- Scene 구성
- Camera
- Light
- Render Setting

Texture, Material Graph 등 기하 정보와 attribute에 직접적으로 관련되지 않은 데이터는 포맷의 범위에 포함하지 않으며,
필요한 경우 외부 시스템에서 `MATERIAL_INDEX` 등의 Attribute를 이용하여 연결한다.

---

## 3. 용어 정리

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

### Byte offset

BIN 청크의 `chunk_data` 시작점으로부터 데이터 배열 시작점까지의 바이트 거리이다.

### Topology

Vertex, edge, face, corner 사이의 연결 관계를 나타내는 필수 메쉬 데이터이다.

### Attribute

특정 domain의 각 element에 대응하는 추가 데이터 배열이다.

---

## 4. 파일 구성

> [!IMPORTANT]
> 
> ```text
> test_mesh.tlyx
> ```
> 
> `1.x.y`부터 통합형 파일(`.tlyx`)이 분리형 파일(`.json + .bin`)을 대체한다.
>

### 컨테이너 구조

> [!TIP] 참고
>
> glTF 포맷(`.glb`)의 저장 구조를 참고했다.
>

1. Header: 12 bytes

    | offset | 이름 | 형식 | 설명 |
    |-|-|-|-|
    | 0 | magic | 4 바이트 ASCII 문자 | `TLYX` (`54 4C 59 58`) |
    | 4 | version | U32 | major 버전 정수. 현재는 `1` |
    | 8 | total_length | U32 | 파일 전체 바이트 길이(헤더 + 모든 청크 포함) |

2. Chunk 0: JSON

    | 이름 | 형식 | 설명 |
    |-|-|-|
    | chunk_length | U32 | chunk_data 길이, 패딩 포함, 4의 배수 |
    | chunk_type | 4 바이트 ASCII 문자 | `JSON` |
    | chunk_data | UTF-8 JSON | 끝을 `0x20`(space)로 패딩해 4바이트 정렬한 JSON |

3. Chunk 1: BIN

    | 이름 | 형식 | 설명 |
    |-|-|-|
    | chunk_length | U32 | chunk_data 길이, 패딩 포함, 4의 배수 |
    | chunk_type | 4 바이트 ASCII 문자 | `BIN\0` |
    | chunk_data | 바이너리 | 끝을 `0x00`으로 패딩해 4바이트 정렬 |

파일에는 이 두 청크만 존재하며, 순서는 JSON → BIN으로 고정된다.

### JSON

> [!IMPORTANT]
> 
> JSON에는 메타 데이터를 저장하며, UTF-8로 인코딩한다.
> 
> ```text
> 포맷 버전
> 오브젝트 이름
> 메쉬 이름
> 좌표계
> 데이터 개수
> 각 데이터 byte_offset
> Attribute 이름/domain/type
> ```
> 

#### 저장 방식 및 필드별 제약

##### 파일 기본 정보

- `header`: 포맷 정보

    - `format`: `Topolyx`
    - `version`: `x.y`

##### 좌표계 정보

- `coordinate_system`: 월드 좌표계 정의. `up_axis`, `forward_axis`, `handedness`, `winding`은 Blender의 월드 좌표계를 그대로 따르는 고정값이며, 모든 Topolyx 파일에서 동일하다.

    - `up_axis`: 항상 `+Z`. 다른 값이면 유효하지 않은 파일이다.
    - `forward_axis`: 항상 `+Y`. 다른 값이면 유효하지 않은 파일이다.
    - `handedness`: 항상 `RIGHT`. 다른 값이면 유효하지 않은 파일이다.
    - `winding`: 항상 `CCW`. 다른 값이면 유효하지 않은 파일이다.
    - `meters_per_unit`: 좌표계에서 1m의 길이. 이 값만 파일마다 다를 수 있다.
        - 0 초과의 유효한 수여야 한다.(NaN, Infinity 불가)

> [!NOTE]
>
> `up_axis`, `forward_axis`, `handedness`, `winding`은 모든 Topolyx 파일에서 고정되므로 파일 간 축/handedness 변환은 필요하지 않다.
> 다만 `meters_per_unit`은 파일마다 다를 수 있으며, 서로 다른 단위 축척 간의 값 변환은 이 포맷이 보장하지 않고 writer/reader 구현의 책임이다.
>

##### 오브젝트 정보

- object: `objects` 배열의 내용물

    - `name`: 오브젝트 이름.
    - `type`: 현재 `MESH`만 지원.
    - `index`: 뒤의 `type`마다 배정된 배열에서의 index. 예를 들어 type이 MESH인 경우 index는 meshes 배열의 index를 의미한다.
    - `transform`: 원본 메쉬에 적용되는 4x4 변환 행렬. 열 우선(Column-Major) 순서.

        - 벡터는 열 벡터(Column Vector)로 취급하며, 변환은 `M × v` 순서로 적용한다.
        - 메쉬 별 local axis는 `transform`을 통해 world space로 변환됨.
        - `transform`이 topology(`positions`) 및 attribute의 각 semantic에 구체적으로 어떻게 적용되는지는 뒤의 "Object Transform 적용 규칙" 절을 따른다.

현재 parenting은 미지원.

##### Attribute 정보

- attribute data: `topology` 데이터와 `attributes`에 공통으로 사용되는 필드

    - `byte_offset`: BIN 청크의 `chunk_data` 시작점 기준 데이터 배열의 byte offset
        - 다음 조건을 만족해야 한다(4바이트 정렬)
            ```text
            byte_offset % 4 == 0
            ````
    - `byte_length`: 데이터 배열이 차지하는 전체 byte 길이. 
        - 다음과 같이 계산한다.
            ```text
            byte_length
            = component_type의 byte 크기
            * component_count
            * element_count
            ```
        - 다음 조건을 만족해야 하며, 다음 계산에서 정수 overflow가 발생해선 안 된다.
            ```text
            byte_offset + byte_length <= BIN 청크의 chunk_length
            ```
    - `component_type`: 각 component를 저장하는 binary scalar type. 현재 지원되는 component는 다음과 같음.

        | Component type |      크기 | 의미                             |
        | -------------- | ------: | ------------------------------ |
        | `F32`          | 4 bytes | IEEE 754 32비트 부동소수점 |
        | `I32`          | 4 bytes | 부호 있는 32비트 정수        |
        |`I8`|1 byte|부호 있는 8비트 정수|
        | `U32`          | 4 bytes | 부호 없는 32비트 정수        |
        |`U8`|1 byte|부호 없는 8비트 정수|
        |`BOOL`|1 byte|1바이트 boolean 값. `0x00`는 `false`, `0x01`는 `true`, 나머지 값은 유효하지 않음.|
    - `component_count`: 하나의 element를 구성하는 component 수. 1 이상이어야 한다.
    - `element_count`: 배열에 저장된 element의 총 개수. 후술할 `face_offsets`의 경우를 제외하면 모두 해당 도메인의 `element_count`와 일치해야 한다.

##### 메쉬 정보

> [!TIP]
>
> [Blender 공식 Mesh 문서](https://developer.blender.org/docs/features/objects/mesh/mesh/)의 배열 구조를 기반으로 한다.

 - mesh: `meshes` 배열의 내용물

    - `name`: 메쉬 이름.

    - `element_counts`

        - `vertices`: 버텍스 개수
        - `edges`: 에지 개수
        - `faces`: 페이스 개수
        - `corners`: 페이스 코너 개수

    - `topology`: 필수 메쉬 데이터.

        - `positions`: attribute data

            - 각 버텍스의 local space 위치
            - `component_type`: `F32`
            - `component_count`: 3
            - `domain`: `POINT`
            - Object `transform` 적용 시의 변환 규칙은 뒤의 "Object Transform 적용 규칙" 절을 따른다.

        - `edges`: attribute data

            - 각 edge를 구성하는 두 vertex의 index
            - `component_type`: `U32`
            - `component_count`: 2
            - `domain`: `EDGE`
            - 두 vertex index의 순서는 edge의 방향을 의미하지 않는다.
            - 중복 edge 및 self-edge는 허용하지 않는다.

        - `corner_vertices`: attribute data

            - 각 face corner가 참조하는 vertex의 index
            - `component_type`: `U32`
            - `component_count`: 1
            - `domain`: `CORNER`
            - 같은 face에 속한 corner들은 polygon을 순회하는 순서대로 저장되어야 하며, 이때 순회 방향은 JSON 파일의 `winding`을 따른다.
            
        - `corner_edges`: attribute data

            - 각 face corner에서 같은 face의 다음 corner 방향으로 이어지는 edge의 index
            - `component_type`: `U32`
            - `component_count`: 1
            - `domain`: `CORNER`

        - `face_offsets`: attribute data

            - 각 face가 사용하는 corner 배열 범위의 시작 index
            - `component_type`: `U32`
            - `component_count`: 1
            - `domain`: `FACE`
            - **`element_count`는 `faces + 1`과 같아야 한다.**
            - face `i`가 사용하는 corner 범위는 다음과 같다.

                ```text
                [face_offsets[i], face_offsets[i + 1])
                ````
            - `face_offsets`의 처음 값은 0, 마지막 값은 전체 corner 수와 같아야 한다.
            - 또한 각 face는 최소 3개의 corner를 가져야 하기 때문에, 다음 조건을 만족해야 한다.
                ```text
                face_offsets[i + 1] - face_offsets[i] >= 3
                ```
            - Non-planar n-gon의 삼각화 및 렌더링 방식은 이 포맷의 범위 밖이며, 이를 처리하는 방식은 reader/엔진의 책임으로 한다.

    - attribute: `attributes` 배열에 저장되는 일반 attribute

        - `name`: attribute 이름.
        - `domain`: `POINT`, `EDGE`, `FACE`, `CORNER` 중 하나.
        - `semantic`: 표준화된 해석 규칙이 필요한 attribute에 지정한다. 현재 지원되는 항목은 다음과 같다.

            |semantic|component type|component count|의미|
            |-|-|-|-|
            |`POSITION`|`F32`|3|3차원 위치 정보.|
            |`DIRECTION`|`F32`|3|3차원 방향 정보.|
            |`NORMAL`|`F32`|3|3차원 법선 정보. 값의 형식은 `DIRECTION`과 같지만, Object `transform` 적용 시 역전치행렬을 사용하는 등 변환 규칙이 다르다. 자세한 내용은 "Object Transform 적용 규칙" 절 참고.|
            |`ROTATION`|`F32`|4|사원수 회전 정보. (x, y, z, w)로 저장됨.|
            |`TANGENT`|`F32`|4|탄젠트 정보. 탄젠트(x, y, z)와 handedness(w)로 저장됨. Bitangent는 `cross(normal, tangent.xyz) * tangent.w`로 복원하며, 여기서 normal은 같은 mesh 안에 존재하는 `NORMAL` semantic attribute를 가리킨다.|
            |`COLOR`|`F32`, `U8`|4|RGBA 색상|
            |`NONE`|Any|Any|그 외 attribute|

        - `data`: attribute data

##### Object Transform 적용 규칙

Object의 `transform`(local → world, 4x4)을 mesh의 topology 및 attribute 값에 적용할 때는 semantic에 따라 아래 규칙을 따른다.

`transform`의 선형(3x3) 부분을 `L`, 이동(translation) 부분을 `t`라 한다.

| 대상 | 적용 방식 |
|-|-|
| `positions`(topology), `POSITION` | `L · p + t` |
| `DIRECTION` | `L · v`. 이동은 적용하지 않으며, 재정규화하지 않는다. |
| `NORMAL` | `transpose(inverse(L)) · n`. 이동은 적용하지 않으며, 적용 후 단위벡터로 재정규화한다. |
| `TANGENT` | `xyz`는 `DIRECTION`과 동일하게 적용한 뒤 재정규화한다. `det(L) < 0`이면 `w` 성분의 부호를 반전한다. |
| `ROTATION` | `q' = quat(R) ⊗ q`. `R`은 `L`에서 추출한 순수 회전 성분이다. |
| `COLOR`, `NONE` | 변환하지 않는다. |

- `L`이 특이행렬(singular, `det(L) = 0`)이면 위 변환은 정의되지 않으며, 이는 5장의 Transform 유효성 조건 위반이다.
- `L`에 반사(reflection)가 포함되어(`det(L) < 0`) `ROTATION` semantic attribute와 함께 쓰이는 경우, `L`에서 순수 회전 성분 `R`을 추출하는 구체적인 방법(극분해 등)은 이 포맷의 범위 밖이며 reader/writer 구현의 책임이다.

<details><summary>Blender의 Default Cube의 저장 예시 (JSON 청크의 chunk_data)</summary>

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

### BIN 청크 데이터

>
> [!IMPORTANT]
> 
> ```text
> positions
> edges
> corner_vertices
> corner_edges
> face_offsets
> attribute values
> ````
> 
> - BIN 청크의 `chunk_data`에는 각 배열을 연속적으로 저장한다.
> 
> - reader는 JSON에 기록된 `byte_offset`을 기준으로 데이터를 읽어야 하며 배열의 실제 저장 순서에 의존해서는 안 된다.
>

#### 저장 방식

- 모든 binary scalar 값은 `little-endian` 방식으로 저장한다.

- 모든 데이터 배열은 4바이트 정렬을 사용한다.
    - 각 데이터 배열의 `byte_offset`은 4의 배수여야 한다.

- 저장된 배열 사이의 사용되지 않은 구역은 허용되지만, reader는 `byte_offset`과 `byte_length`에 의해 참조되지 않은 byte를 데이터로 해석해서는 안 된다.

    - <details><summary>JSON 파일 예시에 해당하는 Binary 파일의 사용되지 않은 구역</summary>

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

    - 또한 사용되지 않은 구역의 byte는 모두 0으로 저장해야 한다.

---

## 5. 유효성 조건

> [!NOTE]
> 
> 여러 필드의 관계를 검사하는 제약 및 공통 제약.

### 컨테이너 유효성

- 파일은 4바이트 `magic`이 정확히 ASCII `TLYX`(`54 4C 59 58`)와 일치해야 한다. reader는 이를 정수로 변환하지 않고 바이트 순서 그대로 비교해야 하며, 일치하지 않으면 파일을 거부해야 한다.
- 컨테이너 `header.version`(U32)은 reader가 지원하는 major 버전과 같아야 한다. 다르면 reader는 파일을 읽어서는 안 된다(6장 참고).
- 컨테이너 `header.version`은 JSON `header.version`(`x.y`)의 `x`와 일치해야 한다.
- `total_length`는 파일 전체의 실제 byte 길이와 정확히 같아야 한다.
- 청크는 정확히 2개이며, 순서는 JSON 청크 → BIN 청크로 고정된다.
- 각 청크의 `chunk_type`은 명세와 정확히 일치해야 한다(`JSON`, `BIN\0`).
- 각 청크의 `chunk_length`는 4의 배수여야 하며, U32 표현 범위를 넘어서는 안 된다(청크 하나의 최대 크기는 약 4GiB로 제한된다).
- JSON 청크의 패딩 바이트는 `0x20`만, BIN 청크의 패딩 바이트는 `0x00`만 허용한다. reader는 패딩 바이트를 데이터로 해석해서는 안 된다.

### Index 범위

- 모든 index 값은 다음 범위에 있어야 한다.(index 값의 유효성)
    ```text
    0 <= index < 배열 길이
    ```

### 이름의 제약 조건

- 이름은 빈 문자열이어선 안 된다.

- object.name은 objects 내에서 유일해야 한다.
- mesh.name은 meshes 내에서 유일해야 한다.
- attribute.name은 하나의 mesh 안에서만 유일해야 한다.

### Attribute data 제약 조건

- `topology` 구조체 의 필드 및 `semantic`이 `NONE`이 아닌 attribute는 저장 시 `component_type`과 `component_count`를 명세와 일치시켜야 한다.

### Transform 유효성

- object.transform의 선형(3x3) 부분은 행렬식이 0이 아니어야 한다(비특이 행렬). 0이면 유효하지 않은 파일이다.

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

## 6. 버전과 호환성

> [!IMPORTANT]
> 
> 버전은 문서에는 `x.y.z` 형식을, JSON `header.version`에는 `x.y` 형식을, 컨테이너 헤더에는 major(`x`)만 담은 U32를 사용한다.

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

- `0.y.z` 단계는 초기 실험 단계이므로, `y` 변경에서도 호환성이 보장되지 않을 수 있다.

- `z` 변경은 포맷 구조를 변경하지 않는 수정이기 때문에, JSON 파일에 포함시키지 않는다.

- Reader는 자신이 지원하지 않는 `x` 버전의 파일을 읽어서는 안 된다.

> [!NOTE]
> 
> 버전 정보는 세 곳에 나타나며 정밀도가 다르다.
> 
> - 문서(CHANGELOG 등) 버전: `x.y.z`
> - JSON `header.version`: `x.y`
> - 컨테이너 `header.version`(binary): `x`만 담은 U32
> 
> 컨테이너 버전은 reader가 JSON을 파싱하기 전, 바이너리 레벨에서 곧바로 지원 여부를 판단할 수 있도록 만든 값이다.
> 컨테이너 버전과 JSON `header.version`의 `x`가 다르면 유효하지 않은 파일이다.

---