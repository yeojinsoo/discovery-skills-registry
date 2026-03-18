## §SUMMARY — PR 본문 생성/조회/수정

현재 브랜치의 변경사항과 프로젝트 플랜 컨텍스트를 종합하여 PR 본문을 생성한다.
버전 파일(`v1.md`, `v2.md`, ...)로 누적 저장되며, 조회 및 수정을 지원한다.

### 스텝별 참조

| 스텝 | 참조 | 설명 |
|------|------|------|
| §2-1 detect_deploy_apps | `.github/pull_request_template.md` (레포 내) | 배포 앱 체크리스트 파싱 |
| §3-2 reconcile_plan_with_diff | 이 파일 내 인라인 (§3-2) | PLAN.md-diff 정합성 검증 |
| §3-3 structural_analysis | `references/analysis-profiles.md` summary 프로파일 | 공유 분석 방법론 (SOT 유지) |
| §4-8 compose_body | 이 파일 내 인라인 (§4-8) | PR 본문 템플릿 + 작성 지침 |
| §9 save_and_output | `rules/history-file-conventions.md` | history.jsonl 스키마 (공유 SOT) |

### 서브커맨드 라우팅

§SUMMARY 진입 시, summary_args의 첫 토큰으로 분기:
- `create`, `current`, `update` → 해당 서브커맨드
- 그 외 텍스트 또는 빈 문자열 → bare case

**주의**: `create`, `current`, `update`는 예약어로 plan_name이 될 수 없다.

```
summary create  [args] → §SUMMARY.CREATE
summary current [args] → §SUMMARY.CURRENT
summary update  [args] → §SUMMARY.UPDATE
summary (bare)         → ls ${SUMMARY_DIR}/v*.md 2>/dev/null 결과로 판단:
                          파일 있으면 → §SUMMARY.CURRENT
                          파일 없으면 → §SUMMARY.CREATE
```

### StateGraph

```
parse_args → resolve_plans
  → [create] collect_diff → detect_deploy_apps → load_plans → load_validation
    → reconcile_plan_with_diff → structural_analysis
    → derive_conditions → group_changes → detect_decisions
    → verify_conditions → compose_body → save_and_output
  → [current] load_latest → output
  → [update] load_version → apply_changes → save_and_output
```

---

### §SUMMARY.CREATE — PR 본문 생성

#### 1. parse_args

- `--plan {name}` 플래그 또는 `플랜\s+(\S+)` 패턴 → plan_name (선택적)
- `#\d+` 패턴 → 참조 PR 번호 (선택적)
- ticket_id: 다음 순서로 추출 (범용 이슈 트래커 패턴 `[A-Z]+-\d+`):
  1. project_name에서 패턴 매칭
  2. 없으면 `git branch --show-current`에서 패턴 매칭
  3. 매칭 실패 시 ticket_id = null (ticket 없는 프로젝트 지원)
- 나머지 자유 텍스트 → 추가 맥락 (diff·프로젝트 플랜과 충돌 시 추가 맥락 우선)

#### 2. collect_diff — git 컨텍스트 수집

--repo 모드(Step 0에서 --repo 플래그로 진입한 경우) 시 아래 git 명령을 모두 skip하고, AskUserQuestion으로 변경 파일 목록과 base 브랜치를 수집한다.

```bash
git branch --show-current

# base 브랜치 동적 감지
BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
# 실패 시 AskUserQuestion으로 base 브랜치 확인

git log ${BASE_BRANCH}..HEAD --oneline
git diff ${BASE_BRANCH}..HEAD --stat
git diff ${BASE_BRANCH}..HEAD | head -300
git status --short | grep '^??'
```

참조 PR 있으면: `gh pr view {PR_NUMBER} --json title,body`

#### 2-1. detect_deploy_apps — 배포 앱 목록 탐지

`.github/pull_request_template.md`에서 배포 애플리케이션 체크리스트를 파싱한다.

```bash
cat .github/pull_request_template.md 2>/dev/null
```

**파싱 규칙**:
1. "배포" 또는 "deploy" 키워드가 포함된 헤딩(`##`, `###`)을 찾는다
2. 해당 헤딩 하위의 `- [ ] {app_name}` 패턴을 추출한다
3. 다음 헤딩이 나오거나 파일이 끝나면 파싱 종료

