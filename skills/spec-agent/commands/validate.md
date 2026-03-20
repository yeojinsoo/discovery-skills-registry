## §VALIDATE — DoD 검증

실행 완료된 스펙들의 구현 결과가 프로젝트의 Definition of Done (PROBLEM.md §3)을 실제로 달성했는지, 분석 방법론을 적용하여 논리적으로 검증한다.

### 공유 SOT (스텝별 참조)

| 스텝 | 참조 | 설명 |
|------|------|------|
| §4 Phase A, §5-1~5-2 Phase C | `rules/analytical-method.md` + `references/analysis-profiles.md` validate 프로파일 | 공유 분석 방법론 (SOT 유지) |
| §3 collect_evidence | `rules/knowledge-conventions.md §3` | Knowledge 스키마 (공유 SOT) |
| §9 write_report | `rules/history-file-conventions.md` | history.jsonl 스키마 (공유 SOT) |
| §11 compose_findings + write_context_file | SKILL.md `resolve_next_filename` + `write_context_file` + `GATE-TO-CREATE_OR_UPDATE` | 공유 유틸리티 (SOT 유지) |
| §4 Phase A 회귀 검사 | `rules/regression-test-strategy.md` §4 (L3) | 3계층 회귀 테스트 전략 — Knowledge 기반 회귀 |

### StateGraph

```
parse_args → load_specs → collect_evidence
  → phase_A(R-ID별, 구축 + 비판)
  → compose_agent_team → phase_C_R1(Agent 병렬) → phase_C_R2(메인 판정)
  → phase_S(종합 판정)
  → compose_report → GATE-VALIDATE-CONFIRM(AskUserQuestion)
  → write_report → report → GATE-TO-SUMMARY
      [A] ALL_ACHIEVED:
          [1] → Skill(/spec-agent summary)
          [2] → 종료
      [B] PARTIAL/NOT_ACHIEVED:
          [1] → Skill(/spec-agent summary)
          [2] → compose_findings → write_context_file → GATE-TO-CREATE_OR_UPDATE
                  [1] → Skill(/spec-agent create)
                  [2] → Skill(/spec-agent update)
                  [3] → 종료
          [3] → 종료
```

### 1. parse_args

- `--spec {name}`: 특정 스펙 (선택적)
- 나머지: 추가 검증 컨텍스트

### 2. load_specs

- spec_name 지정 → 해당 스펙의 SPEC.md + PROBLEM.md + sessions.jsonl + progress.md 로드
- spec_name 미지정 → `${SPECS_DIR}/*/SPEC.md` 전체 스캔, YAML frontmatter `status` 필드가 `DONE`인 스펙 모두 로드
- DONE 상태 스펙 없음 → 에러: "검증 대상 스펙이 없습니다. DONE 상태인 스펙이 필요합니다."

각 스펙에서 추출:
- PROBLEM.md §3 (R-ID 목록 — 검증 대상)
- SPEC.md Section 0 (Contract/Scope/SPEC-TEST)
- sessions.jsonl (실행 결과, checkpoint/CRG 기록)
- progress.md (실행 상태)
- ${KNOW_FILE} (실행 중 학습된 지식)

프로젝트 레벨 컨텍스트 로드:
- `${CTX_DIR}/*.md` 전체 로드 — R-ID의 원래 문제 정의, 설계 원칙 초안, 미결 질문 등 PROBLEM.md에 요약되지 않은 배경 정보를 Phase A/C에서 참조

### 3. collect_evidence — 코드 증거 수집

--repo 모드(Step 0에서 --repo 플래그로 진입한 경우) 시 아래 git 명령을 skip하고, AskUserQuestion으로 변경 파일 목록과 base 브랜치를 수집한다.

```bash
BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
git diff ${BASE_BRANCH}..HEAD --stat
git diff ${BASE_BRANCH}..HEAD
```

R-ID별로 관련 코드를 Read/Grep으로 확인하여 구현 증거를 수집.

> 이 스텝에서 `knowledge-conventions.md §3` 참조

