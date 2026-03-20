# Critical Review Gate (CRG) — 실행 스펙 완료 전 다관점 검토 프로토콜

> ⚠️ **인라인 통합 안내**: §1-3 CRG 프로토콜 + §8-2 skip 조건은 `commands/exec.md` §8에 인라인 통합됨. 수정 시 command 파일을 직접 수정할 것.

> **선행 조건**: 실행 프로토콜(R-004) 이해 필수 (`rules/execution-protocol.md`)
> **트리거**: 모든 서브태스크 `[x]` + Acceptance Gate 통과 후, DONE 전환 직전

---

## 0. 핵심 원리

실행 프로토콜(R-004)은 **서브태스크 단위**의 국소 검증이다. 서브태스크별로 Predicate/Oracle이 충족되어도, 전체 실행 스펙 관점에서 다음 문제가 잔존할 수 있다:

- Oracle이 실제 검증 방법과 불일치 (문서상 Strong이나 실제 Weak)
- Scope Boundary 외부로 영향이 누출
- 코드 수정 자체는 올바르나 SPEC.md 명세와 괴리

CRG는 **전체 실행 스펙 실행 결과**를 서로 다른 관점의 에이전트팀이 홀리스틱하게 검토하여, DONE 선언의 정당성을 검증한다.

---

## 1. CRG 에이전트 역할 정의

| Agent | 역할명 | 책임 | 입력 | 출력 |
|:-----:|--------|------|------|------|
| **X** | Code Verifier | 실제 코드 변경이 SPEC.md 서브태스크 명세와 일치하는지 검증 | SPEC.md, 코드베이스 (변경 파일), sessions.jsonl | 명세-구현 갭 목록 |
| **Y** | Spec Integrity Critic | Oracle 유효성, 증거 품질, SPEC-TEST 정합성을 비판적으로 검토 | SPEC.md, sessions.jsonl, knowledge.jsonl, intermediates | Oracle 등급 재판정 + 문서 정합성 이슈 |
| **Z** | Regression & Risk Assessor | 변경의 회귀 영향과 미검증 영역을 분석. **Invariant 보존 검증 절차** (`regression-test-strategy.md` §3 참조): ① SPEC.md Scope Boundary IN(Invariant) 항목 추출 → ② 코드 변경 diff와 대조 → ③ 위반 시 severity = blocker. 미검증 영역 분석은 `commands/exec.md` §8-1a 가이드라인 참조 | SPEC.md Scope (Invariant 항목 포함), 코드 변경 diff, 관련 모듈 | 회귀 리스크 판정 + Invariant 보존 검증 + 미검증 영역 목록 |

### 역할 간 관계

```
X(Code Verifier)  ──┐
                     ├──→  종합 판정: PASS / NEEDS_FIX
Y(Integrity Critic) ──┤
                     │
Z(Risk Assessor)   ──┘
```

- 3개 에이전트는 **병렬 실행** 가능 (상호 의존 없음)
- 각 에이전트는 독립적으로 이슈를 발견하고 severity를 판정
- 최종 PASS/NEEDS_FIX는 **이슈 severity 기반** (§4 참조)

---

## 2. CRG 실행 흐름

```
1. 트리거: 모든 서브태스크 [x] + Acceptance Gate PASS
2. [병렬] X + Y + Z 실행 → 각각 이슈 목록 출력
3. 이슈 종합 → severity 분류 (blocker / warning / note)
4. 판정:
   ├─ blocker 0건 + warning만 → Conditional PASS (사용자 확인)
   ├─ blocker 0건 + note만 → PASS
   ├─ blocker 1건+ → NEEDS_FIX
   ├─ CRG-N blocker ≈ CRG-N-2 blocker → AskUserQuestion: Oscillation 경고 (§5-1)
   └─ 사이클 3회째 NEEDS_FIX → NEEDS_REVIEW (§5 참조)
5. PASS → DONE 전환
6. NEEDS_FIX → 수정 사이클 (§3 참조)
```

---

## 3. 수정 사이클 (Fix Cycle)