**결과**:
- 앱 목록 추출 성공 → `deploy_apps: ["admin-api", "app-api", ...]`로 저장
- 파일 없음 또는 배포 섹션 없음 → `deploy_apps: null` (compose_body에서 섹션 생략)

**체크 판정** (deploy_apps가 있을 때):
- `git diff --stat`의 파일 경로에서 앱 디렉토리를 추출하여 변경된 앱 식별
- `core/`, `utils/`, `middleware/` 등 공유 모듈 변경 시 → 의존하는 앱 전체를 체크 대상으로 표시하되, `(공유 모듈 변경)` 주석 추가
- 매칭된 앱은 `[x]`, 나머지는 `[ ]`

#### 3. load_plans — 프로젝트 플랜 컨텍스트 로드

- plan_name 지정 → `${PLANS_DIR}/{plan_name}/PLAN.md` + `PROBLEM.md` 로드
- plan_name 미지정 → `${PLANS_DIR}/*/PLAN.md` 전체 스캔, `DONE` 상태인 프로젝트 플랜 모두 로드
- 프로젝트 플랜 0건 → diff에서 변경 의도 역추론 (프로젝트 플랜 없이도 동작)

**GATE-NON-DONE**: `DONE`이 아닌 플랜(`IN_PROGRESS`, `PLANNED` 등)이 1개 이상 존재하면 AskUserQuestion:
```
⚠️ DONE이 아닌 플랜이 있습니다:
  - {plan_name_1} (IN_PROGRESS)
  - {plan_name_2} (PLANNED)

이 플랜의 R-ID가 PR에 누락될 수 있습니다.
```
- [1] 무시하고 DONE 플랜만으로 summary 진행
- [2] 먼저 실행 → Skill 도구로 `/project-planner exec {plan_name} --project {project_name} --repo {repo_slug}` 호출 후 summary 재실행
- [3] summary 중단

[2] 선택 시 대상 플랜이 2개 이상이면 AskUserQuestion으로 실행할 플랜 선택.

각 프로젝트 플랜에서 추출:
- PROBLEM.md §1(증상) → PR "문제" 섹션 입력
- Contract Statement → 해결 조건 도출 입력
- 설계 원칙 (P-N) → 해결 조건 테이블 행(S-N)으로 변환
- Subtask Action/Oracle → 테스트 표 시나리오
- Scope Boundary의 IN (Invariant) 항목 → PR "보존 영역" 섹션 (Invariant 항목이 있을 때만)

**PLAN.md 역할 한정**: PLAN.md는 "왜 변경했는가(intent)"의 보조 컨텍스트로만 사용한다. "무엇이 변경되었는가(what)"는 반드시 diff에서 도출한다. PLAN.md는 rebase·merge로 인해 실제 diff와 불일치할 수 있다 (예: rebase 전 설계에서 추출 대상이던 함수가 base 변경으로 이미 이동된 경우).

#### 3-1. load_validation — 검증 리포트 로드 (선택적)

`${VALIDATION_DIR}/` 내 최신 검증 리포트 탐색:
```bash
ls "${VALIDATION_DIR}" 2>/dev/null | sort | tail -1
```

- 있으면: 해결 조건 달성 현황 테이블 + R-ID별 판정을 로드 → `verify_conditions` 단계의 근거로 활용
- 없으면: skip (기존 로직대로 diff에서 직접 확인)

#### 3-2. reconcile_plan_with_diff — PLAN.md-diff 정합성 검증

PLAN.md의 Subtask Action 서술과 실제 diff를 대조하여 불일치를 탐지한다. rebase·merge로 base 브랜치가 변경되면, PLAN.md 작성 시점의 가정(어떤 파일에 어떤 코드가 있었는지)이 더 이상 유효하지 않을 수 있다.

**검증 항목**:

