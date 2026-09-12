# Requirements Document

## Introduction

The current codebase is a pnpm/Turborepo monorepo rooted at `sitel-tech/` containing three apps (`api/`, `dapp/`, `website/`) and one shared package (`packages/contracts/`). The goal is to split the monorepo into two separate Git repositories while keeping the frontend build pipeline (CI, linting, testing, and staging deployment) fully operational in the new layout.

The two target repositories are:

- **sitel-tech-backend** — contains the Go API (`apps/api/`) and the Rust smart contracts (`packages/contracts/`), along with their CI/CD workflows.
- **sitel-tech-frontend** — contains the Next.js dapp (`apps/dapp/frontend/`) and the marketing/docs website (`apps/website/`), Turborepo/pnpm workspace configuration, and all frontend CI/CD workflows.

## Glossary

- **Monorepo**: The current single `sitel-tech/` Git repository managed with pnpm workspaces and Turborepo.
- **sitel-tech-backend**: The target repository that will contain the Go API and Rust contracts.
- **sitel-tech-frontend**: The target repository that will contain the dapp and website frontends.
- **Split_Tool**: The tooling and scripts responsible for performing the repository split (e.g., `git filter-repo`, shell scripts).
- **CI_Pipeline**: GitHub Actions workflows that build, lint, test, and deploy the codebase.
- **Frontend_Build_Pipeline**: The subset of the CI_Pipeline responsible for building, testing, and deploying the dapp and website.
- **Workspace_Config**: The pnpm workspace manifest (`pnpm-workspace.yaml`), root `package.json`, and Turborepo configuration (`turbo.json`).
- **Lockfile**: The `pnpm-lock.yaml` at the monorepo root that pins all frontend dependency versions.
- **Git_History**: The commit history preserved for files within each target repository.

---

## Requirements

### Requirement 1: Repository Partition

**User Story:** As a developer, I want the monorepo split into exactly two repositories, so that backend and frontend concerns have independent release cadences and access controls.

#### Acceptance Criteria

1. THE Split_Tool SHALL produce exactly two output repositories: `sitel-tech-backend` and `sitel-tech-frontend`.
2. THE `sitel-tech-backend` repository SHALL contain `apps/api/`, `packages/contracts/`, and all root-level configuration files that apply exclusively to those components (e.g., `Makefile`, `docker-compose.yml`, `go.mod`, `go.sum`, `Cargo.toml`, `Cargo.lock`).
3. THE `sitel-tech-frontend` repository SHALL contain `apps/dapp/frontend/`, `apps/website/`, `packages/` (excluding `contracts/`), and the Workspace_Config files (`package.json`, `pnpm-workspace.yaml`, `turbo.json`, `pnpm-lock.yaml`).
4. WHEN the split is complete, THE Split_Tool SHALL verify that no source file exists in both output repositories.

### Requirement 2: Git History Preservation

**User Story:** As a developer, I want the commit history preserved for each file after the split, so that `git log` and `git blame` remain useful.

#### Acceptance Criteria

1. WHEN producing `sitel-tech-backend`, THE Split_Tool SHALL retain the full Git_History for every file included in that repository.
2. WHEN producing `sitel-tech-frontend`, THE Split_Tool SHALL retain the full Git_History for every file included in that repository.
3. THE Split_Tool SHALL NOT include commits that exclusively touched files destined for the other repository.

### Requirement 3: Frontend Build Pipeline Continuity

**User Story:** As a developer, I want the frontend build pipeline to work without modification after the split, so that CI/CD continues to gate every pull request.

#### Acceptance Criteria

