# Topolyx 버전별 변경 기록 및 향후 계획

## 향후 계획

### v1.0 이후

- **우선사항:** 
  - rust importer 개발 시작
  - 레포지토리 분리(명세/blender extension, rust importer)
- parenting 지원
- `MESH` 외 다른 오브젝트 타입(`CURVE`, `POINT_CLOUD`, `INSTANCE` 등) 지원

---

## 변경 기록

### v1.0.0

- 포맷 이름을 `Topolyx`, 확장자명을 `.tlyx`로 변경
- 단일 통합 파일(model.tlyx) 정의, 기존 분리형 파일 대체
- 파일에 저장되는 좌표계 고정(Z+ up, Y+ forward, RIGHT handedness, CCW winding)
- semantic 별 object transform 적용 규칙 명시
- semantic에 `NORMAL` 추가

### v0.3.2

- 6장과 7장을 합치고, 해당 내용을 2장으로 위치 변경(`2. 포맷의 범위`)
  - 포맷 `1.x` 버전의 지원 범위를 명확히 정의

### v0.3.1

- 오타 수정

### v0.3.0

- `component_type`에 `I8`, `U8` 추가
- attribute에 `semantic` 필드 도입 (`POSITION`, `DIRECTION`, `ROTATION`, `TANGENT`, `COLOR`, `NONE`)
- 좌표계 변환이 필요한 semantic에 대한 변환 규칙 정의
- BOOL 표기를 `0b`에서 `0x`로 변경
- 유효성 조건에 "Attribute data 제약 조건" 추가 (semantic이 NONE이 아닌 attribute의 type/count 제약)

### v0.2.0

- 문서 구조 개편: "포맷 규칙"(구 3절), "저장 구조"(구 5절) 섹션 제거 → 필드별 제약으로 통합
- `header.version` 포맷을 `x.y.z`에서 `x.y`로 변경
- topology 각 필드에 `domain` 명시 추가
- `component_type`에 `I32`, `BOOL` 추가
- `coordinate_system`의 `meters_per_unit`에 NaN, Infinity 불가 제약 추가
- 유효성 조건 대폭 단순화: 세분화된 검증 규칙들을 일반화된 규칙으로 통합
    - 이름의 제약 조건 구체화: object.name은 objects 내, mesh.name은 meshes 내, attribute.name은 하나의 mesh 내에서 유일
- topology 필드의 중복 edge 및 self-edge 금지 규칙 명시
- 지원 범위에서 `F32`/`U32` 구체 명기 제거 → "고정 길이 attribute"로 일반화

### v0.1.0

> [!IMPORTANT]
> 
> v0.1 시기는 버전 정책 정의가 명확하지 않아 커밋에서는 0.0.1로 기록되어 있습니다.

- 초안 작성
- 분리형 파일 구조(JSON + Binary) 정의
- 기본 topology 필드 정의: positions, edges, corner_vertices, corner_edges, face_offsets
- 좌표계(`coordinate_system`) 필드 정의: up_axis, forward_axis, handedness, winding, meters_per_unit
- attribute data 공통 필드 정의: byte_offset, byte_length, component_type, component_count, element_count
- `component_type`: `F32`, `U32` 지원
- 4바이트 정렬, little-endian binary 규칙 정의
- 빈 mesh 허용 규칙 정의
- 유효성 조건 정의 (index 범위, 이름 제약, corner-edge 일관성, face offsets, data descriptor, 필수 메쉬 데이터 크기)
- 버전 정책 정의 (`x.y.z`: x=호환성 깨짐, y=기능 추가, z=사소한 수정)
- 지원 범위 및 향후 확장 정의