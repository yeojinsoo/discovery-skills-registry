## §CREATE — 신규 프로젝트 플랜 생성

신규 프로젝트 플랜 생성 워크플로우를 실행한다.

### Gate별 판정 기준 (인라인화됨)

| Gate | 참조 | 설명 |
|------|------|------|
| G0 | 이 파일 내 인라인 (§3) | Ambiguity Score 공식 + 채점 기준 |
| G1~G4 | 이 파일 내 인라인 (§5~§8) | Contract/Scope/SPEC-TEST/정합성 |
| 템플릿 | 이 파일 내 인라인 (§9) | PROBLEM.md/PLAN.md 템플릿 |
| 안티패턴 | 이 파일 내 인라인 (§5) | Contract 안티패턴 목록 |

### 공유 SOT (스텝별 참조)

| 스텝 | 참조 | 설명 |
|------|------|------|
| §2-1 context_load | `rules/knowledge-conventions.md §6` | Knowledge 인출 전략 |
| G0~G4 gate_decision | `rules/session-log-conventions.md §3-4` | gate_decision 스키마 |

### StateGraph

```
parse_input → name_validate → seed_select → context_load
  → G0(PROBLEM HITL)
  → [Wicked?] → YES: wicked_plan(탐색 Contract) → G1 → ...
              → NO: write_problem → G1(Contract) → G2(Scope) → G3(SPEC-TEST) → G4(Q1/Q2/Q3)
  → build_plan → GATE-FINAL → write_artifacts → report
  → GATE-AUTO-EXEC → [YES] invoke /project-planner exec {plan-name} --project {project_name} --repo {repo_slug}
```

### 1. parse_input — 사용자 입력 파싱

사용자가 제공한 텍스트에서 다음을 추출:
- 프로젝트 플랜 이름 후보 (또는 문제 설명에서 kebab-case 생성)
- §1 증상 정보, §2 원인 정보, §3 해결 조건 정보
- Contract/Scope 관련 힌트

### 2. name_validate — 이름 검증

1. kebab-case 변환 (영문 소문자 + 하이픈)
2. `ls "${PLANS_DIR}"` 로 기존 프로젝트 플랜 디렉토리 확인
3. 디렉토리가 이미 존재하는 경우:
   - `PLAN.md`가 이미 있으면 → 이미 생성된 프로젝트 플랜 → AskUserQuestion: "[1] 다른 이름으로 생성 [2] 취소"
   - 그 외 → 신규 프로젝트 플랜으로 진행
4. PLAN_DIR 조기 생성 (G1~G4 gate_decision 기록을 위해):
   ```bash
   mkdir -p "${PLAN_DIR}"
   touch "${PLAN_DIR}/sessions.jsonl"
   ```

### 2-0. seed_select — Seed Context 선택

CTX_DIR 탐색:
```bash
ls "${CTX_DIR}" 2>/dev/null | grep -E '^[0-9]{3}-.*\.md$' | sort
```

[A] context 파일 없음:
  → skip. seed_file = null.

[B] context 파일 있음 + --seed 지정:
  → 파일이 CTX_DIR에 존재하면 seed_file 확정
  → 없으면 에러: "지정한 seed '{file}'이 context/에 없습니다. 존재 목록: {목록}"

[C] context 파일 있음 + --seed 미지정:
  → 목록 출력:
    "[0] (없음 — 사용자 입력 기반으로 생성)
     [1] context/001-initial.md
     [2] context/002-team-review.md"
  → AskUserQuestion: "이 프로젝트 플랜의 Seed Context를 선택하세요 (번호 입력):"
  → 0 → seed_file = null
  → 1+ → seed_file 확정

### 2-1. context_load — 프로젝트 컨텍스트 전체 로드

이 스텝에서 `knowledge-conventions.md §6` 참조 (Knowledge 인출 전략).

