# Cascade Map — 프로젝트 플랜 변경 전파 규칙

프로젝트 플랜 구성 요소 변경 시 하위 요소로 변경이 전파되는 규칙.

## 변경 전파 순서

```
PROBLEM → Contract → Scope → SPEC-TEST → Subtasks → Gate
```

**역방향 전파 불가**: 하위 요소 변경이 상위 요소를 자동으로 변경하지 않음.

## 전파 매트릭스

| 변경 지점 | 영향 범위 | 필수 재검증 |
|-----------|----------|------------|
| §1 증상 | §3 해결 조건 연동 확인 | G0 재판정 |
| §2 원인 | [확인됨]↔[가설] 전환 시 VALIDATE 서브태스크 추가/제거 | G0 + build_plan |
| §3 해결 조건 | Contract → Scope → SPEC-TEST → 전체 | G1~G4 전체 |
| Contract | Scope → SPEC-TEST → Subtasks → Gate | G2~G4 |
| Scope IN 추가 | SPEC-TEST 매트릭스 COVERED 확인 → 서브태스크 추가 필요 | G3~G4 |
| Scope IN 제거 | 대응 서브태스크 삭제/수정 | G3~G4 |
| Scope OUT 변경 | 위임처 갱신만 | G2 재확인 |
| 설계 원칙 추가 | Testability 판정 + SPEC-TEST 매트릭스 갱신 | G3~G4 |
| 설계 원칙 삭제 | 대응 서브태스크 삭제/수정 | G3~G4 |
| Subtask Predicate/Oracle | 해당 서브태스크만 | Q1/Q2 재검증 |
| Acceptance Gate 공식 | 서브태스크 매핑 확인 | Q1/Q2 |

## 변경 규모 판정

### 0단계: 의미적 동등성 판정 (cascade_analysis에서 수행)

변경 전/후 텍스트를 비교하여 다음 중 하나로 분류:

**1차 판정**: R-ID 집합 비교 (기계적)

| R-ID 집합 | 결과 |
|-----------|------|
| 변경 전 ≠ 변경 후 | → Essential. 전파 매트릭스 원래 규모 적용 |
| 변경 전 == 변경 후 | → 2차 판정 진행 |

**2차 판정**: Contract 출력 명사 비교 (R-ID 동일한 경우에만)

| Contract 출력 명사 | 결과 |
|-------------------|------|
| 산출물 형태가 변경됨 (예: "REST API" → "GraphQL API") | → Essential. 구현 전체 재작업 필요 |
| 산출물 형태 동일 (오타, 포맷, 동의어) | → Surface. Minor로 하향. Gate 재검증 불필요 |

이 2단계 판정은 전파 매트릭스의 "필수 재검증" 적용 전에 수행하여, 오타 수정에 G1~G4 전체 재검증이 발생하는 비효율을 방지하면서도, R-ID 동일하지만 구현이 완전히 달라지는 변경(REST→GraphQL 등)이 Surface로 오분류되는 것을 차단한다.

### 경미 (Minor)
- Scope OUT 위임처 변경, Oracle 검증 명령어 수정
- 해당 요소만 수정, Gate 재검증 불필요
- sessions.jsonl 기록 불필요

### 중간 (Moderate)
- 서브태스크 Predicate 수정, 설계 원칙 추가/삭제
- 해당 요소 + 직접 영향 요소 수정
- 부분 Gate 재검증 (G3~G4)
- sessions.jsonl에 `update` 노트 기록 권장

### 대규모 (Major)
- Contract 변경, §3 해결 조건 변경
- 전체 cascade 재실행
- G1~G4 전체 재검증
- sessions.jsonl에 기록 필수 + 기존 진행분 outcome `aborted`
