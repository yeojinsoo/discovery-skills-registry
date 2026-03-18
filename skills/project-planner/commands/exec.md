## §EXEC — 프로젝트 플랜 실행

프로젝트 플랜 실행 워크플로우를 실행한다.

### 스텝별 참조

| 스텝 | 참조 | 설명 |
|------|------|------|
| §6d 워커 | 이 파일 내 인라인 (§6d) | Worker 프롬프트 템플릿 |
| §6e 검증자 | 이 파일 내 인라인 (§6e) | Verifier 프롬프트 템플릿 |
| §6b 인출 | `rules/knowledge-conventions.md §6` | Knowledge 인출 전략 (공유 SOT) |
| §8 CRG | 이 파일 내 인라인 (§8) | CRG 프로토콜 |
| §9-2 세션 | `rules/session-log-conventions.md §2 (스키마) + §7 (ID 부여)` | 세션 레코드 스키마 (공유 SOT) |
| §3 이력 | `rules/session-log-conventions.md §3-4` | gate_decision 스키마 (공유 SOT) |

### StateGraph

```
load_plan → validate_plan → check_history
  → GATE-EXEC-CONFIRM(AskUserQuestion)
  → set_in_progress → init_progress
  → exec_loop:
      context_refresh → pick_subtask → inject_knowledge → drift_check(3+완료시)
      → Agent(worker): 코드 수정
      → Agent(verifier): Tester+Reviewer+Observer 통합 검증
      → blocker? → fix_and_retry (max 5) → [x]_전환 → write_checkpoint → update_progress
  → acceptance_gate_check
  → CRG(X + Y + Z 병렬) → crg_judge
      ├─ PASS / Conditional PASS → set_done + append_session → report → GATE-NEXT-STEP
      │     [단일] [1] summary(Rec) [2] validate [3] 나중에
      │     [다중] [1] validate(Rec) [2] summary [3] 나중에
      ├─ NEEDS_FIX → fix_and_re-CRG (max 2 fix cycles, CRG 총 3회) → 3차 NEEDS_FIX → NEEDS_REVIEW
      └─ NEEDS_REVIEW → escalate
```

### 1. load_plan — 프로젝트 플랜 로드

```
Read: ${PLAN_DIR}/PLAN.md
Read: ${PLAN_DIR}/PROBLEM.md
Read: ${PLAN_DIR}/sessions.jsonl
```

`${PLAN_DIR}/PLAN.md` 미존재 → 에러: "프로젝트 플랜 '{plan_name}'을 찾을 수 없습니다."

서브태스크 8개 이상인 프로젝트 플랜은 sessions.jsonl 마지막 2개 레코드만 로드.

PLAN.md에서 추출:
- Contract Statement, Scope Boundary
- Acceptance Gate 공식
- 서브태스크 목록 + 각 Predicate/Oracle 테이블
- 체크리스트 (`[ ]` / `[x]` 상태)

### 2. validate_plan — 프로젝트 플랜 유효성 검증

- 상태 `ABANDONED` → 중단 + 사유 안내
- Acceptance Gate 미정의 → 중단
- Predicate/Oracle 테이블 없는 서브태스크 → 경고

### 3. check_history — 세션 이력 확인

sessions.jsonl 비어있으면 → 신규 실행.
기존 레코드가 있으면:

1. `type:checkpoint` 레코드로 마지막 완료 서브태스크 확인 → PLAN.md `[x]` 목록과 대조.
   checkpoint 없는 `[x]`가 있으면 "압축 재개 의심" 로그 출력 → 첫 번째 미완료 `[ ]` 서브태스크부터 재개.

2. **[Guard] 모든 서브태스크 `[x]` 완료 상태인데 `type:crg` 레코드가 없으면:**
   → "⚠️ CRG 미실행 감지 (컨텍스트 압축으로 인한 스킵 가능성). CRG를 지금 실행합니다."
   → acceptance_gate_check → CRG 실행 (§8)