CRG가 NEEDS_FIX를 판정하면 다음 사이클을 실행한다:

```
NEEDS_FIX 판정
  → CRG 비평 내용을 crg-{N}.md로 저장
  → sessions.jsonl에 CRG 세션 레코드 append (outcome: "crg_fail")
  → 사용자 확인 (AskUserQuestion): "CRG 권고에 따라 스펙 수정 + 재실행을 진행할까요?"
    ├─ YES → /spec-agent update {spec_name} --project {project_name} --repo {repo_slug} (CRG 권고 기반) → /spec-agent exec {spec_name} --project {project_name} --repo {repo_slug} 재실행 → CRG 재실행
    └─ NO  → 사용자 수동 진행 또는 현 상태 유지
```

### 사이클 제한

| 항목 | 값 |
|------|-----|
| 최대 CRG 실행 횟수 | **3회** |
| 최대 수정+재실행 사이클 | **2회** (1차 CRG 실패 → 수정 → 2차 CRG 실패 → 수정 → 3차 CRG) |
| 3차 CRG도 NEEDS_FIX | 상태를 `NEEDS_REVIEW`로 전환 (§5) |

### 수정 범위 제한

각 수정 사이클에서 변경 가능한 범위는 CRG가 발견한 이슈 항목으로 한정한다. CRG 비평과 무관한 실행 스펙 확장은 별도 실행 스펙으로 분리한다.

---

## 4. 판정 기준

### 4-1. 이슈 severity 정의

| Severity | 정의 | CRG 판정 영향 |
|----------|------|--------------|
| **blocker** | Acceptance Gate Predicate 자체가 미충족, 또는 Oracle이 Invalid | **NEEDS_FIX** 즉시 발행 |
| **warning** | 실행 결과가 설계 의도와 부분 괴리. 기능은 정상 | **Conditional PASS** — 사용자 확인 후 진행 |
| **note** | 개선 권고, 향후 스펙에서 반영 가능 | **PASS** — knowledge.jsonl 기록 권고 |

### 4-2. blocker 예시

- 코드 변경이 SPEC.md 서브태스크 Action과 불일치
- Oracle이 "Strong"으로 기술되었으나 실제 검증이 불가능
- stale 산출물(intermediates)이 검증 근거로 사용됨
- 알려진 regression 발생

### 4-3. Conditional PASS 처리

warning만 존재하는 경우, 각 warning에 의사결정 보조 정보(원인/수정/이점)를 제시한 후 선택지를 제공한다:

1. "인지하고 진행" → PASS로 기록 (warning 내용은 sessions.jsonl `notes`에 포함)
2. "warning 항목만 수정" → warning 목록을 워커에게 전달 → 수정 → CRG 재실행 (CRG 사이클 카운트에 포함, 스펙 재설계/재실행 불필요)
3. "수정 필요 (전체)" → NEEDS_FIX와 동일한 수정 사이클 진행 (스펙 재설계 + 재실행)

---

## 5. NEEDS_REVIEW 상태

### 5-1. 진입 조건

다음 중 하나를 충족하면 NEEDS_REVIEW로 전환한다:

1. **CRG 3회 실패** (기존): 3회 CRG 실행 모두에서 blocker가 해소되지 않은 경우
2. **CRG Oscillation** (신규): CRG 라운드 간 oscillation pattern이 감지된 경우 — CRG-N의 blocker가 CRG-N-2의 blocker와 동일 대상 + 동일 유형으로 재발. AskUserQuestion으로 사용자에게 경고: "[1] 남은 CRG 사이클 진행 [2] NEEDS_REVIEW 전환". 사용자 [2] 선택 시에만 전환
3. **exec_loop Persistent Stagnation** (신규): 2개 이상의 서브태스크에서 stagnation이 연속 감지되고, 각각 5회 재시도를 소진한 경우

### 5-2. 상태 전이

```
IN_PROGRESS → [CRG 3회 실패] → NEEDS_REVIEW
NEEDS_REVIEW → [관리자 검토 후 재설계] → IN_PROGRESS
NEEDS_REVIEW → [관리자 포기 결정] → ABANDONED
```

