# 세션 로그 규약 (`sessions.jsonl`)

> **적용 대상**: `repos/{repo_slug}/projects/{project_name}/plans/{plan_name}/sessions.jsonl` (프로젝트 플랜 전용)

프로젝트 플랜 실행에 사용된 Claude Code 세션의 실행 이력을 기록한다.

---

## 1. 형식

**형식**: 1행 = 1 JSON 객체. `id`는 `S-{NNN}` 형식으로 **플랜 내 고유**.

> JSONL 채택 근거: `knowledge.jsonl`과 동일. append(`>>`) 한 줄로 기록 가능, `jq`로 즉시 필터링, git merge conflict 최소화.

**변경 정책**: append-only (수정·삭제 금지). 오류 정정이 필요하면 새 레코드에 `supersedes` 필드로 이전 id를 참조.

---

## 2. 스키마

```jsonl
{"id":"S-001","ts":"2026-02-26T13:00:00+09:00","slug":"mossy-yawning-dahl","round":1,"outcome":"partial","subtasks":["S0.1","S0.2"],"knowledge_ids":["K-001","K-002"],"commit":"d64ac5b","notes":"Phase 0 버그 수정. BUG-1~3 발견"}
```

### 필수 필드

| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | `S-{NNN}` | 플랜 내 순번. S-001, S-002, ... |
| `ts` | ISO 8601 | 세션 실행 시각 (KST 기준 +09:00) |
| `round` | number | 3-Agent 프로토콜 라운드 번호. 비 3-Agent 실행 시 1부터 순번. (`cycle`은 `round`의 별칭) |
| `outcome` | enum | 세션 결과. 아래 정의 참조 |

### 선택 필드

| 필드 | 타입 | 설명 |
|------|------|------|
| `slug` | string | Claude Code 세션 slug (`~/.claude/plans/{slug}.md`와 매핑) |
| `session_id` | UUID | Claude Code 세션 UUID (알 수 있을 때) |
| `subtasks` | string[] | 이 세션에서 완료한 서브태스크 ID 목록 |
| `knowledge_ids` | string[] | 이 세션에서 생성된 K-NNN 레코드 id 목록 |
| `commit` | string | 세션 종료 시점 git commit SHA (short hash, 7자) |
| `git_branch` | string | 작업 브랜치명 |
| `notes` | string | 세션 요약. 1~2문장, 핵심 행동과 결과를 기술 |
| `tags` | string[] | 검색용 키워드. 2~4개 권장 |
| `supersedes` | string[] | 이 레코드가 정정하는 이전 S-NNN id 목록 |
| `type` | string | 세션 유형. CRG 세션: `"crg"`. 체크포인트: `"checkpoint"`. 생략 시 일반 실행 세션 |
| `crg_file` | string | CRG 비평 파일 참조 (예: `"crg-1.md"`). CRG 세션에서만 사용 |
| `convergence` | object | 수렴 분석 결과 (라운드 2+). `{"verdict": "stagnation"}` |
| `pathological` | string[] | 감지된 병리 패턴. 허용 값은 §2-1 pathological enum 참조 |

### 2-1. `pathological` 허용 값 (Canonical Enum)

이 테이블이 pathological 필드의 **SOT(Single Source of Truth)**이다. 새 값 추가 시 이 테이블을 먼저 갱신하고, 참조하는 파일을 cascade 업데이트한다.

| 값 | 정의 | 감지 위치 | 소비 위치 |
|---|------|----------|----------|
| `stagnation` | 4개 지표 중 2개+ 2회 연속 해당 | execution-protocol.md §3 | SKILL.md §EXEC 6f, §UPDATE drift-entry |
| `oscillation` | N vs N-2에서 4개 지표 중 2개+ 해당 | execution-protocol.md §3 | §UPDATE drift-entry |
| `early_stagnation` | 재시도 3회+ 동일 blocker 2회 연속 미해소 | SKILL.md §EXEC 6f | §UPDATE drift-entry |
| `diminishing_returns` | 개선 정체 지표 3회 연속 해당 | execution-protocol.md §3 | SKILL.md §EXEC 6f, §UPDATE drift-entry |
| `drift` | drift_check에서 Goal/Scope 이탈 감지 | SKILL.md §EXEC 6c | §UPDATE drift-entry |

