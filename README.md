# github-actions-cicd-demo

> Node.js CI/CD lab demonstrating supported Node.js matrix tests, linting, Trivy SARIF upload, Docker Buildx, GHCR publishing, and simulated staged deploys with GitHub Actions.

**Context:** This is the pipeline implementation from [devops-labs Module-3 Mini-Project 9](https://github.com/darestack/devops-labs/tree/main/Module-3/mini-project-09). Full implementation notes live there.

---

## Pipeline Stages

| Stage | What It Does | Key Detail |
|---|---|---|
| `test` | Unit tests across supported Node.js versions | Build matrix: Node 22 and 24 with npm dependency caching. This job also runs the hard lint gate. |
| `code-quality` | ESLint SARIF upload | Publishes lint annotations to GitHub Code Scanning. The hard lint gate lives in the `test` job. |
| `security` | Trivy vulnerability scan | Results uploaded as SARIF to GitHub Code Scanning |
| `build` | Docker image creation | Pushes to GHCR tagged with both `branch-name` and `commit-SHA` |
| `deploy` | Simulated staged rollout | Staging deploy logs -> integration/smoke-test placeholder -> production deploy logs |

---

## Architecture

```
Push to main / PR opened
  │
  ├── test (matrix: Node 22, 24)
  │     └── npm ci (cached) → npm run lint → npm test → coverage report
  │
  ├── code-quality
  │     └── ESLint SARIF → GitHub Code Scanning annotations
  │
  ├── security
  │     └── Trivy filesystem scan → SARIF → GitHub Code Scanning
  │
  ├── build (depends on: test + code-quality + security)
  │     └── docker/setup-buildx → docker/build-push → GHCR
  │
  └── deploy (depends on: build)
        ├── Simulated staging deploy
        ├── Placeholder integration + smoke tests
        └── Production deploy
```

---

## Key Implementation Decisions

**Supported Node versions:** The workflow uses Node 22 and 24 rather than EOL runtime lines. The Docker image uses Node 24 Alpine for the build and runtime stages.

**GHCR image tagging:** Each image is tagged with branch and commit-SHA-derived tags, and `latest` is emitted only for the default branch. That makes rollback easier than relying on `latest` alone.

**ESLint as a hard gate:** The matrix `test` job runs `npm run lint` before tests. The separate `code-quality` job keeps `continue-on-error` for SARIF upload so annotations can still be published for review.

**Trivy SARIF upload:** Required adding `security-events: write` to the job-level permissions block. Without this, the upload fails silently.

**Docker Buildx:** The default GitHub Actions Docker driver does not support cache export. Fixed by adding `docker/setup-buildx-action@v3` before the build step.

**Security scan before build:** The Docker build waits for tests, code-quality SARIF upload, and Trivy filesystem scanning.

**GHCR push permissions:** `GITHUB_TOKEN` requires explicit `packages: write` in the build job permissions to create new container packages.

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

## Current Boundary

The staging and production deploy jobs are intentionally simulated in this lab. They show environment flow, dependency ordering, image handoff, environment URLs, and notification wiring, but they do not deploy to a real target yet. To make this a production deployment project, replace the `echo` deployment steps with a real target such as EC2, ECS, Kubernetes, or a PaaS and add deployment logs.

---

## Stack

`GitHub Actions` · `Docker` · `GHCR` · `Trivy` · `Node.js` · `ESLint`
