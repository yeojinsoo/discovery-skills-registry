# §STATUS — 프로젝트 현황 조회

> 읽기 전용 커맨드. 파일 생성·수정·history 기록 없음. 디렉토리도 생성하지 않는다.
> SKILL.md §0-1-1에 의해 공유 Step 0(0-2~0-4)을 건너뛰고 즉시 이 워크플로우로 진입한다.

---

## Step 0-S: 환경 감지 (인자 우선, git 보조)

SKILL.md Step 0-1의 플래그 파싱(`--repo`, `--project`)은 이미 완료된 상태.
이하 0-S에서는 **인자를 먼저** 확인하고, 없을 때만 git에서 감지한다.

### 0-S-1. repo_slug 결정

1. `--repo` 플래그 → 해당 값 사용 (git 명령 실행 안 함)
2. 플래그 없음 → git 감지 시도:
   ```bash
   git rev-parse --show-toplevel 2>/dev/null | xargs basename
   ```
3. 둘 다 실패 → 안내 출력 후 **즉시 종료** (이후 단계 진행하지 않음):
   ```
   repo를 특정할 수 없습니다. --repo 플래그를 지정하세요.
   예: /project-planner status --repo commerceos-backend-v2 --project AG-1614
   ```

### 0-S-2. project_name 결정

1. `--project` 플래그 → 해당 값 사용
2. 플래그 없음 → git 브랜치 감지 시도:
   ```bash
   git branch --show-current
   ```
   → `feature/{name}` 패턴이면 `{name}` 추출
3. 둘 다 실패 → 안내 출력 후 **즉시 종료**:
   ```
   project를 특정할 수 없습니다. --project 플래그를 지정하세요.
   예: /project-planner status --project AG-1614
   ```

### 0-S-3. 프로젝트 데이터 존재 확인

`${PROJ_DIR}` (`${REPOS_ROOT}/{repo_slug}/projects/{project_name}/`) 디렉토리 존재 여부 확인.

- **미존재** → 한 줄 안내 후 종료:
  ```
  '{project_name}' 프로젝트의 플래너 데이터가 없습니다.
  ```
  디렉토리를 생성하지 않는다.
- **존재** → Step 1 진행.

### 0-S-4. 경로 바인딩

SKILL.md의 경로 상수 테이블에 따라 런타임 경로를 바인딩한다. mkdir 실행하지 않음.

---

## Step 1: 데이터 수집

가능한 한 병렬로 수집한다. 각 항목이 없으면 빈 값으로 처리 (에러 아님).

### 1-1. 기본 정보

| 수집 항목 | 소스 | 비고 |
|----------|------|------|
| repo_slug | Step 0-S에서 확정 | |
| branch | `git branch --show-current` | git 환경일 때만. `--repo`로 진입한 경우 "N/A" |
| project_name | Step 0-S에서 확정 | |

### 1-2. 플랜 목록 + 상세

```bash
ls "${PLANS_DIR}" 2>/dev/null
```

각 플랜 디렉토리에서:
- **PLAN.md 첫 3줄**에서 상태·버전 파싱 (패턴: `**상태**: {STATUS}`, `**버전**: v{N}`)
- **PLAN.md** Contract Statement 본문 첫 1-2문장 추출 (플랜의 목표 요약)
- **PLAN.md** Contract Statement의 `§3 추적` 줄에서 R-ID 목록 추출
- **PLAN.md** Context Sources 체크박스에서 연결된 context 파일 목록 추출
- **progress.md** 읽기:
  - `[x]` 개수 = 완료 서브태스크
  - `[x]` + `[ ]` 총합 = 전체 서브태스크 (progress.md에 `[ ]`가 없으면 PLAN.md의 Section 1에서 서브태스크 총 수 카운트)
  - `## 활성 경고` 섹션 내용
  - `## 현재/다음` 섹션 내용

### 1-3. 검증 결과

```bash
ls "${VALIDATION_DIR}" 2>/dev/null | sort | tail -1
```