3. **[Guard-B] Gate Decision 레코드 검증** (이 스텝에서 `session-log-conventions.md §3-4` 참조):
   create에서 G1/G3/G4 통과 시 기록된 gate_decision 레코드를 확인한다.
   ```bash
   # 조건: sessions.jsonl 레코드가 소수(신규 생성 직후)일 때만 검증
   if [ $(wc -l < "${PLAN_DIR}/sessions.jsonl") -le 3 ]; then
     g1=$(jq 'select(.type=="gate_decision" and .gate=="G1")' "${PLAN_DIR}/sessions.jsonl")
     g3=$(jq 'select(.type=="gate_decision" and .gate=="G3")' "${PLAN_DIR}/sessions.jsonl")
     g4=$(jq 'select(.type=="gate_decision" and .gate=="G4")' "${PLAN_DIR}/sessions.jsonl")

     if [ -z "$g1" ] || [ -z "$g3" ] || [ -z "$g4" ]; then
       echo "⚠️ 불완전한 Gate Decision 레코드 감지."
       echo "  G1: $([ -z "$g1" ] && echo '❌ 누락' || echo '✓')"
       echo "  G3: $([ -z "$g3" ] && echo '❌ 누락' || echo '✓')"
       echo "  G4: $([ -z "$g4" ] && echo '❌ 누락' || echo '✓')"
       # → AskUserQuestion: "[1] 진행 [2] 플랜 재생성"
     fi
   fi
   ```

4. 모든 서브태스크 `[x]` 완료 + `type:crg` 있음 + `outcome:crg_pass` → DONE 확인

5. **progress.md 존재 확인**:
   - 존재: 이전 실행 상태 확인, `context_refresh` 시 활용
   - 미존재 + checkpoint 있음: `init_progress`에서 checkpoint 기반 재구성
   - 양쪽 없음: 신규 실행

상태가 `DONE` → AskUserQuestion:
- [1] 전체 재실행 (PLANNED로 초기화)
- [2] 특정 서브태스크만 재검증
- [3] 취소

### 4. GATE-EXEC-CONFIRM

AskUserQuestion으로 실행 범위 확인.

### 5. set_in_progress

PLAN.md 상태 `PLANNED` → `IN_PROGRESS`.

### 5-1. init_progress

§6g B 스키마에 따라 progress.md 초기 파일 생성.

- `check_history`에서 기완료 서브태스크 감지 시: 해당 목록 포함하여 생성
- `${PROGRESS_FILE}`이 이미 존재하고 PLAN.md 상태가 `IN_PROGRESS`: 이전 실행 상태로 간주, 읽기만 수행
- `${PROGRESS_FILE}`이 이미 존재하고 PLAN.md 상태가 `PLANNED` (DONE→PLANNED 재실행): 기존 progress.md 삭제 후 신규 생성 (이전 실행의 "모두 완료" 상태가 오염되지 않도록)
- Write tool로 `${PROGRESS_FILE}` 생성

> **서브태스크 완료 상태 SOT**: (1) PLAN.md `[x]` 체크리스트 = 최종 권위 (2) progress.md = 실행 중 캐시 (3) sessions.jsonl checkpoint = 감사 로그. 불일치 시 PLAN.md 우선.

### 6. exec_loop — 오케스트레이터-워커 실행 루프

미완료 서브태스크 순서대로 반복:

#### 6-pre. context_refresh (매 서브태스크 시작 시)

대화 기억이 아닌 파일에서 읽은 정보를 진실로 사용한다:

**방법 1 (기본)**: progress.md 존재 시
1. `Read(${PROGRESS_FILE})` — "여기까지 왔다"
2. `Read(${PLAN_DIR}/PLAN.md, 해당 서브태스크 구간)` — "이번에 할 것"

**방법 2 (progress.md 미존재 시)**: checkpoint 기반 복원
1. `jq -s 'map(select(.type=="checkpoint")) | last' ${PLAN_DIR}/sessions.jsonl`
   → 마지막 checkpoint의 summary + decisions + warnings로 상태 복원
2. `Read(${PLAN_DIR}/PLAN.md, 해당 서브태스크 구간)` — "이번에 할 것"

**핵심 원칙**: 압축 여부와 무관하게, 이 파일들이 현재 상태의 SOT.

#### 6a. pick_subtask
체크리스트에서 다음 `[ ]` 서브태스크 선택. DAG 의존성 확인.

#### 6b. inject_knowledge + inject_codebase_context

이 스텝에서 `knowledge-conventions.md §6` 참조 (Knowledge 인출 전략).

관련 knowledge 인출 (2계층):
1. ${REPO_DIR}/knowledge.jsonl (레포 레벨, 있으면 로드)
2. ${KNOW_FILE} (${PROJ_DIR}/knowledge.jsonl, 프로젝트 레벨 — 충돌 시 우선)

