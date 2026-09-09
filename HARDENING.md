<!-- markdownlint-disable -->

# Hardening Report: fjogeleit--http-request-action/v1.14.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **fjogeleit--http-request-action/v1.14.1** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Both workflow files use tag-based (non-SHA-pinned) `uses:` references, making them vulnerable to supply-chain attacks if the referenced tags are moved. Failing references: `actions/checkout@v3` and `actions/setup-node@v3` appear across both files (7 total occurrences).

Locations:

- `.github/workflows/build-action.yml:18`
- `.github/workflows/build-action.yml:20`
- `.github/workflows/ci.yml:30`
- `.github/workflows/ci.yml:32`
- `.github/workflows/ci.yml:56`
- `.github/workflows/ci.yml:58`
- `.github/workflows/ci.yml:65`

### script-injection (severity: high)

Sub-rule (a): GitHub Actions expressions are directly interpolated inside `run:` shell command strings, allowing an attacker to inject arbitrary shell commands.

1. `.github/workflows/build-action.yml` line 35: `git push origin HEAD:${{ inputs.ref }}` — the `inputs.ref` value is injected directly into a shell command without quoting or env-var indirection.

2. `.github/workflows/ci.yml` line 22: `branch="${{ github.event.ref }}"` — attacker-controlled `github.event.ref` is interpolated directly into the shell.

3. `.github/workflows/ci.yml` line 24: `branch="${{ github.event.pull_request.head.ref }}"` — attacker-controlled PR head ref is interpolated directly into the shell.

4. `.github/workflows/ci.yml` line 47: `git push origin HEAD:${{ steps.ref.outputs.branch }}` — step output (derived from attacker-controlled input) is interpolated directly into a shell command.

Locations:

- `.github/workflows/build-action.yml:35`
- `.github/workflows/ci.yml:22`
- `.github/workflows/ci.yml:24`
- `.github/workflows/ci.yml:47`

### github-env-injection (severity: high)

In `.github/workflows/ci.yml`, the `ref` step (around line 20) interpolates `${{ github.event.ref }}` and `${{ github.event.pull_request.head.ref }}` into shell variables (`branch`) and then writes those variables to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). An attacker controlling the PR head ref or event ref can inject newlines to poison GITHUB_OUTPUT and set arbitrary outputs for downstream steps.

Offending lines:
- `branch="${{ github.event.ref }}"` followed by `echo "branch=$branch" >> $GITHUB_OUTPUT`
- `branch="${{ github.event.pull_request.head.ref }}"` followed by `echo "branch=$branch" >> $GITHUB_OUTPUT`

Locations:

- `.github/workflows/ci.yml:22`
- `.github/workflows/ci.yml:24`

### missing-permissions (severity: medium)

`.github/workflows/ci.yml` has no top-level `permissions:` key, and the `integrity` and `test` jobs have no job-level `permissions:` key either. Only the `build` job defines permissions. The `integrity` and `test` jobs run with the default (broad) token permissions, violating the principle of least privilege.

Locations:

- `.github/workflows/ci.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection, missing-permissions

**Notes:**

Fixed all 4 findings across both workflow files:

1. **unpinned-uses**: Pinned all 7 occurrences of actions/checkout@v3 → @a37ce9120846195fa4ece8f58b268e6043cb2f26 and actions/setup-node@v3 → @3235b876344d2a9aa001b8d1453c930bba69e610 in both build-action.yml and ci.yml.

2. **script-injection**: Moved all ${{ }} expressions out of run: shell strings into env: blocks: inputs.ref → INPUT_REF in build-action.yml; github.event.ref → EVENT_REF and github.event.pull_request.head.ref → PR_HEAD_REF in ci.yml ref step; steps.ref.outputs.branch → BRANCH in ci.yml Update dist step. Also moved github.workspace in the 'Output responseFile' step to a WORKSPACE env var.

3. **github-env-injection**: The ref step in ci.yml now sanitizes branch values with `printf '%s' "$VAR" | tr -d '\n\r'` before writing to $GITHUB_OUTPUT.

4. **missing-permissions**: Added `permissions: contents: read` to the integrity and test jobs in ci.yml (build job already had contents: write).

