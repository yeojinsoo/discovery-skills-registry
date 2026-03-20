# 지식 규약 (`knowledge.jsonl`)

> **적용 대상**: `${REPOS_ROOT}/{repo_slug}/projects/{project_name}/knowledge.jsonl` (프로젝트 단위)

---

## §1. 형식

**형식**: 1행 = 1 JSON 객체. `id`는 `K-{NNN}`(프로젝트) / `RK-{NNN}`(레포) 형식으로 **전역 고유**.

> JSONL 채택 근거: append(`>>`) 한 줄로 기록 가능, `jq`로 즉시 필터링, git merge conflict 최소화, 라인 단위 컨텍스트 주입 용이.

**변경 정책**: 라운드 실행 중에는 append-only (수정·삭제 금지). 스펙 간 정리 시점에는 PM 승인 하에 아카이브 허용.

---

## §2. 스키마

```jsonl
{"id":"K-014","spec":"project-file-lineage","round":1,"ts":"2026-02-25T12:00:00+09:00","cat":"constraint","confidence":"observed","insight":"REST API 핸들러에서 Claude SDK를 동기 호출하면 ALB 타임아웃 위험. HTTP 202 즉시 반환 + Worker 비동기 처리 패턴 사용","src":"blocker-B1","tags":["async","worker","alb-timeout"]}
```

### 필수 필드 (에이전트가 직접 입력)

| 필드 | 타입 | 설명 |
|------|------|------|
| `insight` | string | 교훈 본문. 1~2문장, 행동 가능한 형태 |
| `cat` | enum | `pitfall` \| `pattern` \| `constraint` \| `decision` |
| `tags` | string[] | 검색용 키워드. 2~4개 |
| `src` | string | 출처 (서브태스크 ID, 코드 경로 등) |

### 자동 생성 필드 (에이전트가 채울 필요 없음)

| 필드 | 타입 | 생성 방식 |
|------|------|----------|
| `id` | `K-{NNN}` (프로젝트) / `RK-{NNN}` (레포) | 마지막 ID + 1. 레포 레벨은 distill 승격 시 `RK-` 접두사 부여 |
| `ts` | ISO 8601 | 현재 시각 |
| `spec` | string | 현재 스펙명 |
| `round` | number | 현재 라운드 |

### grooming 시에만 추가

| 필드 | 타입 | 설명 |
|------|------|------|
| `confidence` | enum | `observed` → `validated` → `canonical` |
| `supersedes` | string[] | 대체하는 이전 교훈 |
| `refs` | string[] | 관련 교훈 |
| `expires_when` | string | 폐기 조건 |

---

## §3. `cat` 정의

| cat | 행동 신호 | `insight` 작성 패턴 |
|-----|----------|---------------------|
| `pitfall` | "이것을 **하지 마라**" | "X하면 Y 실패한다. Z해야 한다" |
| `pattern` | "이것을 **반복하라**" | "X 상황에서 Y가 효과적이다" |
| `constraint` | "이것은 **바뀌지 않는다**" | "X 환경에서 Y 제약이 존재한다" |
| `decision` | "이렇게 **결정했다**" | "X vs Y에서 X 선택. 근거: Z" |

---

## §4. `confidence` 생명주기

```
[생성]              [검증]              [승격 또는 폐기]
observed ─────────► validated ────────► canonical (CLAUDE.md 반영)
(C가 1회 관찰)     (다른 스펙에서       └► archived (expires_when 충족)
                    재발견/적용 성공)
```

| 값 | 의미 | 전환 조건 |
|----|------|----------|
| `observed` | 1회 관찰 | 기본값 |
| `validated` | 재현 확인 | 다른 스펙에서 재발견 또는 적용 성공 |
| `canonical` | 원칙 승격 | CLAUDE.md 또는 `${CLAUDE_SKILL_DIR}/rules/spec-quality-characteristics.md`에 반영 완료 |

---

## §5. 지식 범위 (2계층 스코프)

knowledge.jsonl은 **레포 레벨**과 **프로젝트 레벨** 두 스코프를 지원한다.

| 스코프 | 경로 | 역할 |
|--------|------|------|
| 레포 레벨 | `${REPO_DIR}/knowledge.jsonl` | 레포 전체 광역 교훈. 동일 레포 내 모든 프로젝트가 공유 |
| 프로젝트 레벨 | `${PROJ_DIR}/knowledge.jsonl` | 특정 프로젝트 내 교훈. 스펙 간 공유 |

### 주입 우선순위

inject_knowledge 시 두 파일을 순서대로 로드:
1. `${REPO_DIR}/knowledge.jsonl` (레포 레벨) — 있으면 로드
2. `${PROJ_DIR}/knowledge.jsonl` (프로젝트 레벨) — 있으면 로드

동일 주제 충돌 시 프로젝트 레벨이 우선한다 (구체적 스코프 > 일반 스코프).

### 기록 위치

- 스펙 실행 중 발생한 교훈 → **프로젝트 레벨** `${PROJ_DIR}/knowledge.jsonl`에 append
- 레포 레벨은 수동 승격 또는 명시적 지정 시에만 기록

