# Spec Quality Characteristics — 결정론적 실행 스펙의 형식적 특성

> **용도**: 실행 스펙 작성 시 준수할 품질 특성 가이드

---

## 0. 핵심 원리

실행 스펙은 **절차서가 아니라 검증 명세**다. "무엇을 해야 하는가"가 아니라 "어떤 상태가 참이어야 하는가"를 선언한다. 이를 통해 실행 스펙의 완료 여부가 **이진적(true/false)**으로 판정되며, 실행자(사람이든 Agent든)가 동일한 입력에 대해 동일한 결과를 재현할 수 있다.

---

## 1. 형식적 특성 (Formal Properties)

### F1. 선언적 DoD (Declarative Done Definition)

**정의**: 완료 조건을 "~한다"(절차)가 아닌 "~이다"(상태)로 기술한다.

**메커니즘**: Predicate + Oracle 2열 테이블

| Predicate (명제) | Oracle (검증 수단) |
|-------------------|-------------------|
| 자연어 상태 서술: "~존재한다", "~비어있다", "~와 같다" | 기계적 검증 명령: `jq`, `file_exists`, exit code |

- Predicate는 항상 **상태 서술형(stative)**으로 작성. "생성한다" X → "존재한다" O
- Oracle은 **결정론적 명령**으로 true/false를 반환. `(수동 확인)` 최소화
- Action(절차)은 테이블 아래 별도 행에 기재. Predicate 테이블과 혼재 금지

**예시**:
```
| `02_nodes.json` 내 노드 수 == 11 | `jq '. | length' == 11` |
```

### F2. Acceptance Gate (이진 판정 공식)

**정의**: 전체 실행 스펙의 완료 조건을 단일 불린 표현식으로 환원한다.

**메커니즘**: 서브태스크 ID의 논리곱(conjunction)

```
ACCEPT iff S1.0 ∧ S1.1 ∧ ... ∧ S3.7
```

- 각 Si.j는 내부 Predicate들의 논리곱
- 하나라도 false이면 전체 REJECT — 부분적 완료 상태 불허
- "대략 되었다"는 존재하지 않음

### F3. 결정론적 검증 (Deterministic Verification)

**정의**: 동일 입력에 대해 검증 과정이 항상 동일한 결과를 산출한다.

**Oracle 3유형의 결정론 보장**:
- `file_exists` / `dir_exists` — 파일시스템 순수 쿼리. 부작용 없음
- `jq` 표현식 — JSON 파싱은 결정론적
- 프로세스 exit code — 환경 전제조건(스펙 Section 0)이 보장하면 결정적

**Oracle 품질 분류** (B-Reviewer 판단 기준):

> Oracle 등급 분류(Strong/Weak/Invalid) 및 별도 플래그(Duplicate/Runtime)의 상세 정의:
> → see `rules/spec-test-alignment.md` §2-B

- B-Reviewer는 Weak Oracle에 대해 `warning`, Invalid Oracle에 대해 `blocker`를 발행한다
- Weak Oracle은 추가 검증 조건 제시(보강 경로)를 권고 사항으로 함께 기록한다
- **False Positive 방지 원칙**: Oracle이 구현 내용과 무관하게 항상 통과할 수 있는 구조(예: 파일 존재만 확인, 빈 파일도 통과)인지 B-Reviewer가 명시적으로 확인한다

### F4. 비결정론 격리 (Non-determinism Isolation)

**정의**: 비결정론 소스를 명시적으로 식별하고, fixture화 또는 범위 제외로 격리한다.

**메커니즘**: 비결정론 격리 매트릭스 + 2-Tier DoD

| 비결정론 소스 | 서브태스크 | 격리 방법 | Tier |
|--------------|----------|----------|:----:|

- **Tier A** (결정론적, 본 계획 DoD): fixture 또는 정적 분석으로 비결정론 제거
- **Tier B** (비결정론적, 별도 계획): LLM 응답, 외부 API 등 DoD 범위 밖으로 명시적 제외

**핵심 원칙**: 증명 불가능한 속성을 증명하려 시도하지 않는다. Tier B를 "별도 계획 필요"로 명시하여 비결정론이 Acceptance Gate에 침투하는 것을 원천 차단.

### F5. 멱등성 (Idempotency)

**정의**: 동일 연산을 여러 번 적용해도 결과가 한 번 적용한 것과 같다. `f(f(x)) = f(x)`.

**메커니즘**:
- **입력 fixture** (overrides.json): 비결정론적 판단을 고정값으로 치환
- **출력 fixture** (expected-output.json): 기대 결과를 golden file로 저장
- **Snapshot Lock**: `normalize → compare → diff == 0` 패턴

**수식**: `execute(overrides, sources) == expected-output` — 파이프라인이 순수 함수처럼 동작.

### F6. 원자적 분할 (Atomic Decomposition)

**정의**: 각 서브태스크가 "더 이상 분리 불가"하며, 독립 실행 & 독립 검증 가능하다.

**분리 기준**:
1. 하나의 서브태스크 = 하나의 Action 단위
2. 서브태스크의 Predicate가 다른 서브태스크의 산출물에 의존하지 않음 (의존 시 별도 서브태스크)
3. 실패 시 실패 지점이 격리됨 — "S1.3이 실패하면 5-Gate 중 어디서 실패했는지 intermediates에서 확인"

**독립성 검증**: 진정으로 독립적인 서브태스크는 의존성 그래프에서 병렬 실행 가능으로 나타남.

---

## 2. 운영적 특성 (Operational Properties)

### O1. 수정 책임 분리 (Compiler vs HITL)

**정의**: 모든 수정 행위를 "자동 추론 가능 → 자동화"와 "사람 판단 필요 → HITL"로 이원 분류한다.

