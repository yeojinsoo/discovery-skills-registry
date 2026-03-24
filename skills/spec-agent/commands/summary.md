## §SUMMARY — PR 본문 생성/조회/수정

현재 브랜치의 변경사항과 실행 스펙 컨텍스트를 종합하여 PR 본문을 생성한다.
버전 파일(`v1.md`, `v2.md`, ...)로 누적 저장되며, 조회 및 수정을 지원한다.

### 스텝별 참조

| 스텝 | 참조 | 설명 |
|------|------|------|
| §2-1 detect_deploy_apps | `.github/pull_request_template.md` (레포 내) | 배포 앱 체크리스트 파싱 |
| §3-2 reconcile_spec_with_diff | 이 파일 내 인라인 (§3-2) | SPEC.md-diff 정합성 검증 |
| §3-3 structural_analysis | `references/analysis-profiles.md` summary 프로파일 | 공유 분석 방법론 (SOT 유지) |
| §4-8 compose_body | 이 파일 내 인라인 (§4-8) | PR 본문 템플릿 + 작성 지침 |
| §8-1 post_compose_verify | 이 파일 내 인라인 (§8-1) | 생성 본문 사실 정합성 기계적 검증 |
| §9 save_and_output | `rules/history-file-conventions.md` | history.jsonl 스키마 (공유 SOT) |

### 서브커맨드 라우팅

§SUMMARY 진입 시, summary_args의 첫 토큰으로 분기:
- `create`, `current`, `update` → 해당 서브커맨드
- 그 외 텍스트 또는 빈 문자열 → bare case

**주의**: `create`, `current`, `update`는 예약어로 spec_name이 될 수 없다.

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
parse_args → resolve_specs
  → [create] collect_diff → detect_deploy_apps → load_specs → load_validation
    → reconcile_spec_with_diff → structural_analysis
    → derive_conditions → group_changes → detect_decisions
    → verify_conditions → compose_body → post_compose_verify → save_and_output
  → [current] load_latest → output
  → [update] load_version → apply_changes → save_and_output
```

---

### §SUMMARY.CREATE — PR 본문 생성

#### 1. parse_args

- `--spec {name}` 플래그 또는 `스펙\s+(\S+)` 패턴 → spec_name (선택적)
- `#\d+` 패턴 → 참조 PR 번호 (선택적)
- ticket_id: 다음 순서로 추출 (범용 이슈 트래커 패턴 `[A-Z]+-\d+`):
  1. project_name에서 패턴 매칭
  2. 없으면 `git branch --show-current`에서 패턴 매칭
  3. 매칭 실패 시 ticket_id = null (ticket 없는 프로젝트 지원)
- 나머지 자유 텍스트 → 추가 맥락 (diff·실행 스펙과 충돌 시 추가 맥락 우선)

#### 2. collect_diff — git 컨텍스트 수집

--repo 모드(Step 0에서 --repo 플래그로 진입한 경우) 시 아래 git 명령을 모두 skip하고, AskUserQuestion으로 변경 파일 목록과 base 브랜치를 수집한다.

```bash
git branch --show-current

# base 브랜치 동적 감지
BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
# 실패 시 AskUserQuestion으로 base 브랜치 확인

git log ${BASE_BRANCH}..HEAD --oneline
git diff ${BASE_BRANCH}..HEAD --stat
git status --short | grep '^??'
```

**커밋별 diff 수집 전략**:

`git log ${BASE_BRANCH}..HEAD --oneline`의 각 커밋에 대해:

1. **변경 파일 목록 수집**: `git show {commit_sha} --stat`으로 커밋별 변경 파일과 변경량(줄 수) 수집
2. **핵심 파일 선별**: 전체 커밋의 변경량을 합산하여, 변경 줄 수 기준 상위 파일을 핵심 파일로 선별 (기준: 변경 줄 수 내림차순 정렬 후 누적 변경량이 전체의 80%에 도달할 때까지, 또는 최대 15개 파일)
3. **핵심 파일 전문 읽기**: 핵심 파일에 대해서만 `git diff ${BASE_BRANCH}..HEAD -- {file_path}`로 전체 diff를 수집
4. **나머지 파일**: `--stat` 정보만 유지 (전문 diff를 수집하지 않음)

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

#### 3. load_specs — 실행 스펙 컨텍스트 로드

- spec_name 지정 → `${SPECS_DIR}/{spec_name}/SPEC.md` + `PROBLEM.md` 로드
- spec_name 미지정 → `${SPECS_DIR}/*/SPEC.md` 전체 스캔, YAML frontmatter `status` 필드가 `DONE`인 실행 스펙 모두 로드
- 실행 스펙 0건 → diff에서 변경 의도 역추론 (실행 스펙 없이도 동작)