| # | 검증 | 방법 | 불일치 시 처리 |
|---|------|------|--------------|
| 1 | PLAN.md에서 "추출/이동/삭제"로 언급한 파일이 diff에 존재하는가 | diff `--stat` 파일 목록과 PLAN.md Subtask Action의 파일 참조를 대조 | diff에 없는 파일의 서술을 narrative에서 제외 |
| 2 | PLAN.md의 구체 수치(줄 수, 함수 개수 등)가 diff와 일치하는가 | diff의 `+`/`-` 줄 수, 실제 함수 정의 수를 확인 | diff 기준 수치로 교체 |
| 3 | PLAN.md에서 "신규 생성"으로 기술한 파일이 diff에서도 신규인가 | diff에서 해당 파일이 `new file` 또는 전체 `+`인지 확인 | base에 이미 존재하면 "변경"으로 재분류 |

**출력**: `plan_diff_conflicts` 리스트. 각 항목은 `{plan_claim, diff_fact, resolution}` 구조.
- 불일치 0건 → 빈 리스트, 이후 단계에 영향 없음
- 불일치 1건+ → `group_changes`, `compose_body`에서 diff 기준으로 서술

**이 단계는 경고를 출력하지 않는다.** 조용히 diff를 SOT로 채택하고 이후 단계에 전달한다.

#### 3-3. structural_analysis — 경량 구조 분석

> 이 스텝에서 `analysis-profiles.md` summary 프로파일 적용. (분석 방법론 원칙은 프로파일에 내장됨)

**Phase T (diff 기반 구조 파악)**:
1. diff에서 변경된 파일의 주요 타입/클래스 추출 (Primary Node)
2. 1-hop 의존 관계 매핑 (Secondary Node)
3. participant 결정 — Primary Node를 서비스/레이어 수준으로 추상화, max 5개
   (클래스명이 아닌 서비스명/레이어명. 예: WorkflowNameValidator → app-api)
4. after_flow 파악 — 변경 후 요청이 participant를 거치는 순서 + 주고받는 데이터/이벤트명
5. before_flow 파악:
   a) diff 삭제 라인(-)에서 호출 관계 추출
   b) PROBLEM.md §1(증상)을 보완 참조 — 기존 흐름의 결과(누락·오류)를 before rect Note에 반영
   c) flow_change 판정:
      - 삭제 라인 없음 + 신규 파일만 추가 → flow_change = false (순수 신규 기능)
      - 삭제 라인 있음 + before_flow 재구성 불충분 → flow_change = true,
        before rect의 Note에 "기존 흐름 정보 부족 — 증상만 표시" 기재 후 §1 증상 1줄 표시
      - 그 외 → flow_change = true

**Phase A (구축 패스만 — 비판 생략)**:
각 논리 단위(변경 그룹)에 대해:
1. 이 변경이 왜 필요한가 (PROBLEM.md §1/§3 연계)
2. 어떤 구조로 구현되었는가 (컴포넌트 관계)
3. 핵심 의사결정 지점 (대안이 있었을 곳)

**출력**: 별도 파일 저장 없음. 내부 컨텍스트로 다음 단계에 전달:
- `participants`, `before_flow`, `after_flow`, `flow_change` → `compose_body`의 sequenceDiagram
- `logical_units` → `group_changes` 단계의 그룹 분류 근거
- `decisions` → `detect_decisions` 단계의 의사결정 식별 근거
- `condition_evidence` → `verify_conditions` 단계의 구현 확인 근거

validation report가 있으면 `condition_evidence`에 검증 판정 결과를 통합.

#### 4. derive_conditions — 해결 조건 도출

**작성 지침** (인라인):
- 각 행: "어떤 입력에 어떤 출력/상태가 보장되는가"를 선언적으로 서술
- 구현 방법이 아닌 **관찰 가능한 결과**로 서술
- 2~6개 행이 적절
- **프로젝트 플랜 연동**: 설계 원칙(P-N) → S-N 행으로 변환. Subtask Predicate에서 검증 가능한 조건 추출.
- 추가 맥락이 있으면 반영

#### 5. group_changes — 구현 논리 단위 도출

**작성 지침** (인라인):

diff 파일을 가장 자연스러운 축으로 그룹화한다:

| 분류 기준 | 적합한 경우 |
|-----------|------------|
| **기능/검증 대상별** | 독립적인 제약/기능을 동시에 구현할 때 |
| **코드 흐름/진입점별** | 동일 로직이 여러 API 경로에 적용될 때 |
| **서비스/레이어별** | 멀티 서비스 변경 시 |
| **인과 단계별** | 변경이 순차적 의존관계를 가질 때 (A를 만들어야 B가 가능) |

