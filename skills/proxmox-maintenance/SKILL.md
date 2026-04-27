---
name: proxmox-maintenance
description: Use when the user wants to perform maintenance, diagnostics, or configuration on their Proxmox VE host via SSH or the Proxmox API. Reads connection details from `$CLAUDE_USER_DATA/proxmox-mgmt/config.json` (populated by the `onboard` skill in this plugin). Triggers on phrases like "proxmox maintenance", "check proxmox", "connect to proxmox", "list VMs", "list containers", "proxmox host".
---

# proxmox-maintenance

Inspect and maintain a Proxmox VE host. All host- and credential-specific values come from the plugin's config — never hard-code them.

## Pre-flight

Resolve the plugin data directory (`${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/proxmox-mgmt/`) and load `config.json`. If it doesn't exist or is incomplete, tell the user to run the `onboard` skill first and stop.

Available fields after load: `host`, `ssh_user`, `ssh_port`, `ssh_key_path`, `web_url`, `node_name`, `api.enabled`, `api.token_id`, `api.token_secret_ref`, `api.verify_tls`.

## SSH access

```bash
SSH_OPTS=( -p "$SSH_PORT" )
[ -n "$SSH_KEY_PATH" ] && SSH_OPTS+=( -i "$SSH_KEY_PATH" )
ssh "${SSH_OPTS[@]}" "$SSH_USER@$HOST" "<command>"
```

## Typical tasks

### Host overview
```bash
pveversion           # PVE version + kernel
pvesh get /nodes/$NODE_NAME/status   # CPU, RAM, uptime
pvesh get /cluster/resources         # All VMs/CTs/storage at once
```

### VMs (KVM)
```bash
qm list                              # All VMs on this node
qm status <vmid>                     # Single VM status
qm config <vmid>                     # VM config dump
qm start  <vmid>                     # Start
qm shutdown <vmid>                   # Graceful shutdown
qm stop   <vmid>                     # Force stop (destructive — confirm first)
```

### LXC containers
```bash
pct list                             # All containers
pct status  <ctid>
pct config  <ctid>
pct start   <ctid>
pct stop    <ctid>
pct enter   <ctid>                   # Interactive shell — only when user explicitly asks
```

### Storage / ZFS
```bash
pvesm status                         # All storage pools
zpool status                         # ZFS pool health
zpool list                           # ZFS capacity per pool
zfs list -t snapshot | head -50      # Snapshots
df -h                                # Filesystem usage
```

### Logs
```bash
journalctl -u pveproxy --since "1 hour ago"
journalctl -u pve-cluster --since "1 hour ago"
tail -100 /var/log/syslog
```

### Updates
```bash
apt update
apt list --upgradable
# apt dist-upgrade — DO NOT run without explicit user confirmation
```

## Proxmox API (when `api.enabled`)

Resolve `token_secret` from `api.token_secret_ref` at runtime (1Password / env / file). Don't log it.

```bash
AUTH="PVEAPIToken=${TOKEN_ID}=${TOKEN_SECRET}"
TLS_FLAG=""
[ "$VERIFY_TLS" = "false" ] && TLS_FLAG="-k"

curl $TLS_FLAG -H "Authorization: $AUTH" \
  "$WEB_URL/api2/json/nodes/$NODE_NAME/qemu"
```

Common endpoints (under `/api2/json`):

| Path | Purpose |
|---|---|
| `/nodes/{node}/qemu` | List VMs |
| `/nodes/{node}/lxc` | List containers |
| `/nodes/{node}/qemu/{vmid}/status/current` | VM live status |
| `/nodes/{node}/storage` | Storage pools |
| `/cluster/resources` | Everything in one call |

## Guidelines

- **Never destroy VMs, containers, or storage without explicit confirmation.** `qm destroy`, `pct destroy`, `zfs destroy`, `pvesm remove` — all require an extra "yes I'm sure" step.
- **Never run `apt dist-upgrade` without confirmation.** Kernel upgrades require a reboot and may interrupt VMs.
- **Prefer SSH for interactive diagnostics; prefer the API for structured queries or scripted actions.**
- **Report what you find before suggesting destructive actions.** Show the user the VM/CT list, current state, recent logs — let them decide.
- **Don't reboot the host.** Ever, unless the user explicitly says "reboot the proxmox host" — it takes every VM/CT down.
