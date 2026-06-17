# tedge-config-plugin — Evaluation Results

A record of how the skill performs against its eval suite, so you can judge its
quality and reproduce the numbers yourself. The suite (prompts + assertions) is
in [`evals.json`](./evals.json) and is fully reproducible — see
[Reproduce these results yourself](#reproduce-these-results-yourself).

## What the skill buys you: with skill vs no skill

This is the comparison that matters if you're deciding whether to install the
skill: the same requests, answered by Claude **with** the skill versus **without**
any skill.

| Metric | With skill | Without skill | Delta |
|--------|-----------|---------------|-------|
| **Pass rate** | **100%** (30/30 assertions) | **43%** (13/30) | **+57 pts** |
| Speed | ~3× faster | slower (more dead-ends) | — |

Per case (assertions passed, out of 10):

| Test case | With skill | Without skill |
|-----------|-----------|---------------|
| `nginx` | 10/10 | 3/10 |
| `agent.env` | 10/10 | 0/10 |
| `mosquitto` | 10/10 | 10/10 |

**Why the gap:** without the skill, the model reliably picks the *wrong mechanism*.
It builds the declarative `tedge-configuration-plugin.toml` file-entry approach
with an ad-hoc interface (e.g. `set`/`get`/`list`/`remove` or
`validate`/`install`/`apply`), omitting the `prepare`/`verify`/`rollback` +
`--work-dir` lifecycle that `tedge-agent` actually drives, and installing to the
wrong location. The one no-skill case that matched (`mosquitto`) did so only
because that run happened to fetch and adapt the public spec — which is exactly
what the skill *guarantees* rather than leaves to luck.

> **On the numbers.** Pass rate is graded against the fixed, objective assertions
> in `evals.json`, so it is directly comparable across runs. The speed figure is
> reported qualitatively: every no-skill run was substantially slower (it burned
> time exploring the wrong design), but exact time/token deltas vary with the
> measurement harness and aren't quoted as precise numbers here.

## Methodology

- **3 test cases**, each a realistic "write me a config plugin for X" request that
  stresses a different part of the spec:
  - `nginx` — the standard validate → backup → apply → verify → rollback flow with
    a real validator (`nginx -t`) and reload.
  - `agent.env` — a config type with **no native validator** (tests that the plugin
    does a sensible KEY=VALUE check instead of inventing a fake one).
  - `mosquitto` — **multi-fragment aggregation** (`get` must return the merged
    effective config from `mosquitto.conf` + `conf.d/*.conf`).
- Each generated plugin is graded against **10 objective assertions** — 8 shared
  contract checks (the six-subcommand `list/get/prepare/set/verify/rollback`
  interface, `--work-dir` handling, exit-code/stderr discipline, install/test
  docs) plus 2 case-specific checks.
- **Fair-baseline control:** the no-skill baseline may use its own knowledge and
  the public internet (including the public thin-edge GitHub repo) but is
  **forbidden from reading any local copy of the thin-edge docs** on the test
  machine, so it can't get an unfair head start.
- Model: Claude Opus 4.8.

## Why the skill is intentionally minimal

The shipped skill does **not** vendor the plugin contract — it points Claude at the
canonical thin-edge spec and has it implement against the live source of truth.
Before adopting that, we verified the lean version is as good as a fuller one that
spelled the whole contract out inline:

| Metric | Minimal (shipped) | Detailed (vendored contract) |
|--------|-------------------|------------------------------|
| Pass rate | 100% (30/30) | 100% (30/30) |

Run over 3 evals × 3 runs each, both restricted to the public spec URL. Vendoring
the contract added **no** correctness with a capable model and a reachable spec
URL — so the skill stays lean and the live spec stays the single source of truth,
avoiding drift. (Caveat: this assumes the spec URL is reachable; the only scenario
where inlined detail would help is a failed fetch — offline, URL down, or a weaker
model that skips reading it.)

## Reproduce these results yourself

You need Claude Code with this plugin installed and the eval suite in
[`evals.json`](./evals.json) — it contains, for each test case, the user `prompt`
and the list of objective `assertions` the generated plugin must satisfy.

**Automated:**

The eval suite is structured for the `skill-creator` eval harness
(invoke the `/skill-creator` skill). It runs every prompt with and without the
skill, grades against the assertions, and emits the benchmark tables above.

**Manual (a few minutes per case):**

1. Install the plugin:
   `/plugin install tedge-config-plugin@thin-edge-skills`
2. For each eval in `evals.json`, open a fresh Claude Code session and paste the
   `prompt`. Confirm the skill triggers, and let Claude produce the plugin.
3. Read the generated plugin and check it against that eval's `assertions` — each
   is a concrete yes/no property (e.g. *"`list` prints `nginx` and nothing else"*).
   Count how many hold; that's the with-skill score.
4. **Baseline:** repeat steps 2–3 with the skill unavailable (`/plugin disable
   tedge-config-plugin@thin-edge-skills`, or a session without it). To keep it
   fair, don't let this run read a local checkout of the thin-edge docs — restrict
   it to the public spec URL, the same source the skill uses.
5. Compare the pass counts.
