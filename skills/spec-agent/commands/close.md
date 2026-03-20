## §CLOSE — 프로젝트 종결

프로젝트를 CLOSED 상태로 전환한다. 어떤 상태에서든 호출 가능.

### StateGraph

```
load_project → confirm_close → set_closed → append_history → report
```

### 1. load_project

`${PROJECT_FILE}` 읽기. 현재 status 확인.

이미 CLOSED → "이 프로젝트는 이미 CLOSED 상태입니다." + 종료.

### 2. confirm_close

현재 상태를 표시하고 AskUserQuestion:

```
프로젝트 '{project_name}' 현재 상태: {status}

CLOSED로 전환하면 더 이상 활성 프로젝트로 표시되지 않습니다.
종결 사유를 입력하세요 (예: "PR 머지 완료", "프로젝트 취소", "다른 프로젝트로 대체"):
```

사유 입력 → 진행.

### 3. set_closed

`${PROJECT_FILE}` 업데이트:
```json
{"status":"CLOSED","statusUpdatedAt":"{ISO 8601}"}
```

### 4. append_history

`${HIST_FILE}`에 append:
```jsonl
{"type":"project_closed","ts":"{ISO 8601}","previousStatus":"{이전 status}","reason":"{사용자 입력 사유}"}
```

### 5. report

```
✅ 프로젝트 '{project_name}' → CLOSED
   이전 상태: {previousStatus}
   사유: {reason}
```
