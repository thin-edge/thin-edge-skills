---
name: tedge-config-plugin
description: >-
  Write a new thin-edge.io (tedge) configuration management plugin to extend
  tedge-agent's config management to a new configuration type or source. Use
  this whenever the user wants to manage a new kind of configuration through
  thin-edge — for example "write a tedge config plugin for nginx", "add a new
  config type to thin-edge", "extend tedge config management to handle my
  database settings", "create a config-plugin so the cloud can update my
  app.conf", or any request about supporting a custom config type, source, or
  service in tedge's config_snapshot / config_update operations. Produces an
  executable plugin that conforms to the tedge config-plugin spec (the
  list/get/prepare/set/verify/rollback contract), plus install and test
  instructions.
---

# Writing a thin-edge config management plugin

A config plugin lets `tedge-agent` read and update a configuration that it
doesn't natively understand. Out of the box, tedge can only read/write plain
files. A plugin extends this to **any source** (a file, several files merged
together, a database, a registry, an API) and lets you wrap an update in
**validation, backup, service reload, and rollback**.

The plugin is just an executable that the agent calls with a fixed set of
subcommands. Your job with this skill is to interview the user about their
specific config type, then generate a correct, robust plugin for it.

## Read the official spec first

The canonical, always-up-to-date specification — including the full **lighttpd
reference plugin**, the discovery/permissions model, and the `config_update`
workflow — lives at:

**https://thin-edge.github.io/thin-edge.io/extend/config-management**

Fetch it at the start of the task (the raw Markdown is at
`https://raw.githubusercontent.com/thin-edge/thin-edge.io/main/docs/src/extend/config-management.md`
if you want the example verbatim). Don't rely on a cached copy — the spec is the
source of truth and may have evolved. The summary below is working guidance to
help you interview and generate; the link is authoritative if anything differs.

## The contract (working summary)

The plugin is an executable implementing these subcommands. Getting this
contract exactly right is the whole point — the agent drives a state machine
across these commands and relies on exit codes to decide what happens next.

| Subcommand | What it must do |
|---|---|
| `list` | Print every config type this plugin handles, **one per line**, to stdout. The agent runs this at startup to discover the plugin; a non-zero exit makes the agent ignore the plugin entirely. |
| `get <type>` | Print the **complete effective configuration** for `<type>` to **stdout** and nothing else. This is what gets uploaded to the cloud on a `config_snapshot`. |
| `prepare <type> <new-config-path> --work-dir <dir>` | Validate the candidate config at `<new-config-path>`. Save whatever rollback will need (normally a backup of the current config) **into `<dir>`**. Do **not** apply anything yet. |
| `set <type> <new-config-path> --work-dir <dir>` | Apply the new config and reload/restart any affected service. Exit non-zero if anything fails — that is the signal that triggers `rollback`. |
| `verify <type> --work-dir <dir>` | Confirm the service is actually healthy after the update (running, responding). Exit non-zero if not — also triggers `rollback`. |
| `rollback <type> --work-dir <dir>` | Restore the backup saved in `<dir>` during `prepare` and reload the service, returning the system to its pre-update state. |

**Universal rules for every subcommand:**

- **Exit `0` on success; non-zero on failure, with a human-readable message on
  `stderr`.** The agent has no other way to know what happened. A plugin that
  fails silently with exit 0 will corrupt config state.
- **Keep stdout clean.** Only `get` (effective config) and `list` (type names)
  are meant to produce stdout. Everywhere else, send progress/diagnostics to
  `stderr` so they don't get mistaken for config content.
- **`--work-dir` is a scratch directory the agent creates and later deletes.**
  It's the *only* shared state between `prepare`, `set`, `verify`, and
  `rollback` for a single update. Anything `rollback` might need must be written
  there by `prepare`. Don't assume it persists beyond one update.
- **One plugin can serve many config types.** Always validate the `<type>`
  argument against the set you support and reject unknown types.

### Why this shape — the update lifecycle

When the cloud requests an update, the agent runs **prepare → set → verify**,
and calls **rollback automatically if any of those steps exits non-zero**. So
think of it as a transaction:

```
prepare   validate + back up   (safe to abort; nothing changed yet)
   │
  set      apply + reload       (the mutation)
   │
verify     health check         (did it actually work?)
   │
 ✓ done        ✗ any failure → rollback → restore backup + reload
```