### 4. Phase A — R-ID별 달성 분석

> 이 스텝에서 `analysis-profiles.md` validate 프로파일 참조

`analysis-profiles.md` validate 프로파일의 Phase A 수행.

각 R-ID에 대해 구축-비판 2패스:

**구축 패스**:
1. R-ID의 정의 확인 (PROBLEM.md §3)
2. 해당 R-ID를 커버하는 서브태스크 식별 (SPEC-TEST 매트릭스)
3. 서브태스크의 Oracle 실행 결과 확인 (sessions.jsonl)
4. 실제 코드에서 구현 증거 수집 (git diff)
5. R-ID 달성 여부 잠정 판정

**비판 패스**:
1. Oracle PASS인데 R-ID 미달성 시나리오 탐색
2. 테스트 미커버 엣지 케이스 식별
3. 구현이 의도와 다르게 동작할 수 있는 조건
4. 회귀 리스크 (기존 기능 영향)

**회귀 검사 Sub-Phase** (비판 패스 완료 후 실행):

> 이 스텝에서 `rules/regression-test-strategy.md` §4 (L3 — Knowledge 기반 회귀) 참조

1. `${KNOW_FILE}`에서 `cat:"pitfall"` 또는 `cat:"constraint"` 태그를 포함하는 레코드를 수집
2. 각 레코드의 내용을 현재 R-ID의 코드 증거와 대조하여 재발 여부 확인:
   - 해당하지 않음 → skip
   - 해당하나 재발 없음 → 회귀 없음
   - 동일 패턴 재발 → **재발 감지** — 해당 R-ID에 warning 추가
3. 결과를 Phase A 비판 결과에 포함:

```
### Pitfall/Constraint 재발 검사 (L3)

| K-ID | 요약 | cat | 재발 여부 | 관련 R-ID | 근거 |
|:---:|---|---|:---:|:---:|---|
| {K-NNN} | {요약} | pitfall/constraint | Y/N | {R-ID} | {현재 코드에서의 근거} |
```

### 5. Phase C — 다관점 검증 (Agent 분리 실행)

`analytical-method.md` Phase C + `references/phase-c-agent-protocol.md` 프로토콜에 따라 4-step 구조로 실행하되, 코드 검증 맥락에 맞게 적응한다.

#### 5-0. compose_agent_team

고정 3 에이전트(구조주의자, 회의론자, 실용주의자) + 동적 1 에이전트(프로젝트 도메인에 맞는 관점).

validate 프로파일에서 동적 에이전트는 1개로 고정한다. 시스템 구조가 SPEC.md에 이미 정의되어 있으므로 구조 탐색보다 코드 증거 검증에 집중하기 위함이다.

#### 5-1. phase_C_R1 — 독립 비판 (Agent 병렬)

팀의 각 에이전트에 대해 Agent tool을 **단일 메시지에서 병렬 호출**한다.

각 Agent에 주입하는 내용:
- `${LA_MODULES}/phase-critique.md` Round 1 프롬프트 템플릿
- Phase A 결과 (R-ID별 달성 분석 결과, 증거 체인)
- R-ID 정의 (PROBLEM.md §3), git diff 관련 부분
- 에이전트별 검사 대상 (`phase-c-agent-protocol.md` §2 "코드 검증용")

Agent 허용 도구: Read, Grep, Glob, Bash(읽기 전용) — 실제 코드 확인 가능.

모든 Agent 완료 후 Round 1 결과를 수집한다.

#### 5-2. phase_C_R2 — 판정 및 구조 수정 (메인 직접 수행)

`${LA_MODULES}/phase-critique.md` Round 2 구조에 따라 종합. 역할: 판정관(Adjudicator).
(코드 검증 맥락에서의 검사 대상은 `phase-c-agent-protocol.md §2` 참조)

validate에서는 R2-a~R2-g 전체 범위를 작성한다 (`analytical-method.md` legitimate divergence 참조). 단, R2-c(분해 손실 점검)와 R2-d(구조 수정안)는 R-ID 달성 판정에 관련된 비판이 있을 때에만 작성한다.