**코드베이스 구조 주입** (Worker 탐색 라운드 절감):
이전 서브태스크에서 Worker가 변경한 파일 경로 누적 목록을 progress.md의 완료 서브태스크에서 추출하여, Worker 프롬프트의 `관련 파일 맵`으로 주입한다. 첫 번째 서브태스크에서는 PLAN.md의 Action에 언급된 파일 경로 또는 `ls` 기반 디렉토리 구조 요약(주요 디렉토리 1~2 depth)을 제공한다.

#### 6c. drift_check (완료 서브태스크 3개 이상 시 활성화)

1. **Goal Alignment**: Contract Statement 출력 산출물을 향해 진행 중인가?
2. **Scope Containment**: Scope Boundary OUT 항목을 침범했는가?

"아니오"이면:
- sessions.jsonl에 `pathological: ["drift"]` 기록
- AskUserQuestion: "[1] 진행 [2] /project-planner update 호출 [3] 중단"
- [2] 선택 시 → Skill 도구로 `/project-planner update {plan_name} --project {project_name} --repo {repo_slug}` 호출

#### 6d. Agent(worker) — 코드 수정 (서브에이전트 위임)

아래 워커 프롬프트 템플릿으로 Agent tool 호출.

**워커 프롬프트 템플릿**:

```
너는 Worker Agent이다. 주어진 서브태스크의 Action을 구현한다.

**입력**:
- 서브태스크: {subtask_id} — {title}
- Action: {action_description}
- 핵심 결정 (progress.md): {key_decisions}
- 활성 경고 (progress.md): {active_warnings}
- Invariant 항목 (Scope Boundary): {invariant_items_or_none}
- 관련 파일 맵: {이전 서브태스크 변경 파일 경로 누적 목록 또는 디렉토리 구조 요약}
- 관련 knowledge: {knowledge_entries}

**규칙**:
- Read, Write, Edit, Bash, Grep, Glob 도구 사용 가능
- Agent, AskUserQuestion 도구 사용 금지 — 판단이 필요한 상황은 needs_decision: true로 반환
- Invariant 항목으로 표시된 영역은 변경 금지. 불가피한 경우 needs_decision: true로 반환
- 비가역 액션(삭제, 리팩터링, 마이그레이션, API 변경, 외부 시스템 호출) → 내부에서 Predict-then-Verify 수행:
  1. 예상 결과 서술
  2. 실행
  3. 결과가 예상과 다르면 rollback 후 needs_decision: true
- 핵심 결정과 활성 경고를 반드시 참고하여 이전 결정과 충돌하지 않도록 구현

**출력 형식** (반드시 이 형식으로만 출력):
## 워커 결과 — {subtask_id}
- 상태: 성공/실패
- 변경 파일: {파일 경로 목록}
- 주요 변경: {1-2문장 요약}
- 결정 사항: {후속 서브태스크에 영향을 미치는 결정. 없으면 "없음"}
- 경고: {주의 사항. 없으면 "없음"}
- needs_decision: {true/false}
```

워커 프롬프트에 주입:
- 서브태스크 Action (PLAN.md)
- progress.md의 핵심 결정 + 활성 경고
- Scope Boundary의 Invariant 항목 (있으면)
- 관련 파일 맵 (6b inject_codebase_context에서 준비)
- 관련 knowledge 레코드

메인 컨텍스트에는 구조화된 결과 요약만 반환 (5~10줄).
`needs_decision: true` 시 오케스트레이터가 AskUserQuestion 후 워커 재호출.

#### 6e. Agent(verifier) — 통합 검증 (서브에이전트 위임)

아래 검증자 프롬프트 템플릿으로 Agent tool 호출.

**검증자 프롬프트 템플릿**:

