# github-actions-cicd-demo

> A complete CI/CD pipeline demonstrating security scanning, multi-version testing,
> Docker image publishing to GHCR, and staged deployment — all built with GitHub Actions.

**Context:** This is the pipeline implementation from [devops-labs Module-3 Mini-Project 9](https://github.com/darestack/devops-labs/tree/main/Module-3/mini-project-09). Full implementation notes live there.

---

## Pipeline Stages

| Stage | What It Does | Key Detail |
|---|---|---|
| `test` | Unit tests across multiple Node.js versions | Build matrix: Node 12, 14, 16 with npm dependency caching |
| `code-quality` | ESLint static analysis | Configured to **fail the build** on lint errors — no `continue-on-error` |
| `security` | Trivy vulnerability scan | Results uploaded as SARIF to GitHub Code Scanning |
| `build` | Docker image creation | Pushes to GHCR tagged with both `branch-name` and `commit-SHA` |
| `deploy` | Staged rollout | Staging deploy → integration + smoke tests → production deploy |

---

## Architecture

```
Push to main / PR opened
  │
  ├── test (matrix: Node 12, 14, 16)
  │     └── npm ci (cached) → npm test → coverage report
  │
  ├── code-quality
  │     └── ESLint → fails build on any error
  │
  ├── security
  │     └── Trivy filesystem scan → SARIF → GitHub Code Scanning
  │
  ├── build (depends on: test + code-quality + security)
  │     └── docker/setup-buildx → docker/build-push → GHCR
  │
  └── deploy (depends on: build)
        ├── Staging environment deploy
        ├── Integration tests + smoke tests
        └── Production deploy
```

---

## Key Implementation Decisions

**GHCR image tagging:** Each image is tagged with both `latest` and the commit SHA — enabling rollback to any previous build without relying on the `latest` tag alone.

**ESLint as a hard gate:** Removed `continue-on-error: true` from the lint step. If ESLint finds errors, the entire pipeline stops — no broken code reaches Docker build.

**Trivy SARIF upload:** Required adding `security-events: write` to the job-level permissions block. Without this, the upload fails silently.

**Docker Buildx:** The default GitHub Actions Docker driver does not support cache export. Fixed by adding `docker/setup-buildx-action@v3` before the build step.

**GHCR push permissions:** `GITHUB_TOKEN` requires explicit `packages: write` in the job permissions to create new container packages.

---

## How to Run

```bash
git clone https://github.com/darestack/github-actions-cicd-demo.git
cd github-actions-cicd-demo
npm ci
npm test
npm run lint
```

Push to `main` or open a PR to trigger the full pipeline.

---

## Stack

`GitHub Actions` · `Docker` · `GHCR` · `Trivy` · `Node.js` · `ESLint`
