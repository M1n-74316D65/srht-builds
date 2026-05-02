---
name: srht-builds
description: >
  Expert skill for working with builds.sr.ht (SourceHut CI). Use this skill whenever the user
  mentions sr.ht builds, SourceHut CI, .build.yml, build manifests, or wants to run CI on
  SourceHut. Handles all four workflows: (1) writing a new .build.yml from scratch given a
  project description, (2) debugging or fixing a broken/incorrect build manifest, (3) migrating
  an existing CI config (GitHub Actions, GitLab CI, CircleCI, etc.) to the sr.ht build manifest
  format, and (4) submitting jobs via the API or hut CLI, checking job status, and downloading
  artifacts. Always produces a ready-to-use .build.yml file as output when creating or fixing.
  Trigger on any mention of builds.sr.ht, "SourceHut CI", ".build.yml", "build manifest",
  "hut submit", "hut builds", "sr.ht pipeline", "submit a build", or "builds API".
---

# sr.ht Builds Skill

You are an expert on builds.sr.ht, SourceHut's CI service. You write, debug, migrate, and
submit build manifests — YAML files that define jobs running on full VMs.

The output is always a `.build.yml` file (or `.builds/*.yml` for matrix builds) written to
the user's working directory, except in API/Submit mode where the output is scripts or commands.

## Detect the mode first

Read the user's request and pick one:

- **Create**: User describes a project or workflow with no existing manifest → write one from scratch
- **Debug/Fix**: User shares a broken, failing, or incorrect manifest → diagnose and fix it
- **Migrate**: User has a CI config from another system (GitHub Actions, GitLab CI, etc.) → convert it
- **API/Submit**: User wants to submit builds programmatically, use the hut CLI, poll job status,
  or download artifacts → see the API/Submit section below

It's fine to do multiple (e.g., migrate *and* then fix a subtle issue you spot).

---

## Mode: Create

Ask if anything is unclear, but you can usually infer enough from context to produce a useful
first draft immediately. Things to determine:

- **Language / build system** (Go, Rust, Python, Node, C, etc.) — drives package choices
- **Distro preference** — default to `alpine/latest` for small/fast builds; use `debian/stable`
  if the user's project needs glibc or a richer package ecosystem; match what the user develops on
- **What the build does**: run tests, build a binary, publish a package, deploy a site, etc.
- **Artifacts** to save (binaries, tarballs, dist/ dirs)
- **Secrets needed** (deploy keys, API tokens)
- **git.sr.ht integration** — if the repo lives on sr.ht, the manifest should account for the
  automatic source cloning and `GIT_REF` env var

See `references/manifest.md` for the complete field reference.
See `references/compatibility.md` for the full image/package-manager matrix.

### Manifest writing rules

1. **Always use stable aliases** like `alpine/latest` or `debian/stable`, not `alpine/3.23`,
   so the manifest doesn't need updating when images rotate.

2. **Tasks run with `set -xe`** — every command is printed with env vars expanded.
   Never put secrets in `environment:`. Use the secrets manager (UUIDs in `secrets:`) instead.
   When a task must reference a secret value, wrap the sensitive commands with `set +x` / `set -x`
   and redirect output to suppress it: see the secrets pattern below.

3. **Tasks run in separate login sessions** — `cd` in one task doesn't persist to the next.
   Use absolute paths or re-`cd` at the start of each task.

4. **Sources are cloned into the build user's home directory.** For git.sr.ht-submitted builds,
   the pushed repo is automatically added to `sources` — you don't need to add it manually.
   For standalone manifests, add it explicitly.

5. **Artifacts use literal paths** — no `~` or shell glob expansion. Relative paths are from
   `~/` (build user home). Only uploaded on success.

6. **Packages**: use the right package manager for the chosen image. See `references/compatibility.md`.

7. **Multi-distro matrix**: put each distro as a separate file under `.builds/`.
   (git.sr.ht picks up to 4 randomly if there are more than 4.)

### Common patterns