그룹 수는 2~4개.

**논리 단위 서술 규칙**: 각 논리 단위의 주 서술(narrative)은 인과 관계 흐름을 따른다:
1. **문제/한계**: 이 단위가 해결하는 구체적 문제 또는 이전 상태의 한계 (1문장)
2. **변경**: 무엇을 어떻게 바꿨는가 (1-2문장)
3. **결과**: 이 변경으로 무엇이 가능해졌는가 또는 어떤 보장이 생겼는가 (1문장)

**diff 우선 규칙** (사실 서술):
- narrative의 "변경" 서술(2번)에서 구체적 사실(파일 수, 함수 개수, 줄 수, 이동 방향)은 **반드시 diff에서 도출**한다. PLAN.md의 Subtask Action 서술을 그대로 옮기지 않는다.
- `reconcile_plan_with_diff`에서 `plan_diff_conflicts`가 존재하면, 해당 항목의 `diff_fact`를 narrative에 반영한다.
- "왜"는 PLAN.md/PROBLEM.md에서, "무엇"과 "얼마나"는 diff에서 가져온다.

서술 금지 사항:
- 파일명을 narrative에 인라인하지 않는다 (파일은 `<details>` 안에서만)
- "~를 추가했다", "~를 변경했다" 나열 금지 — 반드시 "왜"가 선행해야 한다
- 첫 단위의 narrative는 독립적으로 읽히고, 후속 단위는 이전 단위의 결과를 받아서 시작한다

**Before/After 블록** (선택적): 핵심 변환을 직관적으로 보여줄 때 narrative 직후, `<details>` 직전에 포함. 논리 단위당 최대 1개.

**변경 파일 상세** (`<details>` 블록):
- 이유/맥락을 먼저 서술, 파일명은 하위 bullet로 분리
- 파일명에는 **모듈 상대경로**를 포함한다. 레포 루트가 아닌 의미 있는 모듈/패키지 루트 기준으로 경로를 축약하여, 리뷰어가 파일의 위치와 레이어를 즉시 파악할 수 있도록 한다. 모듈/패키지 최상위부터 2–3 depth가 적절.
- 파일명이 있는 줄에는 `` `{모듈 상대경로/파일명}` (신규) `` 또는 `` `{모듈 상대경로/파일명}` (변경) `` 만 올 수 있다. 인라인 부연 금지.
- 세부 변경사항은 파일명 다음 줄에 들여쓰기 하위 bullet으로만 표현
- 소스 파일은 확장자 생략, 설정/마이그레이션/스크립트 파일은 확장자 포함

- `structural_analysis`의 `logical_units`가 있으면 이를 그룹 분류의 1차 근거로 사용

#### 6. detect_decisions — 의사결정 식별

**작성 지침** (인라인):

diff/프로젝트 플랜에서 **여러 대안 중 하나를 선택한 흔적**을 찾는다.

식별 신호:
- fallback 핸들러, 이중 방어 구조
- 신규 컴포넌트 생성 (왜 기존에 inline으로 넣지 않았는가?)
- 기존 동작 변경 (왜 하위 호환이 아닌가?)
- 코드 주석에 "~대신 ~로 처리" 등 대안 언급

포함 기준: 두 가지 이상의 구현 방식이 실제로 고려될 수 있고, 선택 이유가 코드를 읽어도 바로 드러나지 않는 경우.

**의사결정이 0건이면 `## 의사결정` 섹션 전체를 생략한다.**

- `structural_analysis`의 `decisions`가 있으면 이를 의사결정 식별의 1차 근거로 사용

#### 7. verify_conditions — 해결 조건 구현 여부 확인

각 해결 조건이 diff에서 구현되었는지 확인. ❌ 누락 시 PR 본문에 명시.
- `structural_analysis`의 `condition_evidence`가 있으면 이를 구현 확인의 1차 근거로 사용
- validation report가 로드된 경우, R-ID 판정 결과를 해결 조건 확인에 통합

#### 8. compose_body — PR 본문 작성