**값 추가 절차**: (1) 이 테이블에 행 추가 → (2) 감지 위치 파일(위 테이블의 "감지 위치" 컬럼)에 감지 로직 추가 → (3) 소비 위치 파일(위 테이블의 "소비 위치" 컬럼)에 처리 로직 추가 → (4) `execution-protocol.md §5` 검증자 프롬프트의 Observer 관점 인라인 열거 갱신

---

## 3. `outcome` 정의

### 3-1. 실행 세션 outcome

| outcome | 의미 | 판단 기준 |
|---------|------|----------|
| `success` | 계획된 작업 전부 완료 | 해당 라운드의 모든 서브태스크가 `[x]` |
| `partial` | 일부 완료, 잔여 작업 존재 | 서브태스크 일부만 `[x]` |
| `blocked` | 외부 요인으로 진행 불가 | blocker 발견, 환경 문제, 의존성 미충족 |
| `aborted` | 의도적 중단 | 접근 방식 변경, 플랜 재설계 결정 |

### 3-2. CRG 세션 outcome

CRG(Critical Review Gate) 세션에서 사용하는 outcome 값. CRG 세션은 `type: "crg"` 필드로 식별한다.

| outcome | 의미 | 다음 행동 |
|---------|------|----------|
| `crg_pass` | CRG 통과 | DONE 전환 |
| `conditional_pass` | CRG warning만 존재, 사용자 인지 후 통과 | DONE 전환 (warning은 notes에 기록) |
| `crg_fail` | CRG 실패, 수정 필요 | 수정 사이클 진행 (최대 2회) |
| `crg_needs_review` | 3회 CRG 모두 실패 | NEEDS_REVIEW 상태 전환 |

#### CRG 세션 레코드 선택 필드

| 필드 | 타입 | 설명 |
|------|------|------|
| `blocker` | 정수, 선택 | blocker 건수 |
| `warning` | 정수, 선택 | warning 건수 |
| `note` | 정수, 선택 | note 건수 |
| `agents` | 배열, 선택 | CRG 에이전트별 결과 요약 (예: `[{"agent":"X","blocker":1,"warning":0}, ...]`) |

> 상세: [`critical-review-gate.md`](critical-review-gate.md)

### 3-3. Checkpoint 레코드

서브태스크 완료 직후 기록하는 중간 상태 마커.
`check_history` 단계에서 압축 재개 지점 결정 및 CRG 미실행 감지에 사용된다.
`progress.md` 미존재 시 `context_refresh`의 상태 복원 대안으로 사용 가능하다.

| 필드 | 값 | 설명 |
|------|-----|------|
| `type` | `"checkpoint"` | 필수 |
| `subtask` | `"S1"` 등 | 방금 완료한 서브태스크 ID |
| `outcome` | `"success"` | 단일 서브태스크 완료 |
| `ts` | ISO 8601 | 완료 시각 |
| **`summary`** | string | **1줄 완료 요약 (신규)** |
| **`decisions`** | string[] | **후속 서브태스크에 영향을 주는 결정 (신규)** |
| **`warnings`** | string[] | **미해결 경고 (신규)** |

예시:
```jsonl
{"type":"checkpoint","subtask":"S3","outcome":"success","ts":"2026-03-11T09:05:00Z","summary":"Provider 파일 경로 해석 로직 구현","decisions":["glob 대신 정규식 매칭 채택"],"warnings":["기존 테스트 1건 실패 — S4에서 수정 필요"]}
```

> Checkpoint 레코드는 `id`(S-NNN)를 부여하지 않는다. 단순 마커이므로 sessions.jsonl id 시퀀스에 포함하지 않는다.

### 3-4. Gate 판정 근거 레코드

§CREATE의 G0~G4 통과 시 판정 근거를 기록하는 레코드. Gate 판정의 투명성을 확보하고 사후 감사를 가능하게 한다.