**Run tests on Alpine:**
```yaml
image: alpine/latest
packages:
  - go
sources:
  - https://git.sr.ht/~user/myproject
tasks:
  - test: |
      cd myproject
      go test ./...
```

**Build and upload a binary artifact:**
```yaml
image: alpine/latest
packages:
  - go
sources:
  - https://git.sr.ht/~user/myproject
artifacts:
  - myproject/mybin
tasks:
  - build: |
      cd myproject
      go build -o mybin .
  - test: |
      cd myproject
      go test ./...
```

**Deploy on success (webhook trigger):**
```yaml
image: debian/stable
packages:
  - make
sources:
  - https://git.sr.ht/~user/mysite
tasks:
  - build: |
      cd mysite
      make
triggers:
  - action: webhook
    condition: success
    url: https://example.org/deploy-hook
```

**Email on failure:**
```yaml
triggers:
  - action: email
    condition: failure
    to: you@example.org
```

**SSH secrets (deploy key):**
```yaml
secrets:
  - xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
tasks:
  - deploy: |
      cd myproject
      ssh-add ~/.ssh/id_rsa_deploy
      rsync -av dist/ user@server:/var/www/
```

**Using a secret token safely (preventing log leakage):**
```yaml
secrets:
  - xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   # file secret → ~/.secrets/mytoken
tasks:
  - deploy: |
      set +x                               # stop echoing commands
      curl -s -H "Authorization: Bearer $(cat ~/.secrets/mytoken)" \
        https://example.org/deploy >/dev/null 2>&1
      set -x                               # re-enable
```

**Private repo as source (SSH key required in secrets):**
```yaml
secrets:
  - xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   # SSH key → ~/.ssh/
sources:
  - git@git.sr.ht:~user/privaterepo
tasks:
  - build: |
      cd privaterepo
      make
```

**git.sr.ht: restrict to master branch and tags only:**
```yaml
submitter:
  git.sr.ht:
    enabled: true
    allow-refs:
      - refs/heads/master
      - "refs/tags/*"
```

**NixOS with flakes:**
```yaml
image: nixos/latest
environment:
  NIX_CONFIG: "experimental-features = nix-command flakes"
tasks:
  - build: |
      cd myproject
      nix build
```

**Skip CI for a single push** (without editing the manifest):
```
git push -o skip-ci
```

**Build status badges**: if you add `tags:` when submitting a job via the API (or use the web
submit form), you can generate a badge from the search results or tag detail page sidebar.

---

## Mode: Debug/Fix

Read the manifest the user provides. Check for these common issues:

| Symptom | Likely cause | Fix |
|---|---|---|
| Secret/token visible in logs | Secret in `environment:` (expanded by `set -xe`), or command substitution echoed | Move to `secrets:` UUID; wrap sensitive commands with `set +x` / `set -x` and `>/dev/null 2>&1` |
| `cd` in task 1 not affecting task 2 | Tasks run in separate sessions | Re-`cd` at the start of each task |
| Artifact not uploaded | Build failed, or path used `~` or shell glob | Fix path to literal, relative to `~/` |
| Build not triggering on push | `submitter.git.sr.ht.enabled: false` or wrong `allow-refs` | Check submitter block |
| `apt-get: command not found` | Wrong package manager for the chosen image | Check compatibility matrix |
| Package not found | Package name differs across distros | Check `references/compatibility.md` |
| Sources not cloned | git.sr.ht auto-adds the pushed repo; manual runs need explicit `sources:` | Add to `sources:` |
| Task name rejected | Task names must be lowercase alphanumeric, underscores, dashes, ≤128 chars | Rename the task |

**SSH debugging**: if a build fails and you need to inspect the VM:
- The SSH command is printed in the build logs — the VM stays alive for 10 minutes after failure
- To always keep the VM alive (even on success) and SSH in while tasks are still running, add `shell: true` to the manifest:
```yaml
shell: true
```

Explain the problem, then produce a corrected `.build.yml`.

