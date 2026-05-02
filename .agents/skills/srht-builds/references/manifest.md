# Build Manifest Field Reference

Full YAML reference for builds.sr.ht manifests.

## Required fields

### `image` (string)
Which OS image to build in. Use stable aliases (e.g. `alpine/latest`) rather than versioned
names so manifests don't need updating when images rotate.

### `tasks` (list of dict or string)
Scripts to execute sequentially. Each runs in a fresh login shell with `set -xe` pre-applied.

```yaml
tasks:
  - setup: |
      sudo apk add git
  - test: |
      cd myproject
      go test ./...
```

Task name rules: lowercase alphanumeric, underscores, dashes only; ≤128 characters.

## Optional fields

### `arch` (string)
Target architecture. Default is the image's native arch (usually x86_64/amd64).
Non-native builds run under emulation and may be slow.

### `packages` (list of string)
Packages to install before tasks run. Use the package manager of the chosen image.

```yaml
packages:
  - go
  - make
  - sqlite-dev
```

### `repositories` (dict: string → string)
Extra package repositories. Format varies by distro — see compatibility.md.

### `sources` (list of string)
Repos to clone into the build user's home directory before tasks run.
- Default protocol is git; prefix with `hg+` for Mercurial
- Append `#branch-or-commit` to pin a revision
- For git.sr.ht-triggered builds, the pushed repo is injected automatically

```yaml
sources:
  - https://git.sr.ht/~user/myproject
  - https://git.sr.ht/~user/mylib#v2.1
  - hg+https://hg.sr.ht/~user/oldrepo
```

### `artifacts` (list of string)
Files to extract from a successful build and make downloadable (retained 90 days).
- Literal paths only — no `~` expansion, no globs
- Relative paths are from the build user's home directory (`~/`)
- Only uploaded when the build succeeds

```yaml
artifacts:
  - myproject/mybinary
  - myproject/dist/app.tar.gz
```

### `environment` (dict: string → string or list)
Key-value pairs exported to `~/.buildenv` and available in all tasks.

**WARNING**: With `set -xe`, values are printed to build logs. Never put secrets here.
Use the `secrets:` field for sensitive values.

```yaml
environment:
  GO111MODULE: "on"
  GOOS: linux
```

### `secrets` (list of string)
UUIDs of secrets from builds.sr.ht/secrets injected into the build VM.
SSH keys are installed to `~/.ssh/`; other file-type secrets appear in `~/.secrets/`.

```yaml
secrets:
  - xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

### `shell` (boolean)
Keep the VM alive after all tasks finish (even on success) so you can SSH in.
Useful for debugging. Default: false.

```yaml
shell: true
```

### `oauth` (string)
Generates a temporary OAuth 2.0 bearer token with the specified grants.
Used with the `hut` CLI for authenticated API calls within a build.

```yaml
oauth: jobs.sr.ht/PROFILE:RO
```

### `triggers` (list of trigger)
Post-build actions. Each trigger has `action:` and `condition:`.

**Conditions**: `always`, `success`, `failure`

**Email trigger:**
```yaml
triggers:
  - action: email
    condition: failure
    to: you@example.org
    cc: team@example.org          # optional
    in_reply_to: <msg-id>         # optional, for threading
```

**Webhook trigger** (HTTP POST with JSON payload of job metadata):
```yaml
triggers:
  - action: webhook
    condition: always
    url: https://example.org/hook
```

### `submitter` (dict)
Controls which git.sr.ht/hg.sr.ht pushes trigger a build.

```yaml
submitter:
  git.sr.ht:
    enabled: true
    allow-refs:
      - refs/heads/main
      - "refs/tags/*"
```

Set `enabled: false` to stop automatic builds from a specific integration.
For hub.sr.ht patch testing, use `hub.sr.ht` as the key.

Skip a single push without editing the manifest: `git push -o skip-ci`

## Environment variables injected by integrations

| Variable | Set by | Value |
|---|---|---|
| `JOB_ID` | always | Numeric build job ID |
| `JOB_URL` | always | URL to build logs |
| `BUILD_SUBMITTER` | integrations | `git.sr.ht`, `hub.sr.ht`, etc. |
| `GIT_REF` | git.sr.ht | e.g. `refs/heads/main` |
| `BUILD_REASON` | hub.sr.ht | `patchset` |
| `PATCHSET_ID` | hub.sr.ht | Numeric patchset ID |
| `PATCHSET_URL` | hub.sr.ht | URL to patchset |

## git.sr.ht integration

Place `.build.yml` in the repo root. Optionally add up to 4 manifests in `.builds/*.yml`
(if more than 4 exist, 4 are selected randomly). Every push triggers a build automatically
unless `submitter.git.sr.ht.enabled: false` or filtered by `allow-refs`.
