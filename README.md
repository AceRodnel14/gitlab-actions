# gitlab-actions

Shared GitLab CI library. Repos include templates from here; all CI logic lives in
this repo (`jobs/` = hidden building blocks, `templates/` = what repos include).

## Usage

```yaml
include:
  - project: '$CI_PROJECT_NAMESPACE/gitlab-actions'
    file: 'templates/lint_all.yml'
  - project: '$CI_PROJECT_NAMESPACE/gitlab-actions'
    file: 'templates/build_container_images_dockerhub.yml'
  - project: '$CI_PROJECT_NAMESPACE/gitlab-actions'
    file: 'templates/sync_to_github.yml'

stages:      # declare every stage of every included template
  - validate
  - build
  - sync
  - cleanup
```

## Templates

| Template | Stages | What it does |
| :- | :- | :- |
| `lint_all.yml` | validate | Lint + tests for app repos (below); MRs and default branch |
| `lint_infra.yml` | validate | YAML, Kubernetes, shell, OpenTofu lint; secret scan only with a repo `.gitleaks.toml` |
| `build_container_images_dockerhub.yml` | validate, build | **Default.** Check Dockerfile/compose, multi-arch image to Docker Hub |
| `build_container_images_all.yml` | validate, build | Same, to GitLab registry, Docker Hub and GHCR (kept for future use) |
| `build_container_images_no_gitlab.yml` | validate, build | Docker Hub + GHCR (kept for future use) |
| `build_container_images_all_amd64.yml` | validate, build | Like `_all`, amd64 only (only for amd64-only images) |
| `sync_to_github.yml` | sync, cleanup | Mirror the branch to `github.com/$GITHUB_SPACE/<project>` |
| `sync_to_github_remove_sensitive.yml` | sync, cleanup | Mirror with `.substitute` scrubbing (infra repos) |

## When jobs run

- Build and sync jobs: any branch except `ai/*`, plus the tags `latest`, `vX.Y`
  and `vX.Y.Z` (e.g. `v0.1`, `v0.1.0`); never for other tags or `[SyncWithGitHub` commits
  (`jobs/conditions.run.yml`).
- Tag pipelines push the image tagged with the release tag, and the sync job creates the
  tag on GitHub: on the GitHub commit already synced from the tagged commit, or else on a
  new commit (on no branch) holding its files. GitLab commits are never pushed to GitHub.
- Lint jobs: merge request pipelines (including `ai/*` branches) and the default
  branch, and only when matching files exist (`jobs/conditions.merge_request.yml`).

## Lint jobs (`jobs/lint.yml`)

| Job | Runs when the repo has | Tool |
| :- | :- | :- |
| `lint_python` / `test_python` | `*.py` / `tests/` or `test_*.py` | ruff check + format, pytest (uv) |
| `lint_go` / `test_go` | `go.mod` | golangci-lint v2, `go test ./...` |
| `lint_node` | `package.json` | `npm ci`, then `lint`, `typecheck`, `test`, `build` scripts if present |
| `lint_dockerfile` | `Dockerfile*` | hadolint |
| `lint_shell` | `*.sh` | shellcheck |
| `lint_yaml` | `*.yml` / `*.yaml` | yamllint (repo `.yamllint` wins) |
| `lint_kubernetes` | `*.yml` / `*.yaml` (infra template only) | kubeconform + CRD catalog |
| `lint_opentofu` | `*.tf` | `tofu fmt -check`, `tofu validate` |
| `lint_secrets` | always (`lint_all`); `.gitleaks.toml` (`lint_infra`) | gitleaks, redacted; repo `.gitleaks.toml` wins, default skips only `.substitute` |
| `lint_claude_plugin` | `.claude-plugin/*.json` | `claude plugin validate` |

Per-repo variables: `RUFF_FORMAT=false`, `GO_PREPARE="<cmd>"` (e.g.
`mkdir -p web/dist && touch web/dist/.keep` for embedded frontends),
`HADOLINT_THRESHOLD=warning`, `KUBECONFORM_IGNORE=<regex>`.
To turn a job off in one repo: redefine it with `rules: [{when: never}]`.

## CI variables (set in GitLab, never in a repo)

`GITHUB_USERNAME`, `GITHUB_EMAIL`, `GITHUB_TOKEN`, `GITHUB_SPACE`, `DOCKERHUB_USER`,
`DOCKERHUB_TOKEN`. Runner tags: `docker` (container jobs), `shell` (host Docker).