**PR 제목 형식**:
```
{ticket prefix}{type}: {간결한 한국어 설명}
```
- type: `feat` / `fix` / `refactor` / `docs` / `test` / `chore`
- ticket prefix: 이슈 트래커 티켓이 있으면 `[TICKET-ID] ` 형식 (예: `[PROJ-123] feat: ...`). 없으면 생략.

**PR 본문 템플릿** (인라인):

```markdown
## 연관된 이슈 (optional — 이슈 트래커 티켓이 있을 때만 포함)

> {TICKET-ID}

## 문제

> {사용자가 할 수 없는 것 또는 시스템의 잘못된 동작 — "~해도 막히지 않는다", "~할 수 없다", "~오류가 난다" 형식으로 1문장}

## 해결 조건

| # | 해결 조건 |
|---|----------|
| S-1 | {해결 후 보장되는 상태를 선언적으로 서술} |
| S-2 | {해결 후 보장되는 상태를 선언적으로 서술} |

## 구현

{sequenceDiagram — 아래 Mermaid 규칙 참조}

### 1단계 — {논리 단위 이름}

{인과 서술: 문제/한계 → 변경 → 결과. 2-4문장. 파일명 인라인 금지.}

{선택적 Before/After}

<details>
<summary>변경 파일 ({파일 수}개)</summary>

- {이유/맥락 먼저, 결정을 자연스러운 문장으로}
  - `{모듈 상대경로/파일명}` (신규)
- {이유/맥락 + 결정}
  - `{모듈 상대경로/파일명}` (변경)

</details>

## 의사결정

- **{결정 제목}**
  - {검토한 대안 명시 + 왜 이 방향을 선택했는지 1-2문장}

## 테스트

| 조건 | 내용 | TC | 전제 | 유형 | 시나리오 | 확인 항목 | 결과 |
|------|------|----|------|------|---------|---------|------|
| S-1 | {조건 요약 — 첫 행에만} | TC-1 | {선행 상태} | {유형} | {시나리오} | {확인 항목} | ✅ |
| | | TC-2 | | {유형} | {시나리오} | {확인 항목} | ✅ |

## 보존 영역 (optional — Invariant 항목이 있을 때만 포함)

> 이 PR은 다음 영역을 의도적으로 변경하지 않았습니다.

- {Invariant 항목}: {보존 근거 1줄}

## 리뷰 요구사항 (optional)

## 배포할 애플리케이션 (optional — deploy_apps가 있을 때만 포함)

{deploy_apps 목록에서 diff 매칭 결과로 체크리스트 생성}
```

**Mermaid 다이어그램 규칙**:
- 항상 `sequenceDiagram` 사용
- 작성 후 반드시 mmdc로 syntax 검증: `echo "{diagram}" > /tmp/pr-diagram.mmd && mmdc -i /tmp/pr-diagram.mmd -o /tmp/pr-diagram.svg`
- participant: max 5개. 클래스명이 아닌 서비스/레이어명 사용. `participant A as {이름}` alias 형식.
- rect 블록: `flow_change = true`일 때만 사용. 기존: `rect rgb(220,220,240)` + 변경: `rect rgb(220,240,220)`. 두 블록 합쳐 10줄 이하.
- 화살표: `->>`(요청), `-->>`(응답), `-)`(비동기 전송), `--)`(비동기 응답). 레이블에 실제 데이터/이벤트명 명시.

sequenceDiagram 작성:
  - flow_change = true  → before rect + after rect 두 블록
  - flow_change = false → after_flow만 단일 블록 (rect 없음)

**서술 원칙**:
- 주 서술은 인과 관계 흐름: 문제/한계 → 변경 → 결과 (2-4문장)
- 파일명을 narrative 안에 인라인하지 않는다
- "~를 추가했다", "~를 수정했다" 나열 금지 — 반드시 "왜"가 선행
- Before/After 블록: 핵심 변환을 보여줄 때만, 논리 단위당 최대 1개

**테스트 표 규칙**:
- TC 값: `TC-1`, `TC-2`, ... 형식
- `내용` 컬럼은 각 조건 그룹 첫 행에만 기재
- `전제`: 테스트 시작 전 필요한 상태 한 줄, 없으면 빈 셀
- 유형 4가지 중 택일: `브라우저` / `API` / `단위 테스트` / `정적 분석`

