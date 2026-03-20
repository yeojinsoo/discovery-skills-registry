# 실행 프로토콜

> ⚠️ **인라인 통합 안내**: §5 Worker/Verifier 프롬프트 템플릿은 `commands/exec.md` §6d, §6e에 인라인 통합됨. 수정 시 command 파일을 직접 수정할 것.

> **관련 규약**: knowledge.jsonl 스키마/필드 정의 → `${CLAUDE_SKILL_DIR}/rules/knowledge-conventions.md`

---

## §1. 아키텍처: Orchestrator → Worker + Verifier

### 역할 분리

| 역할 | 실행 방식 | 입력 | 출력 | 도구 |
|------|----------|------|------|------|
| **오케스트레이터** (메인) | 직접 | progress.md, SPEC.md, knowledge | 워커/검증자 프롬프트, progress.md 갱신 | Read, Write, Edit, Agent |
| **워커** (구현) | Agent tool | Action + key_decisions + warnings + knowledge | 구조화된 결과 요약 (5~10줄) | Read, Write, Edit, Bash, Grep, Glob |
| **검증자** (A+B+C 통합) | Agent tool | Predicate/Oracle + worker_result + knowledge + sessions | 6상태 판정 + 이슈 + knowledge/sessions 레코드 | Bash, Read, Grep, Glob |

### 핵심 원칙

1. **메인 컨텍스트 보호**: 무거운 작업(파일 탐색, 코드 수정, 테스트 실행)은 서브에이전트에 위임
2. **구조화된 반환**: 서브에이전트는 정해진 포맷으로만 결과를 반환 — 메인 컨텍스트에 raw 데이터가 유입되지 않음
3. **자기 확인 편향 방지**: 구현(워커)과 검증(검증자)을 별도 서브에이전트로 분리하여 같은 에이전트가 자신의 코드를 검증하지 않음

### needs_decision 처리

워커가 `needs_decision: true`를 반환하면:
1. 오케스트레이터가 결정 사항을 요약
2. AskUserQuestion으로 사용자에게 판단 요청
3. 사용자 결정을 포함하여 워커 재호출
4. progress.md 핵심 결정에 사용자 결정 기록

---

## §2. Verifier 내부 3관점: Tester / Reviewer / Observer

실행 스펙의 Acceptance Gate 달성까지, 검증자 내부에서 3개 관점을 운용하여 검증 품질을 보장한다.

### Agent 역할 정의

| Agent | 역할 | 입력 | 출력 | 판단 기준 |
|:-----:|------|------|------|----------|
| **A — Tester** | Predicate + Oracle을 실행하여 6상태 판정(`references/test-verification-guide.md` §1). `knowledge.jsonl`의 기존 교훈을 참조하여 이전에 실패했던 패턴을 사전 회피. 이전 PASS Oracle 회귀 검사(`rules/regression-test-strategy.md` §2) | 실행 스펙, 코드베이스, `knowledge.jsonl`, `sessions.jsonl` | 서브태스크별 6상태 판정(PASS/FAIL/PARTIAL/NOT_COVERED/UNCERTAIN/TIMED_OUT) + 실패 로그 | Oracle 조건 충족 여부 + 이전 PASS 보존 여부 |
| **B — Reviewer** | A의 실행 결과를 비판적으로 검토. `knowledge.jsonl`의 기존 교훈을 참조하여 기존 교훈이 위반된 경우 severity를 상향 | A의 결과 리포트, `knowledge.jsonl` | 이슈 목록 (severity: blocker/warning/note) | 테스트가 Predicate의 의도를 충실히 검증했는가 |
| **C — Observer** | A+B의 논의를 종합하여 수정안 제시 + 지식 축적 판별 | A 결과 + B 이슈 목록, `knowledge.jsonl` | (1) 수정 제안서 + (2) `knowledge.jsonl` append 레코드 | 수정안이 B의 blocker를 해소하는가 |

### 오케스트레이터-워커 모드에서의 3관점 보존

오케스트레이터-워커 모드에서도 3-Agent의 핵심 관점은 보존된다:

1. **Tester 관점**: 검증자 내부 Phase 1에서 L1 회귀 검사(선행) + Predicate/Oracle 6상태 판정
2. **Reviewer 관점**: 검증자 내부 Phase 2에서 비판적 검토 + severity 분류
3. **Observer 관점**: 검증자 내부 Phase 3에서 수정안 + 수렴 분석 + knowledge/sessions 기록

### 실행 루프

```
1. A(Tester): knowledge.jsonl 참조 → Predicate/Oracle 실행
2. B(Reviewer): knowledge.jsonl 참조 → A 결과 비판적 검토
3. C(Observer): 수정안 제시 + 지식 추출 → append 레코드 출력
4. 메인 에이전트: C의 수정안 적용 + knowledge.jsonl append 수행 → 1로 복귀
5. B의 blocker 0건 → Acceptance Gate 통과
```

### 종료 조건

