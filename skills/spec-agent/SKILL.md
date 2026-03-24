---
name: spec-agent
description: "실행 스펙 생성/실행/수정 통합 스킬. 실행 스펙 데이터를 외부 경로($HOME/.discovery-skills/spec-agent/repos/)에 저장하여 레포 무관 재사용 가능. /spec-agent context [name] — create 전 문제 구체화, /spec-agent create [name] — 신규 생성, /spec-agent exec [name] — 실행, /spec-agent update [name] — 수정, /spec-agent validate [name] — DoD 검증, /spec-agent summary — PR 본문 생성/조회/수정, /spec-agent status — 프로젝트 현황 조회, /spec-agent distill — 프로젝트 교훈 정리, /spec-agent close — 프로젝트 종료. 이 스킬은 사용자가 명시적으로 \"/spec-agent\" 커맨드를 입력했을 때만 실행한다. 자연어 요청으로는 트리거하지 않는다."
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - AskUserQuestion
  - Skill
  - Agent
---

# spec-agent

컨텍스트(context), 생성(create), 실행(exec), 수정(update), 검증(validate), PR 본문(summary), 현황 조회(status), 교훈 정리(distill)를 단일 스킬로 처리한다.
모든 실행 스펙 데이터는 `${SPEC_DIR}`에 저장된다. 데이터는 스킬 디렉토리 외부(`$HOME/.discovery-skills/spec-agent/repos/`)에 분리 저장되어 레포에 파일을 남기지 않으며 어느 프로젝트에서도 동일하게 동작한다.

## 경로 상수 (스킬 전체에서 사용)

```
SKILL_DIR  = ${CLAUDE_SKILL_DIR}
RULES_DIR  = ${CLAUDE_SKILL_DIR}/rules
REFS_DIR   = ${CLAUDE_SKILL_DIR}/references
DATA_ROOT  = $HOME/.discovery-skills/spec-agent
REPOS_ROOT = ${DATA_ROOT}/repos
LA_MODULES = ${CLAUDE_SKILL_DIR}/../logical-analysis/modules

# Step 0에서 결정되는 런타임 경로
REPO_DIR  = ${REPOS_ROOT}/{repo_slug}/
PROJ_DIR  = ${REPO_DIR}/projects/{project_name}/
CTX_DIR   = ${PROJ_DIR}/context/
HIST_FILE     = ${PROJ_DIR}/history.jsonl
KNOW_FILE     = ${PROJ_DIR}/knowledge.jsonl
PROJECT_FILE  = ${PROJ_DIR}/project.json
SPECS_DIR     = ${PROJ_DIR}/specs/
SPEC_DIR      = ${PROJ_DIR}/specs/{spec_name}/
PROGRESS_FILE = ${SPEC_DIR}/progress.md
SUMMARY_DIR    = ${PROJ_DIR}/summary/
VALIDATION_DIR = ${PROJ_DIR}/validations/
```

## 파일 매니페스트

> **분류 기준**: `rules/` = 서브커맨드 실행 시 반드시 읽는 규범 (스키마, 프로토콜, 품질 기준). `references/` = 워크플로우 진행 중 필요 시 참조하는 보조 문서 (템플릿, 체크리스트, 가이드).
> 이 테이블에 없는 파일이 rules/ 또는 references/ 에 존재하면 고아(orphan)이므로 삭제 또는 이동한다.

### rules/ (9개)

| 규칙 | 파일 | 참조 서브커맨드 |
|------|------|:-------------:|
| 실행 스펙 품질 특성 | `rules/spec-quality-characteristics.md` | §CREATE, §UPDATE |
| 실행 프로토콜 | `rules/execution-protocol.md` | §EXEC |
| sessions.jsonl 규약 | `rules/session-log-conventions.md` | §EXEC |
| knowledge.jsonl 규약 + 인출 전략 | `rules/knowledge-conventions.md` | §CREATE, §EXEC, §VALIDATE, §DISTILL |
| SPEC-TEST 정합성 | `rules/spec-test-alignment.md` | §CREATE, §UPDATE |
| CRG 프로토콜 | `rules/critical-review-gate.md` | §EXEC |
| history.jsonl 규약 | `rules/history-file-conventions.md` | §CREATE, §EXEC, §UPDATE, §SUMMARY, §VALIDATE, §DISTILL |
| 분석 방법론 | `rules/analytical-method.md` | §VALIDATE, §SUMMARY |
| 3계층 회귀 테스트 전략 | `rules/regression-test-strategy.md` | §EXEC, §VALIDATE |

### references/ (9개)

