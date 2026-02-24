# watcha-actions-platform

Watcha 표준 GitHub Actions reusable workflows 및 composite actions 중앙 레포.

## 구조

```
.github/workflows/
  reusable-create-release.yml   # release 브랜치 생성 + main Draft PR
  reusable-service-cicd.yml     # validate + build + deploy (develop/staging)
  reusable-ci-main.yml          # prod 빌드 + tag + GitHub Release + back-merge
actions/
  helm-values-update/           # Helm values 이미지 태그 업데이트 (GitOps CD trigger)
templates/
  .github/workflows/
    ci.yml                      # 팀 레포 caller: develop + release-* push
    ci-main.yml                 # 팀 레포 caller: main push
    create-release.yml          # 팀 레포 caller: workflow_dispatch
```

## 팀 레포 적용 방법

### 1. `templates/.github/workflows/` 파일 3개를 팀 레포에 복사

```bash
cp templates/.github/workflows/* your-repo/.github/workflows/
```

### 2. `YOUR_APP_NAME`, `YOUR_ORG` 교체

| 파일 | 교체할 값 |
|------|-----------|
| `ci.yml` | `app_name`, `image_repository` (develop + release 각각) |
| `ci-main.yml` | `image_repository` |
| `create-release.yml` | 변경 없음 |

### 3. Secrets 설정

| Secret | 용도 |
|--------|------|
| `GH_PAT` | release 브랜치 push (ci-release 트리거), back-merge PR 생성 |
| `GITHUB_TOKEN` | 이미지 push (GHCR), git tag, GitHub Release, 브랜치 삭제 (자동 제공) |

> `secrets: inherit` 사용 — 팀 레포의 secrets가 reusable workflow에 자동 전달됨.
> 같은 org 내 private 레포 간에만 안전하게 사용 가능.

### 4. 브랜치 생성

```bash
git checkout -b develop
git push origin develop
```

---

## 이미지 태깅 표준

| 브랜치 | 태그 | 배포 환경 |
|--------|------|----------|
| `develop` | `dev-{7sha}` | Development |
| `release-*`, `hotfix-*` | `stg-{7sha}` | Staging |
| `main` | `vX.Y.Z` + `latest` | Production |

---

## TODO (Production 전환 시)

`reusable-service-cicd.yml`, `reusable-ci-main.yml` 내 `# NOTE(production):` 주석 참고:

- GHCR → ECR (`ecr-build-push` composite action)
- GitHub-hosted runner → self-hosted (`arc-shared`)
- `type=gha` 캐시 → `type=registry` (ECR buildcache)
- deploy placeholder → `helm-values-update` composite action

---

## Secrets 참고 (`GITHUB_TOKEN` vs `GH_PAT`)

`GITHUB_TOKEN`으로 push한 커밋은 **다른 workflow를 트리거하지 않음** (GitHub 보안 정책).

| 작업 | 사용 토큰 |
|------|-----------|
| release 브랜치 push (ci-release 트리거 목적) | `GH_PAT` |
| back-merge PR 생성 + auto-merge | `GH_PAT` |
| GHCR 이미지 push, git tag, GitHub Release, 브랜치 삭제 | `GITHUB_TOKEN` |