```
너는 Verifier Agent이다. 워커의 구현 결과를 3개 관점(Tester→Reviewer→Observer)에서 순차 검증한다.

**입력**:
- 서브태스크: {subtask_id} — {title}
- Predicate/Oracle 테이블 (Prerequisites 열 포함): {table}
- SPEC-to-TEST 보강 경로 (Weak Oracle 해당 시): {strengthening_paths_or_none}
- 워커 결과 요약: {worker_result}
- knowledge.jsonl 교훈: {knowledge_entries}
- 이전 라운드 blocker (재시도 시): {previous_blockers}

**규칙**:
- Bash, Read, Grep, Glob 도구만 사용. Write, Edit, Agent 도구 금지 — 코드 수정 불가
- 내부에서 3개 관점을 순차 적용:

### Phase 1 — Tester 관점
- **Prerequisites 확인**: Oracle 실행 전, 테이블의 Prerequisites 열을 확인.
  - "—"이면 즉시 Oracle 실행
  - Safe Prerequisites (빌드, 컴파일 등 부작용 없는 명령) → Verifier가 직접 실행 후 Oracle 실행
  - Stateful Prerequisites (마이그레이션 적용, 서비스 시작 등 상태 변경) → Verifier 실행 불가. blocker로 반환: "Prerequisites 미충족: {조건}. Worker에서 먼저 실행 필요"
- Predicate/Oracle을 실행하여 pass/fail 판정
- knowledge.jsonl의 기존 pitfall/constraint를 테스트 전략에 반영
- 이전에 false-positive가 발생한 Oracle 패턴을 인지하고 보강

### Phase 2 — Reviewer 관점
- Phase 1 결과를 비판적으로 검토
- knowledge.jsonl 교훈 위반 여부 검증
- **Invariant 침범 확인**: 워커의 변경 파일 목록과 Scope Boundary IN (Invariant) 항목을 대조하여 침범 여부 확인. 침범 시 severity = blocker
- 이전 라운드 blocker 재발 여부 명시적 판정
- 동일 유형 실수 재발 시 severity 상향 (warning → blocker)

### Phase 3 — Observer 관점
- Phase 1+2 종합하여 수정안 제시
- 라운드 2+ 수렴 분석 (대상 반복/유형 고착/설명 반복/개선 정체)
- knowledge.jsonl 기록 판별 (2/3 기준 또는 pathological 트리거)
- sessions.jsonl 레코드 출력

**수렴 분석 기준** (라운드 2+ 시):

| 판정 지표 | 관찰 기준 | 의미 |
|----------|---------|------|
| **대상 반복** | 동일 파일/함수/모듈명이 blocker에 2회 이상 등장 | 수정이 근본 원인을 해소하지 못함 |
| **유형 고착** | blocker/warning/note 비율이 이전 라운드와 실질적으로 동일 | 이슈 구조가 변화하지 않음 |
| **설명 반복** | blocker 설명이 이전 라운드와 동일하거나 표현만 다름 | 동일 이슈가 재발 |
| **개선 정체** | blocker 건수가 이전 라운드 대비 1건 이하 감소 (3회 연속) | 수정이 표면적이며 근본 원인에 도달하지 못함 |

**이슈 severity 기준**:
- blocker: 수정 없이는 Predicate 의도 미충족
- warning: 통과하지만 보강 권고
- note: 스타일/가독성/개선 권고

**출력 형식** (반드시 이 형식으로만 출력):
## 검증 결과 — {subtask_id}

### 테스트 (Tester)
| Predicate | Oracle | 결과 | 비고 |
|-----------|--------|:----:|------|
| {predicate} | {oracle} | PASS/FAIL | {비고} |

### 참조한 교훈
- K-{NNN}: {insight} → {어떤 행동을 변경했는지}
- (해당 없으면 "해당 없음")

### 리뷰 (Reviewer)
| # | severity | 대상 | 이슈 | 근거 |
|:-:|:--------:|------|------|------|
| 1 | blocker/warning/note | {대상} | {이슈} | {근거} |

### False Positive 검사
- {Oracle이 구현과 무관하게 항상 통과할 구조인지 확인}
- {Weak Oracle이 PASS인 경우: SPEC-to-TEST 보강 경로가 있으면 즉시 실행하여 진위 확인}

### 종합 (Observer)
- 판정: PASS/FAIL
- blocker: {N}건 / warning: {N}건 / note: {N}건

#### 수정안
(WARNING/STAGNATION/DIMINISHING_RETURNS 시) **[렌즈: {Simplifier|Contrarian|Researcher}]** {선택 사유}
{B의 각 blocker에 대한 구체적 수정 방법}

#### 수렴 분석 (라운드 2+ 시)
- 대상 반복: {해당/비해당} — {반복 대상 파일/함수명}
- 유형 고착: {해당/비해당} — {이슈 유형 비율 변화 요약}
- 설명 반복: {해당/비해당} — {동일 설명 건수}
- 개선 정체: {해당/비해당} — {blocker 건수 변화: N-1→N}
- 판정: {정상/WARNING/STAGNATION/DIMINISHING_RETURNS/OSCILLATION}
- (비정상 시) 근본 원인 분석: {원인}
- (비정상 시) 전략 전환 렌즈: {Simplifier|Contrarian|Researcher} — {사유}

#### knowledge.jsonl 레코드 (해당 시)
```jsonl
{"id":"K-{NNN}","plan":"{plan-name}","round":{N},"ts":"{ISO}","cat":"{cat}","confidence":"observed","insight":"{교훈}","tags":[...]}
```
(2/3 미충족 시: "기록 스킵 — 사유: {미충족 기준}")

#### sessions.jsonl 레코드
```jsonl
{"id":"S-{NNN}","ts":"{ISO}","round":{N},"outcome":"{outcome}","subtasks":[...],"knowledge_ids":[...],"notes":"..."}
```

## Lessons
- {cat}: {insight} | tags: [{tags}] | src: {src}
- (기록할 교훈이 없으면 "없음"이라고 기재)
```

