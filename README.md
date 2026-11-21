<img src="https://manufact.com/branding/app-icon-dark.svg" alt="Manufact" width="64" height="64">

# Manufact Publishing Checks

Run the Manufact publishing checklist for your MCP server in GitHub Actions, and fail the job when checks fail.

This action allows you to run Manufact's MCP Publishing Checks in CI/CD. 

## Before you start

1. **Create a Manufact account.** [Sign up at manufact.com](https://manufact.com/signup).
2. **Deploy or connect your MCP server.** [Deploy it from GitHub](https://docs.manufact.com/dashboard/quickstart), or [connect a server you host elsewhere](https://docs.manufact.com/dashboard/connect-existing-server). Note its [server ID](#find-your-server-id).
3. **Create an API key.** In the Manufact dashboard, open **API Keys** and click **Create key** ([how to create an API key](https://docs.manufact.com/dashboard/api-keys)).
4. **Store the key as a repository secret.** In your GitHub repository, go to **Settings > Secrets and variables > Actions** and add it as `MANUFACT_TOKEN`.

## Usage

```yaml
name: Publishing checklist

on:
  pull_request:

permissions:
  pull-requests: write # to comment the report on the pull request

jobs:
  checklist:
    runs-on: ubuntu-latest
    steps:
      - uses: mcp-use/publishing-checks@v1
        with:
          api-key: ${{ secrets.MANUFACT_TOKEN }}
          server: e3b0c442-98fc-4c14-9afb-f4c8996fb924
```

On pull requests, the checklist runs against the deployment for the pull request's head branch. Run it after that branch has deployed. If your deploy runs in the same workflow, put this step after it. On other events, such as a push to `main`, it checks production.

### Fail on warnings too

```yaml
      - uses: mcp-use/publishing-checks@v1
        with:
          api-key: ${{ secrets.MANUFACT_TOKEN }}
          server: e3b0c442-98fc-4c14-9afb-f4c8996fb924
          fail-on: warnings
```

### Use the result in later steps

```yaml
      - id: checklist
        uses: mcp-use/publishing-checks@v1
        with:
          api-key: ${{ secrets.MANUFACT_TOKEN }}
          server: e3b0c442-98fc-4c14-9afb-f4c8996fb924
          fail-on: none
      - if: steps.checklist.outputs.ready-for-chatgpt == 'true'
        run: echo "Ready for ChatGPT"
```

### External URL servers: check each preview deployment

If your server is hosted elsewhere (for example on Vercel) and connected to Manufact as an External URL server, run the checklist after each preview deployment with `url` set to that deployment's MCP URL:

```yaml
name: Publishing checklist

on:
  deployment_status:

permissions:
  pull-requests: write # to comment the report on the pull request

jobs:
  checklist:
    # Check finished preview deployments only.
    if: github.event.deployment_status.state == 'success' && github.event.deployment.environment == 'Preview'
    runs-on: ubuntu-latest
    steps:
      - uses: mcp-use/publishing-checks@v1
        with:
          api-key: ${{ secrets.MANUFACT_TOKEN }}
          server: e3b0c442-98fc-4c14-9afb-f4c8996fb924
          url: ${{ github.event.deployment_status.environment_url }}/mcp
          profile: 6f1c2a7e-0b5d-4e8a-9c3f-2d4b6a8e1f90 # optional
```

- `url` is the MCP endpoint to check. The run is filed under the server, and the report shows which URL and commit it checked.
- `profile` is the [profile](https://docs.manufact.com/dashboard/profiles) to sign in with, when the server needs authentication. Leave it out to use the server's own profile. The profile must be yours or shared with the organization.
- `deployment_status` events only fire when your host creates GitHub deployments. On Vercel, check that creating GitHub deployments is turned on for the project; it can be off. The action finds the open pull request for the deployed commit and comments there.
- Your preview deployments must accept the profile's token: they should trust the same authorization server as the profile, and accept a token issued for the MCP resource the profile signed in to.

## Find your server ID

Open your server in the Manufact dashboard. The server ID is the UUID after `/servers/` in the URL:

```
https://manufact.com/cloud/<org>/servers/e3b0c442-98fc-4c14-9afb-f4c8996fb924/overview
                                         └─────────── server ID ────────────┘
```

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `api-key` | — | Required. Manufact API key, scoped to the server's organization. Store it as a secret. See [how to create an API key](https://docs.manufact.com/dashboard/api-keys). |
| `server` | — | Required. Server ID: the UUID in the server's dashboard URL. See [Find your server ID](#find-your-server-id). |
| `branch` | PR head branch; production on other events | Preview branch whose deployment is checked. Set `branch: ""` to always check production. Ignored when `url` or `profile` is set. |
| `url` | — | External URL servers only. The MCP URL to check instead of the server's, for example a preview deployment. |
| `profile` | — | External URL servers only. ID of the profile to sign in with instead of the server's. |
| `fail-on` | `failed` | `none`, `failed` or `warnings`. See below. |
| `comment` | `true` | On pull requests, comment the full report. Set `false` to turn it off. |
| `github-token` | `github.token` | Token used to post the comment. |

`fail-on` levels:

| Level | The job fails when |
| --- | --- |
| `none` | The checklist can't run at all (for example, a bad API key) |
| `failed` | A publishing check fails, or the run itself fails or is cancelled |
| `warnings` | Same as `failed`, or a check reports a warning or policy concern |

Policy concerns count as warnings, as on the dashboard: they may need store review but aren't failed publishing checks.

## Outputs

| Output | Description |
| --- | --- |
| `run-id` | Checklist run ID |
| `status` | Final run status, for example `completed` or `failed` |
| `score` | Checks score out of 100, as shown on the dashboard |
| `ready-for-chatgpt` | `true` or `false`; empty when unknown |
| `ready-for-claude` | `true` or `false`; empty when unknown |
| `dashboard-url` | Link to the checklist in the Manufact dashboard |

## Pull request comment

On `pull_request` events, and on `deployment_status` events for a commit with an open pull request, the action comments the report on the pull request, laid out like the checklist page in the Manufact dashboard: the checks score, the last run's results and client readiness, the policy-review banner, and one section per category. Sections with issues start expanded; failing checks show their severity, the affected clients, a policy link where there is one, and how to fix them.

There's one comment per server. Later runs update it rather than adding a new comment. If the checklist can't run at all (for example, the API key is wrong), the comment shows the error instead.

Commenting needs the job to have `pull-requests: write`. Without it, the action logs a warning and carries on; the checklist result is unaffected. Pull requests from forks get a read-only token, so there's no comment for those.

The full report also goes to the job log, and the same summary is added to the job summary page.

## Requirements

- A runner with `jq`. GitHub-hosted runners have it. The action installs Node.js 22 itself.
- `url` and `profile` need `@mcp-use/cli` 5.3.0 or later, which this release pins.

Each release of this action pins the `@mcp-use/cli` version it was tested with.
