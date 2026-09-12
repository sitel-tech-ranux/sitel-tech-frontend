# Implementation Plan: monorepo-split

## Overview

Execute the one-time migration that splits the `sitel-tech/` pnpm/Turborepo monorepo into two standalone Git repositories — `sitel-tech-backend` (Go API + Rust contracts) and `sitel-tech-frontend` (Next.js dapp + website) — hosted under the `sitel-tech` GitHub organisation. The deliverable is a migration script, updated config files for each target repo, a Python verification suite, and property-based tests that prove the split invariants hold.

The implementation language for the migration tooling is **Python + Bash**, matching the design's testing library choice (pytest + hypothesis) and the existing `scripts/` convention.

---

## Tasks

- [ ] 1. Scaffold the split-repo tooling structure
  - Create `scripts/split-repo/` directory containing the main orchestration script and helper modules
  - Create `scripts/split-repo/split-repo.sh` — the top-level bash orchestrator; initially just a skeleton with phase labels and pre-flight checks
  - Create `scripts/split-repo/verify.py` — the Python verification suite (empty, stubs only)
  - Create `scripts/split-repo/requirements.txt` pinning `hypothesis==6.131.13` and `pytest==8.3.5`
  - Create `scripts/split-repo/scan_workspace_deps.py` — scans all `package.json` files for `workspace:*` references to backend packages
  - _Requirements: 1.1, 6.1, 6.2_

- [ ] 2. Implement pre-flight checks in `split-repo.sh`
  - Abort with `ERROR: git-filter-repo is not installed. Run: pip install git-filter-repo` if `git filter-repo --version` fails
  - Abort with `ERROR: Uncommitted changes detected. Commit or stash before running the split.` if `git status --porcelain` is non-empty
  - Abort with `ERROR: Output directory sitel-tech-backend/ already exists. Delete it first.` / same for `sitel-tech-frontend/` if either output directory already exists
  - Abort with `ERROR: pnpm is not installed or not in PATH.` if `pnpm` is not found
  - Abort with `ERROR: gh CLI is not installed or not in PATH.` if `gh` is not found (needed for repo creation)
  - Print `✓ Pre-flight checks passed` on success
  - _Requirements: 1.1, 9.1, 9.2_

- [ ] 3. Implement the Audit phase — cross-dependency scanner
  - Write `scripts/split-repo/scan_workspace_deps.py`:
    - Walk all `package.json` files under `apps/dapp/frontend/` and `apps/website/`
    - Flag any `workspace:*` dependency whose package name contains `contracts` or matches `@sitel-tech/contracts*`
    - Print `FLAGGED: <file> depends on packages/contracts via workspace:*.` and exit non-zero if found
    - Print `OK: No workspace:* references to packages/contracts found.` and exit 0 if clean
  - Invoke `scan_workspace_deps.py` as Phase 1 (Audit) from `split-repo.sh`; halt the split if it exits non-zero
  - _Requirements: 7.3_

  - [ ]* 3.1 Write unit tests for `scan_workspace_deps.py`
    - Test with a package.json that has `"@sitel-tech/contracts": "workspace:*"` — assert flagged
    - Test with a package.json that has only versioned deps — assert OK
    - Test with a package.json that has `workspace:*` on an unrelated package — assert OK
    - _Requirements: 7.3_

- [ ] 4. Implement the Filter phase — `git filter-repo` invocations
  - In `split-repo.sh`, Phase 2 (Filter):
    - Clone the monorepo to a scratch directory `_build/sitel-tech-backend/` using `git clone --local <monorepo> _build/sitel-tech-backend/`
    - Run `git filter-repo` on `_build/sitel-tech-backend/` keeping paths: `apps/api/`, `packages/contracts/`, `docker-compose.yml`, `docker-compose.external.yml`, `docker-compose.override.yml`, `docker-compose.signer.yml`, `docker/`, `Makefile`, `scripts/contract-audit.sh`, `scripts/validate_vulnignore.py`, `tests/load/`, `tests/probes/`, `.vulnignore`, `.gitleaks.toml`, `.gitleaksignore`
    - Clone the monorepo to `_build/sitel-tech-frontend/` and run `git filter-repo` keeping paths: `apps/dapp/frontend/`, `apps/website/`, `pnpm-workspace.yaml`, `turbo.json`, `package.json`, `pnpm-lock.yaml`, `.nvmrc`, `.coderabbit.yaml`
    - After each filter, verify the clone is non-empty (has at least one commit) or abort with an error
  - _Requirements: 1.1, 1.2, 1.3, 2.1, 2.2, 2.3_

