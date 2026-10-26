# Magpie Workspace

English | [简体中文](README_CN.md)

A development workspace for the **Cursor · Magpie Bridge** plugin, combining the plugin, upstream Magpie source, and generated code maps through Git submodules. This repository coordinates those projects; it is not the upstream Magpie application or a standalone plugin distribution.

The bridge lets Cursor use models and routing groups configured in Magpie through a local protocol bridge.

## Repository layout

| Path | Purpose | Repository |
| --- | --- | --- |
| [`magpie/`](magpie/) | Upstream Magpie source, used as a **read-only reference** in this workspace | [yetone/magpie](https://github.com/yetone/magpie) |
| [`plugins/magpie-agent-cursor-bridge/`](plugins/magpie-agent-cursor-bridge/) | Bridge plugin implementation, native engine patch, setup scripts, and tests | [flylink-code/magpie-agent-cursor-bridge](https://github.com/flylink-code/magpie-agent-cursor-bridge) |
| [`doc/`](doc/) | Code map generator, generated indexes, and navigation guides | [flylink-code/magpie-doc](https://github.com/flylink-code/magpie-doc) |
| [`AGENTS.md`](AGENTS.md) | Workspace engineering rules and contribution boundaries | This repository |

## Clone and initialize

Git and Node.js are needed for the workspace workflows below. Plugin runtime setup additionally requires Cursor and Magpie; see the [plugin installation guide](plugins/magpie-agent-cursor-bridge/README.md).

```powershell
git clone --recurse-submodules https://github.com/flylink-code/magpie-workspace.git
Set-Location magpie-workspace
```

For an existing clone, initialize the submodules at the revisions recorded by this repository:

```powershell
git submodule update --init --recursive
```

This does not automatically track the latest submodule branches. Review local changes before updating submodules, and do not use force options to discard work.

## Using the bridge

For normal use, install the plugin from Magpie → **Plugins** with:

```text
github:flylink-code/magpie-agent-cursor-bridge
```

Follow the [plugin README](plugins/magpie-agent-cursor-bridge/README.md) for Windows/macOS setup, local certificate trust, connecting models, status checks, and uninstall instructions. Cloning this workspace alone does not install or start the bridge.

## Development workflow

1. Read [`AGENTS.md`](AGENTS.md) for engineering constraints.
2. Start at the [code map index](doc/CODE_MAP_INDEX.md), select a module, then follow its file/symbol pages to the source. See the [navigation guide](doc/codemap-guide.md) and [generator design](doc/CODE_MAP.md).
3. Make plugin changes under [`plugins/magpie-agent-cursor-bridge/`](plugins/magpie-agent-cursor-bridge/). Keep [`magpie/`](magpie/) unchanged.
4. Run focused tests, then the plugin suite before delivery. Tests must isolate configuration, home directories, and Cursor settings; never run setup/uninstall against the real machine as a test.
5. Regenerate and check the code maps after file changes. Never edit generated map files manually.

Run these commands from the workspace root:

```powershell
# Plugin unit tests
node --test .\plugins\magpie-agent-cursor-bridge\test\*.test.js

# Regenerate and validate code maps
node .\doc\codemap.mjs
node .\doc\codemap.mjs --check

# Code map generator tests, when changing the generator
node --test .\doc\codemap.test.mjs
```

The map check should report `changed: 0` and `diagnostics: 0`.

## Working with submodules

Each submodule has its own history, remote, and working tree. Commit and push changes in the relevant submodule first, then commit its updated revision pointer in this parent repository. A parent-repository push does **not** push submodule commits.

Useful inspection commands:

```powershell
git submodule status
git status --short
git -C doc status --short --branch
git -C plugins\magpie-agent-cursor-bridge status --short --branch
git -C magpie status --short
```

Do not commit machine-specific proxy settings, credentials, or runtime data. Consult each component's repository for its own license and release information.