> NEEDS_REVIEW에서 DONE으로 직접 전환은 불가. 반드시 IN_PROGRESS를 경유한다.

**순환 가드**: 동일 스펙에서 NEEDS_REVIEW **통산 2회** 도달 시, IN_PROGRESS 전환을 차단한다. sessions.jsonl에서 `crg_needs_review` outcome 레코드 수로 카운트. 2회 도달 시 선택지: "[1] 실행 스펙 재설계 (/spec-agent create) [2] ABANDONED".

### 5-3. 허용/차단 행동

| 행동 | 허용 | 근거 |
|------|:----:|------|
| sessions.jsonl / SPEC.md 읽기 | O | 원인 분석용 |
| 에이전트 자율 코드 수정 | **X** | 3회 자동 시도 소진. 무한 루프 방지 |
| CRG 재실행 | **X** | 사이클 한도 소진 |
| 관리자에게 HITL 에스컬레이션 | **필수** | 유일한 탈출구 |
| knowledge.jsonl에 실패 원인 기록 | 권고 | 향후 유사 실패 방지 |

### 5-4. 에스컬레이션 포맷

```markdown
## CRG 판정 실패 에스컬레이션

**실행 스펙**: {spec-name}
**CRG 실행**: 3/3 NEEDS_FIX
**미해결 blocker**:
- [blocker] {내용} — CRG-1부터 지속 / CRG-{N}에서 신규 발생
- ...

**시도한 수정**:
- CRG-1 → {수정 내용 요약}
- CRG-2 → {수정 내용 요약}

**Pathological Pattern** (해당 시):
- 감지 유형: {stagnation / oscillation / persistent_stagnation}
- 재발 대상: {반복된 blocker 대상 요약}
- 시도된 전략 수: {N}회

**관리자 선택지**:
1. SPEC.md Contract/Scope 재설계 후 IN_PROGRESS 복귀
2. 현재 범위 축소 후 재승인
3. ABANDONED 결정
```

### 5-5. IN_PROGRESS 복귀 시

- SPEC.md YAML frontmatter의 `version` 필드를 올린다 (예: v1.1 → v2.0)
- CRG 사이클 카운터가 리셋된다 (신규 CRG 3회 허용)
- sessions.jsonl에 복귀 레코드를 append한다 (outcome: `aborted`, notes: "NEEDS_REVIEW → IN_PROGRESS 복귀, 관리자 결정")

---

## 6. CRG 비평 저장

### 6-1. 파일 구조

CRG 실행 시 비평 내용을 스펙 디렉토리에 저장한다:

```
${REPOS_ROOT}/{repo_slug}/projects/{project_name}/specs/{spec_name}/
├── PROBLEM.md
├── SPEC.md
├── sessions.jsonl
├── crg-1.md          # 1차 CRG 비평 (발생 시 생성)
├── crg-2.md          # 2차 CRG 비평 (발생 시 생성)
└── crg-3.md          # 3차 CRG 비평 (발생 시 생성)
```

> CRG가 발생하지 않는 스펙에는 `crg-*.md` 파일이 없다.

### 6-2. crg-{N}.md 템플릿

```markdown
# CRG-{N} — {spec-name}

> **실행 시각**: {ts} | **결과**: PASS / NEEDS_FIX | **CRG 사이클**: {N}/3

## X(Code Verifier) 판정

{이슈 테이블 또는 "이슈 없음"}

## Y(Spec Integrity Critic) 판정

{이슈 테이블 또는 "이슈 없음"}

## Z(Regression & Risk Assessor) 판정

{이슈 테이블 또는 "이슈 없음"}

## 종합 이슈

| # | Severity | 출처 | 내용 | 권고 조치 |
|:-:|----------|:----:|------|----------|
| 1 | blocker  | X    | ...  | ...      |
| 2 | warning  | Y    | ...  | ...      |

## 최종 판정

{PASS / Conditional PASS / NEEDS_FIX / NEEDS_REVIEW}

## 수정 지시 (NEEDS_FIX 시)

1. {구체적 수정 사항}
2. ...
```

