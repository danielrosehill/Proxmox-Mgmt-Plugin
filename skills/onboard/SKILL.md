---
name: onboard
description: First-run onboarding for the proxmox-mgmt plugin. Interview the user about their Proxmox host — IP, SSH user, web UI URL, and (optionally) Proxmox API token references — and persist the result to `$CLAUDE_USER_DATA/proxmox-mgmt/config.json`. Run this before any other skill in this plugin, or whenever the connection details change. Triggers on phrases like "set up proxmox", "onboard proxmox", "configure proxmox plugin".
---

# proxmox-mgmt: onboard

Establish the persistent connection profile for a Proxmox VE host. The same plugin install can serve multiple Proxmox environments — only the values in `config.json` are environment-specific. No host details should ever be hard-coded into other skills in this plugin.

## Workspace resolution

Resolve the plugin's data directory as `$CLAUDE_USER_DATA/proxmox-mgmt/` if `CLAUDE_USER_DATA` is set; otherwise `$XDG_DATA_HOME/claude-plugins/proxmox-mgmt/` if `XDG_DATA_HOME` is set; otherwise `~/.local/share/claude-plugins/proxmox-mgmt/`. Create the directory if it doesn't exist. See the canonical convention in the `claude-rudder:plugin-data-storage` skill.

Shell form:

```bash
PLUGIN_DATA_DIR="${CLAUDE_USER_DATA:-${XDG_DATA_HOME:-$HOME/.local/share}/claude-plugins}/proxmox-mgmt"
mkdir -p "$PLUGIN_DATA_DIR"
```

The config file is `$PLUGIN_DATA_DIR/config.json`.

## When to use

- User says "set up proxmox", "onboard proxmox", "configure proxmox plugin".
- Any other proxmox-mgmt skill finds no `config.json` at the resolved path — offer to run onboarding first.
- The user changes hosts, IPs, or credentials.

## Procedure

1. **Load** existing config from `$PLUGIN_DATA_DIR/config.json` if present. Show the current values back. Offer **Update** vs **Replace** vs **Cancel**.

2. **Interview** the user. Ask each question; default sensibly when the user just hits enter:

   | Field | Prompt | Notes |
   |---|---|---|
   | `host` | "What's the LAN address of your Proxmox host?" | No default — user must answer. Hostname or IP. |
   | `ssh_user` | "SSH username?" | Default: `root`. Proxmox typically uses root for management. |
   | `ssh_port` | "SSH port?" | Default: `22`. |
   | `ssh_key_path` | "Path to the SSH private key (or leave blank for default agent)?" | Optional. Expand `~`. |
   | `web_url` | "Proxmox web UI URL?" | Default: `https://<host>:8006`. |
   | `node_name` | "Proxmox node name (run `hostname` on the host if unsure)?" | Used by API calls. Default: derived from `host`. |
   | `api_enabled` | "Use the Proxmox API? (y/n)" | If yes, ask for the next two. |
   | `api_token_id` | "API token ID (e.g. `root@pam!claude`)?" | Stored as plain text — it's an identifier, not a secret. |
   | `api_token_secret_ref` | "Where is the API token secret stored? (1Password ref / env var name / file path)" | Reference only. |
   | `verify_tls` | "Verify TLS on the Proxmox web/API URL? (y/n)" | Default: `n` (most homelab Proxmox hosts use self-signed certs). |
   | `notes` | "Any free-text notes about this host?" | Optional (e.g. GPU passthrough, RAID config). |

3. **Test connectivity** (best-effort, don't block on failure):

   ```bash
   ssh -o ConnectTimeout=5 -o BatchMode=yes -p "$SSH_PORT" "$SSH_USER@$HOST" \
     "pveversion; qm list 2>/dev/null | head -5; pct list 2>/dev/null | head -5"
   ```

   If the command fails, surface the error to the user but still write the config.

4. **Write** `$PLUGIN_DATA_DIR/config.json` with this schema (pretty-printed, 2-space indent):

   ```json
   {
     "schema_version": 1,
     "host": "192.168.1.10",
     "ssh_user": "root",
     "ssh_port": 22,
     "ssh_key_path": null,
     "web_url": "https://192.168.1.10:8006",
     "node_name": "pve",
     "api": {
       "enabled": false,
       "token_id": null,
       "token_secret_ref": null,
       "verify_tls": false
     },
     "notes": null,
     "updated_at": "<ISO-8601 timestamp>"
   }
   ```

5. **Summarise**: print every field and its final value. Show the path to `config.json`. List the sibling skills the user can now invoke (`proxmox-maintenance`).

## Re-running

If `config.json` already exists, never silently overwrite. Always confirm. If the user picks **Replace**, back up the existing file with `.bak` suffix first.

## Hard rules

- **Never store the API token secret in `config.json`.** Only store a reference (1Password item, env var name, file path). The seed skill resolves the reference at runtime.
- **Never hard-code IPs, hostnames, or usernames into other skills.** Every operational skill must read from `config.json`.
- **Default values are suggestions, not assumptions.** Always confirm — defaults exist to make the prompt less tedious, not to silently encode someone else's setup.