1. WHEN a pull request is opened against `sitel-tech-frontend`, THE CI_Pipeline SHALL run the `dapp-frontend` job (build, lint, unit tests, coverage) for changes under `apps/dapp/frontend/`.
2. WHEN a pull request is opened against `sitel-tech-frontend`, THE CI_Pipeline SHALL run the `website` job (build, lint) for changes under `apps/website/`.
3. WHEN a pull request is opened against `sitel-tech-frontend`, THE CI_Pipeline SHALL run the `dapp-e2e` job (Playwright) for changes under `apps/dapp/frontend/`.
4. THE Frontend_Build_Pipeline SHALL install JavaScript dependencies using `pnpm install --frozen-lockfile` against the Lockfile contained in `sitel-tech-frontend`.
5. WHEN the `pnpm --filter @sitel-tech/dapp build` command is executed inside `sitel-tech-frontend`, THE Frontend_Build_Pipeline SHALL produce a `.next/` output directory without errors.
6. WHEN the `pnpm --filter @sitel-tech/website build` command is executed inside `sitel-tech-frontend`, THE Frontend_Build_Pipeline SHALL produce a build output directory without errors.

### Requirement 4: Frontend Workspace Configuration

**User Story:** As a developer, I want the pnpm workspace and Turborepo config to be valid in the new standalone `sitel-tech-frontend` repository, so that local development commands work immediately after cloning.

#### Acceptance Criteria

1. THE `sitel-tech-frontend` repository SHALL contain a `pnpm-workspace.yaml` that declares all frontend package paths and excludes any backend paths.
2. THE `sitel-tech-frontend` repository SHALL contain a `turbo.json` that defines the `build`, `dev`, `lint`, and `type-check` tasks without referencing backend packages.
3. WHEN `pnpm install` is run in the root of `sitel-tech-frontend`, THE Workspace_Config SHALL resolve all dependencies without errors.
4. WHEN `pnpm run build` is run in the root of `sitel-tech-frontend`, THE Workspace_Config SHALL execute the Turborepo build pipeline for both `@sitel-tech/dapp` and `@sitel-tech/website` in dependency order.
5. IF a `pnpm-workspace.yaml` entry references a path that does not exist in `sitel-tech-frontend`, THEN THE Workspace_Config SHALL cause `pnpm install` to produce an error.

### Requirement 5: CI/CD Workflow Migration

**User Story:** As a developer, I want the GitHub Actions workflows migrated to the correct repository, so that each repository's CI only runs jobs relevant to it.

#### Acceptance Criteria

1. THE `sitel-tech-frontend` repository SHALL contain GitHub Actions workflows for: change detection, dapp build/lint/test, website build/lint, dapp E2E (Playwright), security scanning (JS/SAST), JS dependency auditing, SBOM generation, and staging deployment.
2. THE `sitel-tech-backend` repository SHALL contain GitHub Actions workflows for: Go API build/test/lint, Rust contracts build/test, security scanning (Go/Rust), dependency auditing, SBOM generation, and staging deployment.
3. WHEN a workflow in `sitel-tech-frontend` references a file path (e.g., `apps/dapp/frontend/`), THE CI_Pipeline SHALL use paths that are valid relative to the root of `sitel-tech-frontend`.
4. WHEN a workflow in `sitel-tech-backend` references a file path (e.g., `apps/api/`), THE CI_Pipeline SHALL use paths that are valid relative to the root of `sitel-tech-backend`.
5. THE CI_Pipeline in `sitel-tech-frontend` SHALL NOT reference any Go, Rust, or `packages/contracts` paths.
6. THE CI_Pipeline in `sitel-tech-backend` SHALL NOT reference any pnpm, Node.js, or frontend-specific paths.

### Requirement 6: Secrets and Environment Variable Continuity

**User Story:** As a platform engineer, I want all required secrets and environment variables documented for each new repository, so that no CI job silently fails due to missing configuration.

#### Acceptance Criteria

1. THE Split_Tool SHALL produce a migration checklist that lists every GitHub Actions secret referenced in `sitel-tech-frontend` workflows (e.g., `SEMGREP_APP_TOKEN`, `SMOKE_TEST_WALLET_SECRET`, `STAGING_DEPLOY_TOKEN`, `SLACK_DEPLOYMENT_WEBHOOK`).
2. THE Split_Tool SHALL produce a migration checklist that lists every GitHub Actions secret referenced in `sitel-tech-backend` workflows.
3. WHEN a secret is referenced in a workflow but absent from the corresponding repository's secret store, THE CI_Pipeline SHALL fail with a descriptive error message rather than silently skipping the step.