### 6. Phase S — 종합 판정

`analysis-profiles.md` validate 프로파일의 Phase S 수행.

각 R-ID에 대한 최종 판정:

| 판정 | 의미 |
|------|------|
| ✅ ACHIEVED | 코드 증거 + Oracle PASS + 다관점 합의 |
| ⚠️ PARTIAL | 일부 달성, 미달성 조건 명시 |
| ❌ NOT_ACHIEVED | 미달성, 갭 상세 설명 |
| 🔍 UNCERTAIN | 증거 부족, 추가 검증 필요 |

전체 프로젝트 판정:
- **ALL_ACHIEVED**: 모든 R-ID가 ✅
- **PARTIAL**: 1개 이상 ⚠️ 또는 🔍, ❌ 없음
- **NOT_ACHIEVED**: 1개 이상 ❌

확신도 선언 (높음/중간/낮음 + 근거). `analysis-profiles.md` validate 프로파일의 확신도 판정 적용 규칙을 따른다.

#### Invariant 침범 판정 하향 규칙

SPEC.md Scope Boundary에서 `IN (Invariant)`로 지정된 항목이 Phase A 회귀 검사 또는 Phase C에서 침범이 확인된 경우, 해당 Invariant와 연관된 R-ID의 판정을 다음과 같이 하향한다:

| 기존 판정 | 하향 판정 | 조건 |
|:---:|:---:|---|
| ACHIEVED | **PARTIAL** | Invariant 침범이 코드 증거로 확인됨 |
| PARTIAL | PARTIAL (유지) | 이미 PARTIAL 이하이므로 추가 하향 없음 |

하향 시 판정 근거에 다음을 명시한다:
- 침범된 Invariant 항목명
- 침범 증거 (diff 또는 코드 위치)
- 원래 판정 및 하향 사유

#### Phase C 합의 검토 의무

Phase C R2-a(합의된 문제) 중 심각도 "중대" 이상인 항목이 있으면, 판정관은 각 항목에 대해 다음 중 하나를 기록해야 한다:

1. **R-ID 판정 반영**: 해당 비판이 특정 R-ID의 달성 근거를 훼손 → R-ID 판정을 PARTIAL 이하로 하향
2. **DoD 범위 외 판정**: 해당 비판이 R-ID DoD 문언의 범위 밖임을 구체적 근거(PROBLEM.md §3 인용 + SPEC.md Scope 참조)와 함께 기록 → 판정 유지, 권고 사항으로 분류

근거 없이 "중대" 합의를 무시하는 것은 금지한다.

### 7. compose_report

`analysis-profiles.md`의 validate 출력 형식에 따라 리포트 작성.
Mermaid 다이어그램 포함 시 mmdc 검증 실행.

### 8. GATE-VALIDATE-CONFIRM

검증 결과를 출력 후 AskUserQuestion:
> "이 검증 결과를 저장할까요?"
- [1] 저장 (Recommended)
- [2] 내용 수정 후 저장
- [3] 취소

### 9. write_report

> 이 스텝에서 `history-file-conventions.md` 참조

```bash
mkdir -p "${VALIDATION_DIR}"
```

파일명: `{YYYY-MM-DD}-{spec_name 또는 project_name}.md`
동일 파일 존재 시 `-v2`, `-v3` 접미사.

Write 도구로 `${VALIDATION_DIR}/{filename}` 저장.

`${HIST_FILE}`에 append:
```jsonl
{"type":"validation_completed","ts":"{ISO 8601}","file":"{filename}","result":"{ALL_ACHIEVED|PARTIAL|NOT_ACHIEVED}","specs":["{spec_name_1}","{spec_name_2}"]}
```

### 9-1. project.json 상태 전환

write_report 완료 후 실행:

현재 `${PROJECT_FILE}`의 status를 읽어, 전체 판정에 따라 전환:
  - 조건: ALL_ACHIEVED → 전환: status → VERIFIED
    - Write: `{"status":"VERIFIED","statusUpdatedAt":"{ISO 8601}"}`
  - 조건: PARTIAL 또는 NOT_ACHIEVED → 전환: status → DEFINING (루프백)
    - Write: `{"status":"DEFINING","statusUpdatedAt":"{ISO 8601}"}`

