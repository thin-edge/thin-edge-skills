# thin-edge skills

The official [Claude Code](https://code.claude.com/docs/) plugin marketplace for
[thin-edge.io](https://thin-edge.github.io/thin-edge.io/) (tedge) development.

Each skill is published as its own plugin, so you can install only the ones you
need.

## Using the marketplace

Add the marketplace, then install a plugin from it:

```shell
/plugin marketplace add thin-edge/thin-edge-skills
/plugin install tedge-config-plugin@thin-edge-skills
```

## Available skills

| Plugin | What it does | Evaluation |
|--------|--------------|------------|
| [`tedge-config-plugin`](plugins/tedge-config-plugin) | Write a new thin-edge configuration management plugin (the `list/get/prepare/set/verify/rollback` contract) to extend `tedge-agent` to a new config type or source. | [100% vs 43% baseline](plugins/tedge-config-plugin/evals/RESULTS.md) |

## Repository layout

```
.claude-plugin/
└── marketplace.json              # marketplace manifest (lists all plugins)
plugins/
└── <plugin>/
    ├── .claude-plugin/plugin.json
    ├── skills/<skill>/SKILL.md   # the skill itself (auto-discovered)
    └── evals/                    # reproducible eval suite + RESULTS.md
```

Transient eval run artifacts live in `*-workspace/` directories and are
git-ignored; only the durable eval suite (`evals/evals.json`) and a curated
`evals/RESULTS.md` are committed. Each skill's `RESULTS.md` reports its benchmark
numbers (with skill vs no skill) and step-by-step instructions to reproduce them
yourself — see, e.g.,
[`tedge-config-plugin` results](plugins/tedge-config-plugin/evals/RESULTS.md#reproduce-these-results-yourself).

## Adding or refining a skill

This repo was built with the [`skill-creator`](https://github.com/anthropics/skills)
skill, and every plugin here follows its conventions (SKILL.md structure,
evals suite, results format). The easiest and most reliable way to add a new
skill or improve an existing one is to use `skill-creator` itself — it
scaffolds the layout, helps you write a well-triggered description, and runs
the eval harness for you, so your contribution stays consistent with the rest
of the marketplace.

In Claude Code:

```
/skill-creator
```

Then describe the tedge skill you want to create or the existing one you want
to refine, and follow its guidance. It will generate the `SKILL.md`, optional
supporting files, and an `evals/` suite, and can benchmark the skill (with vs.
without) so you can capture numbers in `RESULTS.md`.

If you'd rather wire things up by hand, the minimum is:

1. Create `plugins/<new-skill>/` with `.claude-plugin/plugin.json` and
   `skills/<new-skill>/SKILL.md`.
2. Add an entry to `.claude-plugin/marketplace.json`.
3. Validate: `claude plugin validate .` (at the repo root and the plugin dir).
4. Optionally include an `evals/` suite and `RESULTS.md`.