검증자 프롬프트에 주입:
- Predicate/Oracle 테이블 (Prerequisites 열 포함)
- SPEC-to-TEST 보강 경로 (해당 서브태스크의 Weak Oracle에 대한 보강 경로)
- 워커 결과 요약 (6d 반환값)
- knowledge 레코드
- 이전 라운드 blocker (재시도 시)

검증자 내부에서 3개 관점 순차 적용 (Tester→Reviewer→Observer).
자기 확인 편향 방지 = 구현(워커)과 검증(검증자)의 서브에이전트 분리로 강화.

#### 6f. blocker 처리

- blocker 0건 → `[ ]` → `[x]` 전환
- blocker 존재 → 워커 재호출 시 검증자의 수정안 + 이전 blocker를 워커 프롬프트에 포함하여 6d~6e 재실행. 서브태스크당 최대 5회.
  - **슬라이딩 윈도우**: Worker에는 직전 라운드(N-1)의 blocker + 수정안만 주입. 라운드 1~N-2 정보는 "이전 {N-2}라운드에서 해결된 blocker: {건수}건" 1줄 요약으로 대체. Verifier에는 수렴 분석(Oscillation N vs N-2 비교)을 위해 직전 2라운드(N-1, N-2)의 blocker를 유지하되, 그 이전은 건수 요약으로 대체.
- **조기 stagnation 감지**: 재시도 3회+ + 동일 blocker 2회 연속 → 전략 전환 렌즈 적용:
  - **Simplifier**: 더 작은 단위로 분해
  - **Contrarian**: 전제를 뒤집어 대안 경로 탐색
  - **Researcher**: 코드베이스에서 유사 패턴 검색
- 5회 초과 → sessions.jsonl append + AskUserQuestion:
  "[1] 수동 수정 후 재검증 [2] /project-planner update {plan_name} --project {project_name} --repo {repo_slug} [3] 건너뛰고 다음 서브태스크"

#### 6g. write_checkpoint + update_progress

A. **sessions.jsonl checkpoint** (기존):
blocker 0건으로 `[x]` 전환 직후 sessions.jsonl에 즉시 append:

```jsonl
{"type":"checkpoint","subtask":"{S_ID}","outcome":"success","ts":"{ISO 8601}","summary":"{1줄 완료 요약}","decisions":["{결정 사항}"],"warnings":["{미해결 경고}"]}
```

필드 설명:
- `summary`: 워커 결과의 "주요 변경" 1줄 요약
- `decisions`: 워커 결과의 "결정 사항" + 검증자(Observer)의 수정안에서 추출
- `warnings`: 검증자의 warning 중 미해결 사항. 비어있으면 `[]`

이 레코드는 컨텍스트 압축 재개 시 `check_history`의 Guard Clause가 완료 상태를 정확히 감지하는 데 사용되며, `progress.md` 미존재 시 `context_refresh`의 상태 복원 대안으로도 활용된다.
`id` 시퀀스(S-NNN)는 부여하지 않는다.

B. **progress.md 갱신** (신규):
Write tool로 `${PROGRESS_FILE}` 전체 덮어쓰기. 스키마:

