# Shipshape GitHub Action

[![CI](https://github.com/bytestay/shipshape-action/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/bytestay/shipshape-action/actions/workflows/ci.yml)
[![E2E](https://github.com/bytestay/shipshape-action/actions/workflows/e2e.yml/badge.svg?branch=main)](https://github.com/bytestay/shipshape-action/actions/workflows/e2e.yml)
[![Release](https://img.shields.io/github/v/release/bytestay/shipshape-action)](https://github.com/bytestay/shipshape-action/releases)
[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-shipshape-blue?logo=github)](https://github.com/marketplace/actions/bytestay-shipshape-dockerfile-kubernetes-lint)
[![License: MIT](https://img.shields.io/github/license/bytestay/shipshape-action)](LICENSE)

Lint the **Dockerfiles and Kubernetes manifests** a pull request changes, right on the diff:

- root in the final stage;
- unpinned base images;
- `curl | sh`;
- secrets in ENV/ARG;
- the Kubernetes **Pod Security Standards** (baseline and restricted);
- resources and probes.

Every finding comes with the line, the fix and the standard behind it. Every file gets an A–F grade.

It calls the [Shipshape API](https://shipshape.bytestay.pt) with **your own key**, from RapidAPI or API.market.

- The free plan is 15 requests a month on RapidAPI, and a 7-day trial of 15 calls on API.market.
- One file is one request.

```yaml
name: shipshape
on: pull_request
permissions:
  contents: read
  pull-requests: read
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: bytestay/shipshape-action@v1
        with:
          rapidapi-key: ${{ secrets.RAPIDAPI_KEY }}
```

1. Subscribe to the [Shipshape API on RapidAPI](https://rapidapi.com/bytestay-bytestay-default/api/shipshape-dockerfile-and-kubernetes-lint-api). Basic is $0 with a hard limit, so it never runs up a bill.
2. Add your RapidAPI key as the repository secret `RAPIDAPI_KEY`.
3. Add the workflow above.

### With an API.market key

Subscribe to [Shipshape on API.market](https://api.market/store/bytestay/shipshape) instead (the Free plan is a $0 trial: 15 calls for 7 days). Then store the key as `API_MARKET_KEY` and pass it as `api-market-key`:

```yaml
      - uses: bytestay/shipshape-action@v1
        with:
          api-market-key: ${{ secrets.API_MARKET_KEY }}
```

Set one key, not both. The action calls the marketplace the key belongs to.

## What you get

- **Annotations on the pull request diff:**
  - `high` findings as errors, `medium` as warnings, `low` and `info` as notices.
  - Each one has the rule id, the explanation and the fix.
  - GitHub shows at most 10 per level, so high severity comes first.
- **A job summary** with:
  - a table per file (grade, score, counts, passed);
  - every finding with its fix and a link to the rule;
  - the files that were skipped and why;
  - the quota you have left.
- **A pass/fail result:** the step fails when:
  - a finding is at or above `fail-on` (`high` by default);
  - or a file grades worse than `min-grade`.

## Inputs

| input | default | meaning |
|---|---|---|
| `rapidapi-key` | *empty* | your RapidAPI key; set this or `api-market-key` |
| `api-market-key` | *empty* | your API.market key; set this or `rapidapi-key` |
| `mode` | `changed` | `changed`: only the files the pull request (or push) changed; `all`: every matching file |
| `dockerfile-paths` | `**/Dockerfile,**/*.Dockerfile,**/Dockerfile.*,**/Containerfile` | comma-separated globs |
| `kubernetes-paths` | `**/*.yaml,**/*.yml` | comma-separated globs |
| `exclude-paths` | `.github/**,**/templates/**,**/docker-compose*.y*ml,**/compose*.y*ml` | comma-separated globs to skip |
| `fail-on` | `high` | `high`, `medium`, `low`, `info` or `never` |
| `min-grade` | *empty* | also fail when any file grades worse than this (A–F) |
| `ignore` | *empty* | comma-separated rule ids, e.g. `DF-017,K8S-025` |
| `max-requests` | `10` | cap per run; files over it are listed as not analysed |
| `on-quota-exceeded` | `warn` | `warn` or `fail` when the monthly quota runs out |
| `github-token` | `${{ github.token }}` | used to list the files a pull request changed |

## Outputs

| output | meaning |
|---|---|
| `passed` | `true` when every analysed file passed (and met `min-grade`) |
| `worst-grade` | the worst grade among analysed files (empty when none was analysed) |
| `lowest-score` | the lowest score among analysed files |
| `findings-total` | total number of findings across analysed files |
| `requests-used` | API requests used by this run |
| `quota-remaining` | requests left this month, from the `x-ratelimit-requests-remaining` header; empty when the marketplace doesn't send it |

## Saving requests

One file is one request. A multi-document YAML (`---`) is a single request, up to 500 documents.

Before anything is sent, these files are skipped locally at no cost:

- Dockerfiles without `FROM`;
- YAML without `apiVersion` and `kind` (not a Kubernetes manifest);
- Helm templates (anything with `{{ }}`);
- files over 512 KiB.

With `mode: changed`, a typical pull request costs 1 to 3 requests.

When fewer than 3 are left, the action warns you.

## When something goes wrong

| situation | what the action does |
|---|---|
| no key on a pull request from a fork (forks don't get secrets) | warning, step passes |
| key invalid or not subscribed (401/403, or 400 from API.market) | error with the subscribe link, step fails |
| file can't be parsed (422) | error annotation on that file, step fails |
| quota exhausted (429) | `on-quota-exceeded: warn` gives a warning and the step passes; `fail` fails it |
| API down or timing out | one retry, then a warning: **your build is never failed because of us** |

Use `on: pull_request`, not `pull_request_target`:

- with `pull_request_target`, a pull request from a fork would run with your secrets;
- skipping fork pull requests is the safe default.

## Privacy

- The content of each file is sent to the Shipshape API, parsed in memory and discarded.
- Nothing is stored, and Dockerfiles and manifests are never logged.
- Your key is masked in the logs.

See the [privacy policy](https://shipshape.bytestay.pt/privacy.html).

## Limits

- Helm charts must be rendered first, for example `helm template … > rendered.yaml`, with `rendered.yaml` matched by `kubernetes-paths`.
- NetworkPolicies, PodDisruptionBudgets and RBAC are not checked, because they need several objects at once.
- A `USER` set inside a third-party base image can't be seen, and the finding says so.
- The runner needs `bash`, `curl`, `jq` and `git`. GitHub-hosted runners have all four.

## Contributing

Issues and pull requests are welcome: see the [contributing guide](https://github.com/bytestay/.github/blob/main/CONTRIBUTING.md).

To work on the action:

```bash
bats tests/                                 # unit tests (curl is faked, no key needed)
shellcheck -x scripts/*.sh tests/bin/curl
actionlint
```

The end-to-end test (`.github/workflows/e2e.yml`) runs the action against the real API monthly, through each marketplace: with the `RAPIDAPI_KEY` secret and with the `API_MARKET_KEY` secret.

## History

Formerly Hybrid Forge Lab. This repository continues `hybridforgelab/shipshape-action` under the ByteStay name since November 2026, so the older commits come from there.

## Licence

MIT.

Shipshape itself is a paid API with a free plan, run by [ByteStay](https://bytestay.pt).
