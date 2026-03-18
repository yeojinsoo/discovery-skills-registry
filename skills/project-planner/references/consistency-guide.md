# Consistency Guide — 변경 후 정합성 재검증

## 재검증 체크리스트

변경 적용 후 다음 항목을 순서대로 확인:

### 1. PROBLEM ↔ Contract 정합
- §3 해결 조건이 Contract Statement의 "출력"과 일치하는가?
- Contract이 관찰 가능한 산출물(상태)을 서술하는가?

### 2. Contract ↔ Scope 정합
- Contract IN에 명시된 모든 항목이 Scope IN에 포함되는가?
- Scope OUT 항목에 위임처가 명시되어 있는가?
- IN (Invariant) 항목이 보존되어 있는가? 변경/삭제 시 사용자 확인이 있었는가?

### 3. Scope ↔ SPEC-TEST 정합
- SPEC-to-TEST 매트릭스에서 모든 IN 항목이 COVERED인가?
- UNCOVERED 항목 존재 시 → 서브태스크 추가 또는 IN 축소 필요
- Weak Oracle에 보강 경로가 명시되어 있는가? Oracle 등급 변경(Strong↔Weak) 시 보강 경로가 추가/제거되었는가?

### 4. Q1 과소검증
Contract 산출물 중 Gate Predicate로 검증되지 않는 항목이 있는가?
→ 있으면 서브태스크 추가

### 5. Q2 과대검증
Gate 서브태스크 중 Contract IN에 대응하지 않는 것이 있는가?
→ 있으면 서브태스크 삭제 또는 Contract IN 확장

### 6. Q3 원칙-테스트 괴리
런타임 행위를 약속하는 설계 원칙인데 구조만 검증하는 Oracle이 있는가?
→ 있으면 Oracle 보강 또는 설계 원칙을 Scope OUT으로 이동

### 7. Acceptance Gate 공식 + 구조 검증
- Gate 공식의 S1, S2, ... 가 실제 서브태스크 ID와 일치하는가?
- 삭제된 서브태스크가 Gate 공식에 남아있지 않은가?
- 추가된 서브태스크가 Gate 공식에 포함되어 있는가?
- 서브태스크 Predicate/Oracle 테이블에 Prerequisites 열이 존재하는가?

**갱신 절차**:
1. PLAN.md 체크리스트에서 현재 서브태스크 ID 전체 추출
2. 기존 Gate 공식 (`ACCEPT iff S1 ∧ S2 ∧ ...`)에서 변수 목록 추출
3. 차이 비교: (추가된 ID = 체크리스트에 있으나 Gate에 없음) / (삭제된 ID = Gate에 있으나 체크리스트에 없음)
4. Gate 공식 재작성: 추가 ID 포함, 삭제 ID 제거
5. 재작성 결과를 PLAN.md Acceptance Gate 섹션에 반영

### 8. 의존성 그래프(DAG)
- 서브태스크 추가/삭제 시 DAG 갱신
- 순환 의존 없음 확인

## 재검증 결과 처리

| 결과 | 처리 |
|------|------|
| 전항 PASS | 변경 완료. sessions.jsonl에 기록 |
| 일부 FAIL | 실패 항목 수정 후 재검증 |
| 다수 FAIL | cascade 분석 오류 가능. 변경 지점부터 재시작 |
