# SPEC-TEST Alignment — 실행 스펙 계약 경계 + 검증 정합성

> **용도**: 실행 스펙 생성 전 SPEC/TEST 정합성을 강제하는 게이트 규칙

---

## 0. 핵심 원리

실행 스펙의 완료 조건(DoD)은 SPEC이 선언한 계약 범위와 **정확히 일치**해야 한다.

- SPEC이 더 넓으면 → **미달 완료** (false accept). TEST가 PASS해도 SPEC 미이행.
- TEST가 더 넓으면 → **범위 초과** (scope creep). 불필요한 서브태스크가 나중에 DELEGATED/SKIP 처리.

이 규칙은 `spec-quality-characteristics.md`의 F1~F6, O1~O6과 병행 적용된다.

### 안티패턴: Aspirational SPEC

**정의**: 설계 원칙에 야심찬 목표를 선언하지만, TEST는 그보다 좁은 범위만 검증하는 구조.

**사례**:
- 원칙: "원본 동작 충실 재현 — 100% 동일하게 동작해야 한다" (런타임 행위 약속)
- 실제 TEST: JSON 구조 검증, 스키마 유효성, 파일 존재 확인 (산출물 구조 검증)
- 결과: SPEC 약속 범위와 TEST 검증 범위의 불일치로 Acceptance Gate 사후 조정 필요

**교훈**: SPEC에서 약속하지 않은 것을 TEST하지 말고, TEST할 수 없는 것을 SPEC에서 약속하지 말라.

---

## 0-1. G0 PROBLEM Gate — PROBLEM.md 완성 판정

PROBLEM.md §1/§2/§3 각 섹션의 충분/불충분 판정 기준.

### 섹션별 판정 기준

| 섹션 | 충분 | 불충분 (HITL 필요) |
|------|------|-------------------|
| §1 증상 | 제3자가 동일 현상을 관찰할 수 있는 재현 경로 포함 | 현상만 서술, 재현 경로 없음 |
| §2 원인 | [확인됨] 또는 [가설]+확인 방법이 명시됨 | 원인 미상, 가설만 있고 확인 방법 없음 |
| §3 해결 조건 | 상태 서술("~이다") + 검증 방법 1개 이상. 각 조건에 R-ID(R-1, R-2, ...) 부여 | 절차 서술("~한다"), 검증 방법 없음, R-ID 미부여 |

### 유도 질문

- §1: "어떤 명령/경로를 실행하면 이 문제를 재현할 수 있나요?"
- §2: "원인이 확인된 건가요, 아직 가설인가요? 가설이라면 어떻게 확인할 수 있나요?"
- §3: "이 문제가 해결되었다고 판단할 수 있는 관찰 가능한 상태는 무엇인가요?"

### 특수 케이스: 원인 모름

사용자가 "원인 모름"이라고 답하면:
1. §2를 `**[가설]** 원인 미상 — 조사 필요`로 기록
2. SPEC.md에 원인 조사 서브태스크(VALIDATE 단계)를 자동 삽입

### Wicked Problem 판별 기준 + 분기

§2 원인이 [가설]이고 §3 해결 조건에 "탐색 후 결정", "조사 결과에 따라" 등이 포함되면 Wicked Problem 후보.

분기:
- **[1] 탐색 실행 스펙 (Recommended)**: 탐색 Contract로 진행 → G1에서 탐색형 Contract Statement 작성
- **[2] 일반 실행 스펙 강제 진행**: §3을 최대한 구체화한 후 일반 Gate 순서대로 진행

### Ambiguity Score — G0 필수 판정 조건

G0 진입 시 §1/§2/§3 초안에 대해 Ambiguity Score를 산출한다. Ambiguity ≤ 0.2는 **G0 통과의 필수 조건**이다. 섹션별 "충분" 판정과 함께 두 조건 모두 충족해야 G0을 통과한다.

#### 산출 공식

```
Ambiguity = 1 − (symptom_clarity × 0.30 + cause_clarity × 0.30 + resolution_observability × 0.40)
```

#### 채점 기준

