# Mattermost deployment

Example deployment harness for the `ansible-role-mattermost` role. The role lives
in the repository root; this directory holds a runnable playbook plus example
inventory/vars/vault templates.

Only the `*.example` files are committed. Your real inventory, vars, and vault are
git-ignored (see `.gitignore`) so environment-specific values and secrets never
land in git.

## Layout

| File | Purpose |
|---|---|
| `ansible.cfg` | Points `roles_path` at the repo root so the role resolves by name. |
| `site.yml` | The playbook (`hosts: chat`, applies the role). |
| `inventory.ini.example` | Target host(s) + SSH connection details. |
| `group_vars/chat/vars.yml.example` | Non-secret role configuration. |
| `group_vars/chat/vault.yml.example` | Secrets template (encrypt with ansible-vault). |

## Setup

```bash
cd deploy

# 1. Inventory — set your host IP and SSH key.
cp inventory.ini.example inventory.ini
$EDITOR inventory.ini

# 2. Role config — pick install method, DB, TLS, domain, etc.
cp group_vars/chat/vars.yml.example group_vars/chat/vars.yml
$EDITOR group_vars/chat/vars.yml

# 3. Secrets — fill in and encrypt.
cp group_vars/chat/vault.yml.example group_vars/chat/vault.yml
$EDITOR group_vars/chat/vault.yml
ansible-vault encrypt group_vars/chat/vault.yml
```

### Choosing an install method (in `vars.yml`)

- **`apt`** (default) — installs the community edition from the official Mattermost
  APT repository. Leave `mattermost_version` empty for the latest, or pin it.
- **`tarball`** — installs a specific release; set `mattermost_edition` to `team`
  or `enterprise` and `mattermost_version` to the exact release (e.g. `9.11.9`).

## Deploy

```bash
cd deploy
ansible-galaxy collection install -r ../requirements.yml
ansible-playbook site.yml --ask-vault-pass
```

## Verify

```bash
# On the target host:
systemctl status mattermost
curl -s localhost:8065/api/v4/system/ping    # -> {"status":"OK"}
```

Then browse `https://<your-domain>` and create the first admin account.

## Notes

- If you use a Cloudflare Origin certificate (`mattermost_tls_mode:
  cloudflare_origin`), create the proxied DNS record and set the zone SSL mode to
  **Full** in the Cloudflare dashboard — or set `mattermost_cloudflare_enabled:
  true` and let the role manage them via the API.
- Switching an existing host from `apt` to `tarball` removes the APT package
  first (its data under `/opt/mattermost/data` is purged); back up the database
  and any file store before switching.