**판단 기준** (단일 질문):
> "이 정보를 코드나 메타데이터에서 자동으로 추출할 수 있는가?"
> - **Yes** → 자동화가 처리해야 함. 못하면 **자동화 버그**.
> - **No** → 수동 수정이 정당한 **HITL 개입**.

**효과**: 수정 행위의 정당성을 사후 감사 가능. 실행자가 즉시 올바른 행동을 선택.

### O2. 의존성 그래프 (Dependency DAG)

**정의**: 서브태스크 간 선행/후행 관계를 DAG로 표현하여 병렬/순차를 시각적으로 결정.

**3가지 패턴**:
- **Fan-out**: 동일 선행 조건에서 분기 (병렬 실행 가능)
- **Fan-in**: 여러 경로가 합류 (barrier synchronization)
- **Chain**: 직렬 의존 (data dependency)

**표현 방식**: ASCII DAG + 괄호 주석으로 실행 모드 명시.

### O3. 체크리스트 프로토콜 (Checklist Protocol)

**정의**: 서브태스크 완료 상태를 `[ ]` / `[x]`로 추적하며, "구현 → 테스트 → 체크" 워크플로우를 강제.

**메커니즘**:
- Step별 계층 구조 (COMPILE / DEPLOY / VERIFY)
- Round 개념: 실패 시 체크리스트 초기화 후 재시작
- 각 `[ ]`에 대응하는 Predicate/Oracle이 존재

### O4. 테스트 Fixture 전략

**정의**: 비결정론적 요소를 고정된 값으로 치환하여 재현성 보장.

**Fixture 유형**:

| 유형 | 역할 | 생성 시점 |
|------|------|----------|
| **입력 fixture** | 비결정론적 판단 고정 | HITL Review 완료 시 |
| **출력 fixture** | golden file 기반 회귀 탐지 | 최초 성공 실행 시 |
| **환경 fixture** | 외부 의존 시뮬레이션 | 테스트 준비 시 |
| **검토 oracle** | HITL 검토 기준표 | 실행 스펙 문서에 고정 |

### O5. Expected Mapping (충실도 오라클)

**정의**: 구현 결과가 설계와 일치하는지를 기준표 대조로 검증하는 정적 oracle.

**구조**: `| Order | Name | Expected Type | Key Evidence |`

- HITL 검토 시 "표 vs 실제 출력"의 기계적 대조로 환원
- Key Evidence가 "왜 이 분류인가"의 근거를 사람에게 제공

### O6. 원칙-검증 추적성 (Principle-Verification Traceability)

**정의**: 모든 서브태스크가 설계 원칙 중 하나 이상에 의해 지배되며, 원칙 → 서브태스크 → Predicate의 추적 가능한 체인이 존재.

**메커니즘**: 실행 스펙 Section 0(설계 원칙)에 명시된 원칙 각각이 하나 이상의 서브태스크에 대응되어야 한다. 서브태스크의 Action 행에 "적용 원칙: D-3, AX-7" 등으로 명시하거나, 별도 매핑 테이블로 관리한다.

| 원칙 | 서브태스크 | Predicate |
|------|----------|-----------|
| (설계 원칙 ID) | (서브태스크 ID) | (검증할 Predicate 요약) |

- 원칙에 대응하는 서브태스크가 없으면 해당 원칙이 실행 스펙에 불필요하거나, 서브태스크 누락
- 서브태스크에 대응하는 원칙이 없으면 해당 서브태스크의 존재 근거가 불명확

---

## 3. 특성 간 상호 강화

```
F1(선언적 DoD) + F3(결정론적 검증) → DoD가 선언적이므로 검증이 결정론적
F6(원자적 분할) + O3(체크리스트) → 원자적이어야 한 칸에 대응 가능
F4(비결정론 격리) + O4(Fixture) → 격리 방법이 fixture로 구현됨
O1(수정 책임 분리) + O5(Expected Mapping) → HITL 검토 기준이 매핑 표로 형식화
F2(Acceptance Gate) + O3(체크리스트) → 체크리스트 전체 체크 = Acceptance 충족
```

---

## 4. 실행 스펙 작성 체크리스트

새 실행 스펙 작성 시 아래 항목을 확인:

```
--- SPEC-TEST 정합성 (R-006, 선행 필수) ---
[ ] Section 0: Contract Statement (1문장 — 입력 → 출력, 행위 동사 미포함)
[ ] Section 0: Scope Boundary 테이블 (IN/OUT + 위임처 명시)
[ ] Section 0: SPEC-to-TEST 매트릭스 (모든 IN = COVERED)
[ ] 정합성 검사: Q1(과소) Q2(과대) Q3(괴리) 통과

--- 기존 품질 특성 (F1-F6, O1-O6) ---
[ ] Section 0: 설계 원칙 (해당 스펙에 적용되는 원칙 3개 이상, 각 원칙의 Testability 판정)
[ ] Section 1: Acceptance Gate 공식 (ACCEPT iff S1 ∧ S2 ∧ ...)
[ ] 각 서브태스크: Predicate + Oracle 테이블
[ ] 각 서브태스크: Action 행 (Predicate 테이블과 분리)
[ ] Section: 의존성 그래프 (ASCII DAG + 병렬/순차 주석)
[ ] Section: 전체 체크리스트 ([ ] 형식, Step별 계층)
[ ] Section: 비결정론 격리 매트릭스 (소스, 격리 방법, Tier)
[ ] Section: Test Fixture 표 (유형, 용도, 생성 시점)
[ ] 각 HITL 서브태스크: Expected Mapping 또는 검토 체크리스트
[ ] Phase 통합 검증 서브태스크 (각 Phase 마지막)
```