| 차원 | 가중치 | 1.0 (완전 명확) | 0.5 (부분 명확) | 0.0 (불명확) |
|------|:------:|---------------|---------------|-------------|
| **Symptom Clarity** (§1) | 0.30 | 재현 경로 + 관찰 현상 + 환경 명시 | 현상은 서술되었으나 재현 경로 부분적 | 증상 미서술 |
| **Cause Clarity** (§2) | 0.30 | [확인됨] + 근본 원인 + 확인 방법 | [가설] + 확인 방법 명시 | 원인 미상 + 확인 방법 없음 |
| **Resolution Observability** (§3) | 0.40 | 상태 서술 + 검증 방법 + 산출물 명시 + R-ID 부여 | 상태 서술은 있으나 검증 방법 불명 | 절차 서술만 존재 |

Resolution Observability가 최고 가중치인 이유: §3 해결 조건이 Contract Statement의 직접 입력이며, 전체 실행 스펙 품질에 가장 크게 영향을 미친다.

> **용어 구분**: G0의 "Resolution Observability"는 "해결 조건이 관찰 가능한 상태로 서술되었는가?"를 판정한다. G3의 "Oracle Executability"는 "Predicate가 기계적으로 실행 가능한 Oracle로 검증되는가?"를 판정한다. 두 개념은 다르다 — G0은 인간이 관찰할 수 있는 상태 서술의 품질, G3은 자동화 도구가 실행할 수 있는 검증 명령의 품질을 각각 평가한다.

#### 판정 흐름 — 2축 매트릭스

판정은 섹션 판정(충분/불충분)과 Ambiguity 범위의 2축 조합으로 결정한다:

| 섹션별 판정 | Ambiguity | 결과 |
|------------|-----------|------|
| ALL PASS | ≤ 0.2 | AUTO_PASS |
| ALL PASS | > 0.2 | CLARIFY (모호성 해소 후 재채점) |
| 1개+ 불충분 | ≤ 0.5 | CLARIFY (불충분 섹션부터 HITL) |
| 1개+ 불충분 | > 0.5 | INSUFFICIENT (전체 재작성) |

#### Threshold 근거 (0.2)

이 값은 ouroboros(Q00/ouroboros) Interview Phase의 Ambiguity Gate에서 차용하였다. ouroboros에서는 Socratic Interview 후 외부 LLM이 채점하여 0.2 이하를 "충분히 명확"으로 판정한다.

본 시스템에서는 **실행 스펙 작성 에이전트가 자기 채점**하므로, 자기 확증 편향에 의해 실제보다 낮게(명확하다고) 산출될 가능성이 있다. 따라서:
- 경계 구간(0.15~0.25)에서는 **보수적으로 CLARIFY** 판정을 권고한다. 이는 위 매트릭스의 `≤ 0.2 → AUTO_PASS` 판정을 무효화하는 것이 아니라, 자기 채점 시 경계 구간에서 의심되면 CLARIFY 쪽으로 판단하라는 운용 가이드라인이다
- spec-agent create로 G0을 통과한 실행 스펙이 3~5건 누적되면, 사용자가 threshold 적합성을 재검토한다. 재검토 결과는 `${KNOW_FILE}`에 `cat: "decision"` 레코드로 기록한다 (스키마: `${CLAUDE_SKILL_DIR}/rules/knowledge-conventions.md` 참조)
- Threshold 변경 시 이 섹션과 SKILL.md Step 3의 매트릭스를 함께 갱신한다

#### 경계 구간 운용 가이드라인

실행 스펙 작성 에이전트가 자기 채점하므로, 경계 구간(0.15~0.25)에서는 **보수적으로 CLARIFY 판정을 권고**한다. 이는 매트릭스의 `≤ 0.2 → AUTO_PASS` 판정을 무효화하는 것이 아니라, 자기 채점 시 경계 구간에서 의심되면 CLARIFY 쪽으로 판단하라는 운용 가이드라인이다.

#### 주의 사항