```markdown
# Progress — {plan_name}
## 마지막 갱신: {ISO 8601}
## 완료 서브태스크
- [x] S1: {title} — {1줄 요약, max 80자} | 변경: {주요 파일 경로, max 3개}
## 핵심 결정
- {후속 서브태스크에 영향을 미치는 결정만, 최대 10개}
## 활성 경고
- {미해결이지만 후속에서 필요한 사항}
## 현재/다음
- 현재: {진행 중 서브태스크 또는 "없음"}
- 다음: [ ] S{N}: {title}
```

50줄 이내. 완료 서브태스크 15개 초과 시 오래된 항목 축약.

갱신 내용:
- 완료 서브태스크: 현재 서브태스크 추가 (주요 변경 1줄 요약 + 변경 파일 경로 max 3개)
- 핵심 결정: 워커 결과의 "결정 사항" + 검증자(Observer 관점)의 수정안에서 결정 사항 추출하여 반영
- 활성 경고: 검증자의 warning 중 미해결 사항 추가, 해결된 것 제거
- 현재/다음: 다음 미완료 서브태스크

#### 6h. knowledge 기록

검증자 출력의 `#### knowledge.jsonl 레코드` 섹션에서 JSONL을 추출하여 `${KNOW_FILE}`에 append.
- `"기록 스킵"`이 명시된 경우 기록하지 않는다
- 2/3 기록 기준 판단은 검증자 내부에서 수행됨 (`execution-protocol.md §4-1`)


### 7. acceptance_gate_check

모든 서브태스크 `[x]` 완료 후 Acceptance Gate 공식 평가.
일부 FAIL → 실패 서브태스크 재실행 또는 AskUserQuestion.

### 8. CRG — Critical Review Gate

Acceptance Gate 통과 후 DONE 전환 직전에 필수 수행. 전체 프로젝트 플랜 실행 결과를 서로 다른 관점의 에이전트팀이 홀리스틱하게 검토하여, DONE 선언의 정당성을 검증한다.

#### 8-1. CRG 에이전트 역할 정의

| Agent | 역할명 | 책임 | 입력 | 출력 |
|:-----:|--------|------|------|------|
| **X** | Code Verifier | 실제 코드 변경이 PLAN.md 서브태스크 명세와 일치하는지 검증 | PLAN.md, 코드베이스 (변경 파일), sessions.jsonl | 명세-구현 갭 목록 |
| **Y** | Plan Integrity Critic | Oracle 유효성, 증거 품질, SPEC-TEST 정합성을 비판적으로 검토 | PLAN.md, sessions.jsonl, knowledge.jsonl, intermediates | Oracle 등급 재판정 + 문서 정합성 이슈 |
| **Z** | Regression & Risk Assessor | 변경의 회귀 영향과 미검증 영역을 분석. Scope Boundary의 **IN (Invariant)** 항목을 우선 회귀 검증 대상으로 식별. Invariant 위반 시 severity = blocker. 미검증 영역 분석은 §8-1a 가이드라인 참조 | PLAN.md Scope (Invariant 항목 포함), 코드 변경 diff, 관련 모듈 | 회귀 리스크 판정 + Invariant 보존 검증 + 미검증 영역 목록 |

3개 에이전트는 **병렬 실행** 가능 (상호 의존 없음). Agent tool로 X, Y, Z를 병렬 실행한다.

**§8-1a. Z 에이전트 미검증 영역 검사 가이드라인**:

Z는 "미검증 영역 목록" 출력 시, 변경된 코드에 대해 다음 항목을 검사한다 (해당 시):

1. **에러 경로**: 에러/예외 처리 경로가 존재하고 합리적인가 (try-catch, rejection handler 등)
2. **테스트 커버리지**: 변경된 소스 파일에 대응하는 테스트가 존재하는가, 신규 로직에 테스트가 추가되었는가
3. **하위 호환성**: public API, 이벤트 인터페이스, 데이터 스키마 변경이 기존 소비자에게 breaking change를 유발하는가

설정 파일만 변경하는 플랜 등 해당 없는 항목은 "해당 없음"으로 생략한다.

**warning 발행 시 필수 포함 필드**: CRG 에이전트(X/Y/Z)가 warning severity 이슈를 발행할 때, 각 warning에 다음 3필드를 함께 출력한다:
- **원인**: 왜 발생했는가 (1줄)
- **수정**: 무엇을 어떻게 바꾸는가 (1줄)
- **이점**: 수정 전 대비 무엇을 얻는가 (1줄)
- X: 읽기 + 빌드/테스트 명령 실행 권한 (Bash, Grep, Read)
- Y: 읽기 전용 (Read, Grep, Glob)
- Z: 읽기 + diff 분석 (Read, Grep, Glob, Bash — git diff만 허용)

