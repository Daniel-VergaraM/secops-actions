# secops-actions

SAST · SCA · DAST · IAST · Fuzzing as GitHub Actions, ready to drop into any
repo with minimal config. They detect the language on their own (via GitHub's
Linguist API) and, where it applies, how the app starts
(`Dockerfile` / `docker-compose.yml`) — nobody has to tell the pipeline what
the project is written in.

| Technique | File(s) | Runs against | View | Engine |
|---|---|---|---|---|
| SAST | `sast.yml` + `codeql-sast.yml` | Nothing (static) | your own code, from the inside | Semgrep (always) + CodeQL (public repos only, or if forced) |
| SCA | `sca.yml` | Nothing (static) | third-party libraries | Trivy + dependency-review |
| DAST | `dast.yml` | Running app | from the outside, like an attacker | OWASP ZAP |
| IAST | `iast.yml` | Running app | from the inside, with an agent (approximated) | native coverage + ZAP |
| Fuzzing | `fuzzing.yml` | Running app | malformed input | Schemathesis if a schema exists (OpenAPI/GraphQL), otherwise ffuf (black-box) |

## Usage: consume it as a reusable workflow, don't copy `.github/`

This repo ships as **reusable GitHub Actions workflows**
(`on: workflow_call`). The target repo doesn't need a local copy of
`.github/` at all — it just points a thin workflow file at this repo's
`security-pipeline.yml`, which chains the other 6:

```yaml
# .github/workflows/security.yml in your repo
name: Security Pipeline
on:
  pull_request:
  push: { branches: [main] }
  schedule: [{ cron: '0 5 * * *' }]
  workflow_dispatch:

jobs:
  security:
    uses: Daniel-VergaraM/secops-actions/.github/workflows/security-pipeline.yml@v1
    secrets: inherit
    with:
      dast_target_url: ''        # optional: your staging URL
      iast_start_cmd: ''         # optional: e.g. "node dist/server.js"
```

`secrets: inherit` is what lets `semgrep_token` (if you have one, see the
table below) flow from your repo's secrets into `sast.yml` without you
having to name it explicitly.

### Available inputs

All inputs are optional strings; every one has a safe default and the
pipeline runs fine with none of them set.

| Input | Default | What it's for |
|---|---|---|
| `dast_target_url` | `''` | URL to attack with DAST instead of letting the runner spin up the app itself (equivalent to the old `DAST_TARGET_URL` repo variable) |
| `target_url` | `''` | Same as `dast_target_url`, but as a one-off override for a single manual run |
| `scan_type` | `'baseline'` | DAST mode: `baseline` (passive, fast) or `full` (active, slower and more aggressive) |
| `dast_app_port` | `'8080'` | Port the app listens on, shared by DAST, IAST and Fuzzing |
| `iast_start_cmd` | `''` | Command that starts the app for IAST (e.g. `"node dist/server.js"`, `"python -m app"`, `"go run ."`); without it, `iast` skips itself |
| `openapi_path` | `''` | OpenAPI/Swagger spec path, if it's not at one of the common paths Fuzzing already tries |
| `graphql_path` | `''` | GraphQL endpoint path, if it's not `/graphql` |
| `codeql_force_run` | `'false'` | Forces CodeQL on private repos that do have a GitHub Advanced Security license |

Secret: `semgrep_token` (optional) — Semgrep App Token for inline PR comments
via the Semgrep AppSec Platform; passed through `secrets: inherit`. Semgrep
still runs fine without it, just without that integration.

### Calling one technique at a time

You don't have to run the whole pipeline. Each individual workflow is also a
valid reusable workflow on its own, with the same inputs listed above (scoped
to what that file actually uses):

```yaml
jobs:
  sast:
    uses: Daniel-VergaraM/secops-actions/.github/workflows/sast.yml@v1
    secrets: inherit
  sca:
    uses: Daniel-VergaraM/secops-actions/.github/workflows/sca.yml@v1
    secrets: inherit
```

## Unavoidable manual configuration

This is minimum configuration, not zero-config — these three items are
GitHub platform limits, not something this pipeline could automate away:

1. **Settings → Actions → General → Workflow permissions** → "Read and write
   permissions". Without this, the step that uploads results to the Security
   tab (`upload-sarif`) fails silently even though the scan ran fine.
2. **Settings → Code security → Dependency graph** → enabled (free). Used by
   `dependency-review-action` inside `sca.yml`.
3. This repo (`secops-actions`) needs a **`v1` tag/release** to exist before
   `uses: .../security-pipeline.yml@v1` resolves in a consumer repo. Create
   it once (`git tag v1 && git push origin v1`, or a GitHub Release) after
   the workflows here are in the shape you want to pin.

With that, **SAST (Semgrep) and SCA already work on their own**, nothing else
to touch. CodeQL (`codeql-sast.yml`) needs `sast.yml` to already exist on the
default branch before it can trigger (they're chained via `workflow_run`, see
below) — so its first run lands after the first push, not on it.

### Optional repo variables (same thing, named differently per workflow)