최신 검증 파일에서:
- **헤더**: `전체 판정: {RESULT}` 파싱
- **R-ID 판정 요약 테이블**: `| R-{N} | {판정} |` 행들 추출
- **변경 파일 통계** 행 추출 (예: `변경 파일 (5개, +336/-158)`)

### 1-4. 검증 stale 여부 판정

history.jsonl에서 최신 `validation_completed`의 `ts`와, 해당 검증 이후의 `plan_updated` 이벤트 존재 여부를 비교.
- `plan_updated.ts > validation_completed.ts`이면 → stale 플래그 설정

### 1-5. PR Summary

```bash
ls "${SUMMARY_DIR}" 2>/dev/null | sort -V | tail -1
```

최신 Summary 파일의 **첫 줄** (PR 제목)을 추출.

### 1-6. 컨텍스트 파일

```bash
ls "${CTX_DIR}" 2>/dev/null | grep -E '^[0-9]{3}-' | sort
```

각 파일의 **첫 번째 `#` 제목 줄**을 추출하여 요약으로 사용.

플랜의 Context Sources에 연결되지 않은 컨텍스트 파일이 있는지 확인 → unmapped_contexts 목록 생성.

### 1-7. 이력 타임라인

```bash
cat "${HIST_FILE}" 2>/dev/null
```

history.jsonl의 각 JSON 라인을 파싱하여 시간순 이벤트 목록 생성.
**최근 10건만** 유지. 초과분은 카운트만 기록.

### 1-8. 워킹 트리 (git 환경일 때만)

`--repo`로 진입한 경우 이 단계를 건너뛴다.

```bash
git status --short 2>/dev/null
```

---

## Step 2: 포맷팅 + 출력

수집된 데이터를 아래 구조로 마크다운 출력한다. **파일 저장 없음** — 사용자에게 직접 텍스트로 보여준다.

### 출력 레이아웃

> 정보 배치 원칙: **행동 가능한 정보(actionable)**가 위, **참고용 정보(reference)**가 아래.

```markdown
## {project_name} 프로젝트 현황

### 기본 정보

| 항목 | 값 |
|------|------|
| repo | {repo_slug} |
| branch | {branch 또는 N/A} |
| project | {project_name} |

---

### 플랜 ({N}개)

#### {plan_name} — `{STATUS}` v{버전}

> {Contract Statement 첫 1-2문장 — 이 플랜의 목표}

| 파이프라인 | Context | Plan | Exec | Validate | Summary |
|:----------:|:-------:|:----:|:----:|:--------:|:-------:|
| {plan_name} | {✅/—} | {✅/—} | {✅/🔄/—} | {✅/⚠️/—} | {✅/—} |

파이프라인 기호:
- ✅ = 완료 (history.jsonl에 해당 이벤트 존재)
- 🔄 = 진행 중 (IN_PROGRESS 상태)
- ⚠️ = 부분 달성 (PARTIAL)
- — = 미실행

- R-ID: {R-1, R-2, ...} ({총 N}개)
- 서브태스크: {완료}/{전체} ({퍼센트}%)
- 활성 경고: {있으면 내용, 없으면 "없음"}
- 현재/다음: {있으면 내용}

(플랜이 여러 개면 각각 반복. IN_PROGRESS > PLANNED > DONE 순으로 정렬)

(없으면: "플랜 없음")

---

### 검증

- 최신: {파일명} | 판정: **{ALL_ACHIEVED / PARTIAL / NOT_ACHIEVED}**
{stale이면: "  ⚠️ 검증 이후 플랜이 수정됨 — 재검증 권장"}
- 변경 파일: {N}개 ({+lines/-lines})
- R-ID 요약:

전체 ACHIEVED이면 한 줄로: "모든 R-ID ACHIEVED ({N}개)"

ACHIEVED가 아닌 항목이 있으면 테이블로 해당 항목만 표시:
| R-ID | 판정 | 비고 |
|:----:|:----:|------|
| R-2 | PARTIAL | {코드 증거 요약} |

(없으면: "검증 미실행")

---

### PR Summary

- **{v버전}**: {PR 제목 줄}

(없으면: "PR Summary 미생성")

---

### 추천 액션

{Step 3에서 생성한 추천 목록}

---

### 컨텍스트 ({N}개)

- `001-{slug}.md` — {# 제목}
- `002-{slug}.md` — {# 제목}

(전체 목록을 축약 없이 표시)

(없으면: "컨텍스트 파일 없음")

---

### 이력 (최근 {표시 건수}/{전체 건수})

| 시각 | 이벤트 |
|------|--------|
| {ts} | {type 한글 변환}: {요약} |

type 한글 매핑:
- context_added → "컨텍스트 추가"
- plan_created → "플랜 생성"
- plan_completed / plan_done → "플랜 완료"
- plan_updated → "플랜 수정"
- validation_completed → "검증 완료"
- summary_created → "Summary 생성"

최근 10건만 표시. 초과 시: "... +{N}건 이전 이벤트"

(없으면: "이력 없음")

---

### 워킹 트리

{git status --short 결과}

(git 환경 아니면 이 섹션 생략)
(변경 없으면: "클린 상태")
```

