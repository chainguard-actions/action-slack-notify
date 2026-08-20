<!-- markdownlint-disable -->

# Hardening Report: rtCamp--action-slack-notify/v2.3.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **rtCamp--action-slack-notify/v2.3.3** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple unpinned action/image references found using mutable tags instead of full SHA digests:
- action.yml: `uses: "docker://ghcr.io/rtcamp/action-slack-notify:v2.3.3"` (tag `v2.3.3`, not a SHA digest) — appears in both the 'Slack Notification (Formatted)' and 'Slack Notification (Unformatted)' steps.
- .github/workflows/build.yaml: `uses: actions/checkout@v4` (tag `v4`) and `uses: docker/setup-buildx-action@v3` (tag `v3`).
- .github/workflows/release.yaml: `uses: softprops/action-gh-release@v2` (tag `v2`).
All of these should be pinned to a full 40-character commit SHA.

Locations:

- `action.yml:44`
- `action.yml:49`
- `.github/workflows/build.yaml:16`
- `.github/workflows/build.yaml:20`
- `.github/workflows/release.yaml:16`

### script-injection (severity: high)

Sub-rule (a): GitHub Actions expressions are interpolated directly inside `run:` shell command strings in .github/workflows/build.yaml, allowing an attacker to inject shell metacharacters:
- Line 22: `echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin` — `${{ github.actor }}` is unquoted and directly interpolated into the shell command.
- Line 25: `docker buildx build ... -t $(echo 'ghcr.io/${{ github.repository }}:${{ github.ref_name }}' | tr ...)` — both `${{ github.repository }}` and `${{ github.ref_name }}` are interpolated directly inside a `run:` block. Any of these values could contain shell metacharacters if supplied by an attacker.

Locations:

- `.github/workflows/build.yaml:22`
- `.github/workflows/build.yaml:25`

### github-env-injection (severity: high)

In .github/workflows/release.yaml, the `Get Release Name` step writes the inherited process environment variable `GITHUB_REF_NAME` to `$GITHUB_ENV` without sanitization:
  `echo "release_name=${GITHUB_REF_NAME/v/Version }" >> $GITHUB_ENV`
`GITHUB_REF_NAME` is a workflow-controlled value (the tag/branch name that triggered the workflow). A tag name containing newline characters could inject additional key=value pairs into the GitHub environment, leading to environment variable injection. The required sanitization step (`printf '%s' "$GITHUB_REF_NAME" | tr -d '\n\r'`) is absent.

Locations:

- `.github/workflows/release.yaml:14`

### missing-permissions (severity: medium)

.github/workflows/release.yaml has no top-level `permissions:` key and no job-level `permissions:` key on any job. Without explicit permissions, the workflow inherits the repository's default token permissions, which may be overly broad. A `permissions:` block with minimal required scopes (e.g., `contents: write` for creating releases) should be added.

Locations:

- `.github/workflows/release.yaml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection, missing-permissions

**Notes:**

Fixed all four findings:
1. unpinned-uses: Pinned all action/image references to full SHA digests — docker container image in action.yml (both steps) to sha256 digest, actions/checkout and docker/setup-buildx-action in build.yaml, and softprops/action-gh-release in release.yaml.
2. script-injection: In build.yaml, moved github.actor, secrets.GITHUB_TOKEN, github.repository, and github.ref_name expressions out of run: blocks into env: blocks, referencing them as plain shell variables.
3. github-env-injection: In release.yaml, sanitized GITHUB_REF_NAME with `printf '%s' "$GITHUB_REF_NAME" | tr -d '\n\r'` before writing to $GITHUB_ENV, and quoted $GITHUB_ENV.
4. missing-permissions: Added `permissions: contents: write` to release.yaml (minimum needed for creating GitHub releases).

