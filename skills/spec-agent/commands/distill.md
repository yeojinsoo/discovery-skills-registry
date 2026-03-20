## §DISTILL — 프로젝트 교훈 정리

프로젝트 레벨 `knowledge.jsonl`의 교훈을 분류하여 레포 레벨 승격, 아카이브, 또는 유지를 수행한다.

> sessions.jsonl: 해당 없음 (프로젝트 레벨 커맨드, 스펙 단위 세션 기록 대상 아님)

### 공유 SOT (스텝별 참조)

| 스텝 | 참조 | 설명 |
|------|------|------|
| §3 classify | `rules/knowledge-conventions.md §4, §5, §7` | confidence 생명주기, 2계층 스코프, 승격 기준 |
| §6 report | `rules/history-file-conventions.md` | history.jsonl 스키마 (distill_completed) |

### StateGraph

```
parse_args → load_knowledge → classify
  → GATE-CONFIRM(AskUserQuestion)
  → execute(promote/archive/keep)
  → report
```

### 1. parse_args

distill은 서브커맨드 인자 없음 (프로젝트 레벨 커맨드).
`--project`, `--repo` 플래그는 SKILL.md Step 0에서 처리 완료.

### 2. load_knowledge

`${KNOW_FILE}` (`${PROJ_DIR}/knowledge.jsonl`)을 읽는다.

- **파일 없음 또는 0건** → "정리할 교훈이 없습니다" 안내 후 종료
- **레코드 존재** → 전체 로드하여 각 레코드를 JSON 파싱

### 3. classify

각 레코드를 3가지 분류 중 하나로 배정한다.

| 분류 | 기준 | 처리 |
|------|------|------|
| **canonical** | 레포 전체 적용 가능 + confidence가 `validated` 이상 | 레포 레벨로 승격 |
| **active** | 프로젝트 한정 유효 | 프로젝트 레벨에 유지 |
| **expired** | 기술 변경으로 무효화 또는 일회성 교훈 | 아카이브 |

#### 분류 판정 규칙

0. `confidence` 필드가 없는 레코드는 `observed`로 간주한다 (§4 기본값과 일관)
1. `confidence`가 `observed`인 레코드는 **canonical이 될 수 없다** → active 또는 expired만 가능
2. `expires_when` 필드가 존재하고 그 조건이 현재 충족된 레코드 → **expired** (LLM이 현재 프로젝트/코드베이스 상태를 참조하여 판단. 불확실하면 active로 보수적 분류 — 규칙 5 적용)
3. 특정 스펙/기술에만 해당하는 레코드 (spec 필드가 특정 스펙을 가리키고, insight가 해당 스펙의 기술 스택에 한정) → **active**
4. 범용적이고 재현 확인된 교훈 (confidence가 `validated` 이상 + 특정 기술 스택에 한정되지 않음) → **canonical** 후보 (= 레포 레벨 승격 대상. confidence 필드의 `canonical` 값과는 별개 — §7 기준 충족 후 별도 전환)
5. 위 어느 쪽에도 명확히 속하지 않으면 → **active** (보수적 분류)

### 4. GATE-CONFIRM

AskUserQuestion으로 분류 결과를 표시하고 사용자 확인을 받는다.

#### 출력 형식

```
📋 Knowledge 분류 결과 ({총 N}건)

[canonical → 레포 승격] ({N}건)
| # | id | insight | confidence | 근거 |
|---|-----|---------|------------|------|
| 1 | K-003 | {insight 요약} | validated | {적용된 판정 규칙 1줄: 예) "confidence=validated + 범용적"} |

[active → 유지] ({N}건)
| # | id | insight | confidence | 근거 |
|---|-----|---------|------------|------|
| 1 | K-001 | {insight 요약} | observed | {예) "confidence=observed → canonical 불가"} |

[expired → 아카이브] ({N}건)
| # | id | insight | confidence | 근거 |
|---|-----|---------|------------|------|
| 1 | K-005 | {insight 요약} | observed | {예) "expires_when 충족: Next.js 13 마이그레이션 완료"} |
```

AskUserQuestion:
> "분류 결과를 확인하세요. 어떻게 진행할까요?"
- [1] 이대로 실행 (Recommended)
- [2] 분류 수정 후 실행 (변경할 id와 분류를 입력)
- [3] 취소

[2] 선택 시:
  AskUserQuestion: "변경할 내용을 입력하세요 (예: K-003→active, K-005→canonical):"
  → 입력받은 변경 사항을 반영한 후 다시 GATE-CONFIRM 출력

### 5. execute

사용자 확인 후 각 분류별 처리를 수행한다.

#### 5-1. canonical → 레포 승격

각 canonical 레코드에 대해:

1. **중복 감지**: `${REPO_DIR}/knowledge.jsonl`을 읽어 동일/유사 insight가 이미 존재하는지 확인
   - 동일 insight 존재 → 해당 레코드 스킵 + "RK-{NNN}: 이미 레포 레벨에 동일 교훈 존재 — 스킵" 안내
   - 중복 없음 → 승격 진행

2. **ID 변환**: `K-{NNN}` → `RK-{NNN}` 네임스페이스로 변환
   - RK-{NNN}의 시퀀스: `${REPO_DIR}/knowledge.jsonl`의 마지막 `RK-` ID + 1
   - 파일이 없거나 비어 있으면 `RK-001`부터 시작

3. **레포 레벨에 append**:
   ```bash
   # id를 RK-{NNN}으로 변경. confidence는 기존 값(validated) 유지
   # canonical 전환은 §7 승격 기준 충족 + CLAUDE.md 반영 후 별도 수행
   echo '{변환된 레코드 JSON}' >> "${REPO_DIR}/knowledge.jsonl"
   ```

4. **원본에서 제거**: `${KNOW_FILE}`에서 해당 레코드를 제거
   - jq 또는 grep -v로 해당 id 라인 제거 후 임시 파일 → 원본 덮어쓰기

#### 5-2. expired → 아카이브

각 expired 레코드에 대해:

1. `${PROJ_DIR}/knowledge.archive.jsonl`에 append:
   ```bash
   echo '{레코드 JSON}' >> "${PROJ_DIR}/knowledge.archive.jsonl"
   ```

2. 원본 `${KNOW_FILE}`에서 해당 레코드 제거

#### 5-3. active → 유지

아무 조치 안 함. 프로젝트 레벨 `${KNOW_FILE}`에 그대로 유지.

### 6. report

실행 결과 요약 출력:

```
✅ Knowledge 정리 완료

- 승격: {N}건 → ${REPO_DIR}/knowledge.jsonl
  {RK-001, RK-002, ...}
- 아카이브: {N}건 → ${PROJ_DIR}/knowledge.archive.jsonl
- 유지: {N}건 (프로젝트 레벨 유지)
- 스킵: {N}건 (중복)
```

스킵 건이 0이면 해당 줄 생략.

`${HIST_FILE}`에 append:
```jsonl
{"type":"distill_completed","ts":"{ISO 8601}","promoted":{N},"archived":{N},"kept":{N},"skipped":{N}}
```
