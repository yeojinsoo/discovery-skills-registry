# 3계층 회귀 테스트 전략

> **선행 조건**: 실행 프로토콜(`rules/execution-protocol.md`) §2 Verifier 3-Phase, CRG 프로토콜(`rules/critical-review-gate.md`) §1 에이전트 역할, validate Phase A/C/S 구조 이해 필수
> **적용 범위**: 프로젝트 플랜 실행 전 과정에서 회귀를 3개 계층으로 감지

---

## 0. 핵심 원리

서브태스크 실행, CRG, validate는 각각 서로 다른 시점에 서로 다른 관점으로 검증을 수행한다. 단일 계층만으로는 다음 유형의 회귀를 놓칠 수 있다:

- 이전 서브태스크에서 PASS였던 Oracle이 후속 서브태스크 실행 후 FAIL로 전환 (구현 간 간섭)
- Scope Boundary IN(Invariant)으로 보호한 항목이 변경에 의해 파괴
- knowledge.jsonl에 기록된 pitfall이 재발

3계층 전략은 이 세 가지 경로를 각각 전담하여, 회귀가 감지 없이 통과하는 사각지대를 제거한다.

---

## 1. 계층 개요

| 계층 | 명칭 | 감지 대상 | 트리거 시점 | 실행 주체 | 판정 기준 |
|:----:|------|----------|------------|----------|----------|
| **L1** | Oracle 기반 회귀 | 이전 PASS Oracle의 FAIL 전환 | 서브태스크 실행 시 | exec Verifier Phase 1 (Tester) | 이전 PASS → 현재 FAIL = 회귀 감지 |
| **L2** | Invariant 기반 회귀 | Scope Boundary IN(Invariant) 위반 | CRG 실행 시 | CRG Agent Z (Regression & Risk Assessor) | Invariant 항목 보존 여부. 위반 시 severity = blocker |
| **L3** | Knowledge 기반 회귀 | knowledge.jsonl pitfall/constraint 재발 | validate 실행 시 | validate Phase A 비판 패스 | pitfall/constraint 태그 항목의 재발 여부 |

```
서브태스크 실행          CRG 실행              validate 실행
     │                    │                      │
     ▼                    ▼                      ▼
 ┌────────┐          ┌────────┐            ┌──────────┐
 │  L1    │          │  L2    │            │   L3     │
 │ Oracle │          │Invari- │            │Knowledge │
 │  기반  │          │ ant    │            │  기반    │
 │  회귀  │          │  기반  │            │  회귀    │
 └────────┘          └────────┘            └──────────┘
```

---

## 2. L1 — Oracle 기반 회귀

### 2-1. 트리거 시점

서브태스크 실행 시, exec Verifier가 현재 서브태스크의 Predicate/Oracle을 검증하기 **전에** 수행한다.

### 2-2. 실행 주체

exec Verifier Phase 1 (Tester). 기존 3-Phase 구조(`execution-protocol.md` §2) 내에서 Phase 1의 **선행 단계**로 실행한다.

> **Invariant**: exec Verifier의 3-Phase 구조(Phase 1 Tester / Phase 2 Reviewer / Phase 3 Observer)를 변경하지 않는다. L1은 Phase 1 내부 선행 단계로 동작한다.

### 2-3. 실행 절차

```
1. sessions.jsonl에서 이전 라운드의 PASS 판정 서브태스크 목록을 수집
2. 각 PASS 서브태스크의 Oracle을 PLAN.md Predicate/Oracle 테이블에서 추출
3. Prerequisites 분류에 따른 처리:
   ├─ Prerequisites "—" 또는 Safe → Oracle 재실행
   └─ Stateful Prerequisites → skip (재실행 불가 — 환경 의존)
4. 결과 분류:
   ├─ 재실행 대상 전부 PASS → L1 회귀 없음, 현재 서브태스크 검증으로 진행
   └─ 1건 이상 FAIL → 회귀 감지 (§2-4 판정 참조)
```

> **Oracle 추출**: sessions.jsonl에는 Oracle 명령이 저장되지 않는다. 이전 PASS 서브태스크의 S-ID를 sessions.jsonl에서 확인한 후, PLAN.md의 해당 서브태스크 Predicate/Oracle 테이블에서 Oracle 명령을 읽어 재실행한다.

### 2-4. 판정 기준

| 조건 | 판정 | 후속 행동 |
|------|------|----------|
| 이전 PASS Oracle 전부 현재도 PASS | 회귀 없음 | 현재 서브태스크 검증 진행 |
| 이전 PASS Oracle 중 1건 이상 FAIL | **회귀 감지** | 현재 서브태스크 FAIL 처리. 회귀 원인을 Phase 2(Reviewer)에 전달. Phase 3(Observer)가 수정안 제시 |

### 2-5. 기록

회귀 감지 시 Verifier 출력에 다음을 포함한다:

```
## L1 회귀 감지
- 회귀 서브태스크: {S-ID}
- Oracle: {Oracle 내용}
- 이전 PASS 라운드: {round N}
- 현재 FAIL 원인: {요약}
```

---

## 3. L2 — Invariant 기반 회귀

### 3-1. 트리거 시점

CRG 실행 시, Agent Z(Regression & Risk Assessor)의 검증 범위에 포함된다.

### 3-2. 실행 주체

CRG Agent Z. 기존 3-에이전트 구조(`critical-review-gate.md` §1) 내에서 Agent Z의 책임 범위로 동작한다.