The table above covers the `workflow_call` inputs. If you'd rather configure
by repo variable instead of passing `with:` every time (or you're running a
workflow via `workflow_dispatch`/`schedule`, where `with:` isn't available),
the same knobs also exist as `Settings → Secrets and variables → Actions →
Variables` in the consumer repo. No workflow fails for not having them: each
one just skips the part it can't resolve and leaves a `::notice::` explaining
what was missing.

| Variable | Used by | When you need it | If you don't set it |
|---|---|---|---|
| `DAST_TARGET_URL` | `dast.yml` | you don't have a `Dockerfile`/`docker-compose.yml` in the repo, or you'd rather attack your real staging environment instead of having the runner spin up the app | tries to spin up the app with Docker |
| `DAST_APP_PORT` | `dast.yml`, `iast.yml`, `fuzzing.yml` | your app doesn't listen on `8080` | `8080` |
| `IAST_START_CMD` | `iast.yml` | you want this workflow to run (e.g.: `node dist/server.js`, `python -m app`, `go run .`) | skips itself, without failing the pipeline |
| `OPENAPI_PATH` | `fuzzing.yml` | your OpenAPI spec isn't at a common path | tries `/openapi.json`, `/swagger.json`, `/v3/api-docs`, etc. |
| `GRAPHQL_PATH` | `fuzzing.yml` | your GraphQL endpoint isn't `/graphql` | tries `/graphql`, `/api/graphql`, `/v1/graphql`, etc. |
| `CODEQL_FORCE_RUN` | `codeql-sast.yml` | the repo is private but you do have a GHAS license | on private repos, without this, CodeQL skips itself |
| `SEMGREP_APP_TOKEN` **(secret)** | `sast.yml` | you want inline PR comments via the Semgrep AppSec Platform | Semgrep still runs, just without that integration |

If your repo already has a `Dockerfile` or `docker-compose.yml` at the root
and the app listens on `8080`, **you don't need to set any variable**: DAST
and Fuzzing spin up the app themselves and attack it.

## Licensing: the one thing you have to decide, not configure

`codeql-sast.yml` checks the repo's actual visibility
(`repos/{owner}/{repo}` via the GitHub API) before running anything:

- **Public repo** → CodeQL is free with no limits, always runs.
- **Private repo** → Advanced CodeQL requires GitHub Advanced Security (paid).
  The workflow skips itself so it doesn't fail over licensing or burn CI
  minutes on something you won't be able to use. If your organization does
  have GHAS, set `CODEQL_FORCE_RUN=true` (table above) to force it.

In the meantime, Semgrep (`sast.yml`) covers SAST at no cost or license,
whether CodeQL runs or not. Everything else in the pipeline (Trivy,
dependency-review, OWASP ZAP, Schemathesis, ffuf) is also free and open
source, with no account or token required.

`codeql-sast.yml` and `sast.yml` are kept in separate files on purpose:
GitHub Actions doesn't let you use `needs:` across different workflow files,
so they're chained with the `workflow_run` trigger — `codeql-sast.yml`
triggers as soon as `sast.yml` finishes.

## About IAST: why `iast.yml` is an approximation, not "real IAST"

As of today there's no real IAST tool (with *taint-tracking* — following an
attacker-controlled value to see if it reaches a SQL query unsanitized) that's
free/open source and covers Go, TypeScript/Node, and Python at once:

- **Contrast Community Edition**, the only serious free IAST (Java/.NET
  Core), was **discontinued on June 30, 2025**. Contrast Assess is now 100%
  commercial.
- **OpenRASP** (Baidu, Apache 2.0) is still free, but its last release was in
  January 2022 and it only covers Java and PHP well on traditional
  application servers — it doesn't fit containerized microservices.
- **Datadog Code Security (IAST)** is commercial (14-day trial); it's the
  most reasonable option today if you already use Datadog for APM.

`iast.yml` does the most honest thing you can do for free: it spins up the app
with **language-native coverage instrumentation** (a real agent running
inside the process — `go tool cover`, `coverage.py`, `c8`, JaCoCo), attacks it
with ZAP while it's running, and reports which code actually executed during
the attack. That's real, useful *glass-box testing*, but not taint-tracking.
If your team needs real IAST, the honest route is paying for Datadog or
Contrast — there's no free shortcut today.

## About Fuzzing: not every API has a spec

Most real-world APIs don't expose OpenAPI/Swagger. `fuzzing.yml` tries three
levels in order, with nobody choosing anything:

1. **REST with a spec** → Schemathesis reads the OpenAPI/Swagger and
   generates thousands of edge cases on its own.
2. **GraphQL** → self-describing: `fuzzing.yml` tries an introspection query
   (`{__typename}`) against typical paths, and if it responds, Schemathesis
   fuzzes from there with no spec needed.
3. **Neither of the above** (the most common case for undocumented internal
   APIs) → black-box fallback with **ffuf**: discovers endpoints via wordlist
   (SecLists) and hits each one with malformed payloads in the query string
   and body, looking for 5xx errors. Less precise than a schema-aware fuzzer,
   but doesn't depend on anything being documented.

Capped at 20 endpoints processed in step 3 to avoid blowing up CI time. For
fuzzing pure libraries/parsers (not HTTP) in Go/C/C++/Python/Rust/Java, the
reference tool is [ClusterFuzzLite](https://google.github.io/clusterfuzzlite/)
by Google — not included as an active workflow here because it always
requires writing a "fuzz target" by hand.

## Required PR checks (optional, recommended)

Once you confirm SAST and SCA run clean on the target repo, turn them into
*required status checks* under **Settings → Branches → Branch protection
rules** so a merge can't happen with CRITICAL vulnerabilities without someone
explicitly seeing it (the `trivy` job's "gate" step and the `codeql` job
already fail on their own when they find something; Semgrep defaults to
informational mode — remove `continue-on-error` in `sast.yml` if you also
want it to block).

## Repo structure

```
.github/
  actions/detect-stack/action.yml   # language detection + how the app starts
  workflows/
    security-pipeline.yml  # orchestrator — chains the 6 workflows below
    sast.yml           # SAST — Semgrep
    codeql-sast.yml     # SAST — CodeQL (gated on visibility/GHAS)
    sca.yml             # SCA — Trivy + dependency-review
    dast.yml            # DAST — OWASP ZAP
    iast.yml             # IAST — approximation (native coverage + ZAP)
    fuzzing.yml          # Fuzzing — Schemathesis / ffuf
README.md
```
