# AGENTS.md

## 현재 목표

- 기존 모델 포맷(.obj, .glb)에서 vertex attribute만 지원하는 한계에서 벗어나 edge, face 등 다른 도메인의 attribute도 저장되는 포맷 만들기

## 참고 문서
- 프로젝트 내부
    - `SPECIFICATION.md`: 포맷 명세
    - `blender_mattr_exporter/docs/`
        - `overview.md`: Extension 전체 구조 및 설계
        - `architecture.md`: 파일별 api 시그니처 및 역할
- 프로젝트 외부
    - `sources/blender_addon_docs/`: 블렌더 문서의 애드온 부분 발췌
    - `sources/blender_mesh_store/`: 블렌더 문서의 메쉬 저장 방식 부분 발췌
    - `sources/blender_python_reference_5_1/`: 블렌더 문서의 파이썬 레퍼런스 부분 발췌

## 주의사항

- 모호한 점이 있으면 혼자 추측해서 결정하지 말고 사용자에게 물어보기
- 반드시 프로젝트 내부 문서는 참고할 것