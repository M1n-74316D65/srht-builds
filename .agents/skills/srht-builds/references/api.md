# builds.sr.ht API & hut CLI Reference

## hut CLI (recommended)

`hut` is the official SourceHut CLI. It's the preferred way to interact with builds.sr.ht
because it handles authentication and speaks the modern GraphQL API.

### Local submission

```bash
hut builds submit < .build.yml
```

Options:
- `--note "description"` — human-readable label shown on the dashboard
- `--tag mytag` — adds a tag (used for build status badges and filtering)
- `--no-secrets` — disable secrets for untrusted builds

### Using hut from within a build

Add `hut` to `packages:` and set `oauth:` in the manifest. The `oauth:` field generates a
temporary bearer token with the specified grants, available to `hut` automatically:

```yaml
image: alpine/latest
packages:
  - hut
oauth: jobs.sr.ht/PROFILE:RO    # grants read access to builds.sr.ht
tasks:
  - check: |
      # query build info from within the running job
      hut builds job get $JOB_ID
```

Grant scopes follow the pattern `service/RESOURCE:ACCESS`, e.g.:
- `jobs.sr.ht/PROFILE:RO` — read your builds
- `jobs.sr.ht/PROFILE:RW` — read + write (submit, cancel)

The `$JOB_ID` and `$JOB_URL` environment variables are always available in builds.

---

## REST API (legacy, still functional)

The REST API is being replaced by GraphQL but remains fully operational. All requests go to
`https://builds.sr.ht`. Authentication uses OAuth 2.0 bearer tokens from meta.sr.ht.

### Authentication

Obtain a token from https://meta.sr.ht/oauth with scope `jobs:read` and/or `jobs:write`.
Pass it as a header: `Authorization: Bearer <token>`.

### POST /api/jobs — Submit a build

**Scope:** `jobs:write`

```bash
curl -X POST https://builds.sr.ht/api/jobs \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "manifest": "image: alpine/latest\ntasks:\n  - hello: |\n      echo hi\n",
    "note": "my release build",
    "tags": ["release", "v1.0"],
    "execute": true,
    "secrets": true
  }'
```

Request fields:
| Field | Type | Default | Description |
|---|---|---|---|
| `manifest` | string | required | The YAML manifest as a string |
| `note` | string | null | Markdown description shown on dashboard |
| `tags` | list of string | [] | Labels for filtering/badges; lowercase alphanumeric + `-_.` |
| `execute` | boolean | true | Start immediately; false = create in pending state |
| `secrets` | boolean | true | Inject secrets; set false for untrusted builds |

Tip: to safely embed a multi-line YAML manifest in JSON from shell:
```bash
MANIFEST=$(cat .build.yml)
JSON_MANIFEST=$(printf '%s' "$MANIFEST" | python3 -c 'import json,sys; print(json.dumps(sys.stdin.read()))')
curl -X POST https://builds.sr.ht/api/jobs \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"manifest\": $JSON_MANIFEST, \"note\": \"release\"}"
```

Response: the created job object (see GET /api/jobs/:id for shape).

### GET /api/jobs/:id — Get job status

**Scope:** `jobs:read`

```bash
curl -s https://builds.sr.ht/api/jobs/$JOB_ID \
  -H "Authorization: Bearer $TOKEN"
```

Response:
```json
{
  "id": 12345,
  "status": "success",
  "setup_log": "https://...",
  "tasks": [
    {"name": "build", "status": "success", "log": "https://..."},
    {"name": "test",  "status": "failed",  "log": "https://..."}
  ]
}
```

**Job status values:** `pending` → `queued` → `running` → `success` | `failed`  
**Task status values:** `pending` → `running` → `success` | `failed`

### Polling until complete (bash)

```bash
while true; do
  STATUS=$(curl -s -H "Authorization: Bearer $TOKEN" \
    https://builds.sr.ht/api/jobs/$JOB_ID \
    | python3 -c 'import json,sys; print(json.load(sys.stdin)["status"])')
  echo "Status: $STATUS"
  [[ "$STATUS" == "success" || "$STATUS" == "failed" ]] && break
  sleep 10
done
```

### GET /api/jobs/:id/artifacts — List artifacts

**Scope:** `jobs:read`

Returns a paginated list. Each artifact:
```json
{
  "id": 1,
  "created": "2024-01-01T00:00:00Z",
  "path": "/home/build/myproject/mybinary",
  "name": "mybinary",
  "url": "https://...",
  "size": 1234567
}
```

Artifacts are retained for 90 days and only created on successful builds.

### GET /api/jobs/:id/manifest — Retrieve manifest

**Scope:** `jobs:read`  
Returns the original manifest as plain text.

### POST /api/jobs/:id/start — Start a pending job

**Scope:** `jobs:write`  
For jobs created with `execute: false`. Returns `{}` on success.

### POST /api/jobs/:id/cancel — Cancel a running job

**Scope:** `jobs:write`  
Returns `{}` on success.

---

## GraphQL API

builds.sr.ht's GraphQL API is the primary API going forward. Documentation:
- Reference: https://docs.sourcehut.org/builds.sr.ht
- Interactive playground: https://builds.sr.ht/graphql

The `hut` CLI uses the GraphQL API under the hood, so for most use cases `hut` is simpler
than constructing GraphQL queries manually.