#### 8-2. CRG 실행 흐름

```
1. 트리거: 모든 서브태스크 [x] + Acceptance Gate PASS
2. [병렬] X + Y + Z 실행 → 각각 이슈 목록 출력
3. 이슈 종합 → severity 분류 (blocker / warning / note)
4. 판정:
   ├─ blocker 0건 + warning만 → Conditional PASS (사용자 확인)
   ├─ blocker 0건 + note만 → PASS
   ├─ blocker 1건+ → NEEDS_FIX
   ├─ CRG-N blocker ≈ CRG-N-2 blocker → AskUserQuestion: Oscillation 경고
   └─ 사이클 3회째 NEEDS_FIX → NEEDS_REVIEW
5. PASS → DONE 전환
6. NEEDS_FIX → 수정 사이클 (§8-3 참조)
```

**이슈 severity 정의**:

| Severity | 정의 | CRG 판정 영향 |
|----------|------|--------------|
| **blocker** | Predicate 미충족, 또는 Oracle이 Invalid | **NEEDS_FIX** 즉시 발행 |
| **warning** | 설계 의도와 부분 괴리. 기능은 정상 | **Conditional PASS** — 사용자 확인 후 진행 |
| **note** | 개선 권고, 향후 플랜에서 반영 가능 | **PASS** — knowledge.jsonl 기록 권고 |

> severity 체계는 서브태스크 Verifier와 CRG에서 동일하다 (blocker/warning/note). 스코프만 다르다 — Verifier는 단일 서브태스크, CRG는 전체 플랜.

**Conditional PASS 처리**: warning만 존재 시, 각 warning에 대해 의사결정 보조 정보를 제시한 후 사용자에게 선택지를 제공한다.

**warning 보고 형식** (CRG 에이전트가 warning 발행 시 필수 포함):
```
⚠️ warning {N}건:

[W1] {대상}: {이슈 1줄}
  원인: {왜 발생했는가 — 1줄}
  수정: {무엇을 어떻게 바꾸는가 — 1줄}
  이점: {수정 전 대비 무엇을 얻는가 — 1줄}

[W2] ...
```

**선택지**:
- [1] "인지하고 진행" → PASS로 기록 (warning 내용은 sessions.jsonl `notes`에 포함)
- [2] "warning 항목만 수정" → warning 목록을 워커에게 전달 → 수정 → CRG 재실행 (CRG 사이클 카운트에 포함, 플랜 재설계/재실행 불필요)
- [3] "수정 필요 (전체)" → NEEDS_FIX와 동일 수정 사이클 (플랜 재설계 + 재실행)

#### 8-3. 수정 사이클 (Fix Cycle)

```
NEEDS_FIX 판정
  → CRG 비평 내용을 crg-{N}.md로 저장
  → sessions.jsonl에 CRG 세션 레코드 append (outcome: "crg_fail")
  → 사용자 확인 (AskUserQuestion): "CRG 권고에 따라 플랜 수정 + 재실행을 진행할까요?"
    ├─ YES → /project-planner update {plan_name} --project {project_name} --repo {repo_slug} (CRG 권고 기반) → /project-planner exec {plan_name} --project {project_name} --repo {repo_slug} 재실행 → CRG 재실행
    └─ NO  → 사용자 수동 진행 또는 현 상태 유지
```

| 항목 | 값 |
|------|-----|
| 최대 CRG 실행 횟수 | **3회** |
| 최대 수정+재실행 사이클 | **2회** (1차 CRG 실패 → 수정 → 2차 CRG 실패 → 수정 → 3차 CRG) |
| 3차 CRG도 NEEDS_FIX | 상태를 `NEEDS_REVIEW`로 전환. sessions.jsonl에 `{"type":"crg","outcome":"crg_needs_review","ts":"{ISO 8601}","notes":"CRG 3회 실패"}` append |
| **NEEDS_REVIEW 경유 순환 가드** | NEEDS_REVIEW→IN_PROGRESS 전환 후 exec 재진입 시, sessions.jsonl에서 이전 `crg_needs_review` 레코드를 확인. 동일 플랜에서 **통산 NEEDS_REVIEW 2회** 도달 시 자동 재실행 선택지를 제거하고 "프로젝트 플랜 재설계 (/project-planner create) 또는 ABANDONED" 안내 |