> **Invariant**: CRG의 3-에이전트 구조(X Code Verifier / Y Plan Integrity Critic / Z Regression & Risk Assessor)를 변경하지 않는다. L2는 Agent Z의 기존 역할 내에서 수행한다.

### 3-3. 실행 절차

```
1. PLAN.md Scope Boundary에서 IN (Invariant) 항목을 추출
2. 각 Invariant 항목에 대해 코드 변경 diff를 분석:
   ├─ Invariant 항목 관련 코드에 변경이 없음 → 보존 확인
   ├─ 변경이 있으나 Invariant 동작이 보존됨 → 보존 확인 (근거 명시)
   └─ 변경으로 Invariant 동작이 파괴됨 → 위반 감지
3. 위반 항목에 severity = blocker 부여
```

### 3-4. 판정 기준

| 조건 | 판정 | severity |
|------|------|----------|
| 모든 Invariant 항목 보존 | 회귀 없음 | — |
| Invariant 항목 1건 이상 위반 | **회귀 감지** | **blocker** |

Invariant 위반은 CRG 판정에서 무조건 blocker로 분류된다. `critical-review-gate.md` §4-1에 따라 blocker 1건 이상이면 NEEDS_FIX가 발행된다.

### 3-5. 기록

Agent Z 출력의 회귀 리스크 판정 섹션에 다음을 포함한다:

```
### Invariant 보존 검증 (L2)

| Invariant 항목 | 관련 파일 | 변경 여부 | 보존 판정 | severity |
|:---:|---|:---:|:---:|:---:|
| {항목} | {파일} | Y/N | 보존/위반 | —/blocker |
```

---

## 4. L3 — Knowledge 기반 회귀

### 4-1. 트리거 시점

validate 실행 시, Phase A 비판 패스에서 수행한다.

### 4-2. 실행 주체

validate Phase A. 기존 Phase A/C/S 구조 내에서 Phase A의 비판 관점으로 동작한다.

> **Invariant**: validate의 Phase A/C/S 구조를 변경하지 않는다. L3는 Phase A 내부의 비판 항목으로 동작한다.

### 4-3. 실행 절차

```
1. knowledge.jsonl에서 cat:"pitfall" 또는 cat:"constraint" 레코드를 수집
2. 각 레코드의 내용을 현재 분석 대상과 대조:
   ├─ 현재 맥락에 해당하지 않음 → skip
   ├─ 해당하나 재발하지 않음 → 회귀 없음
   └─ 해당하고 동일 패턴으로 재발 → 재발 감지
3. 재발 감지 시 Phase A 비판 결과에 포함
```

> **범위**: validate.md 회귀 검사 Sub-Phase와 동일 — `cat:"pitfall"` + `cat:"constraint"` 모두 대상.

### 4-4. 판정 기준

| 조건 | 판정 | 후속 행동 |
|------|------|----------|
| pitfall/constraint 재발 없음 | 회귀 없음 | Phase A 비판 결과에 "pitfall/constraint 재발 없음" 기록 |
| pitfall/constraint 1건 이상 재발 | **재발 감지** | Phase A 비판 결과에 재발 항목 포함. Phase C에서 severity 판정 |

### 4-5. 기록

Phase A 비판 결과에 다음을 포함한다:

```
### Pitfall 재발 검사 (L3)

| K-ID | pitfall 요약 | 재발 여부 | 근거 |
|:---:|---|:---:|---|
| {K-NNN} | {요약} | Y/N | {현재 코드/분석에서의 근거} |
```

---

## 5. 계층 간 관계

### 5-1. 독립성

3개 계층은 서로 다른 시점에 서로 다른 주체가 실행한다. 계층 간 의존 관계는 없다.

| 속성 | L1 | L2 | L3 |
|------|----|----|-----|
| 트리거 | 서브태스크 실행 | CRG 실행 | validate 실행 |
| 주체 | exec Verifier Phase 1 | CRG Agent Z | validate Phase A |
| 입력 | sessions.jsonl + Oracle | PLAN.md Scope + diff | knowledge.jsonl |
| 출력 | PASS/FAIL + 회귀 목록 | blocker/보존 + 검증 표 | 재발/미재발 + 검사 표 |

### 5-2. 보완성

- **L1**은 **구현 간 간섭**을 감지한다 — 서브태스크 B의 구현이 서브태스크 A의 Oracle을 파괴하는 경우
- **L2**는 **설계 계약 위반**을 감지한다 — 전체 변경이 Scope에서 보호한 Invariant를 파괴하는 경우
- **L3**는 **과거 실수 반복**을 감지한다 — knowledge에 기록된 pitfall과 동일한 패턴이 재현되는 경우

단일 계층만으로는 다른 두 계층의 감지 영역을 커버할 수 없다.

---

## 6. 적용 범위

### 6-1. 기본 규칙

이 전략은 프로젝트 플래너의 모든 실행 흐름에 적용된다:

- **L1**: exec 커맨드 실행 시 자동 적용
- **L2**: CRG 실행 시 자동 적용 (CRG skip 시 L2도 skip)
- **L3**: validate 커맨드 실행 시 자동 적용

### 6-2. L1 skip 조건

첫 번째 서브태스크 실행 시에는 이전 PASS Oracle이 없으므로 L1을 skip한다. sessions.jsonl에 이전 라운드 PASS 레코드가 없으면 자동 skip.

### 6-3. 소급 적용

이 규칙 제정 전에 실행 완료된 프로젝트 플랜에는 소급 적용하지 않는다.
