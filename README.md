# kagami-actions

[![test](https://github.com/zunda-inc/kagami-actions/actions/workflows/test.yml/badge.svg)](https://github.com/zunda-inc/kagami-actions/actions/workflows/test.yml)
[![actionlint](https://github.com/zunda-inc/kagami-actions/actions/workflows/actionlint.yml/badge.svg)](https://github.com/zunda-inc/kagami-actions/actions/workflows/actionlint.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE.md)

A GitHub Action to run [kagami](https://github.com/zunda-inc/kagami), which checks whether there are
any publicly disclosed vulnerabilities in a list of defined software.

kagami is for the software that package audits cannot see in a lockfile: a PHP runtime installed on
a server, Docker Engine on a host, or a controller pinned by its image tag in a Kubernetes manifest.
Advisories are published every day, so run the action on a schedule to find out when one is
published for the software you run.

## Usage

Write the software you run into `kagami.yaml` at the root of your repository. See the
[kagami README](https://github.com/zunda-inc/kagami#configuration) for the format.

```yaml
targets:
  - name: k8s.io/ingress-nginx
    version: 1.11.4
    ecosystem: go
    advisory_type: github
```

Then add a workflow that runs the action:

```yaml
name: kagami

on:
  schedule:
    - cron: '0 0 * * *'
  workflow_dispatch:

permissions:
  contents: read

jobs:
  kagami:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          persist-credentials: false
      - uses: zunda-inc/kagami-actions@v1
        with:
          fail-on-severity: high
```

The report is written to the job log, and the job fails when kagami finds an advisory of `high` or
higher. Without `fail-on-severity`, kagami exits with `0` whether or not it finds advisories, so the
job does not fail.

> [!TIP]
> Pin the actions to commit SHAs, like `zunda-inc/kagami-actions@<commit SHA> # v1.0.0`, so that
> your workflow keeps running the code you have reviewed. Renovate and Dependabot can keep the SHAs
> up to date.

## Inputs

All inputs are optional.

- `config-file`: the config file to read (default: `kagami.yaml`, then `kagami.yml`)
- `output-format`: `text`, `json`, `yaml` or `simple-markdown-text`
- `log-level`: `error`, `warn`, `info`, `debug` or `trace`
- `fail-on-severity`: `unknown`, `low`, `medium`, `high` or `critical`, the lowest severity that makes
  the step fail (not set by default)
- `github-token`: the token for kagami to read the advisories through the GitHub API (default:
  `${{ github.token }}`, see [GitHub API token](#github-api-token))

`output-format`, `log-level` and `fail-on-severity` override the settings in the config file, as the
options of kagami do.

## Outputs

| Output        | Description                                |
|---------------|--------------------------------------------|
| `stdout`      | The report, which kagami writes to stdout. |
| `stderr`      | The logs, which kagami writes to stderr.   |
| `stdout-file` | The path to a file that holds `stdout`.    |
| `stderr-file` | The path to a file that holds `stderr`.    |

> [!IMPORTANT]
> Read the report from `stdout-file` in your scripts. Embedding `${{ steps.<id>.outputs.stdout }}`
> in `run` lets the shell interpret the text of the advisories, and a large report does not fit in
> an environment variable.

For example, to write the report to the job summary even when the step fails:

```yaml
      - uses: zunda-inc/kagami-actions@v1
        id: kagami
        with:
          output-format: simple-markdown-text
          fail-on-severity: high
      - name: Write the report to the job summary
        if: ${{ !cancelled() && steps.kagami.outputs.stdout-file != '' }}
        env:
          REPORT_FILE: ${{ steps.kagami.outputs.stdout-file }}
        run: cat "$REPORT_FILE" >> "$GITHUB_STEP_SUMMARY"
```

## How it works

1. The action picks the kagami binary for the runner from the
   [releases of kagami](https://github.com/zunda-inc/kagami/releases).
2. It restores the binary from the cache, or downloads it.
3. It verifies the SHA-256 checksum of the binary against the one written in [`action.yml`](action.yml).
   A cached binary that does not match is downloaded again, and a downloaded binary that does not
   match fails the step.
4. It runs kagami in the working directory, so check out your repository before the action.

The action runs on Linux (x64 and ARM64), macOS (x64 and ARM64) and Windows (x64 and ARM64). On
Linux, kagami needs glibc 2.38 or newer on x64 and 2.39 or newer on ARM64, so it runs on
`ubuntu-24.04` and `ubuntu-24.04-arm` but not on `ubuntu-22.04` or `ubuntu-22.04-arm`. Self-hosted
runners need `bash`, `curl` and `openssl`, which GitHub-hosted runners have.

## Versioning

kagami-actions has its own version numbers, independent of kagami's. Each release runs a fixed
version of kagami, which its release notes mention. A major version tag such as `v1` follows the
latest release of that major version, and the other tags are never moved.

## GitHub API token

kagami reads the advisories through the GitHub REST API. The action passes `github-token` to kagami
as the `GH_TOKEN` environment variable, so the `GITHUB_TOKEN` of the workflow is used by default.

> [!IMPORTANT]
> kagami only reads public data, so the token needs no extra scopes or permissions.

## Development

CI runs these workflows on every pull request:

- [`test.yml`](.github/workflows/test.yml) runs the action with [`test/kagami.yml`](test/kagami.yml)
  and checks its outputs.
- [`actionlint.yml`](.github/workflows/actionlint.yml) checks the workflows with
  [actionlint](https://github.com/rhysd/actionlint).

To update kagami, change the version in the URLs in `action.yml` and the SHA-256 checksums of the
binaries. GitHub shows the checksums on the release page, and this command lists them:

```sh
gh api repos/zunda-inc/kagami/releases/tags/<version> --jq '.assets[] | "\(.digest | ltrimstr("sha256:"))  \(.name)"'
```

To release, publish a GitHub release with a new version tag such as `v1.2.3`, mention the version of
kagami in its notes, and move the major version tag (`v1`) to it.

The actions are pinned to commit SHAs, and [Renovate](https://docs.renovatebot.com/) keeps them up
to date. See [`renovate.json`](renovate.json) for its settings.

## Security

See [SECURITY.md](SECURITY.md) for how to report a vulnerability in kagami-actions.

## License

kagami-actions is released under the [MIT License](LICENSE.md).

Copyright (c) 2026 ZUNDA Inc.
