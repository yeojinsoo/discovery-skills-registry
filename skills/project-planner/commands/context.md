## §CONTEXT — 프로젝트 플랜 컨텍스트 구체화

`project-planner create` 전/후 언제든 문제·원인·해결 조건·설계 방향·테스트 시나리오를 자유롭게 탐색하고 구체화한다.
컨텍스트는 **세션별 별도 파일**로 축적되며, `project-planner create` / `project-planner update` 시 어떤 파일이 아직 반영되지 않았는지 추적된다.

### 파일 명명 규칙

```
context/{NNN}-{slug}.md
```

- `NNN`: 3자리 시퀀스 (001, 002, ...) — 생성 순서 보장
- `slug`: 선택적 설명 (kebab-case). 생략 시 "context"
- 예: `001-initial.md`, `002-team-review.md`, `003-discovered-constraint.md`

각 `project-planner context` 실행은 **하나의 새 파일**을 생성한다. 기존 파일을 수정하지 않는다.

### SOT 참조

없음 (Gate 없는 자유형 탐색 단계).

### StateGraph

```
parse_args → resolve_next_filename → check_plan_state
  → [seed 있음?]
    → YES: ingest_seed → gap_analysis → [갭 있음?]
              → YES: targeted_ideation_loop → consolidate → ...
              → NO: consolidate → ...
    → NO: [기존 context 다수?]
              → YES: gap_analysis(기존 ctx 기반) → [갭 있음?]
                        → YES: targeted_ideation_loop → consolidate → ...
                        → NO: GATE-IDEATION-START → ideation_loop → consolidate → ...
              → NO: GATE-IDEATION-START → ideation_loop → consolidate → ...
  → GATE-CONTEXT-CONFIRM → write_context_file → report → GATE-TO-CREATE_OR_UPDATE
```

### 1. parse_args

인자에서 추출:
- **슬러그** (선택): 자유 텍스트를 kebab-case로 변환. 없으면 "context" 사용.
- **seed_file** (선택): `--seed {파일경로}` 플래그에서 추출. SKILL.md §0-1 참조.

plan_name은 받지 않는다 (context는 프로젝트 레벨).

### 2. resolve_next_filename

> SKILL.md 공유 유틸리티 `resolve_next_filename` 참조.

### 3. check_plan_state

> SKILL.md 공유 유틸리티 `check_plan_state / load_existing_context` 참조.

### 3-1. seed 분기

check_plan_state 이후 seed_file 존재 여부에 따라 분기:

**seed_file 있음** → `ingest_seed` → `gap_analysis` → 갭 있으면 `targeted_ideation_loop`, 없으면 바로 `consolidate`

**seed_file 없음 + 기존 context 다수 (2개 이상)** → 기존 context 파일들을 모두 읽어 `ctx` 구조체에 병합 → `gap_analysis(기존 ctx 기반)` → 갭 있으면 `targeted_ideation_loop` → `consolidate`, 갭 없으면 `GATE-IDEATION-START` → `ideation_loop` → `consolidate`

**seed_file 없음 + 기존 context 0~1개** → 기존 경로 (`GATE-IDEATION-START` → `ideation_loop` → `consolidate`)

### 3-2. ingest_seed

`--seed {파일경로}` 플래그로 지정된 외부 파일을 Read로 읽어 `ctx` 구조체에 매핑한다.

**파싱 전략:**

1. **context 형식 감지**: 입력에 `^## §[123]` 또는 `^## 반요구사항`으로 시작하는 라인이 있으면 (예: `## §1 증상 (What)`, `## §3 해결 조건`, `## 반요구사항 (Anti-Requirements)` 등) 직접 파싱하여 `ctx`의 각 필드에 매핑
2. **비정형 입력**: 헤더가 없으면 `extract_structure` 로직(§5의 ideation_loop 내 로직과 동일)으로 §1/§2/§3/anti_requirements/principles/test_scenarios/open_questions 추출
3. **잔여 원문 보존**: §1/§2/§3/anti_requirements/principles/test_scenarios/open_questions 어디에도 매핑되지 못한 단락 전체를 `ctx.raw_notes` 필드에 보존. **부분 매핑된 단락도 raw_notes에 전체 보존**한다 (매핑 대상에 포함되면서 동시에 raw_notes에도 보존 — 중복은 정보 소실보다 무해). → consolidate 단계에서 `## 참고 자료` 섹션으로 출력

