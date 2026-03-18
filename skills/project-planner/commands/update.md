## §UPDATE — 프로젝트 플랜 수정

프로젝트 플랜 수정 워크플로우를 실행한다.

### SOT 참조

- `${CLAUDE_SKILL_DIR}/rules/spec-test-alignment.md` — SPEC-TEST 정합성 (R-006)
- `${CLAUDE_SKILL_DIR}/rules/project-plan-quality-characteristics.md` — 품질 특성 (R-001)

Cascade 규칙: `${CLAUDE_SKILL_DIR}/references/cascade-map.md`
버전 관리: `${CLAUDE_SKILL_DIR}/references/version-rules.md`
정합성 검증: `${CLAUDE_SKILL_DIR}/references/consistency-guide.md`

### StateGraph

```
load_plan → validate_state → context_delta_check
  → identify_change → cascade_analysis
  → GATE-CHANGE-CONFIRM(AskUserQuestion)
  → apply_changes → re_verify → update_artifacts → report
  → GATE-AUTO-EXEC → [YES] invoke /project-planner exec {plan-name}
```

### Drift-triggered Entry

project-planner-exec의 drift_check 또는 stagnation 감지에서 호출된 경우:
- `identify_change`에서 sessions.jsonl 마지막 레코드의 `pathological` 필드 자동 로드
- pathological 유형별 변경 방향 권고:
  - `drift` → Scope Boundary 재검토
  - `stagnation`/`early_stagnation`/`diminishing_returns` → Predicate/Oracle 재설계
  - `oscillation` → Contract Statement 실현 가능성 재검토

### 변경 불변식

- **append-only**: sessions.jsonl, knowledge.jsonl은 수정/삭제 금지
- **PLAN.md / PROBLEM.md**: Edit 도구로 수정

### 1. load_plan

```
Read: ${PLAN_DIR}/PLAN.md
Read: ${PLAN_DIR}/PROBLEM.md
Read: ${PLAN_DIR}/sessions.jsonl
Read: ${KNOW_FILE}
# CRG 비평 파일 (존재 시)
ls ${PLAN_DIR}/crg-*.md 2>/dev/null → 존재하면 최신 파일 Read
```

`${PLAN_DIR}/PLAN.md` 미존재 → 에러: "프로젝트 플랜 '{plan_name}'을 찾을 수 없습니다."

### 2. validate_state

`${CLAUDE_SKILL_DIR}/references/version-rules.md` 참조:

| 상태 | 처리 |
|------|------|
| PLANNED | 전체 수정 가능 |
| IN_PROGRESS | 변경 사유 확인 후 진행 (아래 조건부 HITL 참조) |
| DONE | 수정 불가. 새 프로젝트 플랜 생성 안내 또는 사용자 명시 요청 시 IN_PROGRESS 전환 |
| NEEDS_REVIEW | Contract/Scope 재설계만 허용. AskUserQuestion 후 IN_PROGRESS 전환 |
| ABANDONED | 수정 불가 |

**IN_PROGRESS 조건부 HITL**:
- Drift-triggered Entry → sessions.jsonl 마지막 `pathological` 필드를 변경 사유로 자동 채택. AskUserQuestion 생략
- CRG-triggered Entry → crg-{N}.md의 blocker/warning 항목이 변경 사유. AskUserQuestion 생략
- 사용자 직접 호출 + 변경 내용이 입력에 포함됨 → 입력에서 사유 추출. AskUserQuestion 생략
- 사용자 직접 호출 + 변경 내용 미포함 → AskUserQuestion으로 변경 사유 확인

### 2-1. context_delta_check — 신규 컨텍스트 파일 감지

load_plan 직후 수행:

```bash
ls "${CTX_DIR}" 2>/dev/null | grep -E '^[0-9]{3}-.*\.md$' | sort
```

PLAN.md `## Context Sources`의 `[ ]` 항목(미반영 파일)을 식별:
- 미반영 파일 0개 → 일반 update 흐름 진행
- 미반영 파일 1개+ 존재 → 다음 안내 출력 후 진행:

```
📌 미반영 컨텍스트 파일:
  - context/003-discovered-constraint.md
  - context/004-perf-requirement.md

이 컨텍스트를 기반으로 프로젝트 플랜을 보완합니다.
새 컨텍스트 파일을 우선 분석하고, 기존 파일은 배경으로 참조합니다.
```

이후 `identify_change`에서 미반영 파일 내용을 변경 드라이버로 처리:
- 미반영 파일의 §1/§2/§3 내용 → 새 서브태스크 후보 도출
- 미반영 파일의 `## 반요구사항 (Anti-Requirements)` 섹션 → Scope Boundary 갱신 후보:
  - 범위 제외 유형 → OUT 항목 추가 (위임처 필수)
  - 보존/불변 유형 → IN (Invariant) 항목 추가
  - 분류 모호 시 GATE-CHANGE-CONFIRM에서 사용자 확인
- 기존 서브태스크와 충돌하거나 확장이 필요한 항목 식별
- cascade_analysis에서 영향 범위 산출 후 GATE-CHANGE-CONFIRM

update 완료 시 반영된 파일을 PLAN.md `## Context Sources`에서 `[x]`로 업데이트.

### 3. identify_change

사용자 요청을 프로젝트 플랜 구성 요소에 매핑. Contract/Predicate 변경 시 안티패턴 탐지.

**CRG-triggered Entry**: `${PLAN_DIR}/crg-*.md` 파일이 존재하면, 최신 crg-{N}.md를 읽어 blocker/warning 항목을 변경 드라이버로 자동 매핑한다.

