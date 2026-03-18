# Version Rules — 프로젝트 플랜 변경 버전 관리

## 변경 추적 원칙

- PLAN.md / PROBLEM.md는 git 이력으로 버전 추적
- sessions.jsonl은 append-only (변경 이력 자체가 버전)
- knowledge.jsonl도 append-only (`supersedes` 필드로 정정)

## 상태별 변경 허용 범위

| 상태 | PROBLEM.md | PLAN.md 구조 | 체크리스트 | knowledge/sessions |
|------|:----------:|:-----------:|:---------:|:-----------------:|
| PLANNED | 전체 수정 가능 | 전체 수정 가능 | 수정 가능 | append |
| IN_PROGRESS | 조건부 HITL (update.md §2 참조) | cascade 규칙 적용 | `[x]` 전환만 | append |
| DONE | 수정 불가 (새 프로젝트 플랜 생성) | 수정 불가 | 수정 불가 | append |
| NEEDS_REVIEW | Contract/Scope 재설계만 허용 | AskUserQuestion 후 IN_PROGRESS 전환 | 수정 불가 | append |
| ABANDONED | 수정 불가 | 수정 불가 | 수정 불가 | append |

## IN_PROGRESS 변경 절차

1. 변경 사유 확인 (조건부 HITL):
   - Drift-triggered Entry → sessions.jsonl pathological 필드에서 자동 채택. AskUserQuestion 생략
   - CRG-triggered Entry → crg-{N}.md의 blocker/warning 항목이 변경 사유. AskUserQuestion 생략
   - 사용자 직접 호출 + 변경 내용 포함 → 입력에서 사유 추출. AskUserQuestion 생략
   - 사용자 직접 호출 + 변경 내용 미포함 → AskUserQuestion으로 변경 사유 확인
2. `cascade-map.md`에 따라 영향 범위 분석 (§0단계 의미적 동등성 판정 포함)
3. 영향받는 Gate 재검증
4. 이미 완료(`[x]`)된 서브태스크가 영향받는 경우:
   - `[x]` → `[ ]` 되돌리기
   - sessions.jsonl에 `aborted` 레코드 추가 (notes: "프로젝트 플랜 변경으로 인한 서브태스크 리셋")
5. 변경 적용 후 sessions.jsonl에 기록

## DONE 프로젝트 플랜 변경

DONE 프로젝트 플랜은 직접 수정하지 않는다:
- 새 프로젝트 플랜을 생성하고 PROBLEM.md에 이전 프로젝트 플랜 참조
- 사용자 명시적 요청 시에만 `IN_PROGRESS`로 되돌린 후 변경

## 체크리스트 되돌리기 규칙

`[x]` → `[ ]` 되돌리기는 다음 조건에서만 허용:
1. Cascade에 의해 해당 서브태스크의 Predicate가 변경된 경우
2. 사용자가 명시적으로 요청한 경우
3. 3-Agent 재실행 결과 blocker가 발견된 경우