- B(Reviewer)가 **blocker 0건**을 선언할 때 해당 서브태스크의 Predicate가 충족된 것으로 간주
- 모든 서브태스크가 충족되면 Acceptance Gate 공식(`ACCEPT iff S1 ∧ S2 ∧ ...`)이 true
- C(Observer)의 수정안 중 **스펙 자체 변경**이 발생하면, 변경 이력을 스펙 문서 하단에 기록
- C(Observer)는 라운드 종료 시 `sessions.jsonl` 레코드를 텍스트로 출력한다. 메인 에이전트가 append 수행. 스키마는 `${CLAUDE_SKILL_DIR}/rules/session-log-conventions.md` 참조

### 참조 프로토콜

| Agent | 시점 | 참조 방식 |
|-------|------|----------|
| **A(Tester)** | 라운드 시작 시 | 스펙 전용 + 공용 `knowledge.jsonl`을 읽고, 관련 pitfall/constraint를 테스트 전략에 반영. 이전에 false-positive가 발생한 Oracle 패턴을 인지하고 보강 |
| **B(Reviewer)** | 리뷰 시 | 스펙 전용 + 공용 `knowledge.jsonl`을 참조하여 기존 교훈 위반 여부를 검증 항목에 포함. 동일 유형 실수가 재발하면 severity 상향 (warning → blocker) |
| **C(Observer)** | 기록 전 | 기존 파일의 동일 `tags` 레코드를 확인하고 의미적 중복이 있으면 skip. 새 레코드의 `id`는 공용 파일의 마지막 K-NNN + 1 |

### 전략 전환 렌즈

WARNING/STAGNATION/DIMINISHING_RETURNS 시 C(Observer)가 적용:

- **Simplifier**: 문제를 더 작은 단위로 분해하거나 Predicate 자체를 단순화
- **Contrarian**: 현재 접근의 전제를 뒤집어 대안 경로 탐색
- **Researcher**: 코드베이스에서 유사 패턴을 검색하여 기존 해법 참조

C는 선택한 렌즈와 사유를 수정안 상단에 `[렌즈: {이름}] {선택 사유}` 형식으로 명시한다. 상세: SKILL.md §EXEC 6f

---

## §3. 수렴 검사: 4지표, STAGNATION/OSCILLATION/DIMINISHING_RETURNS

### 3-1. 수렴 감지 (Convergence Detection)

exec_loop의 라운드 2 이상에서 C(Observer)는 이전 라운드의 B 이슈 목록과 현재 라운드의 B 이슈 목록을 비교하여 수렴도를 산출한다.

**서술형 판정 기준**:

C(Observer)는 이전 라운드 B 이슈 목록과 현재 B 이슈 목록을 다음 기준으로 비교한다:

| 판정 지표 | 관찰 기준 | 의미 |
|----------|-------------------|------|
| **대상 반복** | 동일 파일/함수/모듈명이 blocker에 2회 이상 등장 | 수정이 근본 원인을 해소하지 못함 |
| **유형 고착** | blocker/warning/note 비율이 이전 라운드와 실질적으로 동일 | 이슈 구조가 변화하지 않음 |
| **설명 반복** | blocker 설명이 이전 라운드와 동일하거나 표현만 다름 | 동일 이슈가 재발 |
| **개선 정체** | blocker 건수가 이전 라운드 대비 1건 이하 감소 (3회 연속) | 수정이 표면적이며 근본 원인에 도달하지 못함 |

**판정**:

| 판정 | 조건 | 행동 |
|------|------|------|
| **STAGNATION** | 4개 지표 중 2개 이상이 2회 연속 해당 | C가 근본 원인 분석 출력 → 전략 전환 렌즈 적용 → sessions.jsonl에 `pathological: ["stagnation"]` 기록 → 렌즈 수정안 미해소 시 AskUserQuestion 에스컬레이션 |
| **DIMINISHING_RETURNS** | 개선 정체 지표가 3회 연속 해당 | C가 "현재 접근의 한계" 분석 출력 → 전략 전환 렌즈 적용 → sessions.jsonl에 `pathological: ["diminishing_returns"]` 기록 |
| **WARNING** | 4개 지표 중 2개 이상이 1회 해당 | C가 수정안에 "이전 라운드와 유사 — 근본적 접근 변경 필요" 경고 포함 + 전략 전환 렌즈 권고 |
| **정상** | 위 조건 미해당 | 정상 진행 |

> pathological 허용 값의 전체 목록 및 값 추가 절차: `${CLAUDE_SKILL_DIR}/rules/session-log-conventions.md` §2-1 참조

> **출처 주석**: 이 수렴 감지는 Ouroboros(Q00/ouroboros)의 Ontology Convergence에서 착안하였다. 원본에서는 고유사도 = 수렴 **성공**(설계가 안정화)을 의미하나, 본 적용에서는 고유사도 = 교착 **실패**(같은 blocker가 반복)로 **의미가 역전**되어 적용되었음에 유의한다.

### 3-2. Oscillation 감지