---

## Step 3: 추천 액션

현황 데이터를 기반으로 **다음에 수행할 수 있는 커맨드**를 제안한다. 해당하는 조건을 **모두** 수집하여 출력한다.

### 판단 로직

아래 조건을 위에서 아래로 순서대로 평가. 해당하는 항목을 모두 수집.

| 우선순위 | 조건 | 추천 |
|:--------:|------|------|
| 1 | IN_PROGRESS 플랜 존재 | `→ /project-planner exec {plan_name}` — 실행 이어서 진행 |
| 2 | PLANNED 플랜 존재 | `→ /project-planner exec {plan_name}` — 실행 시작 |
| 3 | 플랜이 update된 후 재실행되지 않음 (history에 `plan_updated` 이후 `plan_completed` 없음) | `→ /project-planner exec {plan_name}` — 수정된 플랜 재실행 |
| 4 | DONE 플랜 존재 + 해당 플랜의 검증 없음 | `→ /project-planner validate {plan_name}` — DoD 검증 |
| 5 | 검증 결과 PARTIAL 또는 NOT_ACHIEVED | `→ /project-planner update {plan_name}` — 미달 항목 수정 |
| 6 | 검증이 stale (검증 이후 플랜 수정됨) | `→ /project-planner validate {plan_name}` — 재검증 필요 |
| 7 | 모든 플랜 DONE + 검증 ALL_ACHIEVED + Summary 없음 | `→ /project-planner summary create` — PR 본문 생성 |
| 8 | Summary 존재 + 워킹 트리에 변경 있음 | `→ 커밋 & PR 생성 (Summary v{N} 활용 가능)` |
| 9 | 플랜의 Context Sources에 연결되지 않은 새 컨텍스트 파일 존재 | `→ /project-planner update {plan_name}` — 새 컨텍스트 반영 검토 |
| 10 | 컨텍스트만 있고 플랜 없음 | `→ /project-planner create <plan-name>` — 새 플랜 생성 |
| 11 | 아무 데이터도 없음 (빈 프로젝트) | `→ /project-planner context <name>` — 문제 탐색 시작 |
| 12 | 모든 단계 완료 (DONE + ALL_ACHIEVED + Summary 존재 + 클린 상태) | `✅ 모든 단계 완료` |

### 출력 형식

추천 액션은 Step 2 출력 레이아웃의 "추천 액션" 섹션에 배치한다.

```markdown
### 추천 액션

- {추천 내용}
- {추천 내용}
```

우선순위 12(모든 단계 완료)에 해당하면 "✅ 모든 단계 완료"만 표시.

---

## 요약: §STATUS가 하지 않는 것

- 파일 생성·수정·삭제
- history.jsonl / knowledge.jsonl / sessions.jsonl 기록
- 디렉토리 생성 (mkdir)
- AskUserQuestion (정보 조회 목적이므로 사용자 입력 불필요)
- 외부 rules/ 또는 references/ 파일 로드
