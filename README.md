# observo-ai/setup-action

OB-357 sub-task 3d — GitHub Action that authenticates a workflow with Observo via OIDC and creates a test run.

Use it instead of manually plumbing an `OBSERVO_API_KEY` secret + `observo run create` shell steps. It mints a short-lived OIDC token, exchanges it for an Observo session, opens a run, and exports `OBSERVO_RUN_KEY` / `OBSERVO_TOKEN` env vars for subsequent steps.

## Prerequisites

1. Install the **Observo for GitHub** App on your org / user account ([github.com/apps/observo-ai](https://github.com/apps/observo-ai)).
2. After install, visit `observoai.co/install/complete` while logged into Observo to bind the installation to your account.
3. Your workflow must request the `id-token: write` permission (composite actions can't grant it for you).

## Usage

```yaml
name: Observo E2E

on: [pull_request, push]

jobs:
  e2e:
    runs-on: ubuntu-latest
    permissions:
      id-token: write       # required for OIDC
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }

      - uses: observo-ai/setup@v1
        with:
          plan: E2E-MAIN

      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npx playwright test
        # Reporter picks up OBSERVO_RUN_KEY automatically.

      - name: Finalize Observo run
        if: always()
        run: |
          curl -sS -X POST \
            -H "Authorization: Bearer $OBSERVO_TOKEN" \
            -H "Content-Type: application/json" \
            -d '{"status":"auto"}' \
            "${API_BASE:-https://api.observoai.co}/api/runs/$OBSERVO_RUN_KEY:finish"
```

## Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `plan` | ✅ | — | Observo plan key for the run (e.g. `E2E-MAIN`). |
| `api-base-url` | — | `https://api.observoai.co` | Override for self-hosted / staging deployments. |
| `audience` | — | `https://api.observoai.co` | OIDC `aud` claim. Must match the Observo backend's expected audience. |
| `run-name` | — | `<workflow> #<run-number>` | Human-readable name for the run row in Observo. |

## Outputs

| Name | Description |
|---|---|
| `run-key` | The `OBSERVO_RUN_KEY` for this run. Also written to `$GITHUB_ENV`. |
| `session-token` | Short-lived Observo PASETO. Also exported as `OBSERVO_TOKEN`. |

## How it works

1. **Mint OIDC token** — calls the GitHub Actions OIDC issuer endpoint (`ACTIONS_ID_TOKEN_REQUEST_URL`) with the configured audience.
2. **Exchange for session** — `POST /api/auth/oidc/exchange` on the Observo backend, which verifies the JWT against the GitHub Actions JWKS, maps `repository_owner_id` → Observo account via `github_app_installations`, and returns a 1-hour PASETO.
3. **Create run** — `POST /api/runs` with plan key + commit / branch / CI run URL derived from `GITHUB_*` env.
4. **Export env** — `OBSERVO_RUN_KEY`, `OBSERVO_TOKEN`, `OBSERVO_ACCOUNT_ID` written to `$GITHUB_ENV` for downstream steps.

The Observo Playwright reporter (`@observo/playwright-reporter`) automatically attaches to an existing run when `OBSERVO_RUN_KEY` is set — no extra configuration required.

## Versioning

Pin to a major version tag in production:

```yaml
- uses: observo-ai/setup@v1
```

We follow semver. Breaking changes bump the major; new features bump the minor. Patches are auto-tagged from `main`.

## Self-hosted Observo

```yaml
- uses: observo-ai/setup@v1
  with:
    plan: E2E-MAIN
    api-base-url: https://observo.acme.internal
    audience: https://observo.acme.internal
```

The audience must match the Observo backend's `OBSERVO_API_BASE_URL` config.

## Troubleshooting

**"Missing OIDC token request env"** — your workflow lacks `permissions: { id-token: write }`. Add it at the job level.

**"OIDC exchange failed: ... no account mapping"** — the GitHub App is installed but not yet bound to your Observo account. Visit `observoai.co/install/complete` while logged in.

**"OIDC exchange failed: ... invalid OIDC token"** — most often, the `audience` input doesn't match the Observo backend's expected audience. Default is `https://api.observoai.co`.

## License

MIT — see [LICENSE](LICENSE).