| 참조 문서 | 파일 | 참조 서브커맨드 |
|----------|------|:-------------:|
| PR 본문 템플릿 | `references/summary-template.md` | §SUMMARY |
| 실행 스펙 템플릿 | `references/spec-templates.md` | §CREATE |
| 안티패턴 | `references/antipatterns.md` | §CREATE |
| Cascade 규칙 | `references/cascade-map.md` | §UPDATE |
| 버전 관리 | `references/version-rules.md` | §UPDATE |
| 정합성 검증 | `references/consistency-guide.md` | §UPDATE |
| 분석 프로파일 | `references/analysis-profiles.md` | §VALIDATE, §SUMMARY |
| Phase C 에이전트 프로토콜 | `references/phase-c-agent-protocol.md` | §VALIDATE |
| 테스트 검증 방법론 | `references/test-verification-guide.md` | §EXEC |

---

## Step 0 — parse_args + 환경 감지

모든 서브커맨드 실행 전 공통 수행. 단, `status`는 0-1 직후 조기 라우팅된다 (§0-1-1 참조).

### 0-1. 전체 args 선파싱 (플래그 먼저 추출)

$ARGUMENTS 전체에서 플래그를 우선 추출:
  --project {name}  → project_name 강제 지정
  --repo {slug}     → repo_slug 강제 지정 (git 자동감지 대신 사용; git 저장소 밖에서도 실행 가능)
  --seed {path}     → seed_file 지정 (§CONTEXT: 외부 파일 경로를 읽어 ctx에 매핑, §CREATE: context/ 내 파일명을 seed로 선택)
  --spec {name}     → spec_name 강제 지정 (§SUMMARY, §VALIDATE에서 유효)

플래그 제거 후 나머지에서 subcmd + 두 번째 인자 추출. 두 번째 인자의 해석은 subcmd에 따라 다름:
- `create`, `exec`, `update`, `validate` → spec_name으로 파싱
- `context` → slug (컨텍스트 파일 이름)로 파싱. spec_name 아님.
- `summary` → 나머지 args 전체를 §SUMMARY 내부 라우터에 그대로 전달 (spec_name 파싱 안 함)
- `status` → 추가 인자 없음 (잉여 인자는 무시). --repo, --project 플래그만 유효. §0-1-1에서 조기 라우팅.
- `distill` → 추가 인자 없음 (프로젝트 레벨 커맨드). --repo, --project 플래그만 유효.

### 0-1-1. status 조기 라우팅

subcmd가 `status`이면 **즉시** Read(`${CLAUDE_SKILL_DIR}/commands/status.md`)를 로드하고 §STATUS의 Step 0-S를 실행한다.
Step 0-2 이후(repo_slug 감지, project_name 감지, 디렉토리 초기화)는 실행하지 않는다.

> 이유: status는 읽기 전용 커맨드로, 감지 실패 시 안내 메시지 후 종료해야 하며 디렉토리를 생성해서는 안 된다.
> 공유 Step 0-2~0-4는 에러 종료 및 mkdir을 수행하므로 status의 시맨틱스와 충돌한다.

예시:
  "create my-spec --project AG-1614"
    → subcmd=create, spec_name=my-spec, project_name=AG-1614
  "--project AG-1614 exec my-spec"
    → subcmd=exec, spec_name=my-spec, project_name=AG-1614
  "create my-spec --repo commerceos-backend-v2 --project AG-1614"
    → subcmd=create, spec_name=my-spec, repo_slug=commerceos-backend-v2, project_name=AG-1614
  "context initial-design"
    → subcmd=context, slug=initial-design
  "summary create --spec ag-1614"
    → subcmd=summary, summary_args="create --spec ag-1614"

### 0-2. repo_slug 결정

**--repo 플래그 지정 시** → 해당 값을 repo_slug로 사용. 현재 디렉토리가 git 저장소인지 여부와 무관하게 git 명령을 일절 실행하지 않는다.

**--repo 미지정 시** → git에서 자동 감지:
  Bash 실행: git rev-parse --git-dir 2>/dev/null
  비-zero exit → 에러 종료:
    "현재 디렉토리가 git 저장소가 아닙니다.
    --repo {slug} --project {name} 플래그를 함께 지정하거나, git 저장소 디렉토리에서 실행하세요.
    예: /spec-agent context my-context --repo commerceos-backend-v2 --project AG-1614"
  Bash 실행: git rev-parse --show-toplevel 2>/dev/null | xargs basename
  → repo_slug 결정 (예: "commerceos-backend-v2")

### 0-3. project_name 결정

**--project 플래그 지정** → 해당 값 사용.

**--project 미지정 + --repo 지정** (git 미조회 모드) → 에러 종료:
  "--repo 사용 시 git 브랜치에서 project를 추론할 수 없습니다.
  --project {name}을 함께 지정하세요.
  예: /spec-agent create my-spec --repo commerceos-backend-v2 --project AG-1614"