**GATE-NON-DONE**: `DONE`이 아닌 스펙(`IN_PROGRESS`, `PLANNED` 등)이 1개 이상 존재하면 AskUserQuestion:
```
⚠️ DONE이 아닌 스펙이 있습니다:
  - {spec_name_1} (IN_PROGRESS)
  - {spec_name_2} (PLANNED)

이 스펙의 R-ID가 PR에 누락될 수 있습니다.
```
- [1] 무시하고 DONE 스펙만으로 summary 진행
- [2] 먼저 실행 → Skill 도구로 `/spec-agent exec {spec_name} --project {project_name} --repo {repo_slug}` 호출 후 summary 재실행
- [3] summary 중단

[2] 선택 시 대상 스펙이 2개 이상이면 AskUserQuestion으로 실행할 스펙 선택.

각 실행 스펙에서 추출:
- PROBLEM.md §1(증상) → PR "문제" 섹션 입력
- Contract Statement → 해결 조건 도출 입력
- 설계 원칙 (P-N) → 해결 조건 테이블 행(S-N)으로 변환
- Subtask Action/Oracle → 테스트 표 시나리오
- Scope Boundary의 IN (Invariant) 항목 → PR "보존 영역" 섹션 (Invariant 항목이 있을 때만)

**SPEC.md 역할 한정**: SPEC.md는 "왜 변경했는가(intent)"의 보조 컨텍스트로만 사용한다. "무엇이 변경되었는가(what)"는 반드시 diff에서 도출한다. SPEC.md는 rebase·merge로 인해 실제 diff와 불일치할 수 있다 (예: rebase 전 설계에서 추출 대상이던 함수가 base 변경으로 이미 이동된 경우).

#### 3-1. load_validation — 검증 리포트 로드 (선택적)

`${VALIDATION_DIR}/` 내 최신 검증 리포트 탐색:
```bash
ls "${VALIDATION_DIR}" 2>/dev/null | sort | tail -1
```

- 있으면: 해결 조건 달성 현황 테이블 + R-ID별 판정을 로드 → `verify_conditions` 단계의 근거로 활용
- 없으면: skip (기존 로직대로 diff에서 직접 확인)

#### 3-2. reconcile_spec_with_diff — SPEC.md-diff 정합성 검증

SPEC.md의 Subtask Action 서술과 실제 diff를 대조하여 불일치를 탐지한다. rebase·merge로 base 브랜치가 변경되면, SPEC.md 작성 시점의 가정(어떤 파일에 어떤 코드가 있었는지)이 더 이상 유효하지 않을 수 있다.

**검증 항목**:

| # | 검증 | 방법 | 불일치 시 처리 |
|---|------|------|--------------|
| 1 | SPEC.md에서 "추출/이동/삭제"로 언급한 파일이 diff에 존재하는가 | diff `--stat` 파일 목록과 SPEC.md Subtask Action의 파일 참조를 대조 | diff에 없는 파일의 서술을 narrative에서 제외 |
| 2 | SPEC.md의 구체 수치(줄 수, 함수 개수 등)가 diff와 일치하는가 | diff의 `+`/`-` 줄 수, 실제 함수 정의 수를 확인 | diff 기준 수치로 교체 |
| 3 | SPEC.md에서 "신규 생성"으로 기술한 파일이 diff에서도 신규인가 | diff에서 해당 파일이 `new file` 또는 전체 `+`인지 확인 | base에 이미 존재하면 "변경"으로 재분류 |

**출력**: `spec_diff_conflicts` 리스트. 각 항목은 `{spec_claim, diff_fact, resolution}` 구조.
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
- **실행 스펙 연동**: 설계 원칙(P-N) → S-N 행으로 변환. Subtask Predicate에서 검증 가능한 조건 추출.
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
- narrative의 "변경" 서술(2번)에서 구체적 사실(파일 수, 함수 개수, 줄 수, 이동 방향)은 **반드시 diff에서 도출**한다. SPEC.md의 Subtask Action 서술을 그대로 옮기지 않는다.
- `reconcile_spec_with_diff`에서 `spec_diff_conflicts`가 존재하면, 해당 항목의 `diff_fact`를 narrative에 반영한다.
- "왜"는 SPEC.md/PROBLEM.md에서, "무엇"과 "얼마나"는 diff에서 가져온다.

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

diff/실행 스펙에서 **여러 대안 중 하나를 선택한 흔적**을 찾는다.

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

#### 8-1. post_compose_verify — 생성 본문 사실 정합성 검증