inject_knowledge:
```bash
# 레포 레벨
cat "${REPO_DIR}/knowledge.jsonl" 2>/dev/null | jq 'select(.tags | contains(["..."]))'
# 프로젝트 레벨
cat "${PROJ_DIR}/knowledge.jsonl" | jq 'select(.tags | contains(["..."]))'
```

---

## §6. 규모별 인출 전략

### 경로 변수 (이 섹션 전체에서 사용)

```
DATA_ROOT  = $HOME/.discovery-skills/spec-agent
REPOS_ROOT = ${DATA_ROOT}/repos
REPO_DIR   = ${REPOS_ROOT}/{repo_slug}
PROJ_DIR   = ${REPO_DIR}/projects/{project_name}
KNOW_FILE  = ${PROJ_DIR}/knowledge.jsonl
```

> knowledge.jsonl은 **project 단위 단일 파일**이다. per-spec knowledge.jsonl은 존재하지 않는다.

### 규모 판별

```bash
wc -l "${KNOW_FILE}"
```

### ~50건: 전체 읽기

```
Read tool로 ${KNOW_FILE} 전체 로드
```

### 50~200건: 태그/카테고리 필터링

```bash
# pitfall만
jq 'select(.cat=="pitfall")' "${KNOW_FILE}"

# 특정 태그
jq 'select(.tags | index("{tag}"))' "${KNOW_FILE}"

# pitfall + constraint (워커/검증자가 주로 참조)
jq 'select(.cat=="pitfall" or .cat=="constraint")' "${KNOW_FILE}"
```

### 200건+: 인덱스 기반

```bash
# 1. 인덱스 존재 확인
ls "${PROJ_DIR}/knowledge-index.json"

# 2. 인덱스에서 관련 id 추출
jq '.[] | select(.tags | index("{tag}")) | .id' "${PROJ_DIR}/knowledge-index.json"

# 3. 해당 레코드만 주입
jq 'select(.id=="K-001" or .id=="K-005")' "${KNOW_FILE}"
```

### 워커/검증자 Agent 주입 방식

knowledge 레코드를 Agent 프롬프트의 `{knowledge_entries}` 자리에 삽입:

```
knowledge.jsonl 교훈:
- K-001 [pitfall]: REST API 핸들러에서 Claude SDK를 동기 호출하면 ALB 타임아웃
- K-005 [constraint]: E2E 테스트에서 SW가 page.route()를 가로채는 문제
```

### Pathological Pattern 우선 인출

stagnation/oscillation 감지 시, 워커/검증자 Agent에 pathological 태그 레코드를 우선 주입한다:

```bash
# pathological 태그 레코드 인출
jq 'select(.tags | index("pathological"))' "${KNOW_FILE}"

# stagnation 전용
jq 'select(.tags | index("stagnation"))' "${KNOW_FILE}"

# oscillation 전용
jq 'select(.tags | index("oscillation"))' "${KNOW_FILE}"
```

이 레코드들은 일반 knowledge 인출과 별도로, **라운드 시작 시 최우선 주입**한다. 동일 패턴의 과거 해결 전략을 참조하여 반복 실패를 방지한다.

### 세션 기록 후 ID 관리

새 K-NNN 레코드 작성 시:
1. 마지막 id 확인: `tail -1 "${KNOW_FILE}" | jq -r .id`
2. +1 증가하여 새 id 부여

---

## §7. 승격 기준

3개 중 2개 충족 시 `confidence: "canonical"`로 승격 후 CLAUDE.md/spec-quality-characteristics.md에 반영:

1. **반복 인용**: 3개 이상의 서로 다른 스펙에서 동일 교훈이 참조됨
2. **보편성**: 특정 기술 스택이 아닌, 스펙 구조/프로토콜 자체에 대한 교훈
3. **실증**: 해당 교훈을 참조하여 blocker를 사전 회피한 증거 1건 이상

---

## §8. 정리 (Grooming)

**트리거**: 사용자가 `/spec-agent distill` 커맨드를 명시적으로 호출한다.

> 자세한 절차는 `commands/distill.md` 참조.

### 동작 요약

1. **분류** — 프로젝트 레벨 `${PROJ_DIR}/knowledge.jsonl`의 각 레코드를 3가지로 분류한다.

| 분류 | 기준 | 처리 |
|------|------|------|
| **canonical** | confidence `validated` 이상 + 범용적 | `RK-{NNN}` ID를 부여하여 `${REPO_DIR}/knowledge.jsonl`로 승격 (confidence는 `validated` 유지 — §7 기준 충족 후 `canonical` 전환) |
| **active** | 프로젝트 한정 유효 | `${PROJ_DIR}/knowledge.jsonl`에 그대로 유지 |
| **expired** | 무효화 또는 일회성 | `${PROJ_DIR}/knowledge.archive.jsonl`로 아카이브 |

2. **확인** — AskUserQuestion으로 분류 결과를 표시하고, 사용자 확인(또는 수정) 후 실행한다.
3. **실행** — canonical은 레포 레벨로 승격, expired는 프로젝트 아카이브로 이동, active는 현 위치 유지.
