> [!WARNING]
> **`busbar-actions` is under heavy active development — expect breaking changes.**
> These repositories are public, but **not ready for use yet** — please don't depend on them.
> A pilot is starting soon: **[star and watch the busbar-actions organization](https://github.com/busbar-actions)** for the launch of Discussions and the pilot announcement.

# busbar-actions/gh-env-scratch-ensure

Inspect a **stable scratch GitHub Environment slot** and report whether it is
ready for reuse — i.e. whether the named Environment exists and already holds the
secret names a downstream Salesforce job expects. Lets a workflow decide *reuse
the existing scratch org* vs. *fall back to provisioning a fresh one* without
ever touching secret values.

## What it does

Wraps the `gh-env-scratch-ensure` binary, which:

1. Authenticates to the **GitHub API** using either a `github-token` (PAT /
   `github.token`) or a **GitHub App** (`github-app-id` +
   `github-app-private-key`, from which it mints a repo-scoped installation
   token).
2. Checks whether the target Environment (default `scratch`) exists in the
   repository.
3. Lists the Environment's secret **names** (GitHub Environment secrets are
   write-only, so values are never read) and compares them against
   `required-secrets`.
4. Emits `GITHUB_OUTPUT`, a job-summary table, and a `notice`/`warning`
   annotation describing the result.
5. When `fail-if-not-ready` is `true`, exits non-zero if the Environment is
   absent or any required secret name is missing.

This action does **not** authenticate to Salesforce, read any SF token, or mint
an OIDC credential. It only reads GitHub Environment metadata, so it does **not**
require `permissions: id-token: write`.

## Usage

```yaml
permissions:
  contents: read   # plus a token that can read environments/environment secrets
steps:
  - id: slot
    uses: busbar-actions/gh-env-scratch-ensure@main
    with:
      environment-name: scratch
      # required-secrets: SF_ACCESS_TOKEN,SF_INSTANCE_URL,...   # defaults shown below
      # fail-if-not-ready: 'false'
      # github-token: ${{ secrets.GITHUB_TOKEN }}   # defaults to github.token

  - if: steps.slot.outputs.slot_ready != 'true'
    run: echo "Slot not reusable; provisioning a fresh scratch org"
```

Using a GitHub App instead of a PAT (e.g. to read environment secrets across
repositories):

```yaml
  - uses: busbar-actions/gh-env-scratch-ensure@main
    with:
      repository: my-org/target-repo
      github-app-id: ${{ vars.BUSBAR_APP_ID }}
      github-app-private-key: ${{ secrets.BUSBAR_APP_PRIVATE_KEY }}
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `environment-name` | no | `scratch` | GitHub Environment name (the stable named slot). |
| `repository` | no | `` (current repo) | Target repository in `owner/repo` form. Empty = `github.repository`. |
| `required-secrets` | no | `SF_ACCESS_TOKEN,SF_INSTANCE_URL,SF_USERNAME,SF_LOGIN_URL,SF_SCRATCH_ORG_ID,SF_SCRATCH_ORG_INFO_ID,BUSBAR_CREDENTIAL_CONTEXT` | Comma-delimited secret **names** that must be present for the slot to count as reusable. |
| `fail-if-not-ready` | no | `false` | If `true`, fail the step when the Environment is absent or missing any required secret name. |
| `github-token` | no | `` (falls back to `github.token`) | Token able to read environments and environment secrets. Optional when using GitHub App credentials. |
| `github-app-id` | no | `` | GitHub App id used to mint an installation token for the target repo. |
| `github-app-private-key` | no | `` | GitHub App private key PEM used to mint the installation token. |
| `version` | no | `latest` | Release tag of the `gh-env-scratch-ensure` binary to download. |
| `binary-repo` | no | `busbar-actions/actions-dist` | Repo publishing prebuilt binary releases. |

Provide **either** `github-token` **or** `github-app-id` + `github-app-private-key`.

## Outputs

| Output | Description |
|---|---|
| `environment_name` | The Environment name that was inspected. |
| `repository` | The repository that was inspected. |
| `environment_exists` | `"true"` / `"false"` — whether the Environment exists. |
| `slot_ready` | `"true"` / `"false"` — Environment exists **and** all required secret names are present. |
| `missing_secrets` | Comma-delimited required secret names that were missing (empty when ready). |

Outputs carry only secret **names** and booleans — never secret values.

## Auth / permissions model

- Reads GitHub Environment metadata only; **no `id-token: write`** and no
  Salesforce authentication.
- The supplied token (PAT, `github.token`, or minted GitHub App installation
  token) needs read access to the target repo's environments and environment
  secrets. The default `required-secrets` list reflects the slot layout written
  by `gh-env-scratch-save` / `org-auth`.

## Observability

The binary owns all user-facing output via the shared `github-actions-ux`
crate: under GitHub Actions it auto-selects the `GitHubReporter` (workflow
`::notice` / `::warning` commands plus a `$GITHUB_STEP_SUMMARY` table) and the
`CliReporter` locally. On error it calls `github_actions_ux::fail`, which emits a
`::error` annotation and exits non-zero. The action.yml is a thin wrapper:
`setup` installs the binary, and a single step passes inputs through `INPUT_*`
env vars.