- 섹션별 "충분" 판정과 Ambiguity ≤ 0.2은 AND 관계이다. 두 조건 모두 충족해야 AUTO_PASS.
- 섹션별 "충분" ALL PASS이지만 Ambiguity > 0.2이면 **CLARIFY로 진행**. 모호성이 하위 Gate(G1~G4)에 전파되는 것을 차단한다.
- Score는 G0 내부 판정의 필수 조건이지만, Acceptance Gate에 직접 포함하지 않는다 (F3 결정론적 검증 원칙 준수). Score 자체는 비결정론적 산출이므로 Acceptance Gate와 분리한다.
- HITL 반복 시 매 라운드마다 재채점한다. Score **수치**는 G0 내부 판정에만 사용하며 별도 파일에 기록하지 않는다 (threshold 변경 의사결정은 knowledge.jsonl에 기록)

---

## 1. SPEC 필수 구조

모든 실행 스펙의 Section 0에 다음 3개 요소를 포함한다.

### 1-A. Contract Statement (계약 문장)

1문장으로 "이 실행 스펙이 생산하는 것"을 명시한다.

**형식**:

```
이 스펙은 {입력}을 받아 {출력}을 생산한다.
```

| 요소 | 질문 |
|------|------|
| **입력** | 무엇이 주어지는가? |
| **출력** | 무엇이 만들어지는가? |

**규칙**:

- 출력은 **관찰 가능한 산출물**이어야 한다 (파일, DB 상태, API 응답, 배포된 디렉토리)
- "~하게 동작한다"(행위) 금지 → "~가 존재한다", "~에 적합하다"(상태)만 허용
- **안티패턴 탐지어**: "재현한다", "동일하게 동작한다", "100%", "완벽하게" → 재서술 요구

**검증**: Contract Statement의 모든 출력 명사가 Acceptance Gate의 서브태스크 Predicate에 1개 이상 대응되어야 한다.

**§3→Contract R-ID 추적 검증** (G1 기계적 검증):

```
R_all      = {PROBLEM.md §3의 모든 R-ID}
Contract_R = {Contract Statement가 커버하는 R-ID}
Missing    = R_all − Contract_R
Missing ≠ ∅ → G1 실패. 누락 R-ID를 Contract에 포함하거나 OUT으로 명시적 분리.
```

**Contract→Predicate 대응**: Contract의 모든 출력 명사가 Acceptance Gate 서브태스크 Predicate에 1개 이상 대응

### 1-B. Scope Boundary (스코프 경계)

이 실행 스펙이 수행하는 것(IN)과 수행하지 않는 것(OUT)을 명시적으로 구분한다.

| 항목 | IN/OUT | 근거 |
|------|:------:|------|
| (이 실행 스펙의 책임) | **IN** | (왜 이 실행 스펙의 범위인가) |
| (이 실행 스펙의 책임 아님) | **OUT** | (위임처: 스펙 이름/시스템/미정) |

**규칙**:

- IN 1개 이상, OUT 1개 이상 필수
- OUT이 0개인 실행 스펙 → 경계 미설정 경고. 모든 실행 스펙에는 "하지 않는 것"이 있다
- OUT 항목에 **구체적 위임처** 명시 (스펙 이름, 시스템 이름, 또는 "미정 — HITL 결정 필요")
- "별도 계획 필요"만 적고 위임처 미명시 금지

**필수 고려 OUT 항목** (해당 시):

| 유형 | 판단 기준 |
|------|----------|
| 런타임 행위 검증 | 이 실행 스펙이 산출물을 만드는가, 산출물이 동작하는지 확인하는가? |
| 비결정론적 외부 의존 | LLM 응답, 외부 API 가용성 등 → Tier B 격리 |
| 성능/품질 기준 | 수치 기준이 Acceptance Gate에 포함되는가? |

### 1-C. Testability Constraint (검증 가능성 제약)

Section 0에 선언한 각 설계 원칙이 결정론적으로 검증 가능한지 판정한다.

| 원칙 | 검증 가능성 | Oracle 유형 | 미검증 시 처리 |
|------|:-----------:|:-----------:|--------------|
| (원칙 A) | Yes | Strong | — |
| (원칙 B) | Partial | Weak | 보강 Oracle 명시 |
| (원칙 C) | No | — | OUT (Scope Boundary에 명시) |

