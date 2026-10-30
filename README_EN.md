**中文** · [README.md](./README.md)

> [!WARNING]
> **⚠️ This tool has been merged into the [doctors](https://github.com/huanweide/doctors) monorepo and is no longer maintained here.**
>
> The source now lives at [`doctors/packages/ghadoctor/`](https://github.com/huanweide/doctors/tree/main/packages/ghadoctor). Inside doctors it runs as the `doctors gha` subcommand (or `node packages/ghadoctor/index.js`). This repository no longer accepts changes — please use <https://github.com/huanweide/doctors>.

**🔧 Part of the [huanweide zero-dependency CLI family](https://github.com/huanweide/family-showcase) (18 single-file tools). Use it standalone, as a subcommand inside the [doctors](https://github.com/huanweide/doctors) monorepo, or run everything together via [enghealth](https://github.com/huanweide/enghealth).**

# ghadoctor

[![CI](https://github.com/huanweide/ghadoctor/actions/workflows/ci.yml/badge.svg)](https://github.com/huanweide/ghadoctor/actions/workflows/ci.yml)
[![Version](https://img.shields.io/github/package-json/v/huanweide/ghadoctor)](https://github.com/huanweide/ghadoctor)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue)](LICENSE)
[![Stars](https://img.shields.io/github/stars/huanweide/ghadoctor)](https://github.com/huanweide/ghadoctor/stargazers)

**ghadoctor is an open-source, zero-dependency, single-file CLI that audits your GitHub Actions workflows for security & config smells (unpinned actions, script injection, over-broad permissions, `pull_request_target` checkout, missing `permissions`/`timeout`/`concurrency`). One command, no install, with a 0–100 health score and a CI gate (exit code 0 / 2 / 3). See [Install and run](#install-and-run) and [Options](#options).**

← [Back to the zero-dependency CLI family overview](https://github.com/huanweide/family-showcase) · [Hub: doctors](https://github.com/huanweide/doctors)

*Part of the huanweide zero-dependency CLI family, focused on CI-config hygiene (alongside repodoctor / docdoctor / commitdoctor / reldoctor as git-layer governance tools).*

> **Differentiation**: actionlint / zizmor are heavy tools (need install / compile / config); ghadoctor is zero-dependency, single-file, strongly deterministic, runs out of the box, and ships a health score + CI gate — filling the "lightweight zero-dependency" gap (consistent with the family's long-standing pattern: heavy tools occupy the space, so we build the zero-dependency lightweight version).

## Who should use this

- Any team using GitHub Actions that wants a one-command audit of CI security/config smells.
- Projects that want a CI gate on workflow hygiene without installing/compiling a heavy linter.
- Anyone who wants a deterministic, dependency-free check.

## Who should not use this

- If you already run actionlint/zizmor and want their deeper semantic analysis — ghadoctor is deliberately lighter.
- Note: this repo is deprecated — prefer `doctors gha` for new setups.

## Install and run

Zero dependencies, only Node.js (>=18, tests use `node --test`). No `npm install`:

```bash
node index.js [--root <dir>]
```

## Capabilities

Scans `<root>/.github/workflows/*.yml|*.yaml` for 8 categories of CI security / config smells:

| Rule | Severity | What it catches |
| --- | --- | --- |
| `unpinned-action` | HIGH | action reference not pinned to a 40-char commit SHA (`@v4`/`@main`/`@master`/`@latest`/`@head`/`@stable`), can be repointed upstream to malicious code |
| `script-injection` | HIGH | a `run` step directly interpolates untrusted input (`${{ github.event.* }}` etc.) into a shell — script-injection risk |
| `broad-permissions` | HIGH | `permissions: write-all` grants the token repo-wide read/write |
| `pr-target-checkout` | HIGH | `pull_request_target` trigger with a checkout of the PR head, running untrusted code with a privileged token |
| `missing-permissions` | MED | no explicit `permissions` declared, default-inherits repo / org permissions (often too broad) |
| `hardcoded-secret` | MED | a likely hardcoded secret / password in the workflow (should use `${{ secrets.* }}`) |
| `no-timeout` | LOW | `timeout-minutes` not set, a job may hang indefinitely |
| `no-concurrency` | LOW | `concurrency` not declared, concurrent triggers may conflict |

Health score: each file starts at 100 and deducts by severity (HIGH −15 / MED −8 / LOW −4), floored at 0. Overall health score = average of per-file scores.

## Options

```
ghadoctor [--root <dir>]           scan the current/specified dir's .github/workflows
ghadoctor --json                   output pure JSON (no other text; gate via exit code)
ghadoctor --fail-on-high           exit 2 if any HIGH issue exists
ghadoctor --max-high N             exit 2 if HIGH exceeds N
ghadoctor --max-medium N           exit 2 if MEDIUM exceeds N
ghadoctor --max-issues N           exit 2 if total issues exceed N
ghadoctor --min-score N            exit 2 if overall health score below N
ghadoctor --quiet                  print only the summary
ghadoctor --help                   help
```

## Sample output

```
ghadoctor · GitHub Actions config hygiene

.github/workflows/ci.yml
  [HIGH] L12 unpinned-action: actions/checkout@v4 not pinned to SHA
  [MED]  no explicit permissions declared, default-inherits repo/org permissions
  health score: 77

summary: scanned 1 file, 2 issues (1 HIGH / 1 MED / 0 LOW)
overall health score: 77
CI gate: PASS
```

> Three-state gate (frozen in rounds 9.9 / 10.2 / 11.1): `PASS`(0) / `FAIL`(2, threshold configured and unmet) / `INCONCLUSIVE`(3, no threshold configured or scan input empty). **An empty scan never gives a full score** — when no workflow file is found, the health score shows "cannot evaluate", `summary.basisComplete=false`, and exit code 3, rather than a fake "100 PASS". This matches the family gate-tool contract.

## CI integration

```yaml
# .github/workflows/ci-doctor.yml
name: ci-doctor
on: [push, pull_request]
permissions: {}
jobs:
  ghadoctor:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
      - uses: actions/setup-node@a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0
        with:
          node-version: 22
      - run: node index.js --fail-on-high --min-score 90
```

## Constraints

- Pure `fs` read-only, no execution, no network, no YAML-parsing library (line-scanner + light indentation tracking, strongly deterministic).
- Output forbids emoji, so it renders consistently across terminal / CI logs on any platform.
- `--json` mode outputs only pure JSON; the gate result is expressed via the process exit code: `0`=pass / `2`=unmet / `3`=conclusion untrustworthy (no threshold configured or empty input).

## Test

Needs Node.js >=18 (tests use `node --test`):

```bash
node --test test.js && node test/smoke-cli.js
```

---

## Author

Maintained by **huanweide** · [GitHub](https://github.com/huanweide)

## CI gate usage

Runs out of the box in CI: run it in your pipeline, it outputs the health score and severity; if an item is unmet it ends with a non-zero exit code, blocking the problematic commit (see "Quick start" above for parameters).

## License

See [LICENSE](LICENSE).

---

## Portability · zero-dependency promise

- Zero runtime dependencies: uses only one Node built-in module, no `npm install`.
- Single file: the whole tool is one `index.js`, copy it into any repo and it works.
- Offline / cross-platform: Node 18+ is enough, no network, no OS preference.
- One command: `node index.js --root ./src` (see Usage above).

← [Back to the zero-dependency CLI family overview](https://github.com/huanweide/family-showcase)
