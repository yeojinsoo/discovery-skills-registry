# 분석 프로파일 (Analysis Profiles)

> **적용 대상**: §VALIDATE, §SUMMARY
> **의존**: `rules/analytical-method.md` (Phase 정의 및 원칙 참조)

각 서브커맨드가 분석 방법론을 어떻게 적용하는지 정의한다.

---

## 프로파일 요약

| 프로파일 | 목적 | Phase 구성 | 엄밀도 | HITL |
|---------|------|-----------|--------|------|
| validate | DoD 검증 (구현 후 검증) | A → C → S | Full | 없음 (자동 검증) |
| summary | PR 본문 근거 (구조 분석) | T → A(구축만) | Lite | 없음 |

---

## 1. validate 프로파일 — DoD 검증

### 목적

실행 완료된 플랜들의 구현 결과가 프로젝트의 Definition of Done (PROBLEM.md §3)을 실제로 달성했는지 논리적으로 검증한다.

### 입력

- 대상 플랜들의 PLAN.md, PROBLEM.md
- sessions.jsonl (실행 기록)
- progress.md (실행 상태)
- 실제 코드 변경사항 (git diff)
- knowledge.jsonl (실행 중 학습된 지식)

### 사전 질문

없음. 다음 기본값을 자동 적용:
- 분석 목적: "설계 결함 탐색" (구현물 vs DoD 갭 식별)
- 분해 깊이: 심층 (Deep)
- 관심 초점: 각 R-ID별 달성 여부에 집중

### Phase 구성

```
Phase A (R-ID별 달성 분석, 구축 + 비판)
  → Phase C (다관점 검증 — 코드 증거 기반)
  → Phase S (종합 판정)
```

Phase T는 생략한다. 시스템 구조는 이미 PLAN.md에 정의되어 있다.

### Phase A 상세 — R-ID별 분석

각 R-ID(해결 조건)에 대해 구축-비판 2패스를 적용:

**구축 패스**:
1. R-ID의 정의 확인 (PROBLEM.md §3에서)
2. 해당 R-ID를 커버하는 서브태스크 식별 (SPEC-TEST 매트릭스)
3. 서브태스크의 Oracle 실행 결과 확인 (sessions.jsonl)
4. 실제 코드에서 구현 증거 수집 (git diff에서 관련 변경)
5. R-ID 달성 여부 잠정 판정

**비판 패스**:
1. Oracle이 PASS인데 R-ID가 실제로는 미달성인 시나리오 탐색
2. 테스트가 커버하지 못하는 엣지 케이스 식별
3. 구현이 의도와 다르게 동작할 수 있는 조건
4. 회귀 리스크 (기존 기능에 대한 영향)

### Phase C 상세 — 코드 증거 기반 검증

고정 3관점 + 동적 1관점:

| 관점 | 초점 |
|------|------|
| 구조주의자 | R-ID → 서브태스크 → Oracle → 코드 체인의 정합성, 누락된 연결 |
| 회의론자 | Oracle PASS인데 실제 동작이 다른 시나리오, false positive |
| 실용주의자 | 프로덕션 환경에서의 실현 가능성, 운영 리스크 |
| 동적 (1개) | 프로젝트 도메인에 맞는 관점 (예: 보안, 성능, 데이터 정합성, 하위 호환성) |

**Phase C 실행 방식**: `references/phase-c-agent-protocol.md`에 따라 각 에이전트를 Agent tool로 병렬 실행한다. validate에서는 코드 증거 기반 검사 대상을 사용한다 (`phase-c-agent-protocol.md` §2 "코드 검증용" 참조).

**Phase C 구조**: validate는 2라운드로 실행한다: Round 1(독립 비판, Agent 병렬) → Round 2(판정관 종합, 메인 직접). 교차 반박 라운드는 LA 리팩토링에서 제거되었으며 PP도 동일 구조를 따른다.

### Phase S 상세 — 종합 판정

각 R-ID에 대한 최종 판정:

| 판정 | 의미 |
|------|------|
| ✅ ACHIEVED | 코드 증거 + Oracle PASS + 다관점 합의 |
| ⚠️ PARTIAL | 일부 달성, 미달성 조건 명시 |
| ❌ NOT_ACHIEVED | 미달성, 갭 상세 설명 |
| 🔍 UNCERTAIN | 증거 부족, 추가 검증 필요 |

