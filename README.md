# runsweep action

Which CI runs did a supply-chain incident touch, and which secrets do you rotate first?

- Finds the GitHub Actions jobs that ran a compromised npm package or action in the incident window: AFFECTED, POSSIBLE, UNCHECKED or CLEAN.
- Lists the secrets those jobs could read, confirmed ones first, in the job summary.
- Verifies the [runsweep](https://github.com/runsweep/runsweep) binary before running it: SHA-256 against the release checksums and the build provenance attestation from the release workflow.

```yaml
permissions:
  actions: read
  contents: read
steps:
  - uses: runsweep/action@v1
    with:
      incident: chaindrop-2026-08
```

Pin the action to a commit SHA in production workflows (`uses: runsweep/action@<sha> # v1`); the tag is shown for readability.

## A full workflow

Run it by hand when an incident is announced:

```yaml
name: incident-check
on:
  workflow_dispatch:
    inputs:
      incident: {description: 'Incident id or path to a YAML file', required: true}
permissions:
  actions: read
  contents: read
jobs:
  runsweep:
    runs-on: ubuntu-latest
    timeout-minutes: 60
    steps:
      # only needed for an incident file kept in this repository
      - uses: actions/checkout@v7
        with: {persist-credentials: false}
      - id: scan
        uses: runsweep/action@v1
        with:
          incident: ${{ inputs.incident }}
          fail-on: unchecked
      - if: always() && steps.scan.outputs.report
        uses: actions/upload-artifact@v7
        with: {name: runsweep-report, path: '${{ steps.scan.outputs.report }}'}
```

`runsweep incidents` lists the built-in incidents; `runsweep incidents import <OSV id>` builds a file for a new one. See the [runsweep README](https://github.com/runsweep/runsweep#readme) for the incident format and what the statuses mean.

## Inputs

| Input | Default | |
|---|---|---|
| `incident` | (required) | Built-in incident id, or path to an incident YAML file in the workspace |
| `repos` | current repository | Repositories to scan, `owner/name`, separated by spaces or newlines |
| `org` | | Scan every repository of this organization instead |
| `since`, `until` | incident window | Override the window (RFC3339) |
| `lookback` | `7d` | Also check re-runs of runs created this long before the window (max `30d`) |
| `version` | `0.6.0` | runsweep release to run |
| `token` | `github.token` | Token runsweep scans with |
| `fail-on` | `findings` | `findings`: fail when a job is AFFECTED or POSSIBLE. `unchecked`: also fail when not everything could be checked. `never`: only fail on a runsweep error |

A runsweep error (exit code 2: bad incident, token rejected, nothing could be scanned) always fails the step.

## Outputs

| Output | |
|---|---|
| `exit-code` | `0` nothing found and everything checked, `1` AFFECTED or POSSIBLE jobs, `2` error, `3` not everything checked |
| `report` | Path to the full Markdown report |

The report also goes to the job summary. GitHub limits a summary to 1 MiB: a longer report is cut there, and the full one stays at `report`.

## Token

The default `github.token` scans the current repository, given `actions: read` and `contents: read`. To scan other private repositories or a whole organization, pass a fine-grained token with read-only **Actions**, **Contents** and **Metadata** on them:

```yaml
      - uses: runsweep/action@v1
        with:
          incident: chaindrop-2026-08
          org: my-org
          token: ${{ secrets.RUNSWEEP_TOKEN }}
```

runsweep only reads, and talks only to api.github.com; GitHub Enterprise Server is not supported.

## Requirements

GitHub-hosted runners (Linux, macOS, Windows; x64 and arm64). The action uses `gh`, `bash` and `unzip` or `tar` from the runner image; on a self-hosted runner install `gh` 2.49 or newer.

## License

Apache-2.0