### Requirement 7: Shared Dependency Elimination

**User Story:** As a developer, I want no cross-repository pnpm workspace dependencies after the split, so that each repository can be installed and built independently.

#### Acceptance Criteria

1. WHEN the split is complete, THE `sitel-tech-frontend` repository SHALL contain no `workspace:*` dependency references that point to packages in `sitel-tech-backend`.
2. WHEN the split is complete, THE `sitel-tech-backend` repository SHALL contain no `pnpm-workspace.yaml` or Node.js package manifests that reference frontend packages.
3. IF a package in `apps/dapp/frontend/` or `apps/website/` declares a `workspace:*` dependency on `packages/contracts/`, THEN THE Split_Tool SHALL flag this dependency in the migration checklist and propose a versioned npm package or submodule alternative.

### Requirement 8: Backend Build Pipeline Continuity

**User Story:** As a backend developer, I want the Go and Rust build pipelines to work without modification after the split, so that the API and contracts CI continues to gate every pull request.

#### Acceptance Criteria

1. WHEN a pull request is opened against `sitel-tech-backend`, THE CI_Pipeline SHALL run `go build ./...` and `go test -race -timeout 120s -short ./...` for changes under `apps/api/`.
2. WHEN a pull request is opened against `sitel-tech-backend`, THE CI_Pipeline SHALL run `cargo build --target wasm32-unknown-unknown` and `cargo test --lib` for changes under `packages/contracts/`.
3. THE `go.mod` file in `sitel-tech-backend` SHALL remain valid and reference only packages available via public Go module proxy without requiring files from `sitel-tech-frontend`.
4. THE `Cargo.toml` in `sitel-tech-backend` SHALL remain valid and resolve all crates without requiring files from `sitel-tech-frontend`.

### Requirement 9: Local Development Setup

**User Story:** As a developer, I want clear documentation and scripts for setting up the local development environment for each new repository, so that onboarding takes no longer than it does with the current monorepo.

#### Acceptance Criteria

1. THE `sitel-tech-frontend` repository SHALL contain a `README.md` that documents the prerequisites (Node.js version, pnpm version), installation command (`pnpm install`), and development start command (`pnpm dev`).
2. THE `sitel-tech-backend` repository SHALL contain a `README.md` that documents the prerequisites (Go version, Rust/cargo version, Docker), and the development start command (`make dev` or `docker compose up --build`).
3. WHEN `make dev` is executed in `sitel-tech-backend`, THE development environment SHALL start all required services (API, database, Redis) as defined in `docker-compose.yml`.
4. WHEN `pnpm dev` is executed in `sitel-tech-frontend`, THE development environment SHALL start the dapp dev server on port 3001 and the website dev server on the default Next.js port.

### Requirement 10: Staging Deployment After Split

**User Story:** As a platform engineer, I want the staging deployment workflow to deploy the correct artifacts from each repository independently, so that a frontend release does not require a backend deployment and vice versa.

#### Acceptance Criteria

1. THE `sitel-tech-frontend` CI_Pipeline SHALL include a staging deployment workflow that builds the dapp and website and deploys them independently of the backend.
2. THE `sitel-tech-backend` CI_Pipeline SHALL include a staging deployment workflow that builds and deploys the API and signer binaries independently of the frontend.
3. WHEN both repositories have active staging deployments, THE Frontend_Build_Pipeline SHALL configure `NEXT_PUBLIC_API_URL` and `NEXT_PUBLIC_WS_URL` to point to the independently deployed backend staging endpoint.
4. THE staging deployment workflow in `sitel-tech-frontend` SHALL run the existing smoke test suite (`tests/smoke.spec.ts`) against the deployed staging environment as a promotion gate.
