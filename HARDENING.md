<!-- markdownlint-disable -->

# Hardening Report: fjogeleit--http-request-action/v1.15.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **fjogeleit--http-request-action/v1.15.1** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple run: blocks directly interpolate ${{ }} expressions, allowing script injection before the shell ever sees the value. In ci.yml, the 'ref' step uses '${{ github.event.ref }}' and '${{ github.event.pull_request.head.ref }}' directly inside shell commands (lines 21, 22, 25). The 'Update dist' step uses '${{ steps.ref.outputs.branch }}' directly in a git push command (line 48). The 'Output responseFile' step uses '${{ github.workspace }}' directly in a cat command. In build-action.yml, the 'Update dist' step uses '${{ inputs.ref }}' directly in a git push command (line 33). All of these violate sub-rule (a): any ${{ ... }} directly inside a run: script is a script-injection finding.

Locations:

- `.github/workflows/ci.yml:21`
- `.github/workflows/ci.yml:22`
- `.github/workflows/ci.yml:25`
- `.github/workflows/ci.yml:48`
- `.github/workflows/ci.yml:100`
- `.github/workflows/build-action.yml:33`

### github-env-injection (severity: high)

In ci.yml, the 'ref' step sets shell variable 'branch' from ${{ github.event.ref }} (line 22) and ${{ github.event.pull_request.head.ref }} (line 25), then writes it to $GITHUB_OUTPUT via 'echo "branch=$branch" >> $GITHUB_OUTPUT' (lines 23 and 26) without the required sanitization step (printf '%s' "$branch" | tr -d '\n\r'). An attacker-controlled ref value containing newlines could inject arbitrary key=value pairs into the GitHub output context.

Locations:

- `.github/workflows/ci.yml:23`
- `.github/workflows/ci.yml:26`

### unpinned-uses (severity: high)

Multiple uses: references are pinned to mutable version tags instead of immutable 40-character SHA commit hashes, making the workflows vulnerable to supply-chain attacks if the tag is moved. Failing references in build-action.yml: 'actions/checkout@v4' (line 17), 'actions/setup-node@v4' (line 19). Failing references in ci.yml: 'actions/checkout@v4' (lines 29, 57, 74), 'actions/setup-node@v4' (lines 32, 60). All should be pinned to full SHA digests, e.g. actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.

Locations:

- `.github/workflows/build-action.yml:17`
- `.github/workflows/build-action.yml:19`
- `.github/workflows/ci.yml:29`
- `.github/workflows/ci.yml:32`
- `.github/workflows/ci.yml:57`
- `.github/workflows/ci.yml:60`
- `.github/workflows/ci.yml:74`

### missing-permissions (severity: medium)

ci.yml has no top-level permissions: block, and the 'integrity' and 'test' jobs have no job-level permissions: block either. Only the 'build' job defines permissions (contents: write). The 'integrity' and 'test' jobs therefore inherit the default broad repository permissions, violating the principle of least privilege. Every job should declare explicit minimal permissions.

Locations:

- `.github/workflows/ci.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all four findings across ci.yml and build-action.yml:

1. script-injection: Moved all ${{ }} expressions out of run: blocks into env: blocks. Affected steps: 'ref' step (EVENT_REF, PR_HEAD_REF), 'Update dist' in ci.yml (BRANCH_REF), 'Output responseFile' (WORKSPACE), and 'Update dist' in build-action.yml (INPUT_REF).

2. github-env-injection: In the 'ref' step, both branches now sanitize values with `printf '%s' "$VAR" | tr -d '\n\r'` before writing to $GITHUB_OUTPUT.

3. unpinned-uses: Pinned actions/checkout@v4 to SHA 11d5960a326750d5838078e36cf38b85af677262 and actions/setup-node@v4 to SHA 49933ea5288caeca8642d1e84afbd3f7d6820020 in all occurrences across both workflow files.

4. missing-permissions: Added top-level `permissions: {}` to ci.yml, and job-level `permissions: { contents: read }` to both the 'integrity' and 'test' jobs.

