# OffSeq Radar scan – GitHub Action

[![CI](https://github.com/offseq/radar-scan-action/actions/workflows/ci.yml/badge.svg)](https://github.com/offseq/radar-scan-action/actions/workflows/ci.yml)

Checks an SBOM or lockfile against [OffSeq Threat Radar](https://radar.offseq.com) on every push or pull request:

- writes a findings table to the job summary, riskiest first (CISA KEV, then risk score, EPSS, CVSS), with the version to upgrade to;
- uploads a SARIF report to GitHub code scanning, so findings show up under **Security → Code scanning**;
- optionally fails the job when a version-confirmed finding reaches the level you choose.

It calls [`POST /api/v1/scan`](https://radar.offseq.com/api-docs), once per run with `format=all`, so a run costs one API request; the job summary and the SARIF report both come from that response. The file is sent under its repository path, so code-scanning alerts link to it.

## Usage

```yaml
name: Radar
on: [push, pull_request]

permissions:
  contents: read
  security-events: write   # SARIF upload

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: offseq/radar-scan-action@v1
        with:
          api-key: ${{ secrets.RADAR_API_KEY }}
          file: package-lock.json
          fail-on: high
```

Get an API key in [Console → Settings](https://radar.offseq.com/console) and save it as the `RADAR_API_KEY` repository secret.

## Inputs

| Input | Default | |
|---|---|---|
| `api-key` | – | Radar API key. On fork and Dependabot pull requests, which get no secrets, an empty key skips the scan with a warning (`skipped` is `true`); anywhere else an empty key fails the step. |
| `file` | – | SBOM or lockfile to scan. Required. |
| `fail-on` | empty | `critical`, `high`, `medium`, `low`, `kev` or `exploited`. Only version-confirmed findings count. Empty never fails. |
| `fail-on-incomplete` | `true` | Fail when the gate could not evaluate every package, e.g. over the server's candidate cap. The job summary lists the packages. |
| `upload-sarif` | `true` | Upload SARIF to code scanning. Needs `security-events: write`. |
| `api-url` | `https://radar.offseq.com` | Radar base URL. |

## Outputs

| Output | |
|---|---|
| `gate-tripped` | `true` when `fail-on` matched. |
| `gate-incomplete` | `true` when the gate could not evaluate every package. |
| `sarif-file` | Path of the SARIF report. |

## Supported files

CycloneDX (JSON and XML), SPDX (JSON and tag-value), purl-per-line lists, `package-lock.json` / `npm-shrinkwrap.json`, `yarn.lock` (v1 and berry), `pnpm-lock.yaml`, `poetry.lock`, `uv.lock`, `Cargo.lock`, `go.mod`, `go.sum`, `composer.lock`, `Gemfile.lock` and pinned `requirements*.txt`. The format is detected from the file name, then the content.

The number of packages per scan follows your plan's batch cap; see [pricing](https://radar.offseq.com/pricing).

## Without the Action

```sh
jq -n --rawfile content package-lock.json '{content: $content, filename: "package-lock.json"}' |
  curl -sS "https://radar.offseq.com/api/v1/scan?failOn=high" \
    -H "X-API-Key: $RADAR_API_KEY" -H "Content-Type: application/json" --data-binary @-
```

## Versioning

`@v1` follows the latest `v1.x.y` release; pin a full tag such as `@v1.0.0` or a commit SHA for reproducible builds.

## License

[MIT](LICENSE)