**--project 미지정 + --repo 미지정** (git 환경) → 브랜치 확인:
  Bash 실행: git branch --show-current
  → "feature/{name}" 형식이면 project_name = {name}
  → feature 브랜치 아님 → 에러 종료:
    "현재 브랜치({branch})는 feature 브랜치가 아닙니다.
    --project {name} 플래그로 project를 지정하세요.
    예: /spec-agent create my-spec --project AG-1614"

### 0-3-1. repo_slug 존재 검증 (--repo 수동 지정 시에만)

--repo로 지정된 경우에만 `${REPOS_ROOT}/{repo_slug}/` 디렉토리 존재 여부 확인.
(git 자동 감지 시에는 새 레포에서 처음 실행하는 것이 정상이므로 이 단계를 건너뜀.)

**존재** → 정상 진행.

**미존재** →
  기존 레포 목록 조회: `ls -1 "${REPOS_ROOT}/" 2>/dev/null`
  AskUserQuestion으로 확인:
    "'{repo_slug}' 레포가 존재하지 않습니다.
    기존 레포 목록: {기존 레포 목록}

    1) 오타라면 올바른 이름으로 다시 시도하세요.
    2) 새로 생성하려면 'yes'를 입력하세요."
  → "yes" 응답 → `mkdir -p "${REPOS_ROOT}/{repo_slug}"`, 정상 진행.
  → 그 외 → 종료.

### 0-4. 경로 결정 및 디렉토리 초기화

REPO_DIR = ${REPOS_ROOT}/{repo_slug}/
PROJ_DIR = ${REPO_DIR}/projects/{project_name}/

없으면 생성 (DATA_ROOT는 외부 경로이므로 반드시 보장):
  mkdir -p "${REPOS_ROOT}"
  mkdir -p "${PROJ_DIR}/specs"
  mkdir -p "${CTX_DIR}"
  (knowledge.jsonl, history.jsonl은 각 커맨드에서 필요 시 생성)

**project.json 초기화** (미존재 시):
  `${PROJECT_FILE}` 경로에 파일이 없으면 기본값으로 생성:
  ```json
  {"status":"DEFINING","statusUpdatedAt":"{ISO 8601}"}
  ```
  이미 존재하면 읽기만 수행하여 현재 상태를 확인한다.

**project.json 스키마**:
  ```json
  {
    "status": "DEFINING | RUNNING | DONE | VERIFIED | CLOSED",
    "statusUpdatedAt": "{ISO 8601}"
  }
  ```
  - `status`: 프로젝트의 현재 마일스톤 (5값 enum)
  - `statusUpdatedAt`: 마지막 상태 전환 시각

**상태 전환 규칙** (각 커맨드에서 실행):
  | 커맨드 | 전환 | 조건 |
  |--------|------|------|
  | context, create | → DEFINING | CLOSED가 아닐 때 |
  | exec (시작) | → RUNNING | — |
  | exec (모든 스펙 DONE) | → DONE | — |
  | validate (ALL_ACHIEVED) | → VERIFIED | — |
  | validate (PARTIAL/NOT_ACHIEVED) | → DEFINING | 루프백 |
  | close | → CLOSED | 어떤 상태에서든 |

### 0-4-1. §SPEC 마커 출력

모든 변경 커맨드(exec, create, update, validate, summary, distill, close)는 실행 시작 시 아래 형식의 마커 텍스트를 **1행** 출력한다.
`context`와 `status`는 읽기 전용이므로 마커를 출력하지 않는다.

```
§SPEC {repo_slug}/{project_name}/{spec_name} {subcmd} {timestamp+09:00}
```

검증 정규식: `^§SPEC \S+/\S+/\S+ \S+ \d{4}-\d{2}-\d{2}T.+$`

예시:
```
§SPEC commerceos-backend-v2/AG-1614/migration-safety-fixes exec 2026-03-23T15:30:00+09:00
§SPEC skills/spec-agent/project-status create 2026-03-23T16:00:00+09:00
```

**Step 0에서 즉시 출력하는 커맨드** — spec_name이 §0-1 args 파싱에서 이미 확정된 경우:
- `exec`, `update`, `validate` → 여기서 바로 마커 출력 후 §0-5로 진행.

**각 command 파일에서 출력하는 커맨드** — Step 0 시점에서 spec_name이 미확정인 경우:
- `create` → spec_name이 args에 없을 수 있음. §CREATE 내 spec_name 확정 직후 출력.
- `summary` → --spec 지정 시 해당 spec_name 사용. --spec 미지정(프로젝트 레벨) 시 `_project` 사용. 커맨드 파일에서 출력.
- `distill` → 프로젝트 레벨 커맨드. spec_name 자리에 `_project`를 사용하여 커맨드 파일 시작 시 출력.
- `close` → 프로젝트 레벨 커맨드. spec_name 자리에 `_project`를 사용하여 커맨드 파일 시작 시 출력.

