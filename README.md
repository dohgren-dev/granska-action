# Granska Audit — the verified audit, in CI

A GitHub Action that runs [Granska](https://github.com/dohgren-dev/granska) against your repo on every pull request: real scanner CLIs (SAST, secrets, dependency CVEs, IaC) as ground truth, then LLM review lanes that triage, hunt logic/security bugs the scanners miss, and **adversarially verify** every high-severity finding before it ships. Findings gate the merge, upload to the code-scanning tab, and land as one sticky PR comment.

Free for public repos.

## Usage

```yaml
name: granska
on:
  pull_request:
permissions:
  contents: read
  security-events: write   # upload SARIF to code-scanning
  pull-requests: write     # sticky PR comment
  actions: read            # private repos
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - id: granska
        uses: dohgren-dev/granska-action@v2.0.0
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          target: .
          scope: quick        # quick on PRs; run `full` on a schedule
          fail-on: high
          max-cost: '5'       # USD hard cap; a quick audit is ~$2/run
      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        continue-on-error: true
        with:
          sarif_file: ${{ steps.granska.outputs.sarif-path }}
```

See the [full CI guide](https://github.com/dohgren-dev/granska/blob/main/docs/CI.md) for the sticky-comment step, scheduled `full` audits, waivers/baseline, and measured per-run cost.

## Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `anthropic-api-key` | — (required) | Anthropic API key for the headless audit. |
| `target` | `.` | Path to audit, relative to the workspace. |
| `scope` | `quick` | `quick` \| `full` \| `dast`. |
| `fail-on` | `high` | Lowest vetted severity that fails the job. |
| `soft-fail` | `false` | Gate as normal but never fail the job (report-only rollout). |
| `max-cost` | — | Optional USD hard cap on API spend. |
| `baseline` | `.granska/baseline.yml` | Waiver/baseline file, relative to target. |

## Outputs

`grade`, `exit-code`, `sarif-path`, `report-html`, `cost-usd`, `worst-severity`.

## How it works

This action is a thin wrapper: it runs the pinned `ghcr.io/dohgren-dev/granska` container image (Node + Claude Code CLI + a version-locked scanner battery) whose entrypoint drives Granska's audit pipeline headlessly. The engine lives in the [granska](https://github.com/dohgren-dev/granska) repo; this repo exists only to make the action marketplace-listable.

## License

MIT — see [LICENSE](./LICENSE).