- **목적**: compose_body가 생성한 PR 본문의 사실 정합성을 기계적으로 검증한다. 본문에 서술된 파일명·변경 수치·함수명·흐름 설명 등이 실제 diff 데이터와 일치하는지 대조하여, 사실과 다른 서술이 최종 산출물에 포함되는 것을 방지한다.
- **입력**: compose_body의 출력 (PR 제목 + PR 본문 마크다운), collect_diff 단계에서 수집한 diff 데이터, base_branch 이름
- **출력**: 검증 결과 — 오류 목록(`[{location, claim, diff_fact, severity}]`) 또는 통과(빈 리스트)
- **위치**: compose_body 직후, save_and_output 직전

**검증 (a) — S-ID 정합성**:

생성된 PR 본문에서 S-ID(S-1, S-2, ...)가 세 곳에 등장한다. 세 곳의 S-ID 집합이 정확히 일치해야 한다.

1. **해결 조건 테이블**: `## 해결 조건` 섹션의 테이블에서 `S-\d+` 패턴으로 S-ID 집합을 추출한다
2. **테스트 커버리지 테이블**: `## 테스트` 섹션의 테이블에서 `조건` 컬럼의 `S-\d+` 값을 추출한다 (빈 셀은 직전 행의 S-ID를 이어받음)
3. **narrative 참조**: `## 구현` 섹션의 서술 텍스트에서 `S-\d+` 패턴으로 참조된 S-ID 목록을 추출한다

검증 규칙:
- 해결 조건 테이블의 S-ID 집합을 기준(SOT)으로 삼는다
- 테스트 커버리지 테이블에 해결 조건 S-ID가 하나라도 누락되면 → 오류
- narrative에서 해결 조건 테이블에 없는 S-ID를 참조하면 → 오류

오류 출력 예시:
```json
{
  "location": "## 테스트 테이블",
  "claim": "S-3이 커버리지 테이블에 존재해야 함",
  "diff_fact": "테스트 테이블에 S-3에 대한 행이 없음",
  "severity": "error"
}
```

**검증 (b) — 코드 식별자 존재 확인**:

