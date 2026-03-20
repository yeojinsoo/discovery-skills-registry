# Changelog

## 0.2.0 (2026-03-21)
- project-planner → spec-agent 용어 전면 리브랜딩 (plan→spec, 실행 계획→실행 스펙)
- 런타임 데이터 경로를 `$HOME/.discovery-skills/spec-agent/repos/`로 외부 분리
- SPEC.md 메타데이터를 인라인 blockquote에서 YAML frontmatter로 표준화
- project.json 기반 5단계 프로젝트 상태 모델 도입 (DEFINING/RUNNING/DONE/VERIFIED/CLOSED)
- close 커맨드 추가 — 프로젝트 닫기
- distill 커맨드 추가 — 프로젝트 교훈 정리 (canonical/active/expired 분류 → 레포 승격/아카이브)
- knowledge-conventions.md §8 갱신 — 자동 트리거를 distill 명시 호출로 대체
- knowledge-conventions.md §1/§2에 RK-{NNN} 레포 레벨 ID 네임스페이스 반영
- history-file-conventions.md에 distill_completed 이벤트 타입 추가
- CRG, execution-protocol, session-log, spec-quality-characteristics 규칙 보강

## 0.1.0 (2026-03-20)

- project-planner에서 포크 후 spec-agent로 전면 리브랜딩
- 런타임 데이터 경로를 $HOME/.discovery-skills/spec-agent/repos/로 외부 분리
- plan → spec 용어 전면 치환, 파일명 리네이밍