- [ ] 5. Implement the Patch phase — workspace config updates for `sitel-tech-frontend`
  - In `split-repo.sh`, Phase 3 (Patch), applied to `_build/sitel-tech-frontend/`:
    - Overwrite `pnpm-workspace.yaml` with the frontend-only content:
      ```yaml
      packages:
        - 'apps/dapp/frontend'
        - 'apps/website'
      ```
    - Update `package.json`: remove backend-only `devDependencies` (none expected, but ensure `turbo` stays), rename `name` field from `sitel-tech` to `sitel-tech-frontend`, update `scripts` to remove any Go/Rust references
    - Update all `@sitel-tech/` package name references in `apps/dapp/frontend/package.json` and `apps/website/package.json` to `@sitel-tech/`
    - Update all `pnpm --filter @sitel-tech/` invocations in CI workflow files to `pnpm --filter @sitel-tech/`
    - Run `pnpm install --frozen-lockfile` inside `_build/sitel-tech-frontend/` to validate the updated workspace; abort if it fails
    - Commit the patched files as a single migration commit: `chore: update workspace config for sitel-tech-frontend`
  - _Requirements: 3.4, 3.5, 3.6, 4.1, 4.2, 4.3, 4.4, 7.1_

- [ ] 6. Implement the Patch phase — CI workflow split
  - Create `scripts/split-repo/workflows/frontend/` and `scripts/split-repo/workflows/backend/` directories containing the pre-authored split workflow files (written in this task)
  - Write `sitel-tech-frontend/.github/workflows/ci.yml` containing only the jobs: `changes` (frontend-scoped paths only: `apps/dapp/**`, `apps/website/**`, `pnpm-lock.yaml`, `package.json`, `pnpm-workspace.yaml`), `website`, `dapp-frontend`, `dapp-e2e`, and `sbom` (frontend SBOM only)
    - Rename all `@sitel-tech/` filter references to `@sitel-tech/`
    - Update the `dapp-e2e` artifact path from `apps/dapp/frontend/playwright-report/` — this path is unchanged and valid
  - Write `sitel-tech-backend/.github/workflows/ci.yml` containing only the jobs: `changes` (backend-scoped paths: `apps/api/**`, `packages/contracts/**`, `docker/**`, `Makefile`), `api`, `contracts`, `slo`, `security` (Go/Rust parts only), and `sbom` (backend SBOM only)
  - Copy `dependabot-auto-merge.yml` to both repos unchanged
  - Copy `deploy-staging.yml`, `security.yml`, `e2e-testnet-nightly.yml` to frontend repo with path/filter references updated to `@sitel-tech/`
  - Copy `deploy-staging.yml`, `security.yml`, `contract-audit.yml`, `api-property-nightly.yml`, `contracts-property-nightly.yml`, `load-soak.yml`, `synthetic-probes.yml` to backend repo
  - In Phase 3 (Patch), copy these prepared workflow directories into each `_build/` clone and commit: `chore: install split CI workflows`
  - _Requirements: 3.1, 3.2, 3.3, 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 8.1, 8.2_

- [ ] 7. Implement the Patch phase — dependabot, CODEOWNERS, and README files
  - Write `scripts/split-repo/workflows/frontend/.github/dependabot.yml`:
    ```yaml
    version: 2
    updates:
      - package-ecosystem: npm
        directory: /apps/dapp/frontend
        schedule:
          interval: daily
        open-pull-requests-limit: 0
      - package-ecosystem: github-actions
        directory: /
        schedule:
          interval: daily
        open-pull-requests-limit: 0
    ```
  - Write `scripts/split-repo/workflows/backend/.github/dependabot.yml` with `gomod` (`/apps/api`), `cargo` (`/packages/contracts`), and `github-actions` (`/`) entries
  - Write `scripts/split-repo/workflows/frontend/.github/CODEOWNERS` referencing `@sitel-tech/frontend-team` (not the old `@0xDeon` personal handle)
  - Write `scripts/split-repo/workflows/backend/.github/CODEOWNERS` referencing `@sitel-tech/backend-team`
  - Write `scripts/split-repo/templates/frontend-README.md` with prerequisites (Node.js 22, pnpm 9), `pnpm install`, `pnpm dev` commands, and dapp/website port info
  - Write `scripts/split-repo/templates/backend-README.md` with prerequisites (Go 1.23+, Rust stable, Docker), and `make dev` / `docker compose up --build` commands
  - Copy all of the above into the respective `_build/` clones in Phase 3 and commit: `chore: add CODEOWNERS, dependabot, and README`
  - _Requirements: 5.1, 5.2, 8.3, 8.4, 9.1, 9.2, 9.3, 9.4_

