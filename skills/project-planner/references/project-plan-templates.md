# Project Plan Templates — PROBLEM.md + PLAN.md 표준 구조

> ⚠️ **인라인 통합 안내**: 이 내용은 `commands/create.md` §4, §9에 인라인 통합됨. 수정 시 command 파일을 직접 수정할 것.

## PROBLEM.md 템플릿

```markdown
# {plan-name} — Problem Definition

## 1. 증상
{관찰 가능한 실패 현상}

**재현**: {누구든 동일하게 관찰할 수 있는 명령/경로}

## 2. 원인
**[확인됨]** 또는 **[가설]** {근본 원인 서술}
{가설이면 → 확인 방법: ...}

## 3. 해결 조건
- R-1: {상태 서술 "~이다"} → {검증 명령/방법}
- R-2: {상태 서술 "~이다"} → {검증 명령/방법}

## 4. 영향 범위 (선택)
- **지속 시**: {방치하면 뭐가 깨지는가}
- **수정 시**: {고치다가 뭐가 깨질 수 있는가}
```

### 섹션→PLAN 흐름

| 섹션 | 필수 | PLAN.md로의 흐름 |
|------|:----:|-----------------|
| §1 증상 + 재현 | 필수 | 재현 경로를 뒤집으면 Acceptance Gate Oracle |
| §2 원인 + 확인/가설 | 필수 | [확인됨] → FIX→VERIFY. [가설] → VALIDATE→FIX→VERIFY |
| §3 해결 조건 + 검증 | 필수 | Contract Statement의 직접 입력. 각 조건에 R-ID(R-1, R-2, ...) 부여 — 하류 Gate에서 추적 |
| §4 영향 범위 | 선택 | Scope Boundary OUT + Regression 범위 |

---

## VALIDATE 서브태스크 템플릿 (원인 미상 케이스)

§2가 `[가설] 원인 미상`인 경우, PLAN.md 첫 서브태스크로 삽입:

```markdown
### S0: 원인 조사 (VALIDATE) [ ]

**Action**: §2 원인을 특정하기 위해 로그/코드/환경을 조사한다. 결과에 따라 §2를 [확인됨]으로 갱신하고, 후속 서브태스크의 Action을 확정한다.

| Predicate | Oracle |
|-----------|--------|
| §2 원인이 [확인됨]으로 갱신되었다 | PROBLEM.md §2에 `**[확인됨]**` 태그가 존재한다 |
| 확인된 원인에 대응하는 수정 방향이 기술되었다 | PROBLEM.md §2에 "수정 방향: ..." 텍스트가 존재한다 |
```

> VALIDATE 통과 후 `[가설]` → `[확인됨]` 전환. 후속 FIX/VERIFY 서브태스크 진행.

---

## PLAN.md 템플릿

