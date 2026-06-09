# bagx-action

GitHub Action wrapper for [bagx](https://github.com/rsasaki0109/bagx) — evaluate ROS bags in CI, gate on benchmark regressions, post PR readiness diffs, and publish shields.io badges.

Thin composite action: `pip install bagx` + CLI calls. No Docker image.

## Features

- **Benchmark suites** — run `bagx benchmark suite.json` and fail on regressions
- **PR readiness diff** — evaluate a bag, compare against a cached baseline from `main`, post markdown via sticky PR comment
- **Readiness badge** — optional gist publish for [shields.io endpoint badges](https://shields.io/badges/endpoint-badge)

## Quick start — benchmark gate

```yaml
name: bag quality
on: [push, pull_request]

jobs:
  bagx:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: rsasaki0109/bagx-action@v1
        with:
          suite: benchmarks/my_suite.json
          fail-on: warning
```

## PR readiness check

Prime a baseline on `main`, then diff on pull requests:

```yaml
name: bagx baseline
on:
  push:
    branches: [main]

jobs:
  prime:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: rsasaki0109/bagx-action@v1
        with:
          bags: bag/recording.db3
          comment: "false"
      - run: cp .bagx-action/current.json .bagx-action/baseline.json
      - uses: actions/cache/save@v4
        with:
          path: .bagx-action/baseline.json
          key: bagx-eval-baseline-${{ github.sha }}
```

```yaml
name: bagx PR check
on:
  pull_request:
    paths: ["**/*.db3", "**/*.mcap"]

permissions:
  contents: read
  pull-requests: write

jobs:
  readiness:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: rsasaki0109/bagx-action@v1
        with:
          bags: bag/recording.db3
          fail-on: warning
          comment: "true"
```

The action restores `baseline.json` from `actions/cache` keyed on the PR base SHA (`bagx-eval-baseline-<base.sha>`). Save `current.json` from the baseline workflow under that key.

## Readiness badge via Gist

```yaml
- uses: rsasaki0109/bagx-action@v1
  with:
    bags: bag/recording.db3
    badge-gist-id: ${{ vars.BAGX_BADGE_GIST_ID }}
    badge-token: ${{ secrets.BAGX_BADGE_TOKEN }}
    comment: "false"
```

Reference the gist raw URL in your README:

```markdown
![bag readiness](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/you/<gist-id>/raw/badge.json)
```

`badge-token` needs gist write access (a PAT or fine-grained token). The default `GITHUB_TOKEN` cannot update arbitrary gists.

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `bagx-version` | `>=0.5.0` | pip install spec. Set empty to use a pre-installed bagx. |
| `suite` | — | Benchmark manifest JSON path |
| `bags` | — | Comma-separated bag paths (first bag used for diff/badge) |
| `baseline-path` | — | Explicit baseline eval JSON (skips cache restore) |
| `fail-on` | `warning` | Severity floor for benchmark / diff exit code |
| `comment` | `true` | Post sticky PR comment with markdown diff |
| `comment-header` | `bagx-readiness` | Sticky comment grouping key |
| `baseline-cache-key` | `bagx-eval-baseline` | Cache key prefix for baseline JSON |
| `badge-gist-id` | — | Gist ID for badge JSON publish |
| `badge-file-name` | `badge.json` | Filename inside the gist |
| `badge-label` | — | Override shields.io badge label |
| `badge-token` | `GITHUB_TOKEN` | Token with gist write access |
| `rules` | — | Custom rules file or plugin name |
| `working-directory` | `.bagx-action` | Output directory for reports |
| `python-version` | `3.12` | Python version when installing bagx |

## Outputs

| Output | Description |
| --- | --- |
| `report-path` | Primary bag eval JSON path |
| `benchmark-passed` | `true` when suite passed or no suite was run |
| `diff-markdown-path` | Generated diff markdown path |
| `badge-path` | Generated shields.io badge JSON path |

## Marketplace

Published on GitHub Marketplace as **bagx** (composite action).

## License

MIT — see [LICENSE](LICENSE).
