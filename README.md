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

(Replace `thin-edge/thin-edge-skills` with this repo's `owner/repo` if it differs.)

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
`evals/RESULTS.md` are committed.

## Adding a new skill

1. Create `plugins/<new-skill>/` with `.claude-plugin/plugin.json` and
   `skills/<new-skill>/SKILL.md`.
2. Add an entry to `.claude-plugin/marketplace.json`.
3. Validate: `claude plugin validate .` (at the repo root and the plugin dir).
4. Optionally include an `evals/` suite and `RESULTS.md`.
