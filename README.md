# Proxmox Management Plugin

A Claude Code plugin for managing a [Proxmox VE](https://www.proxmox.com/en/proxmox-virtual-environment/overview) host. SSH-based diagnostics and lifecycle ops with optional Proxmox API support.

Per-host details (IP, SSH user, API token references) are stored **outside the plugin** at `$CLAUDE_USER_DATA/proxmox-mgmt/config.json`, so the same install works against any number of Proxmox hosts and survives plugin updates.

## Skills

- `onboard` — interactive first-run setup. Captures host, SSH user, web URL, node name, and (optionally) Proxmox API token references. Writes `config.json`.
- `proxmox-maintenance` — VM/CT lifecycle (`qm`, `pct`), storage and ZFS inspection, log review, update workflows, and Proxmox API access. Reads from `config.json`.

## Installation

```
claude plugins install proxmox-mgmt@danielrosehill
```

## Quick start

1. Install the plugin.
2. Run the `onboard` skill — Claude will interview you for the connection details and write them to `$CLAUDE_USER_DATA/proxmox-mgmt/config.json`.
3. Ask Claude things like *"check proxmox"*, *"list VMs"*, or *"what's the ZFS pool status"* — it'll read the config and connect.

## Storage convention

This plugin follows the [`claude-rudder:plugin-data-storage`](https://github.com/danielrosehill/Claude-Rudder) convention:

- **Plugin code** lives at the install path (read-only, replaced on update).
- **User config** lives at `${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/proxmox-mgmt/config.json`.
- **Secrets** (API token secret) are never stored in the config — only a *reference* to where they live (1Password item, env var, file path). Skills resolve the reference at runtime.

## License

MIT