**규칙**: 검증 불가능(No)한 원칙을 Section 0에 남길 수 있으나, **Acceptance Gate에 포함하면 안 된다**. 포함하면 Aspirational SPEC 안티패턴.

---

## 2. TEST 필수 구조

### 2-A. SPEC-to-TEST 추적 매트릭스

Contract Statement의 각 출력(산출물)이 어떤 서브태스크에서 검증되는지 매핑한다. §3 해결 조건의 R-ID로 추적한다.

| R-ID | SPEC 산출물/조건 | 검증 서브태스크 | Oracle 등급 | 보강 경로 | 커버리지 |
|:----:|-----------------|---------------|:-----------:|----------|:-------:|
| R-1 | (IN 항목 A) | S1.3, S3.1 | Strong | — | COVERED |
| R-2 | (IN 항목 B) | S2.1 | Weak | {추가 검증 명령} | COVERED |
| — | (OUT 항목 C) | — | — | — | OUT |

**커버리지 규칙**:

- IN 항목 중 `COVERED`가 아닌 것이 있으면 → **스펙 작성 BLOCKED** (서브태스크 추가 또는 SPEC 축소)
- OUT 항목이 TEST에 포함되어 있으면 → **범위 초과** (서브태스크 제거 또는 OUT→IN 재분류)

**G3 R-ID 기반 기계적 커버리지 검증**:

```
R_all       = {PROBLEM.md §3의 모든 R-ID} − {G2에서 OUT으로 분리된 R-ID}
R_covered   = {매트릭스에서 COVERED인 R-ID}
R_uncovered = R_all − R_covered
R_uncovered ≠ ∅ → G3 실패 (서브태스크 추가 또는 §3 축소)
```

이 검증은 LLM 의미 판단이 아닌 ID 집합 비교이므로 기계적으로 실행 가능하다.

### 2-B. Oracle 등급 분류

| 등급 | 판정 기준 | Acceptance Gate 포함 |
|------|----------|:-------------------:|
| **Strong** | Predicate의 모든 핵심 명사가 Oracle 명령의 출력에서 **문자열 매칭 또는 수치 비교**로 검증됨 | Yes |
| **Weak** | Predicate의 일부 명사만 Oracle로 검증 가능, 또는 **구조(존재 여부)만 검증** | Yes (SPEC-to-TEST 매트릭스에 보강 경로 필수 기재) |
| **Invalid** | 기계적 실행 불가 또는 항상 true | **No** — 제거 또는 보강 |

> **Strong/Weak 변별 테스트**: "이 Oracle이 PASS인데 Predicate가 실제로는 미충족인 상황이 가능한가?" — 가능하면 **Weak**. 불가능하면 **Strong**.

**보강 경로**: Weak Oracle에는 SPEC-to-TEST 매트릭스의 보강 경로 열에 추가 검증 명령을 기재한다. exec의 Verifier(Phase 2 Reviewer)가 False Positive 검사 시 이 경로를 즉시 실행하여 Weak Oracle의 PASS가 진짜인지 확인한다. Strong Oracle은 "—"으로 기재.

**Prerequisites**: 각 서브태스크의 Predicate/Oracle 테이블에 Prerequisites 열을 포함한다. Oracle 실행 전 충족해야 할 조건을 명시한다. Prerequisites 미명시("—") Oracle은 "즉시 실행 가능"으로 간주한다.

Prerequisites는 Verifier 실행 권한에 따라 2등급으로 구분:

| 등급 | 예시 | Verifier 처리 |
|------|------|-------------|
| **Safe** (부작용 없음) | `npm run build`, `npm run compile` | Verifier가 직접 실행 후 Oracle 실행 |
| **Stateful** (상태 변경) | `prisma migrate deploy`, `docker-compose up` | Verifier 실행 불가 → blocker 반환. Worker가 충족해야 함 |

### 2-B-1. 별도 플래그 (등급과 독립)

