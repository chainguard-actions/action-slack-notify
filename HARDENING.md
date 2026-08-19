<!-- markdownlint-disable -->

# Hardening Report: rtCamp--action-slack-notify/v2.4.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **rtCamp--action-slack-notify/v2.4.0** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Direct ${{ }} expression interpolation inside run: shell commands. In build.yaml line 22, `${{ github.actor }}` is interpolated unquoted directly into a shell command (`docker login ghcr.io -u ${{ github.actor }} --password-stdin`), and `${{ secrets.GITHUB_TOKEN }}` is also interpolated directly. In line 25, `${{ github.repository }}` and `${{ github.ref_name }}` are interpolated directly inside a shell string passed to `echo` and `docker buildx build`. Any of these expressions are substituted by the YAML template engine before the shell ever sees them, enabling command injection if the values contain shell metacharacters.

Locations:

- `.github/workflows/build.yaml:22`
- `.github/workflows/build.yaml:25`

### unpinned-uses (severity: high)

Multiple `uses:` references are pinned to mutable tags rather than full 40-character commit SHAs. In build.yaml: `actions/checkout@v6` (line 15) and `docker/setup-buildx-action@v4` (line 19). In release.yaml: `softprops/action-gh-release@v3` (line 16). These can be silently updated by the upstream maintainer, enabling supply-chain attacks.

Locations:

- `.github/workflows/build.yaml:15`
- `.github/workflows/build.yaml:19`
- `.github/workflows/release.yaml:16`

### github-env-injection (severity: high)

In release.yaml line 14, the run: block writes the inherited environment variable `GITHUB_REF_NAME` directly to `$GITHUB_ENV` without sanitization: `echo "release_name=${GITHUB_REF_NAME/v/Version }" >> $GITHUB_ENV`. `GITHUB_REF_NAME` is a workflow-controlled value (the tag/branch name) that could contain newline characters, enabling an attacker to inject arbitrary environment variable entries. The required sanitization step (`printf '%s' "$GITHUB_REF_NAME" | tr -d '\n\r'`) is absent.

Locations:

- `.github/workflows/release.yaml:14`

### missing-permissions (severity: medium)

release.yaml has no top-level `permissions:` key and its only job (`release`) also has no job-level `permissions:` key. Without explicit permissions, the workflow inherits the repository's default token permissions, which may be overly broad (e.g., write access to contents). A minimal explicit permissions block (e.g., `permissions: contents: write`) should be added.

Locations:

- `.github/workflows/release.yaml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, unpinned-uses, github-env-injection, missing-permissions

**Notes:**

build.yaml: Pinned actions/checkout@v6 to SHA d23441a48e516b6c34aea4fa41551a30e30af803 and docker/setup-buildx-action@v4 to SHA bb05f3f5519dd87d3ba754cc423b652a5edd6d2c. Moved github.actor, secrets.GITHUB_TOKEN, github.repository, and github.ref_name out of run: shell strings into step env: blocks to prevent script injection. release.yaml: Added top-level permissions: contents: write. Fixed github-env-injection by sanitizing GITHUB_REF_NAME with printf/tr before writing to GITHUB_ENV. Pinned softprops/action-gh-release@v3 to SHA 3d0d9888cb7fd7b750713d6e236d1fcb99157228. Also updated the release step to reference env.release_name instead of the now-removed steps output reference.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed the unquoted command substitution in .github/workflows/build.yaml line 27. The `-t $(...)` argument was changed to `-t "$(...)"` so that any shell metacharacters in the output of the command substitution (which includes workflow-controllable values `$GITHUB_REPOSITORY` and `$GITHUB_REF_NAME`) are not interpreted by the shell. The values were already correctly placed in the `env:` block; only the outer quoting of the command substitution was missing.