**timestamp 생성**: Bash 실행 `date +%Y-%m-%dT%H:%M:%S%z` — 시스템 TZ에 따라 오프셋이 자동 결정된다. KST 환경이면 `+0900`이 출력되며, 그대로 사용한다.

### 0-5. 서브커맨드 라우팅

서브커맨드에 따라 해당 command 파일을 Read로 로드한 후 실행:

- context  → Read(`${CLAUDE_SKILL_DIR}/commands/context.md`) → 로드된 §CONTEXT 워크플로우 실행
- create   → Read(`${CLAUDE_SKILL_DIR}/commands/create.md`) → 로드된 §CREATE 워크플로우 실행
- exec     → Read(`${CLAUDE_SKILL_DIR}/commands/exec.md`) → 로드된 §EXEC 워크플로우 실행
- update   → Read(`${CLAUDE_SKILL_DIR}/commands/update.md`) → 로드된 §UPDATE 워크플로우 실행
- validate → Read(`${CLAUDE_SKILL_DIR}/commands/validate.md`) → 로드된 §VALIDATE 워크플로우 실행
- summary  → Read(`${CLAUDE_SKILL_DIR}/commands/summary.md`) → 로드된 §SUMMARY 워크플로우 실행 (나머지 args를 §SUMMARY 내부 라우터에 전달; spec_name 파싱 안 함)
- status   → Read(`${CLAUDE_SKILL_DIR}/commands/status.md`) → 로드된 §STATUS 워크플로우 실행 (읽기 전용; Step 0-S 자체 환경 감지 사용)
- distill  → Read(`${CLAUDE_SKILL_DIR}/commands/distill.md`) → 로드된 §DISTILL 워크플로우 실행
- close    → Read(`${CLAUDE_SKILL_DIR}/commands/close.md`) → 로드된 §CLOSE 워크플로우 실행
- 기타     → "사용법: /spec-agent [context|create|exec|update|validate|summary|status|distill|close] [spec-name]
              옵션: --project {name}, --repo {slug}, --seed {context-file}, --spec {name}"

---

## 공유 유틸리티

§CONTEXT의 로직. 해당 서브커맨드의 command 파일에서 이 섹션을 참조한다.

### resolve_next_filename

```bash
ls "${CTX_DIR}" 2>/dev/null | grep -E '^[0-9]{3}-' | sort | tail -1
```

- 결과 없음 → `001-{slug}.md`
- 마지막이 `NNN-*.md` → `{NNN+1}-{slug}.md`

### check_spec_state / load_existing_context

| 상태 | 처리 |
|------|------|
| CTX_DIR 없음 | 신규 프로젝트 context로 진행 |
| CTX_DIR 있음 | 기존 파일 목록 표시 후 추가로 진행 |
| SPECS_DIR에 실행 스펙 존재 | 기존 실행 스펙 목록 안내 + context 추가 진행 |

기존 컨텍스트 파일이 있으면 파일명과 첫 줄(§1 요약)을 나열하여 현재까지 쌓인 내용을 보여준다.

### write_context_file

```bash
mkdir -p "${CTX_DIR}"
```

생성 파일:
```
${CTX_DIR}/
└── {NNN}-{slug}.md    ← consolidate 결과 (Write 도구로 저장)
```

`${KNOW_FILE}` 없으면 빈 파일 생성.

`${HIST_FILE}`에 append:
```jsonl
{"type":"context_added","file":"{NNN}-{slug}.md","ts":"{ISO 8601}"}
```

> `SPEC.md` / `PROBLEM.md`는 이 단계에서 생성하지 않는다.
> 기존 context 파일은 **절대 수정하지 않는다**. 새 파일만 추가.

### report + GATE-TO-CREATE_OR_UPDATE

완료 안내:
```
✅ 저장됨: {CTX_DIR}/{NNN}-{slug}.md
📋 전체 컨텍스트 파일: {총 N개}
```

AskUserQuestion:
  "다음 단계를 선택하세요:"
  [1] 새 실행 스펙 생성 (Recommended)
  [2] 기존 실행 스펙에 반영
  [3] 나중에 실행

[1] 선택 시:
  AskUserQuestion: "생성할 실행 스펙 이름을 입력하세요 (kebab-case):"
  → spec_name 입력받은 후 → Skill 도구로 /spec-agent create {spec_name} --project {project_name} --repo {repo_slug} 호출

[2] 선택 시:
  실행 스펙 목록 표시 후 AskUserQuestion으로 spec_name 선택
  → Skill 도구로 /spec-agent update {spec_name} --project {project_name} --repo {repo_slug} 호출