| 플래그 | 정의 | 처리 |
|--------|------|------|
| **Duplicate** | 다른 서브태스크와 동일 Predicate 검증 | 경고 — 중복 제거 권고 |
| **Runtime** | 비결정론적 소스 의존 (LLM 응답, 외부 API) | Scope OUT 또는 Tier B 처리 |

---

## 3. SPEC-TEST 정합성 검사

실행 스펙 작성 완료 후, 아래 3가지 질문으로 정합성을 검증한다.

### Q1. 과소 검증 (Under-testing)

> "Contract Statement에 선언한 산출물 중, Acceptance Gate의 Predicate로 검증되지 않는 것이 있는가?"

**R-ID 기반 기계적 검증**:
```
Contract_R   = {Contract Statement §3 추적에 열거된 R-ID}
Predicate_R  = {모든 서브태스크 Predicate의 R-ID 열 합집합}
Under_tested = Contract_R − Predicate_R
Under_tested ≠ ∅ → Q1 실패 (누락 R-ID 명시)
```

**있으면**: 서브태스크 추가 또는 Contract Statement 축소.

### Q2. 과대 검증 (Over-testing)

> "Acceptance Gate에 포함된 서브태스크 중, Contract Statement의 IN 항목에 대응하지 않는 것이 있는가?"

**R-ID 기반 기계적 검증**:
```
Over_tested = Predicate_R − Contract_R
Over_tested ≠ ∅ → Q2 실패 (초과 R-ID 명시)
```

**있으면**: 해당 서브태스크를 OUT으로 이동(위임) 또는 Contract Statement 확장.

### Q3. 원칙-테스트 괴리 (Principle-Test Gap)

> "Section 0의 설계 원칙 중, 런타임 행위/품질/성능을 약속하면서 TEST가 구조/존재/스키마만 검증하는 것이 있는가?"

**있으면**: 원칙을 테스트 가능한 수준으로 재서술하거나, 해당 원칙의 검증을 OUT으로 분리.

**G4 R-ID 기반 기계적 정합성 검증**:

Q1/Q2는 R-ID 집합 연산으로 기계적 판정이 가능하다:

```
Contract_R   = {Contract Statement §3 추적에 열거된 R-ID}
Predicate_R  = {모든 서브태스크 Predicate의 R-ID 열 합집합}

Q1: Under_tested = Contract_R − Predicate_R
    Under_tested ≠ ∅ → Q1 실패 (누락 R-ID 명시)

Q2: Over_tested  = Predicate_R − Contract_R
    Over_tested  ≠ ∅ → Q2 실패 (초과 R-ID 명시)
```

Q3는 의미 판단이 필요하므로 기계화 불가. LLM이 판정하되, 판정 근거를 1~2문장으로 기록한다.

---

## 4. HITL Gate — 실행 스펙 생성 전 필수 체크리스트

(SOT: commands/create.md G0~G4)

SPEC.md 작성 전에 아래 항목에 모두 응답해야 한다. 미응답 항목이 있으면 실행 스펙 생성 **BLOCKED**.

```
[ ] G0. PROBLEM.md §1/§2/§3 완성 (Ambiguity Score ≤ 0.2)
[ ] G1. Contract Statement 1문장 작성 완료 (입력 → 출력, 행위 동사 미포함)
[ ] G2. Scope Boundary 테이블 작성 완료 (IN >= 1, OUT >= 1, 위임처 명시)
[ ] G3. 각 설계 원칙의 검증 가능성 판정 완료 + SPEC-to-TEST 매트릭스에서 모든 IN 항목 COVERED
[ ] G4. Q1/Q2/Q3 정합성 검사 통과
```

### HITL 에스컬레이션 트리거

| 상황 | 행동 |
|------|------|
| Contract Statement에 행위 동사 포함 | 산출물 기반 재서술 요청 |
| OUT 항목 위임처가 "미정" | 위임처 결정 전 스펙 생성 BLOCKED |
| 검증 불가(No) 원칙이 Acceptance Gate에 포함 | 원칙 재서술 또는 Gate 제거 |
| SPEC-to-TEST 매트릭스에 UNCOVERED 항목 존재 | 서브태스크 추가 또는 SPEC 축소 |

전항목 충족 → SPEC.md 작성 진행.