**Scope Boundary 변경 시**: 기존 `**IN** (Invariant)` 표시 항목의 보존 여부를 확인한다. Invariant 항목을 변경/삭제하려면 AskUserQuestion으로 사용자에게 확인 후 진행한다.

### 4. cascade_analysis

`${CLAUDE_SKILL_DIR}/references/cascade-map.md` 참조:
1. **의미적 동등성 판정** (cascade-map.md §0단계): 1차 R-ID 집합 비교 → R-ID 동일 시 2차 Contract 출력 명사 비교로 변경 규모 판정
2. 변경 지점에서 하위 요소로의 전파 경로 도출
3. 영향받는 Gate 목록 산출 (G0~G4)
4. 이미 완료(`[x]`)된 서브태스크 중 영향받는 것 식별 — R-ID 기반: 변경된 R-ID를 SPEC-to-TEST 매트릭스에서 조회하여 해당 R-ID를 커버하는 서브태스크 목록 추출. 이 서브태스크 중 `[x]`인 것 = 리셋 대상. Scope Boundary의 IN (Invariant) 항목이 변경/삭제된 경우, exec Worker(변경 금지 영역 해제), Verifier(침범 확인 대상 변경), CRG Agent Z(우선 회귀 대상 변경)에 대한 영향도 GATE-CHANGE-CONFIRM에 명시

### 5. GATE-CHANGE-CONFIRM

AskUserQuestion으로 cascade 분석 결과 제시 + "이 범위로 변경을 진행할까요?"

### 6. apply_changes

순서대로 적용 (상위 → 하위): PROBLEM.md → Section 0 → Subtasks → Acceptance Gate → DAG → 체크리스트.

**Scope Boundary 수정 시**: 기존 `**IN** (Invariant)` 표시를 보존한다. 신규 IN 항목에 Invariant 분류가 필요하면 create G2 Step 2-1의 반요구사항 유형 기준(범위 제외→OUT, 보존/불변→IN Invariant)을 적용한다.

**Predicate/Oracle 수정 시**: Prerequisites 열을 보존한다. Oracle 변경 시 Prerequisites도 재검토한다 (Safe/Stateful 등급 변경 가능성). **레거시 PLAN.md(Prerequisites 열 미존재)**: apply_changes 시 Prerequisites 열을 추가하고, 기존 서브태스크의 값은 모두 "—"(미지정)으로 채운다.

**SPEC-to-TEST 매트릭스 수정 시**: 보강 경로 열을 보존한다. Oracle 등급 변경(Strong↔Weak) 시 보강 경로를 추가/제거한다.

### 6-A. 구조 검증 (기계적)

apply_changes 완료 후, PLAN.md 필수 구조를 기계적으로 검증한다 (create §9-A와 동일):

| 확인 항목 | 검증 방법 |
|---------|---------|
| Acceptance Gate 공식 존재 | `ACCEPT iff` + "∧" 연산자 존재 확인 |
| 비결정론 격리 매트릭스 존재 | grep: "## 비결정론 격리" |
| 의존성 DAG 존재 | grep: "## 의존성 그래프" |
| Prerequisites 열 존재 | 서브태스크 Predicate/Oracle 테이블에 "Prerequisites" 열 존재 확인 |

검증 결과: 모두 통과 → re_verify 진행 / 1개+ 실패 → 수정 후 재검증.

### 7. re_verify

`${CLAUDE_SKILL_DIR}/references/consistency-guide.md` 체크리스트 실행:
- PROBLEM ↔ Contract ↔ Scope ↔ SPEC-TEST ↔ Q1/Q2/Q3

### 8. update_artifacts + report

변경된 파일 저장. IN_PROGRESS에서 서브태스크 리셋 발생 시:
1. sessions.jsonl에 `aborted` 레코드 append
2. `${PROGRESS_FILE}` 갱신: 리셋된 서브태스크를 완료 목록에서 제거, "현재/다음"을 첫 번째 미완료 서브태스크로 재설정

`${HIST_FILE}`에 append:
```jsonl
{"type":"plan_updated","name":"{plan_name}","ts":"{ISO 8601}","changes":"{변경 요약}"}
```

### 9. GATE-AUTO-EXEC

apply_changes에서 변경이 1건 이상 적용된 경우 AskUserQuestion을 발동한다. 변경 유형에 따라 안내 문구를 분기:

| 우선순위 | 조건 | 안내 문구 |
|---------|------|----------|
| 1 | 리셋된 서브태스크 1개+ | "리셋된 서브태스크가 있습니다. 재실행할까요?" |
| 2 | 신규 서브태스크 추가 (리셋 없음) | "신규 서브태스크가 추가되었습니다. 실행할까요?" |
| 3 | Scope/Contract만 변경 (리셋·신규 없음) | "변경사항이 적용되었습니다. 재실행할까요?" |

- [1] 바로 재실행 → Skill 도구로 `/project-planner exec {plan_name} --project {project_name} --repo {repo_slug}` 호출
- [2] 나중에 실행

apply_changes에서 적용된 변경이 0건(사용자가 GATE-CHANGE-CONFIRM에서 모든 변경을 거부 등)이면 GATE를 발동하지 않고 report에서 종료한다.

**exec↔update 순환 상한**: exec→update→exec 전환은 **최대 3회**. sessions.jsonl의 연속 `aborted` 레코드로 카운트. 3회 도달 시 자동 재실행 선택지를 제거하고 "[1] 나중에 실행 [2] 프로젝트 플랜 재설계 (/project-planner create)" 제시.
