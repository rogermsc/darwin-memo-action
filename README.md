# darwin-memo-action

Settle a [darwin-memo](https://github.com/rogermsc/darwin-memo) CI
lesson store from test results. This composite action wraps the
`darwin-memo settle-ci` subcommand: it diffs two junit XML reports per
test id, settles every ticket with the measured change in passing
tests, runs one tick (upkeep, expiry, consolidation), and optionally
commits the updated store back to the branch. Lessons that keep
shipping green survive. Lessons that keep breaking builds die. Nobody
reviews the store.

The wiring guide lives in the main repo:
[docs/integrations/ci-lesson-store.md](https://github.com/rogermsc/darwin-memo/blob/main/docs/integrations/ci-lesson-store.md).

## Quickstart

Produce junit XML at the base commit and at the head or merge commit
(`pytest --junitxml` at each ref; most CI setups archive these
already), then:

```yaml
- name: Settle memory tickets
  uses: rogermsc/darwin-memo-action@v1
  with:
    junit-base: /tmp/base.xml
    junit-head: /tmp/head.xml
    scale: "2.0"
  env:
    PR_BODY: ${{ github.event.pull_request.body }}
```

Every `darwin-memo-ticket: <id>` line in `PR_BODY` settles with the
measured delta. The job needs `permissions: contents: write` and a
branch checkout (for example `actions/checkout` with `ref: main`) when
`commit-store` is on, which it is by default. The store is a single
JSON file with no locking, so run one settler at a time:

```yaml
concurrency:
  group: darwin-memo-settle
  cancel-in-progress: false
```

For the full measure-then-settle workflow this action slots into, see
[`.github/workflows/memory.yml`](https://github.com/rogermsc/darwin-memo/blob/main/.github/workflows/memory.yml)
in the main repo, which runs this loop on darwin-memo itself.

## Installation note

The action installs the CLI with `pip install` using the
`package-spec` input, which defaults to `darwin-memo>=0.5.0`:
`settle-ci` shipped in 0.5.0, so that is the version floor. To
install from the repository instead of PyPI, point `package-spec` at
a pinned commit:

```yaml
with:
  package-spec: git+https://github.com/rogermsc/darwin-memo@<commit-sha>
```

Pinning a sha gives reproducible installs; this action's own test
workflow installs that way so it never depends on a PyPI release.

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `store-path` | `.darwin-memo/lessons.json` | Ledger file. Created on first use; never created or touched on abstention. |
| `junit-base` | (none) | junit XML from the test run at the base commit. Primary mode: set both junit inputs. |
| `junit-head` | (none) | junit XML from the test run at the head or merge commit. |
| `passes-before` | (none) | Fallback mode: raw passing count at the base commit. |
| `passes-after` | (none) | Fallback mode: raw passing count at the head commit. |
| `scale` | `1.0` | `resource_scale` applied to settle deltas. |
| `expire-after` | `50` | Ticks before unsettled tickets expire at delta zero. |
| `ticket` | (none) | A `darwin-memo-ticket` id (12 hex chars) to settle, typically parsed from the merge commit message or PR body by the caller. When empty, ids are read from the `PR_BODY` environment variable instead. |
| `detail` | the run URL | Settlement detail recorded with each settled ticket. |
| `commit-store` | `true` | Commit and push the updated store (plus `flaky.json` when present) with a `memory: settle and tick` message. Skipped on abstention. |
| `python-version` | `3.12` | Passed to `actions/setup-python`. |
| `package-spec` | `darwin-memo>=0.5.0` | pip requirement used to install the CLI. See the installation note above. |

Exactly one mode must be active: either both junit inputs or both
pass-count inputs. The CLI rejects anything else.

## Outputs

| Output | Description |
| --- | --- |
| `result` | `settled` when the store was settled, ticked, and saved; `abstained` when the run measured nothing and the store was left untouched. Any other failure fails the step. |
| `delta` | The measured change in passing tests, as a float string (for example `-1.0`). Only set when `result` is `settled`. |
| `summary` | The one-line JSON object from `settle-ci`: mode, delta, per-test transitions, quarantined tests, and which tickets settled. On abstention it carries the reason. |

## Abstention is a skip, never a failure

A run with no parseable junit XML, zero collected tests, or a
collection error measured nothing, so settling it at zero would be a
lie. In those cases `settle-ci` exits with code 3, the action maps
that to `result=abstained`, and the step succeeds with the store
untouched: no settle, no tick, no save, no commit. Distinguish the
cases downstream with the output:

```yaml
- name: Warn on abstention
  if: steps.settle.outputs.result == 'abstained'
  run: echo "::warning::Measurement broken; tickets stay open."
```

Abstention means the broken measurement surfaces instead of silently
becoming a fake delta. Tickets stay open and expire at delta zero
after `expire-after` ticks if CI never reports back.

## Flaky tests quarantine themselves

Per-test flip history lives in `flaky.json` next to the store. A test
whose recent history changes direction three times inside a sliding
ten-observation window is excluded from settlement deltas until the
flips slide out of the window; stabilizing on either side is the only
exit. Quarantined tests are reported in the `summary` output, never
settled. When `commit-store` is on, `flaky.json` is committed
alongside the store so the quarantine survives between runs.

Honest caveats: quarantine catches repeat offenders, so a brand-new
flake's first flip still lands on whatever tickets are open, and a PR
that deletes failing tests still moves the delta. That second one is a
review problem, not a measurement problem.

## Fallback mode: raw pass counts

For ecosystems without junit XML, set `passes-before` and
`passes-after` and leave the junit inputs empty. This is the degraded
path: no per-test attribution, no quarantine, and no infra detection
beyond requiring both counts explicitly. Never default a missing count
to zero; if the run produced no count, skip the step entirely.

## License

MIT, same as darwin-memo.
