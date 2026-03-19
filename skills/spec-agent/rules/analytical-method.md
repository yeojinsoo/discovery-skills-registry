# 분석 방법론 (Analytical Method)

> **적용 대상**: §VALIDATE, §SUMMARY
> **원칙 소스**: 각 Phase 모듈에 내장 (phase-analysis.md, phase-critique.md, phase-synthesis.md). 깊이↔실행 매트릭스와 알려진 한계는 logical-analysis SKILL.md에 위치.

> **SA vs LA 적용 범위**: Spec Agent(SA) command에서는 `analysis-profiles.md`의 해당 프로파일만 참조하면 충분하다. 이 문서(analytical-method.md)는 Logical Analysis(LA) 스킬의 원칙 문서이며, SA에서는 프로파일을 통해 간접 참조된다.

---

## SOT 모듈 참조

실행 전 참조할 LA 모듈:

프로파일별 모듈:
- **validate**: `${LA_MODULES}/phase-analysis.md`, `${LA_MODULES}/phase-critique.md`, `${LA_MODULES}/phase-synthesis.md`
- **summary**: 직접 참조 없음 — `analysis-profiles.md` summary 프로파일에 Phase T/A가 인라인 정의됨

> 참고: `principles.md`, `narrative-rules.md`는 LA 리팩토링으로 삭제됨. 핵심 원칙은 각 Phase 모듈에 내장, 서술 규칙은 LA SKILL.md "서술 규칙" 섹션으로 이동.

---

## legitimate divergence (의도적 분기)

다음 항목은 SOT 모듈과 독립적으로 유지한다. SOT 모듈을 업데이트해도 이 항목들은 spec-agent 방식을 따른다:

- **출력 형식**: SA는 프로파일별 (context/validation/summary), LA는 분석 보고서
- **HITL 흐름**: 각 스킬이 독자적으로 관리
- **validate Phase A**: R-ID 기반 구축-비판 (코드 증거 중심) — SOT의 개념 분해와 다름
- **Phase C 검사 대상**: 코드 검증용은 SA 전용 (`phase-c-agent-protocol.md`)
- **validate Phase C**: LA에서 교차 반박 라운드가 제거되었으며, SA validate도 동일하게 2라운드(R1 독립 비판 → R2 판정) 구조를 따른다
- **validate Phase C R2 판정 범위**: validate는 R2-a~R2-g 전체 범위를 작성한다. 단, R2-c(분해 손실 점검)와 R2-d(구조 수정안)는 R-ID 달성 판정에 관련된 비판이 있을 때에만 작성한다.

---

## spec-agent 전용 확장

### Phase A/S validate 모드

`analysis-profiles.md` validate 프로파일 참조.