**에러 처리:**
- seed 파일이 존재하지 않으면 에러 메시지 출력 후 종료: "seed 파일을 찾을 수 없습니다: {seed_file}"
- seed 파일이 비어있으면 경고 후 기존 경로(GATE-IDEATION-START)로 폴백

### 3-3. gap_analysis

seed 또는 기존 context에서 추출한 `ctx`에 대해 6항목 체크리스트를 수행한다.

**체크리스트:**

| # | 항목 | 치명적 | 중요 | 경미 |
|---|------|:-----:|:---:|:---:|
| 1 | §1 완성도: 증상이 서술되었는가? 재현 방법이 있는가? | — | §1 전체 부재 | 재현 방법만 부재 |
| 2 | §2 완성도: 원인이 확인/가설로 구분되었는가? | — | §2 전체 부재 | 확인/가설 구분 없음 |
| 3 | §3 완성도: 관찰 가능한 조건으로 서술되었는가? | §3 전체 부재 | 절차 서술만 존재 | R-ID 미부여 |
| 4 | 설계 원칙: 최소 1개 이상 있는가? | — | — | 부재 |
| 5 | 리스크: 트레이드오프/리스크가 식별되었는가? | — | — | 부재 |
| 6 | 코드베이스 근거: 코드 경로/클래스명 등이 포함되었는가? | — | — | 부재 |
| 7 | 반요구사항: 변경하지 않을 것 / 범위 외 사항이 명시되었는가? | — | — | 부재 |

**출력**: 갭 리포트 (항목별 심각도 + 보완 가이드)

```
📊 Gap Analysis 결과:
  [치명] §3 해결 조건 전체 부재 — 반드시 보완 필요
  [중요] §1 증상 전체 부재 — 보완 권장
  [경미] 설계 원칙 부재 — 선택적 보완
  [경미] 코드베이스 근거 부재 — 선택적 보완
```

**분석 권장 트리거**: 치명적 갭이 1개 이상이고 seed가 없으면 다음 메시지를 출력:
> "체계적 분석이 필요할 수 있습니다. `/logical-analysis`를 먼저 실행하여 분석한 후, 그 결과를 `--seed`로 지정하는 것을 권장합니다. (해당 스킬이 없으면 직접 분석 결과를 정리하여 --seed로 지정할 수도 있습니다)"

### 3-4. targeted_ideation_loop

기존 ideation_loop(§5)의 변형. gap_analysis에서 발견된 갭만을 타겟으로 질문한다.

**기존 ideation_loop과의 차이점:**

1. **타겟 질문**: gap_analysis 결과에서 발견된 갭 항목만 순서대로 질문 (전체 자유형 탐색이 아님)
2. **치명적 갭 필수**: 치명적 심각도의 갭(§3 전체 부재)은 건너뛸 수 없음. 사용자가 "건너뛰기"를 시도하면 한 번 더 요청 (최대 재요청 2회). 2회 거부 시:
   - 경고 메시지 출력: "⚠️ §3(해결 조건)이 미작성 상태입니다. 이 context로 create를 실행하면 불완전한 프로젝트 플랜이 생성될 수 있습니다."
   - context 파일 §3 섹션에 `> ⚠️ 미작성 — 치명적 갭` 경고를 삽입
   - consolidate로 진행
3. **경미한 갭 건너뛰기**: 경미한 심각도의 갭은 사용자가 "건너뛰기", "패스", "skip" 등으로 건너뛸 수 있음
4. **자동 완료**: 모든 갭이 해소(또는 건너뜀)되면 자동으로 consolidate로 진행 (완성 신호 대기 없음)

**질문 형식:**
```
🔍 [{심각도}] {항목명} 보완이 필요합니다.
   {보완 가이드 질문}
   (경미한 갭은 "건너뛰기"로 생략 가능)
```