전체 프로젝트 판정:
- **ALL_ACHIEVED**: 모든 R-ID가 ✅
- **PARTIAL**: 1개 이상 ⚠️ 또는 🔍, ❌ 없음
- **NOT_ACHIEVED**: 1개 이상 ❌

### Phase C 합의 → Phase S 연동

Phase C R2-a에서 "중대" 이상으로 합의된 비판이 특정 R-ID에 귀속되는 경우:
- 해당 비판이 R-ID DoD 문언의 범위 내이면 → R-ID 판정을 PARTIAL 이하로 하향
- R-ID DoD 범위 외이면 → PROBLEM.md §3 및 PLAN.md Scope를 인용하여 범위 외임을 명시, 권고 사항으로 분류

Phase C R2-e에서 "폐기"로 판정된 주장이 특정 R-ID의 유일한 달성 근거였으면, 해당 R-ID는 최대 🔍 UNCERTAIN으로 하향한다.

근거 없이 "중대" 합의를 무시하는 것은 금지한다.

### 확신도 판정 적용 규칙

확신도 판정 시 다음 외부 증거 기준을 적용한다 (PP validate 전용 기준):

**증거 유형 분류**:
- **[실행 검증]**: validate 시점에 빌드/테스트/API를 독립 실행하여 확인한 증거
- **[코드 검증]**: Read/Grep/Glob으로 코드의 존재와 구조를 확인한 증거
- **[기록 참조]**: sessions.jsonl의 과거 Oracle 결과를 참조한 증거
- **[논리적 추론]**: 외부 확인 없이 논리적으로 도출한 판단

**확신도 판정 기준**:
- **높음**: 모든 ✅ 판정 R-ID의 구축 패스에 [코드 검증] 이상의 증거가 포함되고, Phase C 합의 문제 중 "치명적" 심각도가 없을 때
- **중간**: 1개 이상의 R-ID에서 [기록 참조]만으로 판정했거나, Phase C에서 "중대" 미해결 쟁점이 있을 때
- **낮음**: 핵심 R-ID의 증거가 [논리적 추론]에 의존하거나, Phase C에서 "치명적" 미해결 쟁점이 있을 때

**보충 규칙**: [실행 검증] 미수행 R-ID가 있으면 "분석의 한계" 섹션에 해당 R-ID와 미수행 사유를 반드시 기록한다.

### 출력 형식

`${PROJ_DIR}/validations/{YYYY-MM-DD}-{slug}.md`에 저장.

```markdown
# Validation Report: {project_name}

> 검증 시각: {ts}
> 대상 플랜: {plan_name_1}, {plan_name_2}, ...
> 전체 판정: {ALL_ACHIEVED | PARTIAL | NOT_ACHIEVED}
> 확신도: {높음|중간|낮음}

## 해결 조건 달성 현황

| R-ID | 해결 조건 | 판정 | 근거 요약 |
|------|----------|------|----------|
| R-1 | {조건} | ✅ | {1줄 근거} |
| R-2 | {조건} | ⚠️ | {1줄 근거} |

## R-ID별 상세 분석

### R-1: {해결 조건}

**판정: ✅ ACHIEVED**

{Phase A 구축 결과 — 증거 체인 서술}
{Phase A 비판 결과 — 잠재 리스크}
{Phase C 다관점 검증 결과}

### R-2: {해결 조건}

**판정: ⚠️ PARTIAL**

{미달성 부분과 그 근거}
{달성을 위해 필요한 추가 작업}

## 다관점 검증 요약

### 에이전트별 비판 요약

| 에이전트 | 핵심 비판 | 잠정 판정 |
|---------|---------|---------|
| 구조주의자 | {1-2문장 요약} | {판정} |
| 회의론자 | {1-2문장 요약} | {판정} |
| 실용주의자 | {1-2문장 요약} | {판정} |
| {동적} | {1-2문장 요약} | {판정} |

### 합의된 사항

{Phase C Round 2 합의 목록}

### 미해결 쟁점

{Phase C Round 2 미해결 목록}

### 대안 구조 (해당 시)

{Phase C Round 2에서 제시된 대안 구조와 트레이드오프}

## 권고 사항

{추가 작업이 필요한 경우 구체적 권고}
{새 플랜 생성이 필요한 경우 안내}

## 분석의 한계

{이 검증에서 다루지 못한 영역}
```