```markdown
# {plan-name} — Execution Plan

> **버전**: v1.0 | **상태**: PLANNED | **최초 작성**: {YYYY-MM-DD}

---

## Context Sources

<!-- Seed 있는 경우 -->
- [x] **context/003-api-design.md ← Seed** (이 프로젝트 플랜의 기반)
- [x] context/001-initial.md (배경 참조)
- [ ] context/004-new-constraint.md  ← 미반영 (update 대상)

<!-- Seed 없는 경우 -->
_(없음 — 사용자 입력 기반 생성)_

---

## Section 0: Contract Statement / Scope / SPEC-TEST

### Contract Statement

이 프로젝트 플랜은 {입력}을 받아 {출력}을 생산한다.

**§3 추적**: R-1, R-2, ... → 이 Contract가 커버하는 PROBLEM.md §3 R-ID 전체. 누락 R-ID가 있으면 G1 실패.

---

### Scope Boundary

| 항목 | IN/OUT | 근거 / 위임처 |
|------|:------:|--------------|
| {이 프로젝트 플랜의 책임} | **IN** | {왜 이 프로젝트 플랜의 범위인가} |
| {이 프로젝트 플랜의 책임 아님} | **OUT** | {위임처: 프로젝트 플랜 이름/시스템/미정} |

---

### SPEC-to-TEST 매트릭스

| R-ID | SPEC IN 항목 | 검증 서브태스크 | Oracle 등급 | 커버리지 |
|:----:|-------------|--------------|:-----------:|:-------:|
| R-1 | {IN 항목} | {S1, S2...} | Strong/Weak | COVERED |
| R-2 | {IN 항목} | {S3} | Strong/Weak | COVERED |

> **커버리지 집합 연산** (G3 기계적 검증):
> `R_all = {R-1, R-2, ...}` (PROBLEM.md §3 전체 − G2 OUT)
> `R_covered = {매트릭스에서 COVERED인 R-ID}`
> `R_uncovered = R_all − R_covered`
> R_uncovered ≠ ∅ → **G3 실패**
>
> **Q1 (과소검증)**: Contract R-ID 중 Predicate로 미검증인 R-ID = ?
> **Q2 (과대검증)**: Predicate R-ID 중 Contract에 미대응인 R-ID = ?
> **Q3 (원칙-테스트 괴리)**: ...

---

### 설계 원칙

| ID | 원칙 | Testability |
|:--:|------|:-----------:|
| P-1 | {원칙 서술} | Yes/Partial/No (등급) |

---

### 비결정론 격리 매트릭스

| 소스 | 서브태스크 | 격리 방법 | Tier |
|------|----------|----------|:----:|
| {비결정론 소스} | {서브태스크} | {fixture/OUT} | A/B |

---

## Acceptance Gate

\```
ACCEPT iff S1 ∧ S2 ∧ ... ∧ SN
\```

---

## Subtasks

### S1: {title} [ ]

**Action**: {수행 절차}

| Predicate | Oracle | R-ID |
|-----------|--------|:----:|
| {상태 서술} | {검증 명령} | R-1 |

---

## 의존성 그래프 (DAG)

\```
S1 ──┐
     ├── S3 ── S4
S2 ──┘
\```

---

## 전체 체크리스트

\```
[ ] S1: {title}
[ ] S2: {title}
...
\```
```

---

## 상태 전이

| 상태 | 의미 | 전이 조건 |
|------|------|----------|
| PLANNED | 작성 완료, 미실행 | 최초 작성 시 |
| IN_PROGRESS | 실행 중 | 첫 서브태스크 착수 시 |
| DONE | 완료 | Acceptance Gate + CRG 통과 |
| NEEDS_REVIEW | CRG 3회 실패, 인간 검토 필요 | CRG 3회 모두 NEEDS_FIX. `critical-review-gate.md` §5 참조 |
| ABANDONED | 폐기 | 의도적 중단 결정 |

> **Exploration Plan 상태 전이 추가 규칙**: Exploration Plan의 DONE 전이 조건은 Acceptance Gate 통과 **+ 후속 프로젝트 플랜 PROBLEM.md 존재**(S2 Oracle로 검증)이다. 후속 프로젝트 플랜이 존재하지 않으면 DONE으로 전이할 수 없다.

---

## Exploration Plan 템플릿 (Wicked Problem 케이스)

project-planner create G0에서 Wicked Problem으로 분류된 경우 사용하는 템플릿. 일반 PLAN.md와의 차이점:
- Contract 산출물이 **탐색 결과 문서 + 후속 프로젝트 플랜 명세**
- 서브태스크가 **Discover→Define** 2단계로 제한 (구현은 후속 프로젝트 플랜으로 위임)
- Oracle 등급 Weak 허용 (탐색 산출물은 구조만 검증 가능)

