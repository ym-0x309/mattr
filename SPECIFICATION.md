# MATTR 포맷 명세(v0.2.0)

> [!NOTE]
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

### Byte offset

Binary buffer 시작점으로부터 데이터 배열 시작점까지의 바이트 거리이다.

### Topology

Vertex, edge, face, corner 사이의 연결 관계를 나타내는 필수 메쉬 데이터이다.

### Attribute

특정 domain의 각 element에 대응하는 추가 데이터 배열이다.

---

## 3. 파일 구성

```text
test_mesh.mattr.json
test_mesh.mattr.bin
```

통합형 파일(.mattr)은 `0.y.z`의 지원 범위에 포함되지 않는다.
JSON과 Binary를 하나의 파일에 저장하는 chunk 기반 컨테이너 구조는
향후 버전에서 별도로 정의한다.

`0.y.z`는 분리형 파일 구조(model.mattr.json + model.mattr.bin)만 지원한다.

### JSON

JSON에는 메타 데이터를 저장하며, UTF-8로 인코딩한다.

```text
포맷 버전
오브젝트 이름
메쉬 이름
좌표계
데이터 개수
각 데이터 byte_offset
Attribute 이름/domain/type
```

#### 저장 방식 및 필드별 제약

##### 파일 기본 정보

- `header`: 포맷 정보

    - `format`: `MATTR`
    - `version`: `x.y`

- `buffer`: binary 파일 정보

    - `uri`: binary 파일명
    - `byte_length`: binary 파일 길이

##### 좌표계 정보

- `coordinate_system`: 월드 좌표계 정의

    - `up_axis`: `+X`, `-X`, `+Y`, `-Y`, `+Z`, `-Z` 중 하나
    - `forward_axis`: `up_axis`의 조건과 같음. 다만, `up_axis`의 축과 평행하지 않아야 함
    - `handedness`: `RIGHT`, `LEFT` 중 하나
    - `winding`: `CCW`, `CW` 중 하나.
    - `meters_per_unit`: 좌표계에서 1m의 길이.
        - 0 초과의 유효한 수여야 한다.(NaN, Infinity 불가)

##### 오브젝트 정보

- object: `objects` 배열의 내용물

    - `name`: 오브젝트 이름.
    - `type`: 현재 `MESH`만 지원.
    - `index`: 뒤의 `type`마다 배정된 배열에서의 index. 예를 들어 type이 MESH인 경우 index는 meshes 배열의 index를 의미한다.
    - `transform`: 원본 메쉬에 적용되는 4x4 변환 행렬. 열 우선(Column-Major) 순서.

        - 벡터는 열 벡터(Column Vector)로 취급하며, 변환은 `M × v` 순서로 적용한다.
        - 메쉬 별 local axis는 `transform`을 통해 world space로 변환됨.

현재 parenting은 미지원.

##### Attribute 정보

- attribute data: `topology` 데이터와 `attributes`에 공통으로 사용되는 필드

    - `byte_offset`: binary buffer 시작점 기준 데이터 배열의 byte offset
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
            byte_offset + byte_length <= buffer.byte_length
            ```
    - `component_type`: 각 component를 저장하는 binary scalar type. 현재 지원되는 component는 다음과 같음.

        | Component type |      크기 | 의미                             |
        | -------------- | ------: | ------------------------------ |
        | `F32`          | 4 bytes | IEEE 754 32비트 부동소수점 |
        | `I32`          | 4 bytes | 부호 있는 32비트 정수        |
        | `U32`          | 4 bytes | 부호 없는 32비트 정수        |
        |`BOOL`|1 byte|1바이트 boolean 값. `0b00000000`는 `false`, `0b00000001`는 `true`, 나머지 값은 유효하지 않음.|
    - `component_count`: 하나의 element를 구성하는 component 수. 1 이상이어야 한다.
    - `element_count`: 배열에 저장된 element의 총 개수. 후술할 `face_offsets`의 경우를 제외하면 모두 해당 도메인의 `element_count`와 일치해야 한다.

##### 메쉬 정보

> [!IMPORTANT]
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
        - `data`: attribute data

<details><summary>Blender의 Default Cube의 저장 예시</summary>

```json
{
    "header": {
        "format": "MATTR",
        "version": "0.2"
    },

    "buffer": {
        "uri": "test_mesh.bin",
        "byte_length": 612
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
                    "data": {
                        "byte_offset": 420,
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
</details>

### Binary

```text
positions
edges
corner_vertices
corner_edges
face_offsets
attribute values
````

- Binary에는 각 배열을 연속적으로 저장한다.

- reader는 JSON에 기록된 `byte_offset`을 기준으로 데이터를 읽어야 하며 배열의 실제 저장 순서에 의존해서는 안 된다.

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

        padding
        2 bytes

        offset 420

        next array
        ```
        </details>

    - 또한 사용되지 않은 구역의 byte는 모두 0으로 저장해야 한다.

---

## 4. 유효성 조건

여러 필드의 관계를 검사하는 제약 및 공통 제약.

### Index 범위

- 모든 index 값은 다음 범위에 있어야 한다.(index 값의 유효성)
    ```text
    0 <= index < 배열 길이
    ```

### 이름의 제약 조건

- 이름은 빈 문자열이어선 안 된다
- 파일 내의 다른 이름과 겹쳐서는 안 된다.

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

## 5. 버전과 호환성

버전은 문서에는 `x.y.z` 형식을, 포맷에는 `x.y` 형식을 사용한다.

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

---

## 6. 지원 범위

### 현재 버전에서 지원

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
- 고정 길이 attribute
- 빈 mesh
- Little-endian binary
- 4바이트 정렬

### 현재 버전에서 지원하지 않음

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

## 7. 향후 확장

다음 기능은 향후 버전에서 검토한다.

- JSON과 binary를 하나의 파일에 저장하는 chunk 기반 컨테이너
- 자연 정렬 또는 8바이트 정렬
- Attribute semantic
- Material 및 material slot
- Vertex group 및 skin weight
- Shape key
- Object parenting
- Binary 압축
- MESH 외 다른 타입
