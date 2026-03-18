# discovery-skills-registry

[discovery-skills](https://github.com/yeojinsoo/discovery-skills-cli) CLI가 참조하는 스킬 레지스트리.

## Structure

```
registry.toml              # 스킬 메타데이터 (이름, 버전, 설명)
skills/
  logical-analysis/        # 논리적 해체 분석 스킬
  project-planner/         # 실행 계획 생성/실행/수정 스킬
.github/workflows/
  release-skill.yml        # 스킬별 독립 릴리스 CI
```

## Registry Format

```toml
[metadata]
repo = "yeojinsoo/discovery-skills-registry"

[skills.logical-analysis]
version = "1.0.0"
description = "개념이나 시스템을 논리적으로 완전 해체하는 분석 스킬"

[skills.project-planner]
version = "1.0.0"
description = "실행 계획 생성/실행/수정 통합 스킬"
```

## Release Process

스킬별 독립 릴리스. 태그 패턴: `{skill-name}-v{version}`

```bash
# 1. 스킬 파일 수정
# 2. registry.toml 버전 올리기
# 3. 커밋 + 태그 push
git add -A && git commit -m "update logical-analysis v1.1.0"
git tag logical-analysis-v1.1.0
git push && git push --tags

# CI가 자동으로 GitHub Release 생성
# → 사용자는 discovery-skills update 로 수신
```

## For Users

이 레포를 직접 사용하지 않습니다. CLI를 설치하세요:

```bash
curl -fsSL https://github.com/yeojinsoo/discovery-skills-cli/releases/latest/download/discovery-skills-installer.sh | sh
discovery-skills install
```

## License

MIT
