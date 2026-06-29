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

A config plugin is an executable that lets `tedge-agent` read and update a
configuration it doesn't natively understand, with validation, backup, service
reload, and rollback around each update.

## Read the spec, then build to it

The canonical spec — subcommand contract, discovery/permissions model,
`config_update` workflow, and a complete **lighttpd reference plugin** to use as
your template — lives here:

**https://thin-edge.github.io/thin-edge.io/extend/config-management**
(raw Markdown: `https://raw.githubusercontent.com/thin-edge/thin-edge.io/main/docs/src/extend/config-management.md`)

Fetch it at the start of every task and build strictly against it — it is the
source of truth and may have changed since this skill was written. Pay close
attention to the subcommand interface and the exit-code discipline the agent
relies on to drive its state machine.

## Workflow: Staged interview with research-informed follow-ups

Ask in stages, not all at once — each answer narrows the next questions. Research
before you ask, so your questions are informed and come with sensible defaults.

1. **Stage 1: Identify the config type.** Ask what service/application config to
   manage (e.g. nginx, mosquitto, a custom app). This is the only essential
   up-front input.

2. **Initial research.** Fetch the spec and review the reference plugin. Then
   research this specific config type:
   - File location(s).
   - **How to validate** a config of this type — find the actual validator and
     its exact invocation (e.g. `nginx -t`) rather than guessing; some types
     have none.
   - **How a new config is applied.** Do not assume this means reloading a
     service. thin-edge runs on IoT devices where a service manager may not even
     be present, and many config types take effect by other means — re-reading
     the file on next use, a signal (`SIGHUP`), a CLI apply command, a file
     watcher, or nothing at all. Research what's normal for this type so you can
     propose a concrete default rather than asking blind.

3. **Stage 2+: Targeted follow-ups.** Ask only what research couldn't settle,
   each with a suggested default inline. For the application mechanism:
   - If the type is *typically* run as a managed service, propose a reload as the
     default and ask which service manager runs it — **do not assume systemd**;
     it's the most common but confirm (OpenRC, SysVinit, runit, or a plain
     command are all possible on these devices).
   - Otherwise, ask how the app is managed and how config takes effect — service
     manager or not — offering the mechanism your research suggested.
   Also confirm the validator command you found, and how to aggregate when config
   spans multiple files (default: main file). Add stages only as gaps remain.

4. **Generate the plugin.** Adapt the reference plugin from the spec to the
   answers, in the user's chosen language (default: POSIX shell).

5. **Install and local-test instructions.** Give steps to verify the plugin by
   hand before trusting it remotely.