### 10. report + GATE-TO-SUMMARY

검증 결과 요약 출력:
```
✅ 검증 완료: {VALIDATION_DIR}/{filename}
   전체 판정: {result}
   확신도: {confidence}
   R-ID 현황: ✅ {n}개 / ⚠️ {n}개 / ❌ {n}개 / 🔍 {n}개
```

AskUserQuestion (전체 판정에 따라 선택지 분기):

**[A] ALL_ACHIEVED**:
- [1] PR 본문 생성 (Recommended) → Skill 도구로 `/spec-agent summary create --project {project_name} --repo {repo_slug}` 호출
- [2] 나중에

**[B] PARTIAL / NOT_ACHIEVED** (미달성 R-ID 1개+):
- [1] PR 본문 생성 → Skill 도구로 `/spec-agent summary create --project {project_name} --repo {repo_slug}` 호출
- [2] 추가 작업 필요 (미달성 R-ID 대응) → `compose_findings` → `write_context_file` → `GATE-TO-CREATE_OR_UPDATE` (스텝 11)
- [3] 나중에

### 11. compose_findings + write_context_file + GATE-TO-CREATE_OR_UPDATE

[2] 선택 시 실행. Phase A/C에서 발견된 미달성 사항을 context 파일로 정리하고, create 또는 update로 체이닝한다.

#### 11-1. compose_findings

Phase S 판정이 PARTIAL 또는 NOT_ACHIEVED인 R-ID와, UNCERTAIN인 R-ID를 대상으로 다음 구조의 findings를 조합한다:

```markdown
# Context: {project_name} — Validation Findings

> 작성 시각: {ts} | 상태: COMPLETE
> 생성 방법: validate | 검증 파일: {VALIDATION_DIR}/{filename}

## §1 미달성 사항 (R-ID별)

### {R-ID}: {R-ID 제목}
- **판정**: {PARTIAL|NOT_ACHIEVED|UNCERTAIN}
- **달성 증거**: {있으면 요약, 없으면 "없음"}
- **미달성 조건**: {Phase A 구축 패스에서 식별된 미충족 조건}
- **엣지 케이스**: {Phase A 비판 패스에서 식별된 미커버 케이스}

{미달성 R-ID마다 반복}

## §2 원인 분석

{Phase A 비판 패스 + Phase C R2 종합에서 추출한 미달성 원인}

- **{R-ID}**: {원인 요약}
  - Phase A 비판: {관련 비판 요약}
  - Phase C 합의: {R2-a에서 해당 R-ID 관련 합의 항목, 없으면 "해당 없음"}

## §3 보완 필요 조건 (Definition of Done)

{미달성 R-ID의 해결 조건을 검증 결과 기반으로 재정의. 각 조건은 관찰 가능한 상태로 서술.}

- **{R-ID}**: {보완 후 달성 가능한 상태}
  - 구체 조건: {ACHIEVED로 전환하기 위해 충족해야 할 조건}
```

#### 11-2. write_context_file

> SKILL.md 공유 유틸리티 `resolve_next_filename` + `write_context_file` 참조.

slug: `validation-findings`
→ 파일명: `{NNN}-validation-findings.md` (resolve_next_filename으로 결정)

Write 도구로 `${CTX_DIR}/{NNN}-validation-findings.md` 저장.

`${HIST_FILE}`에 append:
```jsonl
{"type":"context_added","file":"{NNN}-validation-findings.md","ts":"{ISO 8601}"}
```

#### 11-3. GATE-TO-CREATE_OR_UPDATE

> SKILL.md 공유 유틸리티 `report + GATE-TO-CREATE_OR_UPDATE` 참조.

완료 안내:
```
✅ 저장됨: {CTX_DIR}/{NNN}-validation-findings.md
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

[3] 선택 시:
  종료
