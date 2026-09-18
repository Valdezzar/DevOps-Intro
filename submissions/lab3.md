# Lab 3 submission

Path: GitHub Actions.

## Task 1

I used GitHub Actions because the repository and previous labs are already on GitHub.

The pipeline has three independent units: vet, test, and lint. The final version also has a `ci-ok` aggregation job.

### CI gate

The workflow runs:

- `go vet ./...`
- `go test -race -count=1 ./...`
- `golangci-lint v2.5.0`

The runner is pinned to `ubuntu-24.04`. Actions are pinned to commit SHAs. Workflow permissions are limited to `contents: read`.

### Failed gate test

I deliberately changed a test so that the test job failed. GitHub showed one failed required check and did not allow the PR to merge.

I then reverted the breaking commit and the CI pipeline became green again.

### Design questions

**a) Why pin the runner version?**

A pinned runner gives a more stable environment. `ubuntu-latest` can move to a newer Ubuntu version and change installed tools or system libraries without any change in the repository.

**b) Why separate vet, test, and lint?**

Separate jobs run in parallel and make failures easier to understand. With one combined job, later commands may never run after the first failure.

**c) Why pin actions by SHA?**

SHA pinning prevents a changed or compromised tag from silently replacing the code used by the workflow. A relevant example is the tj-actions/changed-files supply-chain compromise in March 2025.

**d) What is permissions?**

`permissions` controls what the GitHub Actions token is allowed to access. I use `contents: read` following the principle of least privilege.

## Task 2

I enabled the Go cache, added a Go version matrix, and added path filters.

The vet and test jobs run on Go 1.23 and Go 1.24 with `fail-fast: false`.

The workflow only runs when files in `app/` or `.github/workflows/ci.yml` change.

### Timing

| Scenario | Wall-clock |
|---|---:|
| Baseline | 34 s |
| With cache | 31 s |
| With cache + matrix | 44 s |

The cache only produced a small difference because QuickNotes has no third-party Go dependencies and no `go.sum`. Most of the time is runner startup, checkout, and Go setup.

### Design questions

**f) Why cache inputs instead of build outputs?**

Dependency inputs are deterministic and tied to module files. Build outputs can depend on the OS, Go version, architecture, build flags, and other environment details.

**g) What does fail-fast false do?**

It allows all matrix jobs to finish even if one fails. This makes it possible to see exactly which Go versions work. `fail-fast: true` is useful when later matrix results do not matter after the first failure.

**h) What is the cache poisoning risk?**

A malicious workflow could try to place modified data into a cache and make a trusted workflow restore it later. Cache isolation and restrictions on cache writes reduce this risk, but CI caches should still not be treated as trusted executable content.

## Evidence

- Baseline CI run: 34 s
- Cached CI run: 31 s
- Failed test was blocked by required status checks
- Fixed test passed all required status checks
- Branch protection was enabled on `main`