PR 본문에서 백틱(`` ` ``)으로 감싼 코드 식별자를 추출하여 코드베이스에 실재하는지 확인한다.

1. **추출**: 본문에서 단일 백틱으로 감싼 토큰(`` `identifier` ``)을 모두 추출한다
2. **분류**: 추출된 식별자를 다음으로 분류한다
   - **파일 경로**: `/` 포함 또는 확장자(`.ts`, `.py`, `.md` 등) 포함 → 파일 경로로 판정
   - **코드 식별자**: 그 외 (함수명, 변수명, 클래스명 등)
   - **제외 대상**: 마크다운 서식 키워드(`신규`, `변경`, `x`, 표 셀 내 단순 값 등), S-ID(`S-\d+`), TC-ID(`TC-\d+`), git 명령어, PR 본문 서식 용어는 검증 대상에서 제외
3. **파일 경로 검증**: 핵심 파일(§2 collect_diff에서 선별된 파일)에 해당하면 `git show HEAD:{path}`로 존재 확인. diff에서 `new file`로 생성된 파일은 HEAD에서 확인, 삭제된 파일은 base에서 확인
4. **코드 식별자 검증**: `grep -r '{identifier}' .` 으로 코드베이스에서 존재 여부 확인 (검증 대상은 핵심 파일에 등장하는 식별자로 한정 — §2 collect_diff의 핵심 파일 선별 기준 적용: 변경 줄 수 내림차순 누적 80% 또는 최대 15개)
5. **판정**: 코드베이스에 미존재하는 식별자는 환각(hallucination) 후보로 플래그

오류 출력 예시:
```json
{
  "location": "## 구현 > 1단계 narrative",
  "claim": "`validateWorkflowName` 함수가 존재한다는 서술",
  "diff_fact": "grep -r 'validateWorkflowName' . 결과 0건",
  "severity": "error"
}
```

**검증 (c) — 기존 코드 전제 확인**:

PR 본문에서 '기존' 표현을 사용하여 base 브랜치의 코드 상태를 전제하는 서술이 있을 때, 해당 전제가 사실인지 확인한다.

1. **추출**: 본문에서 `기존`, `기존에`, `기존의`, `기존과`, `이전에`, `기존 코드` 등의 표현이 포함된 문장을 추출한다
2. **참조 식별**: 해당 문장에서 백틱으로 감싼 코드 식별자 또는 파일 경로를 추출한다
3. **base 브랜치 검증**:
   - 파일 경로인 경우: `git show origin/{base_branch}:{path}` 로 base 브랜치에 파일 존재 확인
   - 코드 식별자인 경우: `git show origin/{base_branch}:{관련_파일_경로}` 로 파일을 조회한 뒤, 해당 식별자가 파일 내에 존재하는지 확인 (관련 파일 경로는 diff의 변경 파일 목록에서 추론)
4. **판정**: base 브랜치에 미존재하면 전제 오류(premise error)로 플래그

오류 출력 예시:
```json
{
  "location": "## 구현 > 2단계 narrative",
  "claim": "'기존의 `handleDuplicate` 함수를 확장하여...'",
  "diff_fact": "git show origin/main:src/handlers/workflow.ts 에 handleDuplicate 함수 미존재",
  "severity": "error"
}
```

**검증 흐름 요약**:

```
compose_body 출력 (PR 본문 마크다운)
  │
  ├─ (a) S-ID 정합성 ──────── 해결 조건 vs 커버리지 vs narrative → S-ID 불일치 오류
  ├─ (b) 코드 식별자 존재 ──── 백틱 식별자 vs grep/git show → 환각 후보
  └─ (c) 기존 코드 전제 ───── '기존' 문장 vs git show origin/{base} → 전제 오류
  │
  ▼
  errors: [{location, claim, diff_fact, severity}, ...]
```

**오류 처리 — 자동 수정 루프**:

post_compose_verify는 오류 발견 시 compose_body를 오류 피드백과 함께 재호출하여 자동 수정을 시도한다. compose_body 내부 로직은 변경하지 않으며, 오류 목록을 추가 컨텍스트로 주입하여 동일한 compose_body를 재실행한다.

**상태 변수**:
- `verify_attempt`: 현재 검증 시도 횟수 (초기값 `1`)
- `max_verify_attempts`: 최대 검증 시도 횟수 (`2`)

**흐름**:

```
compose_body 최초 실행
  │
  ▼
post_compose_verify (verify_attempt = 1)
  │
  ├─ 오류 0건 → save_and_output으로 진행
  │
  └─ 오류 1건 이상 & verify_attempt ≤ max_verify_attempts
      │
      ├─ 1) 경고 출력:
      │     ⚠️ post_compose_verify: {N}건 오류 발견 (시도 {verify_attempt}/{max_verify_attempts})
      │     각 오류를 [{location, claim, diff_fact, severity}] 형식으로 나열
      │
      ├─ 2) compose_body 재호출 (오류 피드백 주입):
      │     compose_body(§8)를 동일 입력 + 아래 오류 피드백으로 재실행
      │     ┌──────────────────────────────────────────┐
      │     │ ## 검증 오류 피드백 (자동 수정 요청)      │
      │     │ 이전 생성 본문에서 다음 오류가 발견되었다. │
      │     │ 아래 오류를 수정하여 본문을 재생성하라.    │
      │     │                                          │
      │     │ ```json                                  │
      │     │ [{location, claim, diff_fact, severity}]  │
      │     │ ```                                      │
      │     └──────────────────────────────────────────┘
      │
      ├─ 3) verify_attempt += 1
      │
      └─ 4) post_compose_verify 재검증 (루프 상단으로)
           │
           ├─ 오류 0건 → save_and_output으로 진행
           │
           └─ 오류 1건 이상 & verify_attempt > max_verify_attempts
               │
               ▼
               경고 출력 후 현재 본문으로 진행:
               ⚠️ post_compose_verify: {N}건 오류 잔존 — 최대 재시도 횟수({max_verify_attempts}회) 초과, 현재 본문으로 진행
               → save_and_output으로 진행 (오류 목록 전달)
```

**최종 오류 처리 (save_and_output 전달)**:
- 자동 수정 루프 종료 후 잔존하는 오류가 있는 경우, 오류 목록을 save_and_output에 전달하여 PR 본문 말미에 경고 블록으로 삽입:
  ```markdown
  > ⚠️ **post_compose_verify 검증 경고** ({N}건)
  > - [{location}] {claim} — 실제: {diff_fact}
  ```
- `severity: "error"` 항목과 `severity: "warning"` 항목 모두 동일 경고 블록에 포함하되, warning은 참고 수준으로 표기
- 자동 수정 루프에서 오류가 모두 해소된 경우 → 빈 리스트, save_and_output에 경고 블록 미삽입

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
2. 없으면: "PR 본문이 아직 없습니다. `/spec-agent summary create`로 생성해주세요."
3. 있으면:
   - 첫 줄 → `**PR 제목**: ...`
   - 이후 → 4-backtick `````markdown````` 펜스로 출력

---

### §SUMMARY.UPDATE — 기존 PR 본문 수정

기존 버전을 기준으로 수정 지시사항만 적용한다. diff·실행 스펙을 재분석하지 않는다.

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
/spec-agent summary create
/spec-agent summary create --spec ag-1614-workflow-unique
/spec-agent summary create #1392 의사결정 섹션에 DB fallback 근거 보완

# 조회
/spec-agent summary
/spec-agent summary current

# 수정
/spec-agent summary update 테스트 표에서 TC-3 시나리오 구체화
/spec-agent summary update v2 리뷰어 피드백 반영: 구현 단위 1 간결하게
```