| 필드 | 값 | 설명 |
|------|-----|------|
| `type` | `"gate_decision"` | 필수 |
| `gate` | `"G0"` ~ `"G4"` | 판정한 Gate |
| `result` | `"PASS"` / `"CLARIFY"` / `"FAIL"` | 판정 결과 |
| `rationale` | string | 판정 근거 1~3문장 |
| `r_ids` | object | R-ID 기반 집합 연산 결과 (G1/G3/G4에서 사용) |
| `ts` | ISO 8601 | 판정 시각 |

예시:
```jsonl
{"type":"gate_decision","gate":"G1","result":"PASS","rationale":"§3의 R-1~R-3 모두 Contract 출력에 대응. Missing = ∅","r_ids":{"r_all":["R-1","R-2","R-3"],"contract_r":["R-1","R-2","R-3"],"missing":[]},"ts":"2026-03-12T14:00:00+09:00"}
{"type":"gate_decision","gate":"G3","result":"PASS","rationale":"R_uncovered = ∅. Oracle 등급: S1 Strong, S2 Strong, S3 Weak(보강 경로: grep 추가)","r_ids":{"r_all":["R-1","R-2","R-3"],"r_covered":["R-1","R-2","R-3"],"r_uncovered":[]},"ts":"2026-03-12T14:05:00+09:00"}
{"type":"gate_decision","gate":"G4","result":"PASS","rationale":"Q1: Under_tested = ∅. Q2: Over_tested = ∅. Q3: 런타임 행위 약속 없음.","r_ids":{"contract_r":["R-1","R-2","R-3"],"predicate_r":["R-1","R-2","R-3"],"under_tested":[],"over_tested":[]},"ts":"2026-03-12T14:10:00+09:00"}
```

> Gate 판정 근거 레코드는 `id`(S-NNN)를 부여하지 않는다. Checkpoint과 동일하게 마커 성격.

---

## 4. 기록 시점

| 시점 | 기록 방식 |
|------|----------|
| **3-Agent 프로토콜 실행 시** | C(Observer)가 라운드 종료 시 세션 레코드를 텍스트로 출력. 메인 에이전트가 append 수행 |
| **비 3-Agent 실행 시** | 세션 종료 시 에이전트(또는 PM)가 직접 append |

> knowledge.jsonl과 달리 2/3 기록 기준 없음. **모든 실행 세션은 필수 기록**.

---

## 5. `knowledge.jsonl`과의 관계

- `sessions.jsonl`의 `knowledge_ids` 필드로 해당 세션에서 생성된 knowledge 레코드를 참조
- `knowledge.jsonl`에 `session_id` 필드를 추가하지 않음
- 시간 기반 상관: `sessions.jsonl`의 `ts`와 `knowledge.jsonl`의 `ts` + `round`로 교차 참조 가능

---

## 6. 크로스 플랜 쿼리

공용 `repos/sessions.jsonl` 파일은 생성하지 않는다. 세션은 본질적으로 플랜-로컬이므로 `jq`로 집계:

```bash
# 특정 repo의 전체 프로젝트 플랜 세션 시간순 정렬
cat repos/{repo_slug}/projects/*/plans/*/sessions.jsonl | jq -s 'sort_by(.ts)'

# 특정 project 내 세션 필터
cat repos/{repo_slug}/projects/{project_name}/plans/*/sessions.jsonl | jq 'select(.outcome == "blocked")'

# 플랜별 세션 수 집계
for d in repos/{repo_slug}/projects/{project_name}/plans/*/; do
  echo "$(basename $d): $(wc -l < "$d/sessions.jsonl" 2>/dev/null || echo 0)"
done
```

---

## 7. 빠른 참조

### 새 ID 부여

```bash
# 마지막 세션 ID 확인
tail -1 plans/{plan-name}/sessions.jsonl | jq -r .id
# S-003 → 다음은 S-004
```

### 재개 시 참조

중단된 플랜 재개 시:
1. `sessions.jsonl` 마지막 레코드의 `outcome`과 `subtasks` 확인
2. PLAN.md 체크리스트에서 첫 번째 `[ ]` 서브태스크부터 재개
3. 이전 세션이 `partial`/`blocked`이면 `notes` 필드 맥락 참조
