# Manufact publishing checks

Run the Manufact Cloud publishing checklist for your MCP server in GitHub Actions, and fail the job when checks fail.

The action runs [`mcp-use checklist run`](https://mcp-use.com/docs/typescript/api-reference/cli-reference#checklist-run) from `@mcp-use/cli`. It checks a server that is already deployed on Manufact Cloud. It does not build or deploy anything.

> This is not the Manufact GitHub App. The app runs the checklist from Manufact Cloud after each deployment. Use this action when you want the result as a step in your own workflow, for example to gate a release job.

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
          token: ${{ secrets.MANUFACT_TOKEN }}
          server: weather
```

On pull requests, the checklist runs against the deployment for the pull request's head branch. Run it after that branch has deployed. If your deploy runs in the same workflow, put this step after it. On other events, such as a push to `main`, it checks production.

### Fail on warnings too

```yaml
      - uses: mcp-use/publishing-checks@v1
        with:
          token: ${{ secrets.MANUFACT_TOKEN }}
          server: weather
          fail-on: warnings
```

### Use the result in later steps

```yaml
      - id: checklist
        uses: mcp-use/publishing-checks@v1
        with:
          token: ${{ secrets.MANUFACT_TOKEN }}
          server: weather
          fail-on: none
      - if: steps.checklist.outputs.ready-for-chatgpt == 'true'
        run: echo "Ready for ChatGPT"
```

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `token` | — | Required. Manufact Cloud API key, scoped to the server's organization. Store it as a secret. |
| `server` | — | Required. Server slug (for example `bold-steel-y1vjy`) or server ID, as shown in the Manufact dashboard. |
| `branch` | PR head branch; production on other events | Preview branch whose deployment is checked. Set `branch: ""` to always check production, for example for an external server with no preview deployments. |
| `fail-on` | `failed` | `none`, `failed` or `warnings`. See below. |
| `comment` | `true` | On pull requests, comment the full report. Set `false` to turn it off. |
| `github-token` | `github.token` | Token used to post the comment. |

`fail-on` levels:

| Level | The job fails when |
| --- | --- |
| `none` | The checklist can't run at all (for example, a bad token) |
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

On `pull_request` events, the action comments the report on the pull request, laid out like the checklist page in the Manufact dashboard: the checks score, the last run's results and client readiness, the policy-review banner, and one section per category. Sections with issues start expanded; failing checks show their severity, the affected clients, a policy link where there is one, and how to fix them.

There's one comment per server. Later runs update it rather than adding a new comment. If the checklist can't run at all (for example, the token is wrong), the comment shows the error instead.

Commenting needs the job to have `pull-requests: write`. Without it, the action logs a warning and carries on; the checklist result is unaffected. Pull requests from forks get a read-only token, so there's no comment for those.

The full report also goes to the job log, and the same summary is added to the job summary page.

## Requirements

- A runner with `jq`. GitHub-hosted runners have it. The action installs Node.js 22 itself.
Each release of this action pins the `@mcp-use/cli` version it was tested with.
