<!-- markdownlint-disable -->

# Hardening Report: fjogeleit--http-request-action/v1.16.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **fjogeleit--http-request-action/v1.16.2** was hardened automatically. 4 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Direct ${{ }} expression interpolation inside run: shell commands. In build-action.yml, `${{ inputs.ref }}` is interpolated directly into a git push command: `git push origin HEAD:${{ inputs.ref }}`. In ci.yml, `${{ github.event.ref }}` and `${{ github.event.pull_request.head.ref }}` are interpolated directly into shell conditionals and variable assignments, and `${{ steps.ref.outputs.branch }}` is interpolated into a git push command. These expressions are expanded by the template engine before the shell sees them, allowing an attacker to inject arbitrary shell commands via a crafted branch name or event payload. Sub-rule (a) violated in all cases.

Locations:

- `.github/workflows/build-action.yml:30`
- `.github/workflows/ci.yml:17`
- `.github/workflows/ci.yml:18`
- `.github/workflows/ci.yml:21`
- `.github/workflows/ci.yml:35`

### github-env-injection (severity: high)

Untrusted GitHub context values are written to $GITHUB_OUTPUT without sanitization. In ci.yml, `${{ github.event.ref }}` and `${{ github.event.pull_request.head.ref }}` are interpolated into shell variables (`branch=...`) which are then written to $GITHUB_OUTPUT via `echo "branch=$branch" >> $GITHUB_OUTPUT`. No `printf '%s' ... | tr -d '\n\r'` sanitization is applied before the write, allowing newline injection into the output file.

Locations:

- `.github/workflows/ci.yml:19`
- `.github/workflows/ci.yml:22`

### unpinned-uses (severity: high)

Multiple `uses:` references are pinned to mutable version tags instead of immutable 40-character commit SHAs, making the workflows vulnerable to supply-chain attacks if the tag is moved. Failing references in build-action.yml: `actions/checkout@v4`, `actions/setup-node@v4`. Failing references in ci.yml: `actions/checkout@v4` (twice), `actions/setup-node@v4` (twice).

Locations:

- `.github/workflows/build-action.yml:15`
- `.github/workflows/build-action.yml:17`
- `.github/workflows/ci.yml:23`
- `.github/workflows/ci.yml:26`
- `.github/workflows/ci.yml:47`
- `.github/workflows/ci.yml:50`

### missing-permissions (severity: medium)

Workflow files are missing `permissions:` blocks on one or more jobs, meaning they run with the default (potentially broad) GITHUB_TOKEN permissions. In ci.yml, the `integrity` job has no `permissions:` key (only the `build` job does). In it-tests.yml, there is no top-level `permissions:` key and none of the individual jobs (test-method-get-on-existing-url, test-method-post-on-existing-url, etc., and it-tests) define their own `permissions:` blocks.

Locations:

- `.github/workflows/ci.yml:40`
- `.github/workflows/it-tests.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all four findings across three workflow files:

1. build-action.yml: Pinned actions/checkout and actions/setup-node to full commit SHAs (11d5960a... and 49933ea5...). Fixed script injection in git push by moving inputs.ref to an env var INPUT_REF.

2. ci.yml: Pinned all four action references to full SHAs. Fixed script injection by moving github.event.ref and github.event.pull_request.head.ref to env vars (EVENT_REF, PR_HEAD_REF) and steps.ref.outputs.branch to env var PUSH_REF. Fixed github-env-injection by sanitizing values with printf '%s' | tr -d '\n\r' before writing to GITHUB_OUTPUT. Added permissions: contents: read to the integrity job.

3. it-tests.yml: Added top-level permissions: contents: read block to restrict default GITHUB_TOKEN permissions.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed the 'Output responseFile' step in .github/workflows/it-tests.yml. Moved ${{ github.workspace }} out of the run: block into an env: variable (WORKSPACE) to eliminate script injection risk. Added sanitization using printf '%s' ... | tr -d '\n\r' before writing to $GITHUB_OUTPUT to prevent newline injection attacks. Also quoted $GITHUB_OUTPUT reference for best practice.

### Iteration 1

**Fixes applied:** hardcoded-credentials

**Notes:**

Replaced the hardcoded literal password value 'password' in the test-basic-auth job (line 76 of .github/workflows/it-tests.yml) with a GitHub Actions secret reference: `${{ secrets.POSTMAN_PASSWORD }}`. The username 'postman' was left as-is since it is not a secret credential. The repository owner will need to add a POSTMAN_PASSWORD secret to the repository/organization settings with the value 'password' for the integration test to continue working.