```markdown
# {plan-name} — Exploration Plan

> **버전**: v1.0 | **상태**: PLANNED | **유형**: Exploration | **최초 작성**: {YYYY-MM-DD}

---

## Section 0: Contract Statement / Scope / SPEC-TEST

### Contract Statement

이 프로젝트 플랜은 {미확정 문제/기술적 불확실성}을 입력으로 받아 **탐색 결과 문서 + 후속 실행 프로젝트 플랜 명세**를 생산한다.

---

### Scope Boundary

| 항목 | IN/OUT | 근거 / 위임처 |
|------|:------:|--------------|
| 대안 탐색 및 비교 분석 | **IN** | 탐색 프로젝트 플랜의 핵심 책임 |
| 최적 접근법 선정 및 문서화 | **IN** | Define 단계에서 확정 |
| 실제 구현 (Design/Deliver) | **OUT** | 후속 실행 프로젝트 플랜으로 위임 |

---

### SPEC-to-TEST 매트릭스

| SPEC IN 항목 | 검증 서브태스크 | Oracle 등급 | 커버리지 |
|-------------|--------------|:-----------:|:-------:|
| 대안 탐색 | S1 (Discover) | Weak | COVERED |
| 접근법 선정 | S2 (Define) | Weak | COVERED |

> Oracle 등급 Weak 허용 사유: 탐색 산출물의 "품질"은 주관적이므로 구조(존재 여부, 형식)만 기계적 검증. 내용의 적절성은 후속 프로젝트 플랜의 VALIDATE에서 확인.

> **Q1 (과소검증)**: Contract 산출물(탐색 결과 문서, 후속 프로젝트 플랜 명세) 모두 S1/S2의 Predicate로 커버됨
> **Q2 (과대검증)**: 해당 없음 — S1/S2 모두 Contract IN에 대응
> **Q3 (원칙-테스트 괴리)**: P-1/P-2는 구조 검증 가능하므로 Weak Oracle로 커버 가능

---

### 설계 원칙

| ID | 원칙 | Testability |
|:--:|------|:-----------:|
| P-1 | 최소 2개 이상의 대안을 비교 분석한다 | Yes (대안 수 카운트) |
| P-2 | 각 대안의 장단점과 리스크를 명시한다 | Partial (구조 검증) |

---

### 비결정론 격리 매트릭스

| 소스 | 서브태스크 | 격리 방법 | Tier |
|------|----------|----------|:----:|
| (탐색 프로젝트 플랜은 일반적으로 비결정론 소스 없음 — 해당 시 행 추가) | — | — | — |

---

## Acceptance Gate

\```
ACCEPT iff S1 ∧ S2
\```

---

## Subtasks

### S1: Discover — 대안 탐색 [ ]

**Action**: {문제 영역}에 대해 가능한 접근법을 조사한다. 최소 2개 대안을 식별하고 각각의 특성을 기술한다.

| Predicate | Oracle |
|-----------|--------|
| 2개 이상의 대안이 탐색 결과 문서에 기술되었다 | 탐색 문서에 `## 대안` 섹션이 존재하고 2개+ 항목이 있다 |
| 각 대안에 장단점이 명시되었다 | 각 대안 항목에 `장점:`, `단점:` 또는 동등 구조가 존재한다 |

### S2: Define — 접근법 선정 + 후속 프로젝트 플랜 명세 [ ]

**Action**: S1의 대안을 비교 분석하여 최적 접근법을 선정하고, 후속 실행 프로젝트 플랜의 PROBLEM.md 초안을 작성한다.

| Predicate | Oracle |
|-----------|--------|
| 선정된 접근법과 선정 사유가 문서화되었다 | 탐색 문서에 `## 선정 결과` 섹션이 존재한다 |
| 후속 프로젝트 플랜의 PROBLEM.md 초안이 존재한다 | `plans/{후속-plan-name}/PROBLEM.md`가 §1~§3을 포함한다 (초안이므로 §2 [가설] 허용) |

> **후속 프로젝트 플랜명 규칙**: `{현재-plan-name}-impl` 접미사를 기본으로 사용한다. 탐색 결과가 여러 구현 프로젝트 플랜으로 분기되면 `-impl-{N}` 접미사를 사용한다. 사용자가 다른 이름을 지정하면 해당 이름을 우선한다.

---

## 의존성 그래프 (DAG)

\```
S1 ── S2
\```

---

## 전체 체크리스트

\```
[ ] S1: Discover — 대안 탐색
[ ] S2: Define — 접근법 선정 + 후속 프로젝트 플랜 명세
\```
```