---

## 2. summary 프로파일 — PR 본문 근거 분석

### 목적

최종 구현물의 구조를 PR 리뷰어 관점에서 파악하여, §SUMMARY.CREATE의 `group_changes`, `detect_decisions`, `verify_conditions` 단계에 근거를 제공한다.

### 입력

- git diff (최종 변경사항)
- PLAN.md, PROBLEM.md (프로젝트 플랜 컨텍스트)
- validation report (있으면 — validate 프로파일 결과)

### 사전 질문

없음. 고정 설정:
- 분석 목적: "구현 구조 설명" (PR 리뷰어가 변경사항을 이해할 수 있도록)
- 분해 깊이: 개요 수준
- 관심 초점: diff에 등장하는 컴포넌트에 집중

### Phase 구성

```
Phase T (diff 기반 구조 파악)
  → Phase A (구축 패스만 — 비판 생략)
```

Phase C, S 생략. PR 본문에는 객관적 구조 설명이 필요하지, 비판적 분석은 불필요.

### Phase T 상세 — diff 기반 구조 파악

git diff에서 변경된 컴포넌트를 식별하고 관계를 매핑:

1. 변경된 파일에서 주요 타입/클래스 추출 (Primary Node)
2. 1-hop 의존 관계 매핑 (Secondary Node)
3. 아키텍처 역할 분류 (ENTRY/SVC/VAL/DB/MSG/SCHED/EXT/CACHE — `summary-template.md` §4 기준)
4. 변경 흐름 파악 (어디서 시작해서 어디로 전파되는가)

### Phase A 상세 — 구축 패스만

각 논리 단위(변경 그룹)에 대해:
1. 이 변경이 왜 필요한가 (PROBLEM.md §1/§3 연계) → `problem`
2. 어떤 구조로 구현되었는가 (컴포넌트 관계) → `change` + `consequence`
3. 핵심 의사결정 지점 (대안이 있었을 곳) → `decisions`
4. 핵심 변환이 있는가 (반환 타입·데이터 구조·API 응답 형태 변경) → `before_after` (선택적)

**비판 패스는 생략**한다. summary는 "설명"이지 "검증"이 아니다.

### 출력 형식

별도 파일로 저장하지 않는다. §SUMMARY.CREATE의 내부 컨텍스트로만 사용된다.

분석 결과는 다음 구조로 §SUMMARY.CREATE에 전달:

```
structural_analysis:

  components:
    - name: "{class/type name}"
      role: "{ENTRY|SVC|VAL|DB|...}"
      status: "{NEW|MODIFIED|REFERENCE}"
      relations:
        - target: "{other component}"
          type: "{호출|검증|쿼리|저장|...}"

  logical_units:
    - name: "{논리 단위 이름}"
      problem: "{이 단위가 해결하는 문제/한계 — 1문장}"
      change: "{무엇을 어떻게 바꿨는가 — 1-2문장}"
      consequence: "{이 변경으로 가능해진 것 — 1문장}"
      files: ["{file1}", "{file2}"]
      before_after:        # optional, 핵심 변환이 없으면 null
        before: "{변경 전 상태 — 코드 스니펫, 데이터 구조, API 응답 등}"
        after: "{변경 후 상태}"

  decisions:
    - title: "{의사결정 제목}"
      alternatives: ["{대안1}", "{대안2}"]
      chosen: "{선택한 방향}"
      reason: "{선택 이유}"

  condition_evidence:
    - s_id: "S-1"
      condition: "{해결 조건}"
      implemented: true
      evidence: "{diff에서의 근거}"
```

이 구조가 §SUMMARY.CREATE의 `group_changes`, `detect_decisions`, `verify_conditions` 단계에 근거 데이터를 제공하며, `compose_body`에서 PR 템플릿(`summary-template.md`)에 매핑된다.