---

## 7. 세션 기록

### 7-1. CRG 세션 레코드

CRG 실행은 sessions.jsonl에 별도 레코드로 기록한다. `type: "crg"` 필드로 일반 실행 세션과 구분한다.

```jsonl
{"id":"S-005","ts":"...","type":"crg","round":2,"outcome":"crg_fail","subtasks":[],"knowledge_ids":[],"notes":"CRG-1 실행. crg-1.md 참조. blocker 2건: Oracle Invalid, stale intermediates.","crg_file":"crg-1.md"}
```

### 7-2. CRG 전용 outcome 값

| outcome | 의미 | 다음 행동 |
|---------|------|----------|
| `crg_pass` | CRG 통과 | DONE 전환 |
| `conditional_pass` | CRG warning만 존재, 사용자 인지 후 통과 | DONE 전환 (warning은 notes에 기록) |
| `crg_fail` | CRG 실패, 수정 필요 | 수정 사이클 진행 |
| `crg_needs_review` | 3회 CRG 모두 실패 | NEEDS_REVIEW 상태 전환 |

### 7-3. CRG 실행 횟수 집계

sessions.jsonl에서 `type: "crg"` 레코드 수로 파생한다. 별도 카운터 필드는 두지 않는다.

```bash
# CRG 실행 횟수
jq 'select(.type == "crg")' specs/{spec}/sessions.jsonl | jq -s 'length'

# CRG 실패 이력
jq 'select(.type == "crg" and .outcome == "crg_fail")' specs/{spec}/sessions.jsonl
```

### 7-4. round 번호

CRG 세션의 `round`는 기존 실행 세션과 동일한 연속 번호를 사용한다. CRG 세션이 `type: "crg"`로 명시되어 있으므로, 일반 라운드와의 구분은 `type` 필드로 수행한다.

---

## 8. CRG 적용 범위

### 8-1. 기본 규칙

Acceptance Gate가 정의된 모든 실행 스펙에 CRG를 적용한다.

### 8-2. Skip 허용 조건 (기계적 판정)

다음 **모든 조건**을 기계적으로 검증하여 CRG skip 허용 여부를 판정한다:

| 조건 | 검증 방법 | 유형 |
|------|---------|:----:|
| 서브태스크 ≤ 2개 | `grep -c '^\- \[([ x])\]' SPEC.md` | 기계적 |
| 소스 코드 파일 변경 0건 | `git diff --name-only \| grep -E '\.(py\|js\|ts\|java\|go\|rs)$'` 결과 비어있음 | 기계적 |
| 신규 소스 파일 추가 없음 | 위 결과에서 new file 없음 | 기계적 |
| 사용자 명시적 요청 | AskUserQuestion 확인 | HITL |

**금지 표현**: "단순 실행 스펙", "대략 맞다", "우려 없음" 등 주관적 기준

**skip 시 sessions.jsonl 기록**:
```jsonl
{"type":"crg","outcome":"crg_pass","notes":"CRG skip — 조건: subtask_count=2, code_files=0, new_files=0","ts":"{ISO 8601}"}
```

notes에 기계적 조건 결과를 기록하여 사후 감사 가능.

### 8-3. 소급 적용

이 규칙 제정 전에 DONE 처리된 실행 스펙에는 소급 적용하지 않는다. 해당 실행 스펙의 구현체에서 문제가 발견되면 신규 실행 스펙을 생성하여 CRG를 적용한다.

---

## 9. 구현 방식

- Claude Code의 **Agent tool**로 X, Y, Z를 병렬 실행
- X: 읽기 + 빌드/테스트 명령 실행 권한 (Bash, Grep, Read)
- Y: 읽기 전용 (Read, Grep, Glob)
- Z: 읽기 + diff 분석 (Read, Grep, Glob, Bash — git diff만 허용)
- 각 에이전트의 출력을 종합하여 메인 에이전트가 crg-{N}.md를 작성하고 sessions.jsonl에 append