각 수정 사이클에서 변경 가능한 범위는 CRG가 발견한 이슈 항목으로 한정한다. CRG 비평과 무관한 프로젝트 플랜 확장은 별도 프로젝트 플랜으로 분리한다.

#### 8-4. Skip 허용 조건 (기계적 판정)

다음 **모든 조건**을 기계적으로 검증하여 CRG skip 허용 여부를 판정한다:

| 조건 | 검증 방법 | 유형 |
|------|---------|:----:|
| 서브태스크 ≤ 2개 | `grep -c '^\- \[([ x])\]' PLAN.md` | 기계적 |
| 소스 코드 파일 변경 0건 | `git diff --name-only \| grep -E '\.(py\|js\|ts\|java\|go\|rs)$'` 결과 비어있음 | 기계적 |
| 신규 소스 파일 추가 없음 | 위 결과에서 new file 없음 | 기계적 |
| 사용자 명시적 요청 | AskUserQuestion 확인 | HITL |

**금지 표현**: "단순 프로젝트 플랜", "대략 맞다", "우려 없음" 등 주관적 기준

**skip 시 필수**: sessions.jsonl에 기록:
```jsonl
{"type":"crg","outcome":"crg_pass","notes":"CRG skip — 조건: subtask_count=2, code_files=0, new_files=0","ts":"{ISO 8601}"}
```
notes에 기계적 조건 결과를 기록하여 사후 감사 가능. 이 레코드가 있으면 Step 9-0의 jq 확인을 통과한다.

### 9. set_done + append_session

**Step 9-0 (기계적 확인 — 0 hop)**:
```bash
result=$(jq -r 'select(.type=="crg" and (.outcome=="crg_pass" or .outcome=="conditional_pass"))' ${PLAN_DIR}/sessions.jsonl)
[ -z "$result" ] && echo "⚠️ CRG 미실행. Step 8로 돌아간다." || echo "CRG 확인됨 — Step 9-1 진행"
```

- `result` 비어있음 (`-z`) → Step 8 실행
- `result` 있음 → Step 9-1로 진행

**Step 9-1**: PLAN.md 상태 → `DONE`

**Step 9-2**: sessions.jsonl에 CRG 통과 레코드 append

**Step 9-3**: 이 스텝에서 `session-log-conventions.md §7` 참조. 해당 스키마로 실행 세션 레코드 append

**Step 9-4**: ${HIST_FILE}에 append:
```jsonl
{"type":"plan_completed","name":"{plan_name}","ts":"{ISO 8601}","status":"DONE"}
```

### 10. report + GATE-NEXT-STEP

완료 서브태스크 목록, CRG 결과, 생성된 knowledge 레코드를 출력한다.

**GATE-NEXT-STEP** — 다음 단계를 AskUserQuestion으로 제시:

플랜 수 판정: `ls "${PLANS_DIR}" | wc -l`

**[A] 단일 플랜 프로젝트 (1개)**:

AskUserQuestion:
  "프로젝트 플랜 `{plan_name}` 실행이 완료되었습니다. 다음 단계를 선택하세요:"
  [1] PR 본문 생성 (`/project-planner summary create`) (Recommended)
  [2] DoD 검증 (`/project-planner validate`)
  [3] 나중에

[1] 선택 시 → Skill 도구로 `/project-planner summary create --project {project_name} --repo {repo_slug}` 호출
[2] 선택 시 → Skill 도구로 `/project-planner validate --project {project_name} --repo {repo_slug}` 호출

**[B] 다중 플랜 프로젝트 (2개+)**:

AskUserQuestion:
  "프로젝트 플랜 `{plan_name}` 실행이 완료되었습니다.
   이 프로젝트에 {N}개 플랜이 있습니다. 다음 단계를 선택하세요:"
  [1] DoD 검증 (`/project-planner validate`) (Recommended)
  [2] PR 본문 생성 (`/project-planner summary create`)
  [3] 나중에

[1] 선택 시 → Skill 도구로 `/project-planner validate --project {project_name} --repo {repo_slug}` 호출
[2] 선택 시 → Skill 도구로 `/project-planner summary create --project {project_name} --repo {repo_slug}` 호출