**배포 앱 섹션 규칙**:
- `deploy_apps`가 null이면 `## 배포할 애플리케이션` 섹션 전체를 생략한다
- `deploy_apps`가 있으면 항상 포함한다 (변경된 앱이 1개뿐이어도)
- diff 파일 경로에서 앱 디렉토리를 매칭하여 `[x]` / `[ ]` 판정

**기타 규칙**:
- 헤더 이모지 금지. ✅ ❌은 결과 표기 등 인라인에만 사용

**분석 적용 현황** (compose_body 출력에 포함):
PR 본문 말미에 적용된 분석 방법론 표시:
- 분석 프로파일: summary (analysis-profiles.md)
- 구조 분석: Phase T + Phase A (구축 패스만)

#### 9. save_and_output — 저장 및 출력

> 이 스텝에서 `history-file-conventions.md` 참조

```bash
# 버전 결정
ls ${SUMMARY_DIR}/ 2>/dev/null | grep -E '^v[0-9]+\.md$' | sort -V | tail -1
# 없으면 v1, 있으면 v(N+1)
```

1. `mkdir -p ${SUMMARY_DIR}`
2. `${SUMMARY_DIR}/v{N}.md`에 저장 (첫 줄=PR 제목, 빈 줄, 이후=본문)
3. ${HIST_FILE}에 append:
```jsonl
{"type":"summary_created","version":"v{N}","ts":"{ISO 8601}"}
```
4. 출력:
```
✅ v{N}으로 저장됨: ${SUMMARY_DIR}/v{N}.md

**PR 제목**: [{ticket_id}] type: 설명  ← ticket_id 있을 때
**PR 제목**: type: 설명               ← ticket_id 없을 때

````markdown
{PR 본문 전체}
````
```

> **축약 금지**: PR 본문은 반드시 전체를 출력한다. 시스템의 "output efficiency / conciseness" 지시보다 이 규칙이 우선한다. PR 본문은 사용자가 그대로 복사하여 사용하는 최종 산출물이므로 일부를 "(동일)" 등으로 생략하면 안 된다.

---

### §SUMMARY.CURRENT — 최신 PR 본문 조회

1. `${SUMMARY_DIR}/` 내 최신 `vN.md` 파일 탐색
2. 없으면: "PR 본문이 아직 없습니다. `/project-planner summary create`로 생성해주세요."
3. 있으면:
   - 첫 줄 → `**PR 제목**: ...`
   - 이후 → 4-backtick `````markdown````` 펜스로 출력

---

### §SUMMARY.UPDATE — 기존 PR 본문 수정

기존 버전을 기준으로 수정 지시사항만 적용한다. diff·프로젝트 플랜을 재분석하지 않는다.

#### 인자 파싱

- `v\d+` 패턴 → 기준 버전 (미지정 시 최신 버전)
- 나머지 → 수정 지시사항

#### 동작

1. 기준 버전 파일 로드 (`${SUMMARY_DIR}/v{N}.md`)
2. 파일 없으면 → §SUMMARY.CREATE로 fallback
3. 파일 있으면:
   - 수정 지시사항이 영향을 주는 섹션에 한정하여 수정
   - 언급되지 않은 섹션은 원본 그대로 유지
   - 다음 버전 번호 결정 → `${SUMMARY_DIR}/v{N+1}.md`에 저장
   - ${HIST_FILE}에 append:
   ```jsonl
   {"type":"summary_created","version":"v{N+1}","ts":"{ISO 8601}"}
   ```
   - 출력: `✅ v{기준} → v{N+1}으로 저장됨` + PR 제목 + 4-backtick 본문

---

### 호출 예시

```
# 생성
/project-planner summary create
/project-planner summary create --plan ag-1614-workflow-unique
/project-planner summary create #1392 의사결정 섹션에 DB fallback 근거 보완

# 조회
/project-planner summary
/project-planner summary current

# 수정
/project-planner summary update 테스트 표에서 TC-3 시나리오 구체화
/project-planner summary update v2 리뷰어 피드백 반영: 구현 단위 1 간결하게
```
