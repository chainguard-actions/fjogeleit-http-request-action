<!-- markdownlint-disable -->

# Hardening Report: fjogeleit--http-request-action/v2.0.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **fjogeleit--http-request-action/v2.0.1** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference GitHub Actions by mutable tag/version strings instead of full 40-character commit SHAs, making them vulnerable to supply-chain attacks if the tag is moved. Failing references: build-action.yml — `actions/checkout@v7.0.0` (line 17), `actions/setup-node@v6` (line 20); ci.yml — `actions/checkout@v7.0.0` (line 24), `actions/setup-node@v6` (line 27); release.yml — `actions/checkout@v7.0.0` (line 14).

Locations:

- `.github/workflows/build-action.yml:17`
- `.github/workflows/build-action.yml:20`
- `.github/workflows/ci.yml:24`
- `.github/workflows/ci.yml:27`
- `.github/workflows/release.yml:14`

### script-injection (severity: high)

GitHub Actions expressions are interpolated directly inside `run:` shell command strings, violating sub-rule (a). (1) build-action.yml: `git push origin HEAD:${{ inputs.ref }}` — the `inputs.ref` value is attacker-controlled via workflow_dispatch and is injected directly into a shell command without quoting or env-var indirection. (2) ci.yml: `branch="${{ github.event.ref || github.event.pull_request.head.ref }}"` — attacker-controlled PR head ref is interpolated directly into the shell script. (3) ci.yml: `git push origin HEAD:${{ steps.ref.outputs.branch }}` — a step output (itself derived from the attacker-controlled ref) is interpolated directly into a shell command.

Locations:

- `.github/workflows/build-action.yml:35`
- `.github/workflows/ci.yml:19`
- `.github/workflows/ci.yml:34`

### github-env-injection (severity: high)

In ci.yml, the attacker-controlled value `${{ github.event.ref || github.event.pull_request.head.ref }}` is interpolated into the shell variable `branch`, which is then written to `$GITHUB_OUTPUT` via `echo "branch=$branch" >> $GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). A malicious PR head ref containing newlines could inject arbitrary key=value pairs into the GitHub output environment.

Locations:

- `.github/workflows/ci.yml:19`
- `.github/workflows/ci.yml:20`

### missing-permissions (severity: medium)

The workflow file it-tests.yml has no top-level `permissions:` key and none of its jobs define a `permissions:` block. Without explicit permissions, the workflow inherits the repository's default token permissions, which may be overly broad (e.g., write access to contents). Every job should declare minimal required permissions.

Locations:

- `.github/workflows/it-tests.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection, missing-permissions

**Notes:**

Fixed all four findings: (1) Pinned actions/checkout@v7.0.0 to SHA 9c091bb21b7c1c1d1991bb908d89e4e9dddfe3e0 and actions/setup-node@v6 to SHA 249970729cb0ef3589644e2896645e5dc5ba9c38 in build-action.yml, ci.yml, and release.yml. (2) Fixed script injection in build-action.yml by moving inputs.ref into an env var INPUT_REF; in ci.yml by moving github.event.ref and github.event.pull_request.head.ref into env vars EVENT_REF/PR_HEAD_REF, and steps.ref.outputs.branch into env var BRANCH. (3) Fixed github-env-injection in ci.yml by sanitizing the branch value with printf/tr before writing to GITHUB_OUTPUT. (4) Added top-level 'permissions: {}' to it-tests.yml to enforce least-privilege.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection, hardcoded-credentials

**Notes:**

Fixed all three findings in hardened/action/.github/workflows/it-tests.yml: (1) script-injection at line 163 — moved `${{ github.workspace }}` into an env: variable RESPONSE_FILE and referenced it as $RESPONSE_FILE in the shell; (2) github-env-injection at line 163 — added `printf '%s' ... | tr -d '\n\r'` sanitization before writing the HTTP response content to $GITHUB_OUTPUT; (3) hardcoded-credentials at line 86 — replaced the literal `password: 'password'` with `password: ${{ secrets.POSTMAN_ECHO_PASSWORD }}` to store the credential in GitHub Secrets.