**사용자 입력 처리**: 각 응답은 extract_structure로 파싱하여 `ctx`에 누적. 갭 해소 여부는 §3-3의 6항목 체크리스트를 타겟 항목에 대해 재수행하여 판정한다 (§5의 gap_detect 4항목이 아닌 §3-3의 gap_analysis 6항목 기준).

### 4. GATE-IDEATION-START

AskUserQuestion:
> "프로젝트 `{project_name}` 컨텍스트를 구체화합니다.
> 문제 상황, 원인, 해결 조건, 설계 방향, 테스트 시나리오, 반요구사항(변경 금지 사항) 등 가지고 있는 정보를 자유롭게 말씀해주세요.
> 완성되면 '저장', '완성', '다음' 중 하나를 말씀해주세요."

### 5. ideation_loop — 자유형 탐색

**사용자 입력마다 수행:**

#### extract_structure
입력에서 다음 항목을 식별하여 `ctx` 구조체에 누적:
- `§1`: 관찰된 현상, 재현 방법, 영향 범위
- `§2`: 원인 (확인/가설 구분), 조사 필요 항목
- `§3`: 해결 후 관찰 가능한 상태 (완료 기준)
- `anti_requirements`: 변경하지 않을 것, 범위에 포함하지 않을 것, 보존해야 할 기존 동작
  - 탐지 신호: "~하지 않는다", "~은 건드리지 않는다", "범위 밖", "변경 불가", "유지해야", "보존", "out of scope" 등 부정/보존 표현
  - §3과의 구분: "달성해야 할 새로운 상태"이면 §3, "변경하지 말아야 할 기존 상태/제약"이면 anti_requirements
- `principles`: 설계 원칙 후보 (P-N)
- `test_scenarios`: 테스트 시나리오 후보 (조건 + 예상 결과)
- `open_questions`: 미결 질문 / 결정 필요 사항

#### gap_detect
아직 불명확한 항목을 파악하여 **제안** 형식으로 질문 (강제 아님):
- §1 불명확: "어떤 상황에서 발생하나요? 재현 방법이 있으면 알려주세요."
- §2 불명확: "원인이 확인된 건가요, 아직 가설인가요?"
- §3 불명확: "해결됐다고 판단할 수 있는 관찰 가능한 기준이 있나요?"
- anti_requirements 없음: "이 작업에서 변경하지 않아야 할 것이나, 범위에서 명시적으로 제외할 사항이 있나요?"
- test_scenarios 없음: "예상하는 테스트 시나리오가 있으면 알려주세요."

> **규칙**: gap_detect 질문은 대화당 최대 2개. 사용자가 무시해도 진행.

#### show_summary
현재까지 누적된 `ctx`를 간략히 요약하여 대화 중 보여준다:
```
📋 현재 컨텍스트:
  §1: {한 줄 요약 또는 "미작성"}
  §2: {한 줄 요약 또는 "미작성"}
  §3: {한 줄 요약 또는 "미작성"}
  설계 원칙: {N}개 | 테스트 시나리오: {N}개 | 미결 질문: {N}개
```

#### 완성 신호 감지
사용자 입력에 "완성", "저장", "다음", "create", "충분해", "이 정도면" 등이 포함되면 → consolidate로 이동.

### 6. consolidate — context 파일 조합

**정보 보존 원칙**: 구조화(헤더 부여, 불릿 정리, 테이블화)는 허용하되, 아래 D1~D5에 해당하지 않는 한 정보를 삭제하거나 추상화하지 않는다. 의심스러우면 보존한다. upstream 정보 소실은 하류(create→exec) 전체에 오류가 전파되므로, 허용된 정제만 수행하고 맥락을 최대 보존한다.

**삭제 금지 대상**: 코드 경로, 함수명, 에러 메시지, 재현 절차, SQL 스키마, 수치, 고유명사, 발화자 정보, 시간 정보, 조건부 맥락("~인 경우"). 이들이 포함된 진술은 D1~D5 어떤 규칙으로도 단독 삭제 불가 (D1/D2에 의한 병합만 허용).

**허용되는 정제 (consolidate 최종 출력에만 적용)**:

| # | 규칙 | 조건 (모두 충족) | 처리 |
|---|------|----------------|------|
| D1 | 완전 중복 제거 | 동일 ctx 필드 + 실질적으로 동일한 내용의 반복 | 후출만 보존, 선출 삭제 |
| D2 | 포섭 병합 | 동일 ctx 필드 + 후출이 선출의 삭제 금지 대상을 모두 포함하면서 추가 정보를 가짐 + 선출에 후출 미포함 삭제 금지 대상 없음 | 후출만 보존 |
| D3 | 모순 태그 | 동일 대상 + 반대 방향 서술 | 양쪽 보존. 후출에 `[최신 방향]`, 선출에 `[이전 — 번복됨]` 태그 |
| D4 | 메타 발화 축약 | ctx 어떤 필드에도 매핑할 구체 명사 없음 + 프로세스 맥락 포함 ("회의에서", "슬랙에서" 등) | 맥락 1줄 요약을 raw_notes에 보존, 원문 삭제 |
| D5 | 순수 필러 삭제 | 구체 명사 없음 + 프로세스 맥락도 없음 ("음...", "잠깐만요" 등) | 삭제 |

**미구체화 표현 처리**: 구체 명사/수치 없이 주제만 언급된 서술("성능이 안 좋다" 등)은 원문 보존하되 `[미구체화]` 태그를 부여한다. 하류 G0에서 CLARIFY 판정으로 보완된다.

**§3/anti_requirements 교차 검증**: consolidate 직전에, §3의 각 항목이 "달성해야 할 새로운 상태"인지, anti_requirements의 각 항목이 "변경하지 말아야 할 기존 상태/제약"인지 재확인한다. 오분류 발견 시 항목을 올바른 카테고리로 이동한다.

`ctx` 구조체를 다음 포맷으로 조합:

```markdown
# Context: {project_name}

> 작성 시각: {ts} | 상태: COMPLETE
> {seed_file 사용 시: "생성 방법: seed | 원본: {seed_path}"}

## §1 증상 (What)

{관찰된 현상 및 재현 방법. 없으면 섹션 생략.}

## §2 원인 분석 (Why)

{확인된 원인 또는 가설. 가설이면 **[가설]** 표시.}

{미확인 항목이 있으면:}
- [ ] 확인 필요: {항목}

## §3 해결 조건 (Definition of Done)

{해결 후 관찰 가능한 상태. 없으면 섹션 생략.}

## 반요구사항 (Anti-Requirements)

- {변경하지 않을 것 / 보존할 기존 동작 / 범위 외 사항}

{없으면 섹션 생략. create G2에서 OUT 초안 및 Invariant로 활용됨.}

## 설계 원칙 초안

- P-1: {원칙}
- P-2: {원칙}

{없으면 섹션 생략.}

## 테스트 시나리오 초안

| # | 조건 | 예상 결과 |
|---|------|---------|
| T-1 | {조건} | {결과} |

{없으면 섹션 생략.}

## 미결 질문 / 결정 사항

- [ ] {질문}

{없으면 섹션 생략.}

## 참고 자료

{raw_notes가 있을 때만 출력. ingest_seed에서 §1/§2/§3 등으로 매핑되지 못한 원문을 여기에 보존.}
{raw_notes 없으면 섹션 생략.}
```

### 7. GATE-CONTEXT-CONFIRM

context 파일 초안을 출력하고, 입력 통계를 병기한 후 AskUserQuestion:

```
📊 추출 통계: §1 {N}항목 | §2 {N}항목 | §3 {N}항목 | 반요구사항 {N}항목 | 설계원칙 {N}개 | 테스트 {N}개 | 미결질문 {N}개 | 참고자료 {N}행
```

> "이 내용으로 컨텍스트를 저장할까요?"
- [1] 저장 (Recommended)
- [2] 내용 수정 후 저장 → 수정 내용을 받아 `ctx` 업데이트 → consolidate 재실행
- [3] 취소

### 8. write_context_file

> SKILL.md 공유 유틸리티 `write_context_file` 참조.

### 9. report + GATE-TO-CREATE_OR_UPDATE

> SKILL.md 공유 유틸리티 `report + GATE-TO-CREATE_OR_UPDATE` 참조.
