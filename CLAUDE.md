# watcha-actions-platform

## Project Overview

Watcha 표준 GitHub Actions reusable workflows 및 composite actions 중앙 레포. 팀 레포에서 workflow_call로 호출하여 CI 파이프라인을 표준화한다.

## 레포 구조

```
.github/workflows/
  reusable-service-cicd.yml      # validate → build (develop/staging)
  reusable-ci-main.yml           # build → release (production)
  reusable-create-release.yml    # release 브랜치 생성 + Draft PR
actions/
  helm-values-update/action.yml  # Helm values 이미지 태그 업데이트 (GitOps CD trigger)
templates/                       # 팀 레포 복사용 caller workflow 템플릿
```

## Reusable Workflows

### reusable-service-cicd.yml (develop/staging)

validate → build 파이프라인. `workflow_call` inputs:

| Input | 필수 | 설명 |
|-------|------|------|
| `app_name` | Y | 서비스명 |
| `environment` | Y | 배포 환경 (dev/staging) |
| `image_repository` | Y | 이미지 URI (e.g. ghcr.io/org/service) |
| `image_tag_prefix` | N | 태그 prefix (e.g. `dev-`, `stg-`) |
| `setup_command` | N | lint/test 전 셋업 커맨드 |
| `run_lint` / `run_test` | N | lint/test 실행 여부 |
| `lint_command` / `test_command` | N | lint/test 커맨드 |

Output: `image_tag` (e.g. `dev-abc1234`)

### reusable-ci-main.yml (production)

build → release 파이프라인. `workflow_call` inputs:

| Input | 필수 | 설명 |
|-------|------|------|
| `image_repository` | Y | 이미지 URI |
| `back_merge_enabled` | N | develop 역병합 활성화 (default: true) |
| `initial_version` | N | 태그 없을 때 초기 버전 (default: v0.1.0) |

처리 순서: version resolve → Docker build (`vX.Y.Z` + `latest`) → git tag + GitHub Release → release 브랜치 삭제 → 조건부 back-merge

### reusable-create-release.yml

release 브랜치 생성 + main 대상 Draft PR.

| Input | 필수 | 설명 |
|-------|------|------|
| `base_branch` | N | 분기 기준 (default: develop) |
| `version_bump` | Y | patch / minor / major |

## Composite Actions

### helm-values-update

Helm values 파일의 image tag를 업데이트하고 push (GitOps CD trigger).

| Input | 설명 |
|-------|------|
| `helm-repo` | Helm values 레포 |
| `helm-values-path` | values 파일 경로 |
| `image-tag` | 새 이미지 태그 |
| `token` | push용 토큰 |
| `app-name` / `environment` | 커밋 메시지용 |

## 이미지 태깅 표준

| 브랜치 | 태그 | 환경 |
|--------|------|------|
| `develop` | `dev-{7sha}` | Development |
| `release-*` / `hotfix-*` | `stg-{7sha}` | Staging |
| `main` | `vX.Y.Z` + `latest` | Production |

## Secrets 정책

| 토큰 | 용도 | 이유 |
|------|------|------|
| `GH_PAT` | release 브랜치 push, back-merge PR | GITHUB_TOKEN push는 다른 workflow 트리거 안 됨 |
| `GITHUB_TOKEN` | GHCR push, git tag, GitHub Release, 브랜치 삭제 | 자동 제공 |

팀 레포에서 `secrets: inherit`로 전달.

## Production 전환 TODO

workflow 파일 내 `# NOTE(production):` 주석 참고:

- [ ] GHCR → ECR (`ecr-build-push` composite action 구현)
- [ ] runner: `ubuntu-latest` → `arc-shared` (self-hosted)
- [ ] cache: `type=gha` → `type=registry` (ECR buildcache)
- [ ] deploy placeholder → `helm-values-update` composite action 연결
- [ ] `reusable-service-cicd.yml` → `reusable-ci.yml` 파일명 변경 (표준 문서 정합성)
- [ ] Release Notes: `--generate-notes` → PR 제목 + 브랜치 prefix 카테고리 분류
- [ ] Trivy scan 단계 추가 (conditional, staging only)

## 팀 레포 적용 방법

1. `templates/.github/workflows/` 파일 3개를 팀 레포에 복사
2. `app_name`, `image_repository` 교체
3. `GH_PAT` secret 설정
4. `develop` 브랜치 생성

## 참조

- CI 표준화 전략: `~/Workspace/docs/obsidian/Projects/CICD/final/CI-CD 표준화 전략.md`
- 테스트 소비자: `/Users/imjonghyeon/Workspace/code/test/rca-agent/`