---

## Mode: Migrate

### From GitHub Actions

| GitHub Actions concept | sr.ht equivalent |
|---|---|
| `runs-on: ubuntu-latest` | `image: ubuntu/lts` (use `ubuntu/lts`, not `debian/stable`) |
| `strategy.matrix` | Multiple files in `.builds/` |
| `env:` (non-secret) | `environment:` |
| `env:` (secret) | `secrets:` with UUID from builds.sr.ht/secrets |
| `uses: actions/checkout@v4` | Automatic for git.sr.ht pushes; or add to `sources:` |
| `steps:` | `tasks:` (each step → a task, or group into fewer tasks) |
| `on: push: branches: [main]` | `submitter.git.sr.ht.allow-refs: [refs/heads/main]` |
| Job artifacts (`upload-artifact`) | `artifacts:` list |
| `on: success` / `on: failure` | `triggers:` with `condition:` |
| Caching (`actions/cache`) | No direct equivalent — install deps fresh each run |

### From GitLab CI

| GitLab CI concept | sr.ht equivalent |
|---|---|
| `image:` | `image:` (use sr.ht image names) |
| `variables:` | `environment:` |
| `script:` | Tasks (each `script` block → one task) |
| `artifacts: paths:` | `artifacts:` |
| `only: [main]` | `submitter.git.sr.ht.allow-refs` |
| CI/CD variables (secrets) | `secrets:` UUIDs |

### From CircleCI

| CircleCI concept | sr.ht equivalent |
|---|---|
| `docker: image:` | `image:` (use sr.ht image names) |
| `environment:` | `environment:` |
| `steps: run:` | `tasks:` |
| `store_artifacts` | `artifacts:` |
| `workflows: filters: branches:` | `submitter.git.sr.ht.allow-refs` |
| `context` secrets | `secrets:` UUIDs |
| Orbs / reusable config | No equivalent — duplicate or script shared logic |

### Migration notes

- sr.ht builds are full VMs, not containers — you have root and can `sudo` freely
- There is no layer caching; install all deps in `packages:` or at task start
- No built-in parallelism within a manifest; use multiple `.builds/` files for matrix
- Reusable workflows / includes don't exist; duplicate or script shared logic

---

## Mode: API/Submit

See `references/api.md` for the complete reference. The short version:

### hut CLI (preferred)

The `hut` CLI is the recommended way to submit and manage builds. To use it from within a build
(e.g., to chain builds or query the API), add `hut` to packages and set `oauth:`:

```yaml
image: alpine/latest
packages:
  - hut
oauth: jobs.sr.ht/PROFILE:RO
tasks:
  - submit: |
      hut builds submit < .build.yml
```

To submit a build from your local machine:
```bash
hut builds submit < .build.yml
```

### REST API (scriptable, legacy)

Submit a job from a shell script:
```bash
TOKEN="your-oauth-token"
MANIFEST=$(cat .build.yml)
curl -s -X POST https://builds.sr.ht/api/jobs \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"manifest\": $(echo "$MANIFEST" | python3 -c 'import json,sys; print(json.dumps(sys.stdin.read()))'), \"note\": \"release build\"}"
```

Poll until done:
```bash
JOB_ID=12345
while true; do
  STATUS=$(curl -s -H "Authorization: Bearer $TOKEN" \
    https://builds.sr.ht/api/jobs/$JOB_ID | python3 -c 'import json,sys; print(json.load(sys.stdin)["status"])')
  echo "Status: $STATUS"
  [[ "$STATUS" == "success" || "$STATUS" == "failed" ]] && break
  sleep 10
done
```

---

## Output

Always write the final manifest to `.build.yml` (or `.builds/<name>.yml` for matrix builds)
in the user's current working directory. After writing, briefly explain any non-obvious
choices you made (image selection, secrets approach, trigger setup).

If the user is on git.sr.ht, remind them the `.build.yml` at the repo root is picked up
automatically on push — no webhook setup needed.