로딩 순서:
1. seed_file → seed_ctx (G0~G4 Primary Driver)
2. 나머지 CTX_DIR 파일 (정렬순) → bg_ctx
3. KNOW_FILE → knowledge_ctx
4. PLANS_DIR/*/PLAN.md의 Section 0 + sessions.jsonl tail -1 → completed_plans_ctx

Knowledge 표시 시 confidence 레벨 구분:
- `[canonical]` — 검증된 교훈 (최우선 참조)
- `[validated]` — 2회+ 확인된 교훈
- `[observed]` — 1회 관찰된 교훈

프롬프트 내 표시:
```
📌 Seed Context: context/003-api-design.md (또는 "없음 — 사용자 입력 기반")
⬜ Background Context (N개): context/001-*.md, context/002-*.md ...
📚 Project knowledge: K-{N}건
📋 완료 프로젝트 플랜 참조: ag-1614 (DONE), ag-1614-fixes (DONE)
```

반영 여부 추적 (기존 로직 유지, 단 경로를 CTX_DIR 기준으로):
  [x] : 이미 이전 create/update에서 반영됨
  [ ] : 이번 create에서 신규 반영 대상

**핵심 원칙: G0~G4는 context 파일이 있어도 항상 실행된다.**
context 파일들은 각 Gate의 초안 데이터를 제공할 뿐이며, Gate를 대체하거나 스킵하지 않는다.
context가 있으면 "백지에서 정보 수집"이 아닌 "초안 확인/보완" 모드로 Gate를 실행한다.

**Gate별 context 활용 방식:**

| Gate | context에서 가져오는 초안 | Gate가 여전히 검증하는 항목 |
|------|--------------------------|--------------------------|
| G0 | §1/§2/§3 내용 → Ambiguity Score 산출 입력 | 불충분한 섹션 보완, 사용자 확인 |
| G1 | §3 해결 조건 → Contract Statement 초안 | "관찰 가능한 산출물인가?" 검증, 행위 동사 제거 |
| G2 | §2 원인/§3 해결 조건 + **반요구사항** → IN/OUT 항목 초안 | OUT 항목 확인, 위임처 확인, Invariant 분류 |
| G3 | `test_scenarios`, `principles` → SPEC-TEST 초안 | SPEC-to-TEST 매트릭스 COVERED 여부 — Draft이므로 정식 검증 필수 |
| G4 | 전체 ctx → Q1/Q2/Q3 검사 입력 | 항상 실행 (정합성 검사는 ctx와 무관하게 필수) |

> G2의 OUT 경계는 context의 **반요구사항(Anti-Requirements)**에서 초안을 도출할 수 있다. 반요구사항이 없으면 Gate에서 백지 설계한다. G4의 Q1/Q2/Q3는 context와 무관하게 필수 실행.

### 3. G0 — PROBLEM.md 완성 (§1/§2/§3)

3개 필수 섹션의 충분/불충분 판정 + Ambiguity Score 산출.

**Step 0-0. Context 활용 여부 확인**:
- seed_ctx 존재 → "📌 Seed Context 감지 → 초안 확인/보완 모드로 Gate 실행"
  - DRAFT 모드: context 초안 검증 + 불충분 영역만 HITL
- seed_ctx 없음 → "📋 Context 없음 → 백지 생성 모드로 Gate 실행"
  - FULL 모드: 전체 Gate 기준 검증

**섹션별 판정 기준** (인라인):

| 섹션 | 충분 | 불충분 (HITL 필요) |
|------|------|-------------------|
| §1 증상 | 제3자가 동일 현상을 관찰할 수 있는 재현 경로 포함 | 현상만 서술, 재현 경로 없음 |
| §2 원인 | [확인됨] 또는 [가설]+확인 방법이 명시됨 | 원인 미상, 가설만 있고 확인 방법 없음 |
| §3 해결 조건 | 상태 서술("~이다") + 검증 방법 1개 이상. 각 조건에 R-ID(R-1, R-2, ...) 부여 | 절차 서술("~한다"), 검증 방법 없음, R-ID 미부여 |

**Step 0-1 — Ambiguity Score 산출**:

§1/§2/§3 초안에 대해 아래 공식을 적용한다.

```
Ambiguity = 1 − (symptom_clarity × 0.30 + cause_clarity × 0.30 + resolution_observability × 0.40)
```

**채점 기준**:

| 차원 | 가중치 | 1.0 (완전 명확) | 0.5 (부분 명확) | 0.0 (불명확) |
|------|:------:|---------------|---------------|-------------|
| **Symptom Clarity** (§1) | 0.30 | 재현 경로 + 관찰 현상 + 환경 명시 | 현상은 서술되었으나 재현 경로 부분적 | 증상 미서술 |
| **Cause Clarity** (§2) | 0.30 | [확인됨] + 근본 원인 + 확인 방법 | [가설] + 확인 방법 명시 | 원인 미상 + 확인 방법 없음 |
| **Resolution Observability** (§3) | 0.40 | 상태 서술 + 검증 방법 + 산출물 명시 + R-ID 부여 | 상태 서술은 있으나 검증 방법 불명 | 절차 서술만 존재 |

Resolution Observability가 최고 가중치인 이유: §3 해결 조건이 Contract Statement의 직접 입력이며, 전체 프로젝트 플랜 품질에 가장 크게 영향을 미친다.

**판정 흐름 — 2축 매트릭스**:

| 섹션별 판정 | Ambiguity | 결과 |
|------------|-----------|------|
| ALL PASS | ≤ 0.2 | AUTO_PASS |
| ALL PASS | > 0.2 | CLARIFY |
| 1개+ 불충분 | ≤ 0.5 | CLARIFY |
| 1개+ 불충분 | > 0.5 | INSUFFICIENT |

> **자기 채점 편향 주의**: 프로젝트 플랜 작성 에이전트가 자기 채점하므로, 경계 구간(0.15~0.25)에서는 보수적으로 CLARIFY 판정을 권고한다.

**섹션별 판정 → 불충분 시 AskUserQuestion**:
- §1 증상: "어떤 명령/경로를 실행하면 이 문제를 재현할 수 있나요?"
- §2 원인: "원인이 확인된 건가요, 아직 가설인가요?"
- §3 해결 조건: "해결되었다고 판단할 수 있는 관찰 가능한 상태는?"
  - §3 각 해결 조건에 R-ID(R-1, R-2, ...) 부여 — 하류 Gate(G1~G4)에서 추적 체인의 시작점

**MUST**: 각 Gate는 반드시 AskUserQuestion 도구를 호출한다.

**원인 모름 처리**: §2를 `**[가설]** 원인 미상 — 조사 필요`로 기록하고 PLAN.md에 VALIDATE 서브태스크 자동 삽입.

**Wicked Problem 판별** (0 hop):
조건: §2 원인이 [가설]이고 §3에 "탐색 후 결정", "조사 결과에 따라", "원인 파악 후 판단" 등 포함
→ AskUserQuestion: "[1] 탐색 프로젝트 플랜(Recommended) [2] 일반 프로젝트 플랜 강제 진행"
→ [1] 선택 시: Contract를 "탐색 계약"으로 전환 — 탐색 범위와 판단 기준만 명시

### 4. write_problem — PROBLEM.md 작성

아래 PROBLEM.md 템플릿에 따라 작성. 저장: `${PLAN_DIR}/PROBLEM.md`

**PROBLEM.md 템플릿** (인라인):

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

**섹션→PLAN 흐름**:

| 섹션 | 필수 | PLAN.md로의 흐름 |
|------|:----:|-----------------|
| §1 증상 + 재현 | 필수 | 재현 경로를 뒤집으면 Acceptance Gate Oracle |
| §2 원인 + 확인/가설 | 필수 | [확인됨] → FIX→VERIFY. [가설] → VALIDATE→FIX→VERIFY |
| §3 해결 조건 + 검증 | 필수 | Contract Statement의 직접 입력. 각 조건에 R-ID(R-1, R-2, ...) 부여 — 하류 Gate에서 추적 |
| §4 영향 범위 | 선택 | Scope Boundary OUT + Regression 범위 |

### 5. G1 — Contract Statement

**Step 1-0. Context 활용 여부 확인**:
- seed_ctx 존재 → "📌 Seed Context 감지 → 초안 확인/보완 모드로 Gate 실행"
  - DRAFT 모드: context 초안 검증 + 불충분 영역만 HITL
- seed_ctx 없음 → "📋 Context 없음 → 백지 생성 모드로 Gate 실행"
  - FULL 모드: 전체 Gate 기준 검증

PROBLEM.md §3 해결 조건(R-ID 포함)을 입력으로 Contract Statement 1문장 초안 작성.

**§3→Contract R-ID 추적 검증** (기계적):
```
R_all     = {PROBLEM.md §3의 모든 R-ID}
Contract_R = {Contract가 커버하는 R-ID}
Missing   = R_all − Contract_R
Missing ≠ ∅ → G1 실패. 누락 R-ID를 Contract에 포함하거나 OUT으로 명시적 분리.
```

검증: 출력이 관찰 가능한 산출물인가? 행위 동사("~한다") 미포함인가?

**Contract 안티패턴 검사** (인라인):

**1. Aspirational SPEC 안티패턴**:
- 탐지 신호: Contract에 런타임 행위 약속 ("~하게 동작한다", "~를 재현한다")
- 안티패턴 탐지어: "재현한다", "동일하게 동작한다", "100%", "완벽하게"

**2. 절차→상태 변환 (행위 동사 금지)**:

| 절차 표현 (X) | 상태 표현 (O) |
|--------------|-------------|
| "파일을 생성한다" | "파일이 존재한다" |
| "데이터를 마이그레이션한다" | "데이터가 새 스키마에 적합하다" |
| "테스트를 실행한다" | "모든 테스트가 통과 상태이다" |
| "API를 배포한다" | "API 엔드포인트가 200을 반환한다" |
| "코드를 리팩토링한다" | "리팩토링된 코드가 존재하고 기존 테스트를 통과한다" |
| "원본과 동일하게 동작한다" | "출력 JSON의 구조가 기대 스키마와 일치한다" |

**3. Scope Boundary 안티패턴**:

| 안티패턴 | 문제 | 수정 |
|---------|------|------|
| OUT 0개 | 경계 미설정 — 범위 무한 확장 위험 | 최소 1개 OUT 명시 |
| "별도 계획 필요" (위임처 없음) | 위임처 불명 — 추적 불가 | 구체적 플랜 이름/시스템/"미정 — HITL" 기재 |
| IN에 비결정론 포함 | Acceptance Gate 침투 | Tier B로 격리 후 OUT 이동 |

**4. SPEC-TEST 매트릭스 안티패턴**:

| 안티패턴 | 탐지 | 수정 |
|---------|------|------|
| UNCOVERED IN 항목 | IN인데 검증 서브태스크 없음 | 서브태스크 추가 또는 SPEC 축소 |
| OUT인데 TEST 포함 | OUT 항목이 서브태스크에 매핑됨 | 서브태스크 제거 또는 IN 재분류 |
| Invalid Oracle | 항상 true 반환 (`file_exists` + 빈 파일도 통과) | Strong Oracle로 보강 |
| Weak Oracle 미보강 | `file_exists`만 사용, 내용 검증 없음 | jq/grep 등 내용 검증 추가 |

실패 시 → 위 안티패턴 목록을 참조하여 재서술 후 AskUserQuestion.

G1 통과 시 `gate_decision` 레코드를 sessions.jsonl에 append (스키마: `session-log-conventions.md` §3-4).

### 6. G2 — Scope Boundary

**Step 2-0. Context 활용 여부 확인**:
- seed_ctx 존재 → "📌 Seed Context 감지 → 초안 확인/보완 모드로 Gate 실행"
  - DRAFT 모드: context 초안 검증 + 불충분 영역만 HITL
- seed_ctx 없음 → "📋 Context 없음 → 백지 생성 모드로 Gate 실행"
  - FULL 모드: 전체 Gate 기준 검증

**Step 2-1. 반요구사항 → OUT/Invariant 초안 매핑**:

context의 `## 반요구사항 (Anti-Requirements)` 섹션이 존재하면, 각 NOT-ID를 다음 기준으로 분류하여 IN/OUT 테이블 초안에 반영:

| 반요구사항 유형 | Scope 매핑 | 예시 |
|---------------|-----------|------|
| 범위 제외 ("~안 다룸", "범위 밖") | **OUT** + 위임처 | "provider 통합 안 다룸" → OUT, 위임처: 별도 플랜 |
| 보존/불변 ("~유지", "~변경 금지") | **IN** (Invariant) | "기존 API 스키마 불변" → IN (Invariant) |

분류가 모호한 경우(범위 제외인지 보존인지 불명확) → AskUserQuestion으로 사용자에게 확인.

Invariant로 분류된 항목은 Scope Boundary 테이블에 `**IN** (Invariant)` 표시로 기재. exec의 CRG Agent Z가 이 항목들을 우선 회귀 검증 대상으로 취급하며, Worker 프롬프트에도 변경 금지 영역으로 주입된다.

반요구사항 없으면 기존 로직(Contract 기반 백지 설계)으로 진행.

Contract를 기반으로 IN/OUT 테이블 초안 작성.
필수 조건: IN >= 1, OUT >= 1, OUT 항목에 구체적 위임처.
AskUserQuestion으로 IN/OUT 분류 확인.

### 7. G3 — SPEC-TEST + Testability (Oracle 실행 가능성)

**Step 3-0. Context 활용 여부 확인**:
- seed_ctx 존재 → "📌 Seed Context 감지 → 초안 확인/보완 모드로 Gate 실행"
  - DRAFT 모드: context 초안 검증 + 불충분 영역만 HITL
- seed_ctx 없음 → "📋 Context 없음 → 백지 생성 모드로 Gate 실행"
  - FULL 모드: 전체 Gate 기준 검증

1. 각 설계 원칙의 Testability 판정 (Yes/Partial/No)
2. SPEC-to-TEST 매트릭스 작성 (R-ID + 보강 경로 열 포함): 모든 IN 항목 → COVERED 필수
3. **R-ID 기반 기계적 커버리지 검증**:
   ```
   R_all = {PROBLEM.md §3의 모든 R-ID} − {G2에서 OUT으로 분리된 R-ID}
   R_covered = {매트릭스에서 COVERED인 R-ID}
   R_uncovered = R_all − R_covered
   R_uncovered ≠ ∅ → G3 실패
   ```
4. **Oracle 등급 변별**: "이 Oracle이 PASS인데 Predicate가 미충족인 상황이 가능한가?" — 가능하면 Weak
5. **Weak Oracle 보강 경로**: Weak 등급 Oracle에는 SPEC-to-TEST 매트릭스의 보강 경로 열에 추가 검증 명령을 명시. Verifier가 False Positive 검사 시 이 경로를 즉시 실행하여 확정/부정 가능
6. **Oracle Prerequisites**: 각 서브태스크의 Predicate/Oracle 테이블에 Prerequisites 열을 설계. Oracle 실행 전 충족 조건(마이그레이션 적용, 빌드 완료, 서비스 시작 등)을 명시하여 Verifier의 false-negative blocker 방지
7. UNCOVERED 항목 또는 등급 이슈 존재 시 → AskUserQuestion

G3 통과 시 `gate_decision` 레코드를 sessions.jsonl에 append.

### 8. G4 — 정합성 검사 (Q1/Q2/Q3)

**Step 4-0. Context 활용 여부 확인**:
- context 유무 무관하게 **항상 Q1/Q2/Q3 실행** (R-ID 집합 연산은 context와 무관)
- seed_ctx 존재 시에도 축약 금지 — 정합성 검사는 전수 검증 필수

**Q1/Q2 R-ID 기반 기계적 검증**:
```
Contract_R  = {Contract §3 추적에 열거된 R-ID}
Predicate_R = {모든 서브태스크 Predicate의 R-ID 열 합집합}

Q1: Under_tested = Contract_R − Predicate_R → ≠ ∅이면 실패
Q2: Over_tested  = Predicate_R − Contract_R → ≠ ∅이면 실패
```

- Q3 원칙-테스트 괴리: 런타임 행위 약속인데 구조만 검증? (LLM 의미 판단, 근거 1~2문장 기록)
1개라도 실패 → 원인 제시 + 수정안 + AskUserQuestion.

G4 통과 시 `gate_decision` 레코드를 sessions.jsonl에 append.

### 9. build_plan — PLAN.md 구성

아래 PLAN.md 템플릿에 따라 구성:

**PLAN.md 템플릿** (인라인):

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

| R-ID | SPEC IN 항목 | 검증 서브태스크 | Oracle 등급 | 보강 경로 | 커버리지 |
|:----:|-------------|--------------|:-----------:|----------|:-------:|
| R-1 | {IN 항목} | {S1, S2...} | Strong | — | COVERED |
| R-2 | {IN 항목} | {S3} | Weak | {추가 검증 명령} | COVERED |

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

ACCEPT iff S1 ∧ S2 ∧ ... ∧ SN

---

## Subtasks

### S1: {title} [ ]

**Action**: {수행 절차}

| Predicate | Oracle | Prerequisites | R-ID |
|-----------|--------|---------------|:----:|
| {상태 서술} | {검증 명령} | {Oracle 실행 전 충족 조건. 없으면 "—"} | R-1 |

---

## 의존성 그래프 (DAG)

S1 ──┐
     ├── S3 ── S4
S2 ──┘

---

## 전체 체크리스트

[ ] S1: {title}
[ ] S2: {title}
...
```

구성 항목:
- Section 0: Contract (§3 추적 R-ID 명시) + Scope + SPEC-TEST (R-ID 열 포함) + 설계 원칙
- Acceptance Gate: `ACCEPT iff S1 ∧ S2 ∧ ...`
- Subtasks: 각각 Predicate + Oracle + R-ID 테이블 + Action
- 의존성 그래프 (ASCII DAG)
- 전체 체크리스트 (`[ ]` 형식)

**Context Sources 섹션** (build_plan에서 생성):

seed_file이 있는 경우:
```markdown
## Context Sources
- [x] **context/003-api-design.md ← Seed** (이 프로젝트 플랜의 기반)
- [x] context/001-initial.md (배경 참조)
- [x] context/002-team-review.md (배경 참조)
```

seed_file이 null인 경우:
```markdown
## Context Sources
_(없음 — 사용자 입력 기반 생성)_
```

#### 9-A. 구조 검증 (기계적)

build_plan 완료 후, PLAN.md 필수 구조를 기계적으로 검증한다:

| 확인 항목 | 검증 방법 |
|---------|---------|
| Acceptance Gate 공식 존재 | `ACCEPT iff` + "∧" 연산자 존재 확인 |
| 비결정론 격리 매트릭스 존재 | grep: "## 비결정론 격리" |
| 의존성 DAG 존재 | grep: "## 의존성 그래프" |
| Prerequisites 열 존재 | 서브태스크 Predicate/Oracle 테이블에 "Prerequisites" 열 존재 확인 |

검증 결과: 모두 통과 → GATE-FINAL 진행 / 1개+ 실패 → AskUserQuestion

### 10. GATE-FINAL — 최종 확인

완성된 PLAN.md 구조를 요약하여 AskUserQuestion으로 사용자 최종 승인.

### 11. write_artifacts — 파일 생성

생성 파일:
```
${PLAN_DIR}/
├── PLAN.md
├── PROBLEM.md
└── sessions.jsonl   ← name_validate에서 이미 생성됨. 여기서는 미존재 시에만 touch
```

sessions.jsonl 보존: G1/G3/G4에서 기록한 gate_decision 레코드가 이미 존재하므로 덮어쓰지 않는다.
```bash
[ -f "${PLAN_DIR}/sessions.jsonl" ] || touch "${PLAN_DIR}/sessions.jsonl"
```

생성하지 않는 파일:
- knowledge.jsonl    ← per-plan 없음 (PROJ_DIR에 있는 것 사용)
- context/           ← per-plan 없음 (CTX_DIR 공유)

`${KNOW_FILE}` 없으면 빈 파일 생성 (경로: ${PROJ_DIR}/knowledge.jsonl).

`${HIST_FILE}`에 append:
```jsonl
{"type":"plan_created","name":"{plan_name}","ts":"{ISO 8601}","seed":"{seed_file 또는 null}"}
```

### 12. report + GATE-AUTO-EXEC

파일 목록 안내 후 AskUserQuestion:
- "프로젝트 플랜 `{plan_name}`이 생성되었습니다. `/project-planner exec {plan_name}`을 실행할까요?"
- [1] `/project-planner exec {plan_name}` 실행 (Recommended)
- [2] 나중에 실행

[1] 선택 시 → Skill 도구로 `/project-planner exec {plan_name} --project {project_name} --repo {repo_slug}` 호출