- [ ] 8. Implement the Patch phase — staging deploy workflow env vars
  - In `sitel-tech-frontend/deploy-staging.yml`, ensure the `build-dapp` step sets `NEXT_PUBLIC_API_URL: ${{ vars.STAGING_API_URL }}` and `NEXT_PUBLIC_WS_URL: ${{ vars.STAGING_WS_URL }}`
  - Ensure the smoke test step in `sitel-tech-frontend/deploy-staging.yml` runs `tests/smoke.spec.ts` against `${{ vars.STAGING_URL }}` as the promotion gate
  - In `sitel-tech-backend/deploy-staging.yml`, ensure the workflow builds and deploys only the API and signer Docker images, with no pnpm/Node.js steps
  - _Requirements: 10.1, 10.2, 10.3, 10.4_

- [ ] 9. Implement the Publish phase — create GitHub org repos and push
  - In `split-repo.sh`, Phase 5 (Publish):
    - Create the backend repo in the `sitel-tech` org:
      ```bash
      gh repo create sitel-tech/sitel-tech-backend --private --description "Go API and Rust smart contracts"
      ```
    - Create the frontend repo in the `sitel-tech` org:
      ```bash
      gh repo create sitel-tech/sitel-tech-frontend --private --description "Next.js dapp and marketing website"
      ```
    - Set the `origin` remote on `_build/sitel-tech-backend/` to `git@github.com:sitel-tech/sitel-tech-backend.git`:
      ```bash
      git -C _build/sitel-tech-backend remote set-url origin git@github.com:sitel-tech/sitel-tech-backend.git
      ```
    - Set the `origin` remote on `_build/sitel-tech-frontend/` to `git@github.com:sitel-tech/sitel-tech-frontend.git`
    - Push both repos with full history mirror:
      ```bash
      git -C _build/sitel-tech-backend push --mirror origin
      git -C _build/sitel-tech-frontend push --mirror origin
      ```
    - Print `✓ Both repositories pushed to github.com/sitel-tech` on success
  - _Requirements: 1.1_

