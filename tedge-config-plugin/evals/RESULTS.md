# tedge-config-plugin — Evaluation Results

A record of how the skill performs against its eval suite, so consumers can
judge its quality. The suite itself (prompts + assertions) is in
[`evals.json`](./evals.json) and is fully reproducible.

## Methodology

- **3 test cases**, each a realistic "write me a config plugin for X" request
  that stresses a different part of the spec:
  - `nginx` — the standard validate → backup → apply → verify → rollback flow
    with a real validator (`nginx -t`) and reload.
  - `agent.env` — a config type with **no native validator** (tests that the
    plugin does a sensible KEY=VALUE check instead of inventing a fake one).
  - `mosquitto` — **multi-fragment aggregation** (`get` must return the merged
    effective config from `mosquitto.conf` + `conf.d/*.conf`).
- Each case was run **with the skill** and, as a baseline, **without the skill**
  (same prompt, no skill) to isolate the skill's contribution.
- **Fair-baseline control:** the no-skill baseline was allowed to use its own
  knowledge and the public internet (WebSearch / WebFetch, including the public
  thin-edge GitHub repo), but was **forbidden from reading any local copy of the
  thin-edge docs** on the test machine. This removes a confound from an earlier
  run where a baseline had found a local docs clone on disk.
- Each output plugin was graded against **10 objective assertions** — 8 shared
  contract checks (the six-subcommand `list/get/prepare/set/verify/rollback`
  interface, `--work-dir` handling, exit-code/stderr discipline, install/test
  docs) plus 2 case-specific checks.

## Headline result

| Metric | With skill | Without skill (fair baseline) | Delta |
|--------|-----------|-------------------------------|-------|
| **Pass rate** | **100%** (30/30 assertions) | 43% (13/30) | **+57 pts** |

Per case (assertions passed):

| Test case | With skill | Baseline |
|-----------|-----------|----------|
| nginx | 10/10 | 3/10 |
| agent.env | 10/10 | 0/10 |
| mosquitto | 10/10 | 10/10 |

**Where the skill helps most:** pinning the exact six-subcommand contract.
Without the skill, the model reliably picked the *wrong mechanism* — it built
the declarative `tedge-configuration-plugin.toml` file-entry approach with an
ad-hoc interface (e.g. `set`/`get`/`list`/`remove` or
`validate`/`install`/`apply`/`restart`/`snapshot`), omitting the
`prepare`/`verify`/`rollback` + `--work-dir` lifecycle that `tedge-agent`
actually drives, and installing to the wrong location.

**The one baseline that matched (mosquitto)** did so legitimately: that run
proactively fetched the spec from the public thin-edge GitHub repo and adapted
the reference `lighttpd` example. In other words, a no-skill agent *can* reach
parity — but only if it happens to find and read the exact public spec. The
skill's value is making the correct contract the *guaranteed* outcome rather
than a lucky one.

## Reproducing

Invoke the skill on each prompt in `evals.json` and grade the generated plugin
against the assertions listed for that eval, or run the suite through the
skill-creator eval harness.

---
*2 iterations · clean-baseline numbers from iteration 2 · 2026-06-17 · model: Claude Opus 4.8 · 1 run per configuration.*