Designing each step around "what does rollback need from me, and how do I prove
success?" is what makes a plugin trustworthy.

## How to use this skill

### 1. Pick the implementation language

Ask the user which language they want **unless they've already said**. Default
to **POSIX shell (`/bin/sh`)** — it matches the official example, needs no build
step or runtime on the device, and installs by copy + `chmod +x`. Offer
alternatives if their logic is complex or they prefer a compiled binary:

- **Shell** — best default; portable; ideal when the work is "run a few commands".
- **Rust / Go** — a compiled binary; good for heavy logic, but remember it must
  be cross-compiled for the target device's architecture.
- **Python** — readable, but the device needs a Python runtime.

The contract is identical regardless of language — only the syntax changes.

### 2. Interview the user about their config type

You can't write a correct plugin without knowing how *their* config behaves.
Gather these, asking only what you can't reasonably infer:

1. **Type name(s)** — the identifier(s) `list` prints and the cloud uses (e.g.
   `nginx`, `app.conf`, `mosquitto.conf`). The plugin's filename becomes the
   plugin name; the cloud sees `<type>::<plugin-name>`.
2. **Read path (`get`)** — Where does the effective config come from? A single
   file? Several fragments to merge? A command that prints it (like
   `lighttpd -p`)? A DB/registry query?
3. **Validation (`prepare`)** — Is there a way to check a candidate config is
   well-formed before applying it (e.g. `nginx -t`, `visudo -c`, a JSON/TOML
   parse)? If yes, run it and fail `prepare` when it fails. If there's no
   validator, say so — skip validation rather than inventing a fake check.
4. **Apply (`set`)** — How does the new config take effect? Move a file into
   place? Write to a DB? Then which service must reload/restart, and how
   (`systemctl reload X`, a signal, an API call)?
5. **Verify (`verify`)** — How do you know it worked? Service active? A port
   open? A health endpoint returning 200?
6. **Backup/restore granularity** — What exactly must be saved in `prepare` and
   put back in `rollback` to fully undo the change?

If the user is vague, propose sensible defaults from the service's conventions
and let them correct you — don't stall on a long questionnaire.

### 3. Generate the plugin

Write the executable implementing all six subcommands per the contract. Use the
**lighttpd reference plugin from the official doc** as the structural template
for shell plugins — it already handles argument dispatch, robust `--work-dir`
parsing, type validation, and the backup/restore pattern correctly. Adapt its
bodies to the user's answers rather than starting from scratch. For other
languages, mirror the same structure and the same exit-code discipline.

Guard against the common mistakes:

- `prepare` that applies the change (it must only validate + back up).
- `set`/`verify` that exit 0 even when the reload failed (must exit non-zero so
  rollback fires).
- `get` that prints logs or "OK" to stdout mixed with the config.
- `rollback` that assumes a backup exists without checking (handle the
  "prepare didn't run / nothing to restore" case).
- Forgetting to make the file executable, or hardcoding a single type when the
  user wanted several.

### 4. Provide install + test instructions

After writing the plugin, give the user:

**Install** — copy into the plugin directory, make executable, and (only if
their tedge wasn't installed via the official packages) add the sudoers entry
shown in the spec:

```sh
sudo cp ./<plugin-name> /usr/share/tedge/config-plugins/<plugin-name>
sudo chmod +x /usr/share/tedge/config-plugins/<plugin-name>
# Tell the agent to pick it up immediately:
tedge mqtt pub te/device/main/service/tedge-agent/signal/sync_config '{}'
```

**Local test** — exercise the contract by hand before trusting it remotely.
Note that `prepare`/`set` mutate the system:

```sh
PLUGIN=/usr/share/tedge/config-plugins/<plugin-name>
sudo "$PLUGIN" list
sudo "$PLUGIN" get <type>

WORK_DIR=$(mktemp -d)
sudo "$PLUGIN" prepare  <type> /path/to/new-config --work-dir "$WORK_DIR"
sudo "$PLUGIN" set      <type> /path/to/new-config --work-dir "$WORK_DIR"
sudo "$PLUGIN" verify   <type> --work-dir "$WORK_DIR"
# If something looks wrong, undo:
sudo "$PLUGIN" rollback <type> --work-dir "$WORK_DIR"
```

Encourage the user to deliberately feed `prepare` a broken config and confirm it
exits non-zero — a plugin whose failure paths are untested isn't done.