- [ ] 10. Generate the migration checklist artifact
  - In Phase 3 (Patch) or as a standalone step, write `scripts/split-repo/gen_checklist.py`:
    - Parse all workflow YAML files in both `_build/` clones with a regex for `\$\{\{\s*secrets\.([A-Z0-9_]+)\s*\}\}` and `vars\.([A-Z0-9_]+)`
    - Group findings by repo and print the migration checklist (matching the format defined in the design's Data Models section) to stdout and to `_build/migration-checklist.md`
    - Print `WARNING: Secret XYZ referenced in workflow but missing from checklist` for any secret not already documented
  - Invoke `gen_checklist.py` from `split-repo.sh` at the end of Phase 3
  - _Requirements: 6.1, 6.2, 6.3_

  - [ ]* 10.1 Write unit tests for `gen_checklist.py`
    - Test with a minimal workflow YAML containing two secret refs — assert both appear in the output grouped correctly
    - Test with a workflow referencing `vars.STAGING_URL` — assert it appears in the variables section
    - _Requirements: 6.1, 6.2_

- [ ] 11. Checkpoint — verify pre-publish state
  - Ensure all tests pass and ask the user if questions arise before continuing to the verification suite.

- [ ] 12. Implement the verification suite helper functions in `verify.py`
  - Write `parse_workflow_paths(yaml_content: str) -> list[str]` — extracts all path string literals from `on.push.paths`, `on.pull_request.paths`, and `working-directory` fields in a workflow YAML
  - Write `parse_workflow_secrets(yaml_content: str) -> set[str]` — extracts all `secrets.X` identifiers using regex `\$\{\{\s*secrets\.([A-Z0-9_]+)\s*\}\}`
  - Write `paths_are_disjoint(set_a: set[str], set_b: set[str]) -> bool` — returns `len(set_a & set_b) == 0`
  - Write `path_matches_pattern(path: str, patterns: list[str]) -> bool` — uses `fnmatch.fnmatch` for each pattern
  - Write `collect_file_paths(repo_dir: str) -> set[str]` — walks a directory tree and returns all relative file paths
  - _Requirements: 1.4, 2.3, 4.1, 5.3, 5.4, 5.5, 5.6, 6.1, 6.2, 7.1, 7.2_

  - [ ]* 12.1 Write unit tests for `parse_workflow_paths`
    - Test with a workflow YAML containing `paths: ['apps/api/**', 'packages/contracts/**']` — assert both paths returned
    - Test with a workflow with no path filters — assert empty list
    - _Requirements: 5.3, 5.4_

  - [ ]* 12.2 Write unit tests for `parse_workflow_secrets`
    - Test with `${{ secrets.STAGING_DEPLOY_TOKEN }}` — assert `{'STAGING_DEPLOY_TOKEN'}`
    - Test with multiple distinct secret refs — assert all are returned
    - Test with no secret refs — assert empty set
    - _Requirements: 6.1, 6.2_

  - [ ]* 12.3 Write unit tests for `paths_are_disjoint`
    - Disjoint sets → True; overlapping sets → False; empty sets → True
    - _Requirements: 1.4_

  - [ ]* 12.4 Write unit tests for `path_matches_pattern`
    - `apps/api/main.go` vs `apps/api/**` → True
    - `apps/dapp/frontend/page.tsx` vs `apps/api/**` → False
    - _Requirements: 5.3, 5.4_

- [ ] 13. Write property-based tests in `verify.py` using Hypothesis
  - [ ]* 13.1 Write property test for Property 1 — Repository File Disjointness
    - Use `@given(st.frozensets(st.text()), st.frozensets(st.text()))` to generate two path sets
    - Assert `paths_are_disjoint(a, b) == (len(a & b) == 0)`
    - **Property 1: Repository File Disjointness**
    - **Validates: Requirements 1.4**

  - [ ]* 13.2 Write property test for Property 3 — Frontend Workspace Path Validity
    - Generate random lists of path strings using `st.lists(st.text(min_size=1))`
    - For each path matching `apps/api/**` or `packages/contracts/**`, assert `path_matches_pattern` returns True against the backend patterns
    - Assert `path_matches_pattern` returns False against frontend patterns for those same paths
    - **Property 3: Frontend Workspace Path Validity**
    - **Validates: Requirements 4.1, 4.2, 7.1**

  - [ ]* 13.3 Write property test for Property 4 — Workflow Path Cross-Contamination Absence
    - Generate random YAML strings that include a mix of `apps/api/`, `apps/dapp/` paths
    - Assert that `parse_workflow_paths` extracts them correctly and that the cross-contamination classifier correctly identifies which belong to which repo
    - **Property 4: Workflow Path Cross-Contamination Absence**
    - **Validates: Requirements 5.3, 5.4, 5.5, 5.6**

  - [ ]* 13.4 Write property test for Property 5 — Secrets Checklist Completeness
    - Generate random workflow YAML strings with 1–10 `${{ secrets.X }}` refs using `st.from_regex(r'\$\{\{ secrets\.[A-Z0-9_]{3,20} \}\}')`
    - Assert `parse_workflow_secrets` returns a set whose length matches the number of distinct secret names in the input
    - **Property 5: Secrets Checklist Completeness**
    - **Validates: Requirements 6.1, 6.2**

  - [ ]* 13.5 Write property test for Property 6 — No Cross-Repository Workspace Dependencies
    - Generate random `package.json`-like dicts with `st.dictionaries` where values can include `workspace:*`
    - Assert the scanner correctly flags those whose keys match `@sitel-tech/contracts*` and does not flag others
    - **Property 6: No Cross-Repository Workspace Dependencies**
    - **Validates: Requirements 7.1, 7.2, 7.3**

- [ ] 14. Implement the Verify phase integration checks in `verify.py`
  - Write `verify_file_disjointness(backend_dir, frontend_dir)` — enumerates real paths in both `_build/` dirs (excluding `.git/`) and asserts intersection is empty; prints offending paths and raises on failure
  - Write `verify_git_history_smoke(repo_dir, sample_files)` — for each path in `sample_files`, runs `git log --oneline -- <path>` inside `repo_dir` and asserts at least one line of output
  - Write `verify_workspace_paths(frontend_dir)` — parses `pnpm-workspace.yaml` in `frontend_dir` and asserts each declared path (a) exists as a directory and (b) does not match `apps/api/**` or `packages/contracts/**`
  - Write `verify_workflow_cross_contamination(backend_dir, frontend_dir)` — uses `parse_workflow_paths` on all workflow YAMLs in each repo and asserts no cross-contamination per Property 4
  - Write `verify_secrets_checklist(backend_dir, frontend_dir, checklist_path)` — uses `parse_workflow_secrets` on all workflow YAMLs and asserts all found secrets appear in the checklist
  - Write `verify_no_cross_repo_workspace_deps(frontend_dir)` — finds all `package.json` files under `frontend_dir` and asserts no `workspace:*` reference to backend package names
  - Wire all checks into a `run_all_checks(backend_dir, frontend_dir, checklist_path)` function that runs them in order and exits non-zero on first failure
  - Invoke `python3 scripts/split-repo/verify.py _build/sitel-tech-backend _build/sitel-tech-frontend _build/migration-checklist.md` as Phase 4 (Verify) from `split-repo.sh`
  - _Requirements: 1.4, 2.1, 2.2, 2.3, 4.1, 4.2, 5.3, 5.4, 5.5, 5.6, 6.1, 6.2, 7.1, 7.2_

- [ ] 15. Add pnpm install dry-run and Go/Rust build smoke checks to the Verify phase
  - In `verify.py`, write `verify_pnpm_install_dryrun(frontend_dir)` — runs `pnpm install --frozen-lockfile --dry-run` in `frontend_dir` and asserts exit code 0
  - Write `verify_go_build(backend_dir)` — runs `go build ./...` in `backend_dir/apps/api/` and asserts exit code 0
  - Write `verify_cargo_check(backend_dir)` — runs `cargo check --target wasm32-unknown-unknown` in `backend_dir/packages/contracts/` and asserts exit code 0
  - Add these to `run_all_checks`
  - _Requirements: 3.4, 3.5, 3.6, 8.1, 8.2, 8.3, 8.4_

- [ ] 16. Checkpoint — final verification
  - Ensure all unit tests and property tests pass (`pytest scripts/split-repo/`). Ensure all pre-flight checks pass against a test invocation of the script. Ask the user if questions arise.

- [ ] 17. Wire everything together in `split-repo.sh` and do an end-to-end dry run
  - Ensure `split-repo.sh` calls all five phases in order: Pre-flight → Audit → Filter → Patch (workspace + workflows + dependabot + CODEOWNERS + READMEs + deploy config + checklist) → Verify → Publish
  - Add a `--dry-run` flag: when set, skip Phase 5 (Publish) and instead print the `git push --mirror` commands that would be run
  - Add a `--skip-verify` flag for emergency use only (print a loud warning when used)
  - Run `bash -n scripts/split-repo/split-repo.sh` to syntax-check the script
  - Verify the script is executable (`chmod +x`)
  - _Requirements: 1.1, 1.2, 1.3, 1.4, 2.1, 2.2, 2.3_

- [ ] 18. Final checkpoint — Ensure all tests pass
  - Run `pytest scripts/split-repo/ -v` and confirm all unit and property tests pass. Ask the user if questions arise before declaring the workflow complete.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster first pass; the core split script is fully functional without them.
- The project name is `sitel-tech`; npm scope is `@sitel-tech`; both output repos live in the `sitel-tech` GitHub organisation.
- Remote URLs are `git@github.com:sitel-tech/sitel-tech-backend.git` and `git@github.com:sitel-tech/sitel-tech-frontend.git`.
- `CODEOWNERS` files reference `@sitel-tech/backend-team` and `@sitel-tech/frontend-team` — not the original `@0xDeon` personal handle.
- The `pnpm --filter @sitel-tech/...` references in workflow files must all be updated to `@sitel-tech/...` during the Patch phase (Task 5 and 6).
- Property tests use `pytest-hypothesis`; minimum 100 iterations per property (the default Hypothesis `max_examples=100`).
- The split script never modifies the source monorepo in place — all work happens in `_build/` scratch directories.
- The `--dry-run` flag in Task 17 allows safe rehearsal of the full migration without pushing to GitHub.