라운드 N의 blocker 목록이 라운드 N-2의 blocker 목록과 비교하여, 위 4개 지표 중 2개 이상이 해당하면 **oscillation**(fix→revert 패턴)으로 판정한다. 개선 정체 지표는 N vs N-2 비교에서는 '라운드 N과 N-2의 blocker 건수 차이가 1건 이하'로 적용한다. C(Observer)는 재발 blocker에 대해 이전과 다른 수정 전략을 제안하거나, 서브태스크 Predicate 자체의 재설계를 권고한다

---

## §4. knowledge 기록 기준: 2/3 threshold + pathological 트리거

### 4-1. knowledge.jsonl (교훈 기록)

Verifier 출력의 ## Lessons 섹션에 기재. 없으면 "없음" 명시.

2/3 기준은 "기록 판단의 가이드"로 유지하되, 최종 판단은 Verifier의 출력 형식 준수에 의해 강제된다:

1. **반복성**: 동일 실수가 재발했거나, 향후 라운드에서 재발 가능성이 높은가?
2. **비자명성**: SPEC.md나 CLAUDE.md에 이미 명시되어 있지 않은 교훈인가?
3. **행동 가능성**: 읽는 Agent가 즉시 행동을 바꿀 수 있는 구체적 지침인가?

**4번째 기록 트리거 (pathological pattern)**: stagnation, diminishing_returns, 또는 oscillation이 감지되면 2/3 기준과 무관하게 knowledge 레코드를 **필수 기록**한다.
- `cat`: `pitfall` (stagnation/diminishing_returns) 또는 `pattern` (oscillation)
- `tags`에 `pathological`, `stagnation`/`diminishing_returns`/`oscillation` 포함
- `insight`에 재발 blocker의 구체적 대상과 시도된 수정안 요약을 포함

### 4-2. sessions.jsonl (세션 로그)

매 라운드 종료 시 **필수 기록**. 2/3 기록 기준 없음. C의 출력에 다음을 포함:

```jsonl
{"id":"S-{NNN}","ts":"...","round":N,"outcome":"success|partial|blocked|aborted","subtasks":[...],"knowledge_ids":[...],"notes":"..."}
```

> `slug`, `session_id`, `commit` 등 선택 필드는 메인 에이전트가 보완하여 append.

### 4-3. 소비 프로토콜 (규모별)

| 규모 | A/B 인출 방식 |
|------|-------------|
| ~50건 | 전체 읽기 (Read tool) |
| 50~200건 | `jq 'select(.scope=="cross-spec" and .cat=="pitfall")'` 등 태그/카테고리 필터링 |
| 200건+ | knowledge-index.json 인덱스 → 관련 id 추출 → 해당 레코드만 주입 |

**필수 출력**: A/B는 결과 리포트에 `## 참조한 교훈` 섹션을 포함하여 어떤 레코드를 참조했고 어떤 행동을 변경했는지 명시. 해당 없으면 "해당 없음"으로 표기.

---

## §5. Worker/Verifier 프롬프트 템플릿

> **SOT**: `commands/exec.md` §6d (Worker), §6e (Verifier). 프롬프트 전문은 SOT를 직접 참조할 것.
>
> 이 섹션에는 프롬프트 사본을 유지하지 않는다. SOT와의 비동기 위험을 원천 차단하기 위함.

### 워커 개요

- **역할**: 서브태스크 Action 구현
- **입력**: Action, progress.md (핵심 결정 + 활성 경고), Invariant 항목, knowledge
- **출력**: 구조화된 결과 요약 (상태/변경 파일/주요 변경/결정 사항/경고/needs_decision)
- **전문**: → `commands/exec.md` §6d

### 검증자 개요

- **역할**: 워커 결과를 Tester→Reviewer→Observer 3관점 순차 검증
- **입력**: Predicate/Oracle 테이블 (Prerequisites 열 포함), SPEC-to-TEST 보강 경로, 워커 결과, knowledge, 이전 blocker
- **출력**: PASS/FAIL + 이슈 테이블 + False Positive 검사 + 수렴 분석 + knowledge/sessions 레코드
- **전문**: → `commands/exec.md` §6e

---

## §6. progress.md 조율

### progress.md 연동

오케스트레이터는 매 서브태스크에서:
1. `context_refresh`: progress.md 읽기 → 현재 상태 복원
2. 워커 프롬프트에 핵심 결정 + 활성 경고 주입
3. `update_progress`: 검증 통과 후 progress.md 전체 갱신

이를 통해 컨텍스트 압축이 발동해도 progress.md에서 정확한 상태를 복원할 수 있다.

### 실행 흐름

```
오케스트레이터(메인):
  context_refresh → pick_subtask → inject_knowledge
  → Agent(worker): {Action + key_decisions + warnings + knowledge}
      ← 워커 결과 요약 (5~10줄)
  → [needs_decision?] → AskUserQuestion → Agent(worker) 재호출
  → Agent(verifier): {Predicate/Oracle + worker_result + knowledge}
      ← 검증 결과 요약 (PASS/FAIL + 이슈 + knowledge/sessions)
  → [blocker?] → fix_and_retry
  → [x] 전환 → write_checkpoint + update_progress
```
