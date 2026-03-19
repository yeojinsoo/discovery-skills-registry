# 이력 파일 규약 (`history.jsonl`)

> **적용 대상**: `repos/{repo_slug}/projects/{project_name}/history.jsonl` (프로젝트 단위)

프로젝트의 context/spec 추가 이력을 시간순으로 기록한다.

---

## 1. 형식

**형식**: 1행 = 1 JSON 객체. `id` 없음 (시간순 append-only 로그).

**변경 정책**: append-only (수정·삭제 금지).

---

## 2. 이벤트 타입

| type | 발생 시점 | 기록 위치 (SKILL.md) |
|------|----------|---------------------|
| `context_added` | §CONTEXT write_context_file | §CONTEXT 8 |
| `spec_created` | §CREATE write_artifacts | §CREATE 11 |
| `spec_completed` | §EXEC set_done | §EXEC 9 |
| `spec_updated` | §UPDATE update_artifacts | §UPDATE 8 |
| `summary_created` | §SUMMARY save_and_output | §SUMMARY 9 |
| `validation_completed` | §VALIDATE write_report | §VALIDATE 9 |

---

## 3. 스키마

### context_added

```jsonl
{"type":"context_added","file":"001-initial.md","ts":"2026-03-11T09:00:00+09:00"}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `type` | `"context_added"` | 고정 |
| `file` | string | 추가된 context 파일명 |
| `method` | string \| 생략 | 생성 방법. 기본적으로 생략 |
| `ts` | ISO 8601 | 기록 시각 |

### spec_created

```jsonl
{"type":"spec_created","name":"ag-1614","ts":"2026-03-11T09:05:00+09:00","seed":"001-initial.md"}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `type` | `"spec_created"` | 고정 |
| `name` | string | 스펙 이름 |
| `seed` | string \| null | seed context 파일명 (없으면 null) |
| `ts` | ISO 8601 | 기록 시각 |

### spec_completed

```jsonl
{"type":"spec_completed","name":"ag-1614","ts":"2026-03-11T15:00:00+09:00","status":"DONE"}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `type` | `"spec_completed"` | 고정 |
| `name` | string | 스펙 이름 |
| `status` | enum | `DONE` \| `ABANDONED` |
| `ts` | ISO 8601 | 기록 시각 |

### spec_updated

```jsonl
{"type":"spec_updated","name":"ag-1614","ts":"2026-03-11T12:00:00+09:00","changes":"Scope 확장: API 검증 추가"}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `type` | `"spec_updated"` | 고정 |
| `name` | string | 스펙 이름 |
| `changes` | string | 변경 요약 (1문장) |
| `ts` | ISO 8601 | 기록 시각 |

### summary_created

```jsonl
{"type":"summary_created","version":"v3","ts":"2026-03-11T20:00:00+09:00"}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `type` | `"summary_created"` | 고정 |
| `version` | string | 생성된 버전 (예: "v3") |
| `ts` | ISO 8601 | 기록 시각 |

### validation_completed

```jsonl
{"type":"validation_completed","ts":"2026-03-12T18:00:00+09:00","file":"2026-03-12-provider-file-support-v2.md","result":"ALL_ACHIEVED","specs":["provider-file-support-v2"]}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| `type` | `"validation_completed"` | 고정 |
| `file` | string | 검증 리포트 파일명 |
| `result` | enum | `ALL_ACHIEVED` \| `PARTIAL` \| `NOT_ACHIEVED` |
| `specs` | string[] | 검증 대상 스펙 이름 목록 |
| `ts` | ISO 8601 | 기록 시각 |

---

## 4. 공통 필드

| 필드 | 모든 이벤트 | 설명 |
|------|:---------:|------|
| `type` | 필수 | 이벤트 유형 |
| `ts` | 필수 | ISO 8601 기록 시각 |
