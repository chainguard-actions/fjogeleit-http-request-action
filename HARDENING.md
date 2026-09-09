<!-- markdownlint-disable -->

# Hardening Report: fjogeleit--http-request-action/v1.15.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **fjogeleit--http-request-action/v1.15.2** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple `uses:` references are pinned to mutable version tags instead of immutable 40-character commit SHAs, making the workflow vulnerable to supply-chain attacks if the tag is moved.

In `.github/workflows/build-action.yml`:
- `uses: actions/checkout@v4` (line 17)
- `uses: actions/setup-node@v4` (line 20)

In `.github/workflows/ci.yml`:
- `uses: actions/checkout@v4` (line 29, build job)
- `uses: actions/setup-node@v4` (line 32, build job)
- `uses: actions/checkout@v4` (line 59, integrity job)
- `uses: actions/setup-node@v4` (line 62, integrity job)
- `uses: actions/checkout@v4` (line 80, test job)

All should be pinned to full SHA digests, e.g. `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4`.

Locations:

- `.github/workflows/build-action.yml:17`
- `.github/workflows/build-action.yml:20`
- `.github/workflows/ci.yml:29`
- `.github/workflows/ci.yml:32`
- `.github/workflows/ci.yml:59`
- `.github/workflows/ci.yml:62`
- `.github/workflows/ci.yml:80`

### script-injection (severity: high)

GitHub Actions expressions (`${{ ... }}`) are interpolated directly inside `run:` shell command strings, allowing an attacker to inject arbitrary shell commands.

**`.github/workflows/ci.yml` — `build` job, `ref` step (lines 21–26):**
- Sub-rule (a): `if [[ -n '${{ github.event.ref }}' ]]; then` — `github.event.ref` is interpolated directly into the shell condition.
- Sub-rule (a): `branch="${{ github.event.ref }}"` — attacker-controlled ref value injected into shell variable assignment.
- Sub-rule (a): `branch="${{ github.event.pull_request.head.ref }}"` — PR head ref (attacker-controlled on pull_request events) injected directly.

**`.github/workflows/ci.yml` — `build` job, `Update dist` step (line 48):**
- Sub-rule (a): `git push origin HEAD:${{ steps.ref.outputs.branch }}` — step output (derived from attacker-controlled PR ref) interpolated directly into shell command.

**`.github/workflows/ci.yml` — `test` job, `Output responseFile` step (line ~142):**
- Sub-rule (a): `cat "${{ github.workspace }}/response.json"` — `github.workspace` interpolated directly into a shell command.

**`.github/workflows/build-action.yml` — `build` job, `Update dist` step (line 35):**
- Sub-rule (a): `git push origin HEAD:${{ inputs.ref }}` — `workflow_dispatch` input (user-controlled) interpolated directly into shell command.

Locations:

- `.github/workflows/ci.yml:21`
- `.github/workflows/ci.yml:22`
- `.github/workflows/ci.yml:25`
- `.github/workflows/ci.yml:48`
- `.github/workflows/ci.yml:142`
- `.github/workflows/build-action.yml:35`

### github-env-injection (severity: high)

Untrusted GitHub context values are written to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`), enabling newline injection that could allow an attacker to set arbitrary output variables.

In `.github/workflows/ci.yml`, the `build` job's first step (`id: ref`) assigns attacker-controlled values to a shell variable and then writes them directly to `$GITHUB_OUTPUT`:

```bash
branch="${{ github.event.ref }}"                          # line 22 — attacker-controlled
echo "branch=$branch" >> $GITHUB_OUTPUT                   # line 23 — FAIL: no sanitization
```

and:

```bash
branch="${{ github.event.pull_request.head.ref }}"        # line 25 — attacker-controlled PR head ref
echo "branch=$branch" >> $GITHUB_OUTPUT                   # line 26 — FAIL: no sanitization
```

`github.event.pull_request.head.ref` is fully attacker-controlled on `pull_request` events. A branch name containing a newline (e.g. `main\nFOO=injected`) would inject an extra key into `$GITHUB_OUTPUT`.

Locations:

- `.github/workflows/ci.yml:23`
- `.github/workflows/ci.yml:26`

### missing-permissions (severity: medium)

`.github/workflows/ci.yml` has no top-level `permissions:` key, and two of its three jobs (`integrity` and `test`) also have no job-level `permissions:` key. This means those jobs run with the default token permissions, which may be broader than necessary (e.g. `write` on public repositories or as configured by the organization). Every job should declare the minimal permissions it requires.

The `build` job does declare `permissions: contents: write`, but `integrity` and `test` do not declare any permissions.

Locations:

- `.github/workflows/ci.yml:1`

### hardcoded-credentials (severity: high)

A literal hardcoded password value is present in `.github/workflows/ci.yml`. In the `test` job's "Request Postman Echo BasicAuth" step, the `password` input is set to the literal string `'password'`:

```yaml
- name: Request Postman Echo BasicAuth
  uses: ./
  with:
    url: 'https://postman-echo.com/basic-auth'
    method: 'GET'
    username: 'postman'
    password: 'password'   # hardcoded literal credential
```

Even if this is a well-known public test endpoint, hardcoding credentials in workflow files is a bad practice and violates the check. Credentials should be stored in GitHub Secrets and referenced via `${{ secrets.POSTMAN_PASSWORD }}`.

Locations:

- `.github/workflows/ci.yml:115`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection, missing-permissions, hardcoded-credentials

**Notes:**

Fixed all 5 findings across .github/workflows/build-action.yml and .github/workflows/ci.yml:

1. unpinned-uses: Pinned actions/checkout@v4 to SHA 11d5960a326750d5838078e36cf38b85af677262 and actions/setup-node@v4 to SHA 49933ea5288caeca8642d1e84afbd3f7d6820020 in both workflow files (7 occurrences total).

2. script-injection: Moved all ${{ }} expressions out of run: shell strings into env: blocks:
   - ci.yml ref step: github.event.ref and github.event.pull_request.head.ref moved to env: as EVENT_REF and PR_HEAD_REF
   - ci.yml Update dist step: steps.ref.outputs.branch moved to env: as BRANCH_REF
   - ci.yml Output responseFile step: github.workspace moved to env: as WORKSPACE
   - build-action.yml Update dist step: inputs.ref moved to env: as INPUT_REF

3. github-env-injection: In ci.yml ref step, sanitized both branch values with 'printf \'%s\' "$VAR" | tr -d \'\n\r\'' before writing to $GITHUB_OUTPUT.

4. missing-permissions: Added 'permissions: contents: read' to both the integrity and test jobs in ci.yml.

5. hardcoded-credentials: Replaced hardcoded password: 'password' with password: ${{ secrets.POSTMAN_PASSWORD }} in the BasicAuth test step.